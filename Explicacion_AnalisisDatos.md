# Guía del notebook de análisis de calidad del aire

Este documento explica, en el mismo orden del notebook, qué hace cada apartado y qué significa cada operación de pandas. El notebook principal es [AnalisisDatos.ipynb](./AnalisisDatos.ipynb).

## 1. Importación y verificación de los datos

```python
from pathlib import Path
import pandas as pd
```

- `Path` permite localizar archivos usando rutas relativas a la carpeta del proyecto.
- `pandas` se importa con el alias habitual `pd`; se usa para leer y analizar tablas.

El notebook busca primero `calidad_aire_estaciones.csv`. Si no existe, carga `calidad_aire_estaciones.xlsx`. La tabla leída se guarda en `df_aire`:

- `pd.read_csv(..., index_col=False)` lee un CSV sin interpretar una columna como índice.
- `pd.read_excel(..., index_col=None)` hace lo mismo para el archivo Excel.
- Si ninguno de los archivos está presente, `FileNotFoundError` detiene la ejecución e informa del problema.

Luego se quitan las columnas con nombres como `Unnamed: 0`, que suelen ser índices guardados accidentalmente como una columna. Se imprimen el archivo elegido, las dimensiones y los nombres de las columnas. `display(df_aire.head())` muestra las primeras cinco filas, y `df_aire.info()` resume tipos de datos y valores no nulos.

### Conversión de las mediciones a números

Las columnas PM2.5, ozono, temperatura y viento se recorren con un `for`. Para cada valor:

- `pd.to_numeric(..., errors="coerce")` convierte números (incluidos números guardados como texto) a tipo numérico.
- Si una celda contiene algo que no es una medición numérica, como una fecha, se convierte en `NaN` en lugar de provocar un error.
- `valores_invalidos` cuenta y anuncia las celdas que tenían contenido, pero no se pudieron convertir.

Esta conversión es importante para poder comparar las mediciones con números, calcular promedios y aplicar filtros. No recupera los valores originales ni adivina una medición: las celdas convertidas a `NaN` quedan como datos faltantes.

## 2. Selección de una columna y de varias columnas

```python
serie_pm25 = df_aire["Material_Particulado_PM25"]
tabla_sector_clima_alerta = df_aire[
    ["Sector", "Temperatura_C", "Alerta_Ambiental"]
]
```

- Al elegir una sola columna con corchetes, pandas devuelve una **Serie**. `serie_pm25` contiene las mediciones de PM2.5.
- Al elegir una lista de nombres de columnas, pandas devuelve un **DataFrame**. `tabla_sector_clima_alerta` conserva las columnas de sector, temperatura y alerta.
- `display(...)` permite ver la Serie o la tabla en el notebook.

## 3. Acceso por etiqueta con `.loc[]`

```python
registro_175 = df_aire.loc[175]
```

`.loc[]` selecciona por **etiqueta del índice**, no por el número de posición de memoria. Como el archivo se carga con un índice consecutivo que empieza en cero, la etiqueta `175` corresponde inicialmente a la fila cuyo índice es 175. La línea se ejecuta antes de quitar duplicados para consultar el registro original.

## 4. Detección y eliminación de duplicados

```python
cantidad_duplicados = int(df_aire.duplicated().sum())
df_aire = df_aire.drop_duplicates(keep="first")
```

- `duplicated()` marca las filas repetidas, comparando todas las columnas.
- `sum()` cuenta cuántas filas repetidas hay.
- `drop_duplicates(keep="first")` conserva la primera aparición y elimina las repeticiones posteriores.
- El resultado se asigna de nuevo a `df_aire`, que pasa a ser la tabla sin duplicados.

El índice original se conserva; eliminar filas no renumera automáticamente el índice.

## 5. Filtros de datos

Cada filtro crea una tabla nueva con las filas que cumplen la condición. Los filtros de este apartado se calculan antes de imputar los valores faltantes.

### PM2.5 por encima del umbral

```python
pm25_critico = df_aire[
    df_aire["Material_Particulado_PM25"] > 55.0
]
```

La comparación `> 55.0` produce una condición verdadera o falsa para cada fila. Se conservan las que tienen PM2.5 estrictamente mayor que 55; los valores `NaN` no cumplen la comparación.

### Sectores seleccionados con `.isin()`

```python
sectores_seleccionados = df_aire[
    df_aire["Sector"].isin(["Zona Industrial", "Corredor Vial"])
]
```

`.isin(lista)` comprueba si el sector de cada fila aparece en la lista. El resultado contiene solo las filas de Zona Industrial o Corredor Vial.

### Intervalo de humedad con `.between()`

```python
humedad_intervalo = df_aire[
    df_aire["Humedad_Relativa"].between(60, 80, inclusive="both")
]
```

`.between()` selecciona los valores entre los límites. `inclusive="both"` indica que se incluyen los dos extremos: humedad desde 60% hasta 80%, inclusive.

### Consulta con `.query()`

```python
alerta_roja_viento_bajo = df_aire.query(
    "Alerta_Ambiental == 'Roja' and Velocidad_Viento_kmh < 10.0"
)
```

`.query()` expresa el filtro como texto. `and` exige que ambas condiciones sean verdaderas: la alerta debe ser Roja y el viento debe ser menor que 10 km/h. Un viento exactamente igual a 10 no entra en el resultado.

### Condición compuesta y búsqueda de texto

```python
zona_ozono_alto = df_aire[
    df_aire["Sector"].str.startswith("Zona", na=False)
    & (df_aire["Nivel_Ozono_ppb"] > 40.0)
]
```

