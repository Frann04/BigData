# Apuestas deportivas con MapReduce

Trabajo Práctico N°1 de la asignatura **Big Data**, correspondiente a la carrera **Ingeniería en Computación** de la **UNLP** (Universidad Nacional de La Plata).

Cuatro consultas sobre grandes volúmenes de apuestas, resueltas con el modelo **MapReduce** usando el emulador `MRE` en Python (Google Colab).

## Datasets

Hay tres datasets, uno por cada grupo de usuarios: **Estándar**, **Premium** y **Platinium**. Cada línea es un partido, con los campos separados por tabulador:

| Campo | Descripción |
|-------|-------------|
| `ID_equipo_local` | Identificador del equipo local |
| `ID_equipo_visitante` | Identificador del equipo visitante |
| `apuestas_local` | Usuarios que apostaron al local como ganador |
| `apuestas_visitante` | Usuarios que apostaron al visitante como ganador |

Ejemplo:

```
65	31	123	487
1	73	1896	186
```

> Se asume que la cantidad de usuarios y de equipos es **Big Data**: ninguna solución carga listas completas de equipos o partidos en memoria.

## Consultas

| # | Consulta | Operación de conjuntos | Datasets |
|---|----------|------------------------|----------|
| 1 | Equipos con **más de X apuestas** como ganador (locales o visitantes) | Unión | Premium ∪ Estándar |
| 2 | Equipos que **de visitante** recibieron más apuestas que el local | Intersección | Estándar ∩ Platinium |
| 3 | Equipos que **de local** recibieron más de **X veces** las apuestas del visitante | Diferencia | Platinium − Premium |
| 4 | Equipos que pertenecen a **exactamente un** conjunto | `[Plat − (Prem ∪ Est)] ∪ [Prem − (Plat ∪ Est)] ∪ [Est − (Plat ∪ Prem)]` | Los tres |

## Cómo se resuelve cada punto

### Punto 1: Unión con suma por origen
- **Map:** emite dos pares por partido, `(equipo, (origen, apuestas))`, uno para el local y otro para el visitante.
- **Combiner:** suma parcial por equipo y origen. No filtra por X porque solo ve datos parciales.
- **Reduce:** suma el total por origen y emite el equipo si supera X en estándar **o** en premium.

### Punto 2: Intersección
- **Map:** si `ap_visitante > ap_local`, emite `(visitante, etiqueta del dataset)`.
- **Combiner:** deduplica etiquetas con `set()`.
- **Reduce:** emite el equipo si llegaron **ambas** etiquetas (`EST` y `PLAT`). No hace falta que sea el mismo partido.

### Punto 3: Diferencia
- **Map:** si `ap_local > X * ap_visitante`, emite `(local, etiqueta del dataset)`. Se multiplica en lugar de dividir para evitar la división por cero.
- **Combiner:** deduplica etiquetas.
- **Reduce:** emite el equipo si está `PLAT` y **no** está `PREM`. La diferencia no es simétrica.

### Punto 4: Exactamente un conjunto
- **Map:** no filtra. Platinium emite solo locales, Premium solo visitantes y Estándar ambos.
- **Combiner:** deduplica etiquetas.
- **Reduce:** traduce la fórmula a lógica booleana (resta = `and not`, unión = `or`) y escribe el equipo junto con el conjunto al que pertenece.

## Ideas clave de diseño

- **Etiquetado de origen:** cada map marca de qué dataset viene el dato, y el reduce decide con esas etiquetas. Así las operaciones de conjuntos se resuelven en un solo job.
- **El combiner nunca decide:** solo ve una parte de los datos, por lo que se limita a sumar (punto 1) o deduplicar (puntos 2, 3 y 4). Las decisiones finales se toman en el reduce.
- **El shuffle agrupa, no suma:** el reduce recibe una lista de largo variable por equipo, con tantos valores como mappers donde apareció.
- **Un map por dataset:** el emulador permite asociar una función map distinta a cada ruta de entrada con `addInputPath`.

## Estructura del repositorio

```
.
├── MRE.py                    # Emulador de MapReduce
├── punto1.py                 # Unión (Premium ∪ Estándar)
├── punto2.py                 # Intersección (Estándar ∩ Platinium)
├── punto3.py                 # Diferencia (Platinium − Premium)
├── punto4.py                 # Exactamente un conjunto
├── apuestasEstandar.txt
├── apuestasPremium.txt
├── apuestasPlatinium.txt
└── README.md
```

## Cómo ejecutarlo en Google Colab

1. Subir `MRE.py` y los tres `.txt` directamente a `/content` (al lado de la carpeta `sample_data`, sin crear ninguna carpeta).
2. Cargar el código del punto deseado en una celda y ejecutarla.
3. Revisar el resultado en la carpeta de salida (por ejemplo `/content/salida_punto1`).

El emulador lee **carpetas**, no archivos sueltos. Por eso cada driver crea una carpeta por dataset, copia el `.txt` adentro y borra la salida anterior para poder re-ejecutar sin errores:

```python
os.makedirs(input_est, exist_ok=True)
shutil.copy("/content/apuestasEstandar.txt", input_est + "/apuestasEstandar.txt")
shutil.rmtree(output_dir, ignore_errors=True)
```

### Parámetros

| Punto | Parámetro | Dónde se configura |
|-------|-----------|--------------------|
| 1 | `X`: mínimo de apuestas | `job1.setParams({"X": 1000})` |
| 3 | `X`: factor de apuestas local / visitante | `job3.setParams({"X": 2})` |

## Salidas

| Punto | Formato de salida |
|-------|-------------------|
| 1, 2, 3 | ID del equipo |
| 4 | ID del equipo y conjunto al que pertenece (`PLAT`, `PREM` o `EST`) |

## Tecnologías

- Python 3
- Emulador MapReduce `MRE`
- Google Colab
