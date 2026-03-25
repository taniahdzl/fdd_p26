---
title: "Pipeline E2E en Polars"
---

# Pipeline E2E: limpieza de datos completa en Polars

Esta sección construye un pipeline de limpieza de datos **completo y realista** usando exclusivamente Polars en modo lazy. El objetivo es ver cómo las piezas individuales (expresiones, contextos, tipos, lazy eval) se combinan en un flujo de trabajo real.

## El escenario

Tienes dos archivos CSV con datos sucios de un sistema de ventas:

1. **ventas.csv** — transacciones con errores típicos:
   - Nombres con espacios extra y mayúsculas inconsistentes
   - Fechas como strings en formatos mixtos
   - Precios negativos (errores de captura)
   - Nulls en columnas críticas
   - Columnas que no necesitas (50 columnas, solo usas 8)

2. **productos.csv** — catálogo de productos para enriquecer las ventas:
   - Categoría, subcategoría, proveedor

El pipeline toma estos datos sucios y produce un **resumen limpio por categoría** en Parquet.

## El pipeline completo

```
ventas.csv (sucio, 50 cols)    productos.csv
        │                           │
        ▼ scan_csv                  ▼ scan_csv
        │                           │
        ▼ Validación de esquema     │
        │ (cast tipos)              │
        ▼ Filtrar filas inválidas   │
        │ (precios negativos,       │
        │  nulls en campos clave)   │
        ▼ Limpiar strings           │
        │ (trim, lowercase)         │
        ▼ Parsear fechas            │
        │ (str → Date)              │
        ▼ Join ◄──────────────────────┘
        │ (enriquecer con categoría)
        ▼ Agregar
        │ (por categoría + mes)
        ▼ .explain()
        │ (inspeccionar plan optimizado)
        ▼ .collect()
        │ (EJECUTAR todo)
        ▼ write_parquet
        │
        resultado.parquet (limpio)
```

## Paso 1: Scan — no leer, solo registrar

```python
import polars as pl

# NO lee los datos — solo registra la intención de leer
lf_ventas = pl.scan_csv("ventas.csv")
lf_productos = pl.scan_csv("productos.csv")
```

En este punto, Polars no ha tocado los archivos. Solo sabe que existen y (opcionalmente) qué esquema tienen.

**Diferencia con pandas:** `pd.read_csv("ventas.csv")` lee TODO el archivo en memoria inmediatamente — las 50 columnas, todas las filas, sin importar cuántas necesites.

## Paso 2: Validación y casting de tipos

```python
lf_ventas = lf_ventas.with_columns(
    pl.col("precio").cast(pl.Float64),
    pl.col("cantidad").cast(pl.Int32),
    pl.col("producto_id").cast(pl.Int64),
)
```

En Polars, los casts son explícitos y fallan si el dato no es convertible. No hay conversiones silenciosas como en pandas.

## Paso 3: Filtrar filas inválidas

```python
lf_ventas = lf_ventas.filter(
    pl.col("precio").is_not_null()
    & pl.col("cantidad").is_not_null()
    & (pl.col("precio") > 0)            # precios negativos = error
    & (pl.col("cantidad") > 0)
)
```

El optimizador combinará este filtro con el scan — si el formato lo permite (Parquet), puede saltar bloques enteros que no cumplan la condición.

## Paso 4: Limpiar strings

```python
lf_ventas = lf_ventas.with_columns(
    pl.col("cliente")
        .str.strip_chars()               # quitar espacios al inicio/fin
        .str.to_lowercase()              # normalizar a minúsculas
        .alias("cliente"),
    pl.col("vendedor")
        .str.strip_chars()
        .str.to_titlecase()              # "juan pérez" → "Juan Pérez"
        .alias("vendedor"),
)
```

Cada operación `.str.*` es una expresión nativa — se ejecuta en Rust, sin pasar por Python. Compara con pandas donde `df["col"].str.lower()` opera sobre un array de objetos Python.

## Paso 5: Parsear fechas

