# Tarea #3 — Clasificación binaria con Apache Spark (PySpark)

Entrenamiento de **dos modelos de clasificación binaria** con Apache Spark para
predecir el **abandono de clientes** (*customer churn*). Todo el flujo se ejecuta en
contenedores Docker y usa **PostgreSQL** como almacenamiento intermedio.

- Notebook principal: [`notebooks/Binary classification.ipynb`](notebooks/Binary%20classification.ipynb)
- Datos de entrada: [`data/dataset.csv`](data/dataset.csv) (10 000 registros)
- Instrucciones detalladas: [`INSTRUCCIONES.md`](INSTRUCCIONES.md) / `INSTRUCCIONES.pdf`
- Resultados: [`RESULTADOS.md`](RESULTADOS.md) 

## Uso de IA

* Se utilizo el modelo Claude Opus 4.8 en modo de Plan para crear un plan de trabajo con el fin de completar todos los pasos de la tarea. Claude me fue guiando y explicando las instrucciones. Esto fue de gran ayuda ya que paso a paso iba entrando en detalle sobre como funcionaba el procesamiento de los datos y los requerimientos.


## Arquitectura

El `docker-compose.yaml` define dos servicios:

| Servicio | Imagen | Rol |
|---|---|---|
| `postgres` | `postgres:16` | Base de datos `tarea3` (usuario/clave `spark`/`spark`), puerto `5432`. |
| `bigdata` | `./big-data/Dockerfile` | Jupyter + PySpark 3.5.1 + driver JDBC PostgreSQL, puerto `8888`. |

## Requisitos

- Docker y Docker Compose instalados y en ejecución.
- Puertos `8888` y `5432` libres en la máquina anfitriona.

## Levantar el stack

```bash
docker compose up -d --build
docker compose ps        # verificar que 'postgres' está healthy y 'bigdata' arriba
```

> La primera construcción puede tardar varios minutos (descarga de dependencias).

## Acceder a Jupyter

Obtener el token de acceso:

```bash
docker compose logs bigdata
```

Abrir en el navegador la URL mostrada (`http://127.0.0.1:8888/?token=...`) o
`http://localhost:8888` e introducir el token.

## Ejecutar el notebook

Abrir `notebooks/Binary classification.ipynb` y ejecutar **todas las celdas de arriba
a abajo** (*Run → Run All Cells*). El notebook se entrega ya ejecutado con todas las
salidas visibles. Alternativa por línea de comandos:

```bash
docker compose exec -T bigdata \
  jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=600 \
  "/src/notebooks/Binary classification.ipynb"
```

## Tablas generadas en PostgreSQL

El notebook escribe tres tablas (modo `overwrite`):

- **`tarea3`** — conjunto limpio, solo columnas escalares (sin vectores):
  `id, edad, gasto_mensual, tickets_soporte, meses_cliente, usa_app, etiqueta`.
- **`modelo1`** — predicciones de la Regresión Logística sobre validación
  (columnas de `tarea3` + `prediccion`).
- **`modelo2`** — predicciones del Random Forest sobre validación
  (columnas de `tarea3` + `prediccion`).

Las figuras del análisis exploratorio se guardan en [`output/`](output/)
(`histogramas.png`, `boxplots.png`, `correlacion.png`).

### Inspeccionar las tablas

```bash
docker compose exec -T postgres psql -U spark -d tarea3 -c "\dt"
docker compose exec -T postgres psql -U spark -d tarea3 \
  -c "select count(*) from tarea3; select count(*) from modelo1; select count(*) from modelo2;"
docker compose exec -T postgres psql -U spark -d tarea3 -c "\d modelo1"
```

## Detener el stack

```bash
docker compose down      # conserva el volumen 'postgres_data' con las tablas
```

> No usar `-v`: eso eliminaría los datos de PostgreSQL.
