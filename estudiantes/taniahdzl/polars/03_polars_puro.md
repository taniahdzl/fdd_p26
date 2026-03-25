---
title: "Polars puro"
---

# Polars puro

Esta sección enseña Polars en sus propios términos — no como "pandas con sintaxis diferente". El concepto central es la **expresión**: un objeto que describe una transformación, no la ejecuta.

## 1. Series y DataFrame

Un `DataFrame` es una colección de `Series` (columnas) con nombre:

```python
import polars as pl

df = pl.DataFrame({
    "nombre": ["Alice", "Bob", "Carol", "David"],
    "edad": [30, 25, 35, 28],
    "ciudad": ["Madrid", "Lima", "Tokyo", "Madrid"],
    "salario": [50000, 42000, 68000, 55000],
})

print(df)
# shape: (4, 4)
# ┌────────┬──────┬────────┬─────────┐
# │ nombre ┆ edad ┆ ciudad ┆ salario │
# │ ---    ┆ ---  ┆ ---    ┆ ---     │
# │ str    ┆ i64  ┆ str    ┆ i64     │
# ╞════════╪══════╪════════╪═════════╡
# │ Alice  ┆ 30   ┆ Madrid ┆ 50000   │
# │ Bob    ┆ 25   ┆ Lima   ┆ 42000   │
# │ ...    ┆ ...  ┆ ...    ┆ ...     │
# └────────┴──────┴────────┴─────────┘
```

Observa que Polars:
- Muestra los **tipos** debajo de cada nombre de columna (`str`, `i64`)
- Usa `┆` como separador (no `|`)
- Infiere tipos estrictos: `i64` para enteros, `str` para texto

### Inspección básica

```python
df.shape         # (4, 4)
df.columns       # ['nombre', 'edad', 'ciudad', 'salario']
df.dtypes        # [String, Int64, String, Int64]
df.schema        # {'nombre': String, 'edad': Int64, ...}
df.describe()    # estadísticas descriptivas
df.head(2)       # primeras 2 filas
df.glimpse()     # vista transpuesta (útil para muchas columnas)
```

## 2. Expresiones: el concepto central

Una **expresión** en Polars es un objeto que describe una computación. No ejecuta nada — describe qué hacer.

```python
# Esto NO es una operación — es una DESCRIPCIÓN
expr = pl.col("salario") * 1.1

# Solo se ejecuta cuando la pones en un contexto:
df.select(expr)
```

Las expresiones básicas:

```python
pl.col("edad")                    # referencia a una columna
pl.col("edad") + 1                # operación aritmética
pl.col("nombre").str.to_lowercase()  # operación de texto
pl.col("salario").mean()          # agregación
pl.lit(42)                        # valor literal (constante)
pl.col("edad").alias("age")       # renombrar el resultado
```

### Composición de expresiones

Las expresiones se componen por encadenamiento — cada método retorna una nueva expresión:

```python
# Una expresión compuesta:
(
    pl.col("nombre")
    .str.to_lowercase()
    .str.replace(" ", "_")
    .alias("nombre_limpio")
)
```

### ¿Por qué expresiones y no indexación?

En pandas, filtras con indexación booleana:
```python
# pandas
df[df["edad"] > 30]              # crea una máscara booleana, luego indexa
df[df["edad"] > 30]["nombre"]    # encadenamiento que puede fallar (SettingWithCopyWarning)
```

En Polars, usas expresiones:
```python
# polars
df.filter(pl.col("edad") > 30)
df.filter(pl.col("edad") > 30).select("nombre")
```

Las expresiones son más explícitas, no tienen efectos secundarios, y el optimizador puede razonar sobre ellas.

## 3. Contextos: dónde viven las expresiones

Una expresión sola no hace nada. Necesita un **contexto** — un método del DataFrame que le dice "ejecuta aquí":

### `select` — seleccionar y transformar columnas

Retorna **solo** las columnas especificadas:

```python
df.select(
    "nombre",                              # columna sin transformar
    pl.col("salario") * 12,                # salario anual
    (pl.col("edad") >= 30).alias("senior") # columna booleana nueva
)
# shape: (4, 3) — solo 3 columnas
```

### `with_columns` — agregar o modificar columnas

Retorna **todas** las columnas + las nuevas/modificadas:

```python
df.with_columns(
    (pl.col("salario") * 12).alias("salario_anual"),
    pl.col("nombre").str.to_uppercase().alias("nombre_upper"),
)
# shape: (4, 6) — las 4 originales + 2 nuevas
```

### `group_by(...).agg(...)` — agregación

Agrupa y aplica expresiones de agregación:

```python
df.group_by("ciudad").agg(
    pl.col("salario").mean().alias("salario_medio"),
    pl.col("nombre").count().alias("n_personas"),
    pl.col("edad").max().alias("edad_max"),
)
```

### Resumen de contextos

```
┌─────────────────┬─────────────────────────────────────────────┐
│ Contexto        │ Qué hace                                    │
├─────────────────┼─────────────────────────────────────────────┤
│ select          │ Retorna SOLO las columnas que especificas    │
│ with_columns    │ Retorna TODAS las columnas + nuevas          │
│ group_by.agg    │ Agrupa y agrega — cada expr es una columna   │
│ filter          │ Filtra filas (la expr debe ser booleana)     │
│ sort            │ Ordena por la expresión dada                 │
└─────────────────┴─────────────────────────────────────────────┘
```

## 4. Filter, sort, unique, nulls

### Filtrar filas

```python
# Filtro simple
df.filter(pl.col("edad") > 30)

# Filtros combinados
df.filter(
    (pl.col("edad") > 25) & (pl.col("ciudad") == "Madrid")
)

# Filtro con OR
df.filter(
    (pl.col("ciudad") == "Madrid") | (pl.col("ciudad") == "Lima")
)

# Filtro con is_in (más limpio para múltiples valores)
df.filter(pl.col("ciudad").is_in(["Madrid", "Lima"]))
```

### Ordenar

```python
df.sort("edad")                          # ascendente
df.sort("edad", descending=True)         # descendente
df.sort("ciudad", "edad")               # múltiples columnas
```

### Valores únicos y duplicados

```python
df.select("ciudad").unique()                     # valores únicos
df.unique(subset=["ciudad"])                     # filas únicas por columna
df.n_unique()                                    # conteo por columna
df.is_duplicated()                               # máscara de duplicados
```

### Manejo de nulls

```python
df.filter(pl.col("salario").is_not_null())        # eliminar nulls
df.with_columns(pl.col("salario").fill_null(0))   # reemplazar con valor
df.with_columns(
    pl.col("salario").fill_null(strategy="forward")  # forward fill
)
df.with_columns(
    pl.coalesce("salario", "salario_default")     # primer no-null
)
df.drop_nulls()                                   # eliminar filas con cualquier null
df.drop_nulls(subset=["salario", "edad"])         # solo en columnas específicas
```

## 5. UDFs: funciones definidas por el usuario

A veces las expresiones nativas no son suficientes y necesitas una función Python personalizada. Polars ofrece tres niveles con rendimiento decreciente:

### El árbol de decisión

```
¿Necesitas una función personalizada?
│
├─ ¿Existe una expresión nativa?
│   → ÚSALA. 100x más rápido.
│   pl.col("x").str.*, .dt.*, .list.*, .cast(), operadores
│
├─ ¿Puedes combinar expresiones?
│   → pl.when(...).then(...).otherwise(...)
│   → Sigue siendo Rust, sigue siendo rápido
│
├─ ¿Necesitas operar sobre la Serie completa?
│   → map_batches(fn)
│   → Recibe una Series, retorna una Series
│   → El overhead es una sola llamada Python por columna
│
└─ ¿Necesitas operar fila por fila?
    → map_elements(fn)
    → Recibe un escalar, retorna un escalar
    → ÚLTIMO RECURSO: una llamada Python POR FILA
    → Rompe el pipeline Rust → pierde paralelismo
```

### Ejemplo comparativo