- `.str.startswith("Zona")` identifica sectores cuyo texto empieza por `Zona`.
- `na=False` hace que un sector faltante no sea seleccionado.
- `&` combina ambas condiciones con el significado de **Y**: deben cumplirse las dos.
- Los paréntesis alrededor de la comparación del ozono ayudan a que pandas evalúe correctamente la condición.

Así se seleccionan sectores iniciados por “Zona” con ozono estrictamente mayor que 40 ppb.

## 6. Imputación de valores faltantes con `.fillna()`

```python
media_pm25 = df_aire["Material_Particulado_PM25"].mean()
df_aire["Material_Particulado_PM25"] = (
    df_aire["Material_Particulado_PM25"].fillna(media_pm25)
)
df_aire["Nivel_Ozono_ppb"] = (
    df_aire["Nivel_Ozono_ppb"].fillna(0.0)
)
```

- `mean()` calcula el promedio de PM2.5 con los valores numéricos disponibles; pandas omite `NaN` al calcularlo.
- `fillna(media_pm25)` reemplaza cada faltante de PM2.5 por ese promedio.
- `fillna(0.0)` reemplaza los faltantes de ozono por cero.
- La asignación a cada columna guarda los valores rellenados en `df_aire`.

La imputación es una decisión de análisis: el cero de ozono significa “reemplazado por cero” y no una medición observada. Conviene tenerlo presente al interpretar los resultados.

## 7. Clasificación de temperatura con `.apply()`

La función `clasificar_temperatura()` devuelve una categoría:

- Menos de 18 °C: `Frío`.
- De 18 °C a 25 °C, incluidos ambos extremos: `Templado`.
- Más de 25 °C: `Cálido`.
- Si la temperatura falta (`NaN`), devuelve `pd.NA` para conservar esa ausencia y no clasificarla incorrectamente.

```python
df_aire["Clima"] = df_aire["Temperatura_C"].apply(
    clasificar_temperatura
)
```

`.apply()` ejecuta la función para cada valor de la columna. Los resultados se guardan en la nueva columna `Clima`.

## 8. Renombrado y reordenamiento de columnas

```python
df_aire = df_aire.rename(
    columns={
        "Material_Particulado_PM25": "PM25",
        "Velocidad_Viento_kmh": "Viento_KMH",
    }
)
```

`.rename(columns=...)` cambia esos nombres sin alterar los datos. Después, `columnas_reordenadas` forma una lista con `ID_Estacion` y `Clima` primero, seguida de las demás columnas en su orden actual. Al indexar `df_aire` con esa lista se reordena la tabla.

## 9. Promedios y máximo por sector con `.groupby()` y `.agg()`

```python
resumen_sector = df_aire.groupby("Sector").agg(
    PM25_promedio=("PM25", "mean"),
    Temperatura_maxima=("Temperatura_C", "max"),
)
```

- `groupby("Sector")` divide las filas por sector.
- `.agg()` calcula varias estadísticas para cada grupo.
- `("PM25", "mean")` obtiene el promedio de PM2.5.
- `("Temperatura_C", "max")` obtiene la temperatura máxima.
- Los nombres `PM25_promedio` y `Temperatura_maxima` nombran las columnas resultantes.

El resultado tiene una fila por sector.

## 10. Rango de mediciones por alerta

```python
def rango(series):
    return series.max() - series.min()
```

La función `rango()` resta el menor valor al mayor. Mide la amplitud entre extremos, no la desviación estándar.

```python
dispersion_alerta = df_aire.groupby("Alerta_Ambiental").agg(
    {"PM25": rango, "Humedad_Relativa": rango}
)
```

Se agrupan las filas por nivel de alerta y se aplica `rango()` a PM2.5 y humedad relativa. Si una columna no tiene valores disponibles en un grupo, su rango no se puede calcular.

## 11. Agrupación multinivel por sector y alerta

```python
promedios_sector_alerta = df_aire.groupby(
    ["Sector", "Alerta_Ambiental"]
).agg(
    PM25_promedio=("PM25", "mean"),
    Humedad_promedio=("Humedad_Relativa", "mean"),
)
```

Al agrupar por una lista de dos columnas, pandas forma grupos para cada combinación de sector y alerta. Para cada combinación se calculan el promedio de PM2.5 y el promedio de humedad. El índice del resultado contiene ambos niveles de agrupación.

## Nota sobre el archivo de datos actual

En la carpeta del proyecto está `calidad_aire_estaciones.xlsx`, pero no `calidad_aire_estaciones.csv`. El notebook está preparado para usar el CSV cuando esté disponible y, en caso contrario, utiliza el Excel.

En el Excel hay celdas con fechas en columnas que deberían contener mediciones. El notebook informa cuántas encuentra y las convierte en `NaN` porque no son mediciones numéricas válidas. En la copia inspeccionada, la conversión detectó 210 valores no numéricos en PM2.5, 251 en ozono, 889 en temperatura y 805 en velocidad del viento. Por eso:

- PM2.5 y ozono se pueden rellenar posteriormente según las reglas indicadas en el notebook.
- Muchas temperaturas quedan ausentes, así que `Clima` también queda ausente en esas filas.
- Los filtros y estadísticas de temperatura o viento se basan únicamente en los valores numéricos que sí están disponibles.

Para analizar correctamente todas las mediciones, se necesita el CSV original correcto o un Excel corregido. La conversión evita errores y hace explícitos los datos inválidos, pero no puede reconstruir los valores perdidos.
