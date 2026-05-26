# TP1 — EDA: Recorridos de Ecobici en la Ciudad de Buenos Aires

Trabajo Práctico Nº 1 de la Diplomatura en Ciencia de Datos e Inteligencia Artificial dictada por Martin Jaureguy.

**Alumno:** Matías Valdez
**Año analizado:** 2025

---

## Objetivo

Hacer un análisis exploratorio de datos sobre un dataset real, plantear entre 4 y 5 hipótesis y responderlas con evidencia. La idea es repasar todo lo visto en la materia hasta ahora: chequeo de integridad, limpieza, variables derivadas, distribuciones, detección de outliers, correlaciones, visualización y armado de conclusiones.

## Contexto del dataset

Los datos vienen del portal de **datos abiertos del Gobierno de la Ciudad de Buenos Aires**. Cada vez que un usuario hace un viaje en una bicicleta del sistema público Ecobici, el sistema registra automáticamente cuándo arrancó, cuándo terminó, en qué estación retiró la bici, en qué estación la devolvió y algunos datos del usuario (género, modelo de bici).

Elegí este dataset por tres motivos concretos:

1. **Tamaño.** El año 2025 completo tiene **3.156.182 viajes** registrados, así que cumple holgadamente el mínimo de 50.000 que pedía la consigna.
2. **Variedad de tipos de variable.** Hay fechas, numéricas continuas (duración, lat/long), categóricas (género, modelo de bici, nombres de estación) e identificadores. Permite mostrar distintos tipos de gráfico.
3. **Es un dataset bien documentado** por el portal de BA Data, y además es un sistema que conozco como usuario, lo que me ayuda a interpretar los resultados con sentido común.

El archivo se descarga directo del CDN del GCBA en formato ZIP (contiene un único CSV de unos 800 MB descomprimido), sin login ni token.

Link a la fuente: <https://data.buenosaires.gob.ar/dataset/bicicletas-publicas>

## Diccionario de datos

### Variables originales

| Columna | Tipo | Descripción |
|---|---|---|
| `Id_recorrido` | str | Identificador único del viaje. |
| `duracion_recorrido` | str (numérico) | Duración del viaje en **segundos**. Cuidado: viene como string con coma como separador de miles (`"1,016"` significa 1016 segundos). |
| `fecha_origen_recorrido` | datetime | Fecha y hora en que el usuario retira la bici. |
| `id_estacion_origen` | str | Identificador de la estación de retiro. |
| `nombre_estacion_origen` | str | Nombre legible de la estación de retiro. |
| `direccion_estacion_origen` | str | Dirección postal de la estación de retiro. |
| `long_estacion_origen` | float | Longitud geográfica de la estación de retiro. |
| `lat_estacion_origen` | float | Latitud geográfica de la estación de retiro. |
| `fecha_destino_recorrido` | datetime | Fecha y hora en que se devuelve la bici. |
| `id_estacion_destino` | str | Identificador de la estación de devolución. |
| `nombre_estacion_destino` | str | Nombre legible de la estación de devolución. |
| `direccion_estacion_destino` | str | Dirección postal de la estación de devolución. |
| `long_estacion_destino` | float | Longitud geográfica de la estación de devolución. |
| `lat_estacion_destino` | float | Latitud geográfica de la estación de devolución. |
| `id_usuario` | str | Identificador anonimizado del usuario. |
| `modelo_bicicleta` | str | Tipo de bici. Valores observados: `ICONIC`, `FIT`. |
| `género` | str | Género declarado por el usuario al registrarse. Valores: `MALE`, `FEMALE`, `OTHER`, nulo. |

### Variables derivadas (creadas durante el análisis)

| Columna | Cómo se calcula | Para qué se usa |
|---|---|---|
| `duracion_min` | `duracion_recorrido / 60` | Hipótesis H1, H3, H5 |
| `hora` | hora del `fecha_origen_recorrido` | Hipótesis H2 |
| `dia_semana` | día de la semana del `fecha_origen_recorrido` | Hipótesis H3, H4 |
| `es_finde` | flag binario, `1` si el día es sábado o domingo | Hipótesis H3, H4 |
| `mes` | mes del `fecha_origen_recorrido` | Evolución estacional |
| `fecha` | solo la fecha del `fecha_origen_recorrido` | Evolución diaria |
| `circular` | flag binario, `1` si la estación de origen y destino son la misma | Hipótesis H4 |
| `distancia_km` | distancia haversine entre las coordenadas de origen y destino | Hipótesis H5 |

## Hipótesis

1. El género del usuario influye en la duración promedio del viaje.
2. Existen picos horarios definidos en la cantidad de viajes a lo largo del día.
3. Los viajes de fin de semana son más largos que los de día hábil.
4. Hay un porcentaje significativo de viajes "circulares" (misma estación de origen y destino), y este patrón se concentra los fines de semana.
5. La distancia geográfica entre estaciones correlaciona con la duración del viaje.

## Metodología aplicada