Tarea: convertir temperaturas de Celsius a Fahrenheit.

```python
# ✓ MEJOR — expresión nativa (Rust, paralelo)
df.with_columns(
    (pl.col("temp_c") * 9 / 5 + 32).alias("temp_f")
)

# ✓ OK — map_batches (una llamada Python, opera sobre toda la Serie)
df.with_columns(
    pl.col("temp_c").map_batches(
        lambda s: s * 9 / 5 + 32  # s es una Series completa
    ).alias("temp_f")
)

# ✗ EVITAR — map_elements (una llamada Python POR FILA)
df.with_columns(
    pl.col("temp_c").map_elements(
        lambda x: x * 9 / 5 + 32,  # x es un escalar
        return_dtype=pl.Float64
    ).alias("temp_f")
)
```

### Cuándo SÍ necesitas map_elements

- Llamar a una API externa por fila
- Usar una librería Python que no entiende Series (regex compilado, geopy, etc.)
- Lógica de negocio compleja que no se expresa con expresiones

Siempre especifica `return_dtype` — Polars no puede inferir el tipo de retorno de una función Python arbitraria.

> **Verifica en el notebook:** Notebook 01 — Sección 5 mide los tiempos exactos de expresión nativa vs `map_batches` vs `map_elements` sobre 1M de filas.

## 6. Columnas de tipo List

Una de las diferencias más importantes entre Polars y pandas: Polars tiene un tipo `List` nativo. Cada celda puede contener una lista de valores del mismo tipo.

### Crear columnas List

```python
# Directamente
df = pl.DataFrame({
    "grupo": ["A", "A", "B", "B", "B"],
    "valor": [10, 20, 30, 40, 50],
})

# Agrupar valores en listas
agrupado = df.group_by("grupo").agg(
    pl.col("valor")  # sin .sum() ni .mean() → crea una lista
)
# shape: (2, 2)
# ┌───────┬───────────────┐
# │ grupo ┆ valor         │
# │ ---   ┆ ---           │
# │ str   ┆ list[i64]     │  ← tipo List
# ╞═══════╪═══════════════╡
# │ A     ┆ [10, 20]      │
# │ B     ┆ [30, 40, 50]  │
# └───────┴───────────────┘
```

### Operaciones sobre listas

```python
agrupado.with_columns(
    pl.col("valor").list.len().alias("n"),            # longitud
    pl.col("valor").list.sum().alias("total"),         # suma
    pl.col("valor").list.mean().alias("promedio"),     # media
    pl.col("valor").list.get(0).alias("primero"),      # primer elemento
    pl.col("valor").list.get(-1).alias("ultimo"),      # último elemento
    pl.col("valor").list.contains(20).alias("tiene_20"),  # contiene
    pl.col("valor").list.sort().alias("ordenado"),     # ordenar
)
```

### `.list.eval()` — expresiones dentro de listas

La operación más poderosa: ejecutar una expresión **dentro de cada lista**:

```python
agrupado.with_columns(
    # Normalizar cada lista (restar la media de ESA lista)
    pl.col("valor").list.eval(
        pl.element() - pl.element().mean()
    ).alias("normalizado"),

    # Diferencias consecutivas dentro de cada lista
    pl.col("valor").list.eval(
        pl.element().diff()
    ).alias("diffs"),
)
```

`pl.element()` dentro de `.list.eval()` se refiere a cada elemento de la lista — es como `pl.col()` pero dentro del contexto de una lista.

### Explode e implode

```python
# explode: deshacer listas → una fila por elemento
agrupado.explode("valor")
# ┌───────┬───────┐
# │ grupo ┆ valor │
# │ str   ┆ i64   │
# ╞═══════╪═══════╡
# │ A     ┆ 10    │
# │ A     ┆ 20    │
# │ B     ┆ 30    │
# │ B     ┆ 40    │
# │ B     ┆ 50    │
# └───────┴───────┘

# implode: reagrupar → de vuelta a listas (inverso de explode)
# equivale a group_by + agg sin agregación
```

