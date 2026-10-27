# Resumen de resultados — Tarea #3 (Spark ML)

Clasificación binaria de abandono de clientes (*churn*) con Apache Spark y PostgreSQL, ejecutado dentro de Docker.

- **Objetivo:** predecir `etiqueta` (0 = el cliente se queda, 1 = el cliente abandona).
- **Fecha de ejecución:** 26 de octubre de 2026.
- **Entorno:** notebook ejecutado de punta a punta en el contenedor `big-data` (PySpark 3.5.1) con PostgreSQL 16 como almacén intermedio vía JDBC.

## Datos

| Concepto | Valor |
|---|---|
| Filas totales | 10 000 |
| Features | `edad`, `gasto_mensual`, `tickets_soporte`, `meses_cliente`, `usa_app` |
| Variable objetivo | `etiqueta` (binaria) |
| Clase 0 (se queda) | 5 879 (58.79 %) |
| Clase 1 (abandona) | 4 121 (41.21 %) |
| Partición | Entrenamiento 7 976 / Validación 2 024 (`seed=27`) |

El dataset limpio se almacenó en la tabla `tarea3` (10 000 filas) con columnas escalares individuales (sin columnas de tipo vector).

## Preparación

- Carga del CSV con **schema explícito** (`StructType`), sin `inferSchema`.
- Análisis exploratorio: estadísticas descriptivas, balance de clases, histogramas, boxplots y matriz de correlación (3 figuras guardadas en `output/`).
- **Estandarización** con `StandardScaler` dentro de un `Pipeline` (`VectorAssembler` → `StandardScaler`).

## Modelos entrenados

Ambos modelos se entrenaron leyendo los datos desde PostgreSQL, con **K-Fold Cross Validation (k=5)** usando `CrossValidator` y un `ParamGridBuilder` de 4 combinaciones cada uno.

### Modelo 1 — Regresión Logística
- Mejores hiperparámetros: `regParam=0.1`, `elasticNetParam=0.0`
- `avgMetrics` (CV): 0.7045, 0.7045, 0.7046, 0.7011
- AUC en entrenamiento: **0.7051**

### Modelo 2 — Random Forest
- Mejores hiperparámetros: `numTrees=50`, `maxDepth=5`
- `avgMetrics` (CV): 0.7036, 0.6838, 0.7053, 0.6896
- AUC en entrenamiento: **0.7217**

## Evaluación en el conjunto de validación

| Métrica | Modelo 1 (Reg. Logística) | Modelo 2 (Random Forest) |
|---|---|---|
| AUC | **0.7001** | 0.6992 |
| Accuracy | 0.6551 | **0.6571** |

Las predicciones sobre el conjunto de validación se persistieron en las tablas `modelo1` y `modelo2` (2 024 filas cada una), con las mismas columnas que `tarea3` más la columna `prediccion` (sin columnas de tipo vector).

## Conclusión

El desempeño de ambos modelos es prácticamente equivalente. La **Regresión Logística** obtiene el mejor AUC de validación (0.7001 frente a 0.6992), una diferencia de apenas **0.0009**, mientras que **Random Forest** alcanza una accuracy ligeramente superior (0.6571 frente a 0.6551). Se trata de un **empate técnico**: ambos modelos capturan una señal moderada (AUC ≈ 0.70), lo que sugiere que las variables disponibles explican parcialmente el abandono y que habría margen de mejora con más features o ingeniería de atributos.

## Tablas resultantes en PostgreSQL

| Tabla | Filas | Contenido |
|---|---|---|
| `tarea3` | 10 000 | Dataset limpio y estandarizado (columnas escalares) |
| `modelo1` | 2 024 | Validación + `prediccion` (Regresión Logística) |
| `modelo2` | 2 024 | Validación + `prediccion` (Random Forest) |

## Reproducibilidad

Instrucciones completas de ejecución en `README.md` e `INSTRUCCIONES.pdf`. Resumen:

```bash
docker compose up -d --build
docker compose logs -f bigdata   # obtener el token de Jupyter (puerto 8888)
```

Luego abrir `notebooks/Binary classification.ipynb` y ejecutarlo de principio a fin.