### Carga
El dataset se descarga directo del CDN de la Ciudad en formato ZIP. La primera celda del notebook se encarga de descargarlo y descomprimirlo solo si no está presente. Para leer el CSV alcanza con pandas.

### Chequeo de integridad
Antes de tocar nada, miré los `dtypes`, conté nulos por columna, busqué duplicados y corrí un `describe()` general. Detecté:

- La columna `duracion_recorrido` viene como **string con coma como separador de miles**. Si se intenta convertir a número con `pd.to_numeric()` directamente, casi la mitad de los valores quedan como NaN (justamente los más largos, los de 4 o más dígitos en segundos).
- Cero duplicados exactos.
- Algunas estaciones con lat/long nulos (estaciones que se dieron de baja y quedaron en histórico).
- Unos 8.600 viajes con `género` nulo.

### Limpieza
A partir del chequeo apliqué estos pasos, todos con justificación:

| Paso | Por qué |
|------|---------|
| Reemplazar la coma por nada en `duracion_recorrido` antes de convertir a número | Es separador de miles, no decimal. Sin este paso se pierden ~1,5 M de viajes. |
| Convertir fechas a datetime | Imprescindible para análisis temporal. |
| Descartar filas sin lat/long de origen o destino | Necesario para calcular distancia geográfica (H5). |
| Descartar filas con duración nula | No tiene sentido analizar un viaje sin saber cuánto duró. |

Tras la limpieza quedan **3.143.227 viajes**, un 99,6 % del dataset original.

### Visualizaciones

Para responder las hipótesis se generan las visualizaciones que pide la consigna:

- 3 histogramas con KDE (duración, distancia, hora de inicio).
- 3 boxplots (duración por género, por tipo de día, por día de la semana).
- 2 scatterplots (distancia vs duración, distancia vs duración solo en no circulares).
- 4 visualizaciones adicionales (barras por hora, barras de % circular por día, línea de evolución mensual, barras por género).

Los scatterplots usan una muestra aleatoria de 5.000 puntos por una cuestión de rendimiento.

## Conclusiones y hallazgos relevantes

El detalle completo, con tablas y números, está en la sección 5 del notebook. Acá va el resumen.

Las cinco hipótesis se confirmaron, algunas con datos especialmente fuertes.

- **H1** (género ↔ duración). Las mujeres hacen viajes ~16 % más largos que los hombres en promedio (FEMALE 23,1 min vs MALE 19,8 min). La diferencia es robusta dado el tamaño muestral.
- **H2** (picos horarios). Pico claro a las 17 hs (300.490 viajes), valle a las 4 hs (13.967). Diferencia de ~21× entre pico y valle. Llamativamente, el pico vespertino domina sobre el matinal.
- **H3** (finde más largo). Los viajes de fin de semana duran 31,5 min en promedio, contra 19,3 min los días hábiles. Un 63 % más largos.
- **H4** (viajes circulares y fin de semana). El 8,7 % del total son circulares (mismo origen y destino), pero la cifra sube a 16,2 % los domingos. Confirma uso recreativo de fin de semana.
- **H5** (distancia ↔ duración). Correlación moderada, `r = 0,49`. La velocidad efectiva en bici varía mucho entre usuarios, lo que limita el poder predictivo de la distancia sobre la duración.

### Hallazgos a destacar

1. **El sistema se usa fundamentalmente entre las 16 y las 19 hs.** Esto es información directamente accionable para distribuir bicis y operar el rebalanceo de estaciones.

2. **El uso del sistema cambia de carácter el fin de semana.** No solo hay menos viajes: los que hay son más largos y más circulares. Es un sistema con dos usos distintos según el día.

3. **Diferencia de género clara y consistente.** Las mujeres son apenas un tercio del total de viajes (32,4 %), pero los que hacen son sistemáticamente más largos. Es un hallazgo útil tanto para entender perfiles de uso como para campañas de promoción del sistema.

4. **Estacionalidad fuerte**: octubre fue el mes con más viajes (343.257) y junio el de menos (150.054), siguiendo el patrón climático esperado de Buenos Aires.

## Cómo correr el notebook

Requiere Python 3.9 o superior, con `pandas`, `numpy`, `matplotlib`, `seaborn` y `scipy`. La primera celda del notebook se encarga de descargar y descomprimir el dataset solo si no está presente localmente.

```bash
jupyter notebook TP1_Ecobici.ipynb
```

Tiempo aproximado de ejecución: 3 a 5 minutos si el dataset ya está descomprimido localmente; sumarle 30 a 60 segundos extra para la primera descarga del ZIP (~200 MB) si no está.

## Estructura del repo

```
.
├── README.md             # Este archivo
├── TP1_Ecobici.ipynb     # Notebook con todo el desarrollo y las salidas ya ejecutadas
└── .gitignore            # Excluye CSV/ZIP descargados en runtime
```

El dataset no está versionado en el repo: pesa unos 800 MB descomprimido y se baja en runtime desde el CDN público del GCBA.