```python
lf_ventas = lf_ventas.with_columns(
    pl.col("fecha_str")
        .str.to_date("%Y-%m-%d")         # string → Date
        .alias("fecha"),
).with_columns(
    pl.col("fecha").dt.year().alias("anio"),
    pl.col("fecha").dt.month().alias("mes"),
)
```

El namespace `.dt.*` da acceso a todas las operaciones temporales — extracción de componentes, aritmética de fechas, truncamiento.

## Paso 6: Join con catálogo

```python
lf_enriquecido = lf_ventas.join(
    lf_productos,
    on="producto_id",
    how="left",
)
```

Este join es lazy — no se ejecuta todavía. El optimizador puede aplicar projection pushdown al join: si después del join solo usas 3 columnas de `productos`, Polars solo lee esas 3.

## Paso 7: Agregación

```python
lf_resultado = (
    lf_enriquecido
    .group_by("categoria", "mes")
    .agg(
        pl.col("precio").sum().alias("revenue"),
        pl.col("precio").mean().alias("ticket_promedio"),
        pl.col("producto_id").n_unique().alias("productos_distintos"),
        pl.len().alias("n_transacciones"),
    )
    .sort("categoria", "mes")
)
```

## Paso 8: Inspeccionar el plan

Antes de ejecutar, veamos qué va a hacer Polars:

```python
# Plan sin optimizar (lo que escribimos):
print(lf_resultado.explain(optimized=False))

# Plan optimizado (lo que Polars ejecutará):
print(lf_resultado.explain())
```

En el plan optimizado deberías ver:
- **Projection pushdown**: de las 50 columnas del CSV, solo lee las ~8 que necesitas
- **Predicate pushdown**: los filtros de precio > 0 se aplican durante la lectura
- **Join optimizado**: solo las columnas necesarias de `productos` participan

## Paso 9: Ejecutar

```python
df_resultado = lf_resultado.collect()
print(df_resultado)
```

**Todo el pipeline se ejecuta aquí** — lectura, limpieza, join, agregación — en una sola pasada optimizada y paralelizada.

## Paso 10: Escribir resultado

```python
df_resultado.write_parquet("resumen_ventas.parquet")
```

Parquet es el formato ideal de salida:
- Columnar (como Arrow) → lectura selectiva de columnas
- Comprimido → archivos más pequeños
- Tipado → no hay ambigüedad de tipos al releer
- Estándar → lo leen Polars, pandas, Spark, DuckDB, etc.

## El punto clave

Todo el pipeline — desde la lectura hasta la escritura — se define como una serie de expresiones lazy. Polars ve el plan completo, lo optimiza, y lo ejecuta en paralelo. No hay DataFrames intermedios en memoria, no hay copias innecesarias, no hay columnas que se leen y se descartan.

Compara con el equivalente en pandas:

```python
# pandas: cada línea crea un DataFrame nuevo en memoria
df = pd.read_csv("ventas.csv")                      # lee TODO, 50 cols
df = df.dropna(subset=["precio", "cantidad"])        # copia
df = df[df["precio"] > 0]                            # copia
df["cliente"] = df["cliente"].str.strip().str.lower() # modifica in-place (a veces)
df["fecha"] = pd.to_datetime(df["fecha_str"])         # copia
df = df.merge(productos, on="producto_id")            # copia
resultado = df.groupby(["categoria", "mes"]).agg(...) # copia
```

Cada línea materializa un DataFrame completo. Con datos grandes, esto puede significar múltiples copias de GBs en memoria.

> **Verifica en el notebook:** Notebook 02 construye exactamente este pipeline paso a paso, mostrando `.explain()` después de cada transformación para ver cómo el plan crece y el optimizador lo reescribe.

:::exercise{title="Extender el pipeline"}

Agrega al pipeline:
1. Una columna `dia_semana` extraída de la fecha (lunes=1, domingo=7)
2. Un filtro que excluya ventas de domingo
3. Una columna calculada `total = precio * cantidad`
4. Usa `.explain()` para verificar que el filtro de domingo se integró con los otros filtros (predicate pushdown)

:::