El round-trip `group_by → agg → explode` es un patrón fundamental para operaciones que necesitan contexto de grupo.

> **Verifica en el notebook:** Notebook 01 — Sección 6 construye columnas List, aplica `.list.eval()` con diferentes expresiones, y hace el round-trip explode/implode.

## 7. Struct: datos anidados

Un `Struct` es un registro con campos nombrados — piensa en un diccionario tipado por fila:

```python
df = pl.DataFrame({
    "nombre": ["Alice", "Bob"],
    "coordenadas": [{"lat": 40.4, "lon": -3.7}, {"lat": -12.0, "lon": -77.0}],
})
# La columna "coordenadas" tiene tipo Struct con campos lat y lon

# Acceder a campos del struct:
df.with_columns(
    pl.col("coordenadas").struct.field("lat").alias("latitud"),
    pl.col("coordenadas").struct.field("lon").alias("longitud"),
)

# Crear structs desde columnas existentes:
df2 = pl.DataFrame({
    "nombre": ["Alice", "Bob"],
    "lat": [40.4, -12.0],
    "lon": [-3.7, -77.0],
})
df2.with_columns(
    pl.struct("lat", "lon").alias("coordenadas")
)
```

Los Struct son útiles cuando tienes datos anidados (JSON, APIs) y quieres mantener la estructura sin aplanar todo en columnas separadas.

## 8. LazyFrame: el flujo de trabajo lazy

El modo lazy es donde Polars brilla. El flujo completo:

### Paso 1: crear un LazyFrame

```python
# Desde archivo (NO carga datos):
lf = pl.scan_csv("datos.csv")
lf = pl.scan_parquet("datos.parquet")

# Desde un DataFrame existente:
lf = df.lazy()
```

### Paso 2: encadenar transformaciones

```python
lf = (
    pl.scan_csv("ventas.csv")
    .filter(pl.col("monto") > 100)
    .with_columns(
        pl.col("fecha").str.to_date("%Y-%m-%d"),
        (pl.col("monto") * 0.16).alias("iva"),
    )
    .group_by("categoria").agg(
        pl.col("monto").sum().alias("total"),
        pl.col("monto").count().alias("n_ventas"),
    )
    .sort("total", descending=True)
)
```

Nada se ha ejecutado todavía.

### Paso 3: inspeccionar el plan

```python
# Plan optimizado (lo que Polars ejecutará):
print(lf.explain())

# Plan sin optimizar (lo que tú escribiste):
print(lf.explain(optimized=False))
```

### Paso 4: ejecutar

```python
result = lf.collect()       # ejecuta y retorna un DataFrame
# o
lf.collect(streaming=True)  # ejecuta en modo streaming (para datos grandes)
```

### Cuándo usar lazy vs eager

```
¿Cuándo usar lazy?
├─ Lectura de archivos → SIEMPRE (scan_csv > read_csv)
├─ Pipeline de múltiples pasos → SIEMPRE (el optimizador ayuda)
├─ Datos grandes → SIEMPRE (projection/predicate pushdown)
└─ Exploración rápida en Jupyter → eager puede ser más cómodo

Regla general: usa lazy por defecto, eager solo para exploración interactiva.
```

> **Verifica en el notebook:** Notebook 01 — Sección 8 construye un LazyFrame paso a paso, muestra `.explain()` en cada etapa, y compara el plan optimizado vs el no optimizado.

:::exercise{title="Predecir el plan optimizado"}

Dado este código:

```python
lf = (
    pl.scan_csv("productos.csv")         # 20 columnas
    .filter(pl.col("precio") > 50)
    .with_columns(
        (pl.col("precio") * 1.16).alias("precio_con_iva")
    )
    .select("nombre", "precio_con_iva")
    .sort("precio_con_iva", descending=True)
)
```

1. ¿Cuántas columnas leerá Polars del CSV? (pista: projection pushdown)
2. ¿En qué momento se aplica el filtro `precio > 50`? (pista: predicate pushdown)
3. Escribe lo que esperarías ver en `lf.explain()`.

:::
