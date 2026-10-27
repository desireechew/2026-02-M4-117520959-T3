# Tarea #3 — Clasificación binaria con Apache Spark

Instrucciones completas para ejecutar la tarea. Todo el flujo corre dentro de
contenedores Docker: un contenedor con **Jupyter + PySpark** y un contenedor con
**PostgreSQL** que se usa como almacenamiento intermedio.

## Requisitos previos

- **Docker** y **Docker Compose** instalados y en ejecución.
- Puertos libres en la máquina anfitriona: `8888` (Jupyter) y `5432` (PostgreSQL).

## 1. Levantar el stack

Desde la carpeta raíz del proyecto (donde está `docker-compose.yaml`):

```bash
docker compose up -d --build
```

Esto construye la imagen `big-data` (Jupyter + PySpark 3.5.1 + driver JDBC de
PostgreSQL) y arranca dos servicios:

- `postgres` — PostgreSQL 16, base de datos `tarea3`, usuario `spark`, contraseña `spark`.
- `bigdata` — servidor Jupyter en el puerto `8888`.

> La primera construcción descarga varias dependencias y puede tardar varios
> minutos.

Comprobar que ambos servicios están arriba (y `postgres` *healthy*):

```bash
docker compose ps
```

## 2. Abrir Jupyter

Obtener el token de acceso desde los registros del contenedor:

```bash
docker compose logs bigdata
```

Buscar en la salida una URL como:

```
http://127.0.0.1:8888/?token=xxxxxxxxxxxxxxxxxxxxxxxx
```

Abrir esa URL en el navegador (o `http://localhost:8888` e introducir el token).

## 3. Ejecutar el notebook

En Jupyter abrir `notebooks/Binary classification.ipynb` y ejecutarlo
**de arriba a abajo** (menú *Run → Run All Cells*).

El notebook ya se entrega **ejecutado**, con todas las salidas visibles. Para
reproducirlo desde la línea de comandos sin abrir el navegador:

```bash
docker compose exec -T bigdata \
  jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=600 \
  "/src/notebooks/Binary classification.ipynb"
```

## 4. ¿Qué produce el notebook en la base de datos?

El notebook escribe tres tablas en la base `tarea3` (modo `overwrite`):

- **`tarea3`** — conjunto de datos limpio (solo columnas escalares, sin vectores):
  `id, edad, gasto_mensual, tickets_soporte, meses_cliente, usa_app, etiqueta`.
- **`modelo1`** — predicciones del Modelo 1 (Regresión Logística) sobre el conjunto
  de validación: las mismas columnas de `tarea3` más `prediccion`.
- **`modelo2`** — predicciones del Modelo 2 (Random Forest) sobre validación: las
  mismas columnas de `tarea3` más `prediccion`.

Las figuras del análisis exploratorio se guardan en `output/`
(`histogramas.png`, `boxplots.png`, `correlacion.png`).

## 5. Inspeccionar las tablas con psql

```bash
# Listar las tablas
docker compose exec -T postgres psql -U spark -d tarea3 -c "\dt"

# Conteos
docker compose exec -T postgres psql -U spark -d tarea3 \
  -c "select count(*) from tarea3; select count(*) from modelo1; select count(*) from modelo2;"

# Estructura de las tablas de predicción (debe aparecer la columna 'prediccion')
docker compose exec -T postgres psql -U spark -d tarea3 -c "\d modelo1"
docker compose exec -T postgres psql -U spark -d tarea3 -c "\d modelo2"

# Primeras filas
docker compose exec -T postgres psql -U spark -d tarea3 -c "select * from modelo1 limit 5;"
```

## 6. Detener el stack

Para detener los contenedores **conservando** los datos de PostgreSQL (volumen):

```bash
docker compose down
```

> No usar la opción `-v`: eso borraría el volumen `postgres_data` con las tablas.

## Resumen del flujo del notebook

1. Introducción: dominio (abandono de clientes), atributos y variable de predicción (`etiqueta`).
2. Carga y limpieza con esquema explícito (sin `inferSchema`), conteo de nulos y duplicados.
3. Análisis exploratorio: estadísticas, balance de clases, histogramas, boxplots y correlaciones.
4. Normalización/estandarización con `VectorAssembler` + `StandardScaler` dentro del Pipeline.
5. Escritura del conjunto limpio en la tabla `tarea3`.
6. Lectura desde la base de datos, división 80/20 y entrenamiento de dos modelos con
   K-Fold Cross Validation (`numFolds=5`).
7. Evaluación sobre validación y persistencia en `modelo1` y `modelo2`.
8. Análisis comparativo de resultados (AUC, accuracy, matriz de confusión).
