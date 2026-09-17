# Generador de planos de assemblies de ACERO — Revit 2027

Pipeline de graphs de Dynamo (4.0, CPython3, **sin paquetes externos**) para documentar
assemblies de estructura metálica: plantas N.I.P.B., P.T. y T.A. sobre niveles que modela el
proyectista, elevaciones por eje, distribución en láminas y tabla de cantidades.

> **El modelo declara, el pipeline obedece.** Dos convenciones sostienen todo 00: el
> parámetro **`ASSEMBLY`** de cada eje decide qué elevaciones se hacen y qué ejes se ven, y
> el **nombre de cada nivel** (`N.I.P.B._`, `P.T._`, `T.A._`) decide qué plantas se hacen y a
> qué cota. Sin ese marcado, 00 no inventa nada: avisa en el log y no crea la vista.

> **Origen**: este proyecto (T003) es una adaptación de `T001-Generador de planos`
> (fundaciones). T001 conserva intacta la lógica de fundaciones — aquí se reemplazó,
> no se extendió. Si buscas el comportamiento de zapatas/pedestales/capa de cimiento,
> está en T001.

## Requisitos

- Revit 2027 (los .dyn están guardados en formato Dynamo 4.0).
- **Assemblies ya creados en el proyecto.** Todo el pipeline es assembly-driven: un
  assembly = una zona/sector a documentar = un bloque de láminas propio.
- Viñetas (title blocks) cargadas en el proyecto.
- Ningún paquete de Dynamo: todo es out-of-the-box + Python embebido.

> **Estado al 2026-07-28**: el modelo de referencia (`LDS25-099058-M3D-340-1000`) ya tiene
> los assemblies **ES-1001, ES-1002 y ES-1003**, y 00 genera 40 vistas sobre ellos
> (13 plantas + 27 elevaciones). Sin assemblies ningún graph produce nada: 00 avisa en el
> log y termina.
>
> ⚠️ **Desde el 2026-08-10 esas 40 vistas ya no salen solas.** Hay dos preparaciones en el
> modelo que ahora son requisito, no opcionales:
>
> 1. **Parámetro `ASSEMBLY` en los ejes** (proyecto, texto, categoría *Grids*), con el nombre
>    del assembly. Sin él: cero elevaciones y plantas sin ejes.
>    Ver [00 § D](#d-elevaciones-de-eje).
> 2. **Niveles nombrados** `{PREFIJO}_{assembly}` o `{PREFIJO}_{N}_{assembly}`, con
>    `N.I.P.B._`, `P.T._` y `T.A._`.
>    Sin ellos: cero plantas. Ver [§ Los niveles los modela el proyectista](#los-niveles-los-modela-el-proyectista).
>
> **Estado al 2026-09-16.** Se midieron dos modelos con el MCP de Revit, y cada uno escribe
> los niveles distinto — de ahí las dos pasadas del cambio de formato.
>
> `MODELO DE PRUEBA_KPF (1)` — 34 niveles, 4 assemblies. T.A. con correlativo adelante y el
> primero pelado (`T.A._ES-1001`, `T.A._2_ES-1001`…); P.T. sin correlativo, uno por assembly:
>
> | Assembly | NIPB | P.T. | T.A. |
> |---|---|---|---|
> | ES-1001 | ✅ | 1 | 4 |
> | ES-1002 | ✅ | ❌ | 2 |
> | ES-1003 | ❌ | 1 | 1 |
> | ES-1004 | ✅ | ❌ | 3 |
>
> `1818-2120-S-MOD-001_detached` — 5 niveles, 1 assembly. Todo con correlativo de 2 dígitos
> adelante, y **dos P.T.** en el mismo assembly:
>
> ```
> N.T.                 EL. 57,600   (no es de assembly)
> N.I.P.B._ES-1001     EL. 57,655
> T.A._01_ES-1001      EL. 64,030
> P.T._01_ES-1001      EL. 69,155
> P.T._02_ES-1001      EL. 70,355
> ```
>
> Las dos formas calzan con el matcher actual. Faltan `P.T._ES-1002`, `N.I.P.B._ES-1003` y
> `P.T._ES-1004` en el primero: esos assemblies salen sin esa planta hasta que alguien
> modele el nivel.

**Primera vez en una máquina nueva**: abrir cada .dyn en Dynamo (no en Player), correr y
guardar. Eso registra los inputs para Dynamo Player. Después, todo se opera desde Player.

## Orden de ejecución

| # | Graph | Estado | Qué hace |
|---|-------|--------|----------|
| 0 | `00_Vistas de assembly.dyn` | ✅ acero | Por assembly: 1 planta NIPB + N plantas P.T. + N plantas T.A., **una por nivel modelado**, + 1 elevación por cada eje **marcado con el parámetro `ASSEMBLY`** |
| 1 | `01_Calcular y crear laminas.dyn` | ✅ acero | Calcula cuántas láminas hacen falta (1 assembly por lámina) y las crea |
| 2 | `02_Colocar vistas en laminas.dyn` | ✅ acero | Coloca las vistas en flujo, más las leyendas |
| 3 | `03_Ejes y cotas entre ejes.dyn` | 🧪 acero | Ejes y cadenas de cotas en las plantas, más un tag `ELEMENTO_TBL` por conexión en la NIPB (una por familia contenedora, sin bajar a sus piezas internas); en las elevaciones deja los dos ejes extremos y agrega cotas de altura, marcas de nivel (desde los `Level` modelados), el eje de cada viga y línea de terreno — **lo de elevaciones y los rótulos de la NIPB, sin probar en Revit** |
| 4 | `04_Grating en plantas.dyn` | 🧪 acero | Dibuja el grating de las T.A. como Filled Regions recortadas contra las vigas — **sin probar en Revit** |
| 5 | `05_Rotulos de perfil.dyn` | 🧪 acero | Multi-Category Tag `C-MultiCat` sobre cada viga de las T.A. y **uno por tipo de familia** en las elevaciones (vigas, columnas, escaleras y barandas), eligiendo tipo corto o largo según lo que entre en la pieza — **plantas OK, elevaciones sin volver a probar** |
| 6 | `06_Tabla de assemblies.dyn` | 🧪 acero | Una *lista de materiales* por assembly, en su primera lámina — **sin probar en Revit** |

**Flujo**: 00 (crear vistas) → 01 (calcular y crear láminas) → 02 (colocar vistas y
leyendas) → 03 (ejes y cotas entre ejes) → 04 (grating) → 05 (rótulos de perfil) → 06
(tabla). 06 va al final porque necesita las láminas ya armadas para saber cuál es la
primera de cada assembly.

> El `04_Cotas de ejes.dyn` de fundaciones (cadena borde → eje de elemento → borde) se
> descartó y el número quedó libre; el 04 de acero es otra cosa. Si hace falta recuperarlo,
> está en el historial de git y en T001.

> `03` se llamaba `03_Cotas generales.dyn` y hacía cotas de ancho/largo/altura y spot
> elevations sobre `Structural Foundation`. Esas anotaciones no sirven en acero, así que
> el graph se **reemplazó** por completo y se renombró. No confundir con `04`, que acota
> el eje de cada **elemento** (fundaciones); `03` acota entre **ejes estructurales**.

---

## 00 — Vistas de assembly

### Por qué las plantas NO son vistas de assembly

`AssemblyViewUtils.CreateDetailSection` es el camino natural, pero **una vista de assembly
limita lo que muestra a los miembros del ensamble y por eso nunca dibuja datums**. Medido
en el modelo real (2026-07-28):

| Vista | Clase | Ejes visibles |
|---|---|---|
| `ES-1001 - NIPB` creada como vista de assembly | `ViewSection` | **0** |
| `ESTRUCTURA ES-1001 - PLANTA` del proyecto | `ViewPlan` | **54** |

Sin ejes visibles tampoco se ven las **cotas entre ejes de 03**, porque sus referencias son
justamente los ejes: las 26 cadenas se creaban y sobrevivían al commit, pero la vista no
dibujaba ninguna. El síntoma en el log era `vistas con Grids encendido: 0` — 03 preguntaba
`GetCategoryHidden`, Revit contestaba «no está oculta», y no había nada que encender.

Por eso las plantas son **`ViewPlan` normales**, recortadas al assembly por `CropBox` y
**aisladas** a sus miembros + **los ejes marcados con `ASSEMBLY`** + los marcadores de sus
elevaciones (`IsolateElementsTemporary` → `ConvertTemporaryHideIsolateToPermanent`). Es la
misma técnica que ya funcionaba en las elevaciones de eje.

**Consecuencias**:

- Las plantas dejan de colgar del nodo *Assemblies* del Project Browser y aparecen como
  plantas estructurales normales (tipo `02_ESTRUCTURAS`). El resto del pipeline no se
  entera: 01, 02 y 03 buscan las vistas por nombre.
- El tipo de vista de las plantas es un input (default `02_ESTRUCTURAS`).

### Carpeta del Project Browser

Se escribe **una palabra** en `04. Subcarpeta del Project Browser` y todas las vistas del
pipeline quedan agrupadas bajo ese nombre. Cómo funciona:

El Project Browser agrupa las vistas por **Family and Type**. La API de Revit no puede crear
carpetas directamente, pero sí puede crear **tipos de vista**, y eso produce exactamente el
mismo resultado visible. Así que 00 duplica el tipo base (`05. Tipo de vista BASE`, default
`02_ESTRUCTURAS`) en un tipo nuevo con el nombre de la subcarpeta y se lo asigna a todo lo
que crea. Sin esquemas, sin parámetros compartidos, sin configuración manual.

- Vacío = las vistas usan directamente el tipo base.
- Si el tipo ya existe, se reutiliza (no se duplica en cada corrida).
- El grafismo se hereda del tipo base, así que las vistas se ven igual que antes.

#### ⚠️ La carpeta define un JUEGO de vistas, no solo una carpeta

**Cambio 2026-09-17.** El nombre de la carpeta entra también en el **nombre de cada vista**:

```
sin carpeta:      ES-1001 - T.A. 01
carpeta REV_B:    ES-1001 - T.A. 01 [REV_B]
```

Antes la identidad de una vista era **su nombre pelado**, y `view_ids` se indexaba sólo por
ahí. Volver a correr 00 en un proyecto ya maquetado caía sobre **las mismas vistas**: se las
reutilizaba y se les reaplicaban todos los ajustes, o con «recrear» se las borraba y salían
de la lámina. El modelador que ya había acotado y ordenado sus planos perdía el trabajo.

Con la marca, correr 00 con **otro nombre de carpeta crea un juego nuevo** y deja el anterior
intacto. Revit exige nombres de vista únicos, así que meter la marca en el nombre no es una
preferencia: es la única forma de que dos juegos convivan.

**La marca va al final a propósito.** 01 a 05 buscan sus vistas con
`startswith(tname + SUF_*)`, y un sufijo no rompe ninguno de esos prefijos.

**No hizo falta un input nuevo en ningún graph**, porque los tres de anotación ya saltan lo
que está hecho: 03 con `ya_cotas` / `ya_marcas`, 04 y 05 con su input «rehacer». Con dos
juegos conviviendo anotan el nuevo y respetan el viejo. Lo único que cambió en 01 y 02 es que
la NIPB pasó a buscarse **por prefijo** como las demás — con la marca al final, un
`get()` por nombre exacto ya no la encontraba.

> ⚠️ **`endswith(MARCA)` no alcanza.** Con la carpeta vacía la marca es `''` y
> `endswith('')` da `True` para **todo**, así que «recrear» sin carpeta se llevaría puestas
> las vistas de todos los juegos del proyecto. Por eso `es_de_este_juego()` compara la marca
> **completa** (`juego_de()` la extrae del final del nombre) en vez de usar `endswith`.
> Cazado por el test de simulación antes de llegar a Revit.

> Para volver a generar un juego ya existente, se corre 00 con **ese mismo** nombre de
> carpeta: ahí sí cae sobre sus vistas, que es lo que se quiere.

> ⚠️ **Da dos nodos, no uno.** La *ViewFamily* (Floor Plans / Sections) es un nivel de
> agrupación **por encima** del tipo, así que se obtiene `Floor Plans > MI_CARPETA` con las
> plantas y `Sections > MI_CARPETA` con las elevaciones. Una carpeta única que mezcle
> plantas y elevaciones solo se consigue con un esquema de *Browser Organization* que agrupe
> por un parámetro compartido, y eso sí requiere configuración manual en Revit.

### Un solo campo para el tipo de vista

`05. Tipo de vista BASE` es **un único campo**. Plantas y elevaciones usan `ViewFamilyType`
distintos (Floor Plan vs Section), pero en este proyecto ambos se llaman igual
(`02_ESTRUCTURAS`), así que el graph resuelve el nombre dentro de cada `ViewFamily` por
separado. Antes eran dos campos y era fácil confundirlos entre sí y con el de la carpeta.

### Orden de los campos en Dynamo Player

Player ordena los inputs **alfabéticamente por etiqueta**, no por su posición en el canvas.
Por eso las etiquetas van numeradas con dos dígitos (`01.`, `02.`, … `16.`): sin el cero a
la izquierda, `10.` se ordenaría antes que `02.`. El orden resultante va de lo que se toca
siempre (qué procesar, carpeta, tipo de vista) a lo que casi nunca se toca (márgenes y
far clip).

> ⚠️ **Son 14 inputs y la numeración salta de la `10.` a la `13.`.** El 2026-08-10 se
> eliminaron `11. Tolerancia de agrupación T.A.` y `12. Mínimo de un nivel T.A.` junto con el
> clustering geométrico. Se dejó el hueco a propósito: renumerar habría invalidado todas las
> referencias por número que hay en este README. Los índices `IN[]` del código **sí** se
> corrieron, así que la etiqueta y el `IN[]` no coinciden — el mapeo vive en la cabecera del
> nodo Python.

### ⚠️ View Range: `Top` nunca puede igualar al `Cut`

Revit exige el orden estricto `Top ≥ Cut ≥ Bottom ≥ View Depth`. La primera versión ponía
**`Top = Cut`** («así no se ve nada por encima del plano de corte»), y eso rompía la vista
entera por un camino nada evidente:

1. Con el rango de altura colapsado a cero, el `CropBox` que Revit deriva del View Range
   queda con espesor nulo.
2. `ajustar_crop_a_assembly()` corre **justo después** y conserva ese `Min.Z`/`Max.Z` al
   reconstruir el crop → `BoundingBoxXYZ` degenerado.
3. Una vista con crop degenerado **no dibuja nada y Revit ni siquiera puede abrirla**.

Lo engañoso del síntoma: `FilteredElementCollector(doc, view.Id)` seguía contando elementos
visibles con normalidad (49 y 62 elementos en las T.A. de ES-1001, con 15 vigas, 5 ejes y
4 columnas cada una), porque el colector respeta el View Range y el aislamiento pero **no**
el crop. Todo indicaba que la vista tenía contenido; en pantalla estaba en blanco.

Ahora `Top = Cut + 10 cm` (constante `HOLGURA_TOP`): suficiente para que el rango sea
válido sin arrastrar el nivel de arriba a la planta. Además:

- si el fondo calculado queda por encima del corte, se fuerza a un rango sano y se avisa;
- `ajustar_crop_a_assembly()` tiene una última línea de defensa: si recibe un crop de
  espesor cero, le da 2 m y lo avisa;
- el log ahora imprime el **tamaño real del crop** de cada planta (`crop de planta T.A. 01:
  12.40 x 8.15 m`), para poder detectar de un vistazo un crop absurdo.

### ⚠️ Cota compartida: usar `ProjectElevation`, nunca `Elevation`

`ViewPlan.Create` exige un `Level`, que se usa **solo como origen de los offsets del View
Range** (el criterio de altura sigue siendo geométrico; el nivel no decide nada). Pero este
modelo tiene **«Elevation Base = Shared»**:

| Dato | Valor |
|---|---|
| Columna `HN20x46,0`, bounding box interno | Z = 3,84 → 10,83 pies |
| Parámetro `Elevation` de su nivel | 7549,6 pies |
| Diferencia | 7545,8 pies = **2300 m**, la cota del salar |

Los niveles reportan la cota real del sitio mientras la geometría vive cerca del origen del
proyecto. Anclar el View Range con `Level.Elevation` mandaba el plano de corte ~7538 pies
fuera del modelo: **las plantas salían completamente vacías**, y sin plano de corte que
cruzara sus extensiones los ejes tampoco se dibujaban. `Level.ProjectElevation` va siempre
en coordenadas de proyecto, el mismo sistema en el que se mide la geometría — es la que usa
`elev_nivel()` para ordenar niveles, elegir el ancla y calcular los offsets. Si el modelo
usa cota compartida, 00 lo avisa en el log al arrancar.

### Los niveles los modela el proyectista

Desde el **2026-08-10** las tres clases de planta salen de `Level` reales del modelo, con un
nombre que declara a qué assembly pertenecen. 00 ya no deduce cotas de la geometría.

| Nivel | Nombre | Cuántos | Planta que genera |
|---|---|---|---|
| Nivel inferior placa base | `N.I.P.B._{assembly}` | 1 | `{assembly} - NIPB` |
| Punto de trabajo | `P.T._{assembly}` y `P.T._{N}_{assembly}` | N | `{assembly} - P.T. 01`, `02`… |
| Tope de acero | `T.A._{assembly}` y `T.A._{N}_{assembly}` | N | `{assembly} - T.A. 01`, `02`… |

Ejemplo para `ES-1001`: `N.I.P.B._ES-1001`, `P.T._01_ES-1001`, `P.T._02_ES-1001`,
`T.A._01_ES-1001`.

#### ⚠️ El correlativo va ADELANTE, y el primero puede no llevarlo

**Una sola regla para las tres clases**: `{PREFIJO}_{assembly}` o
`{PREFIJO}_{correlativo}_{assembly}`. El nombre tiene que **empezar** con el prefijo y
**terminar** con el nombre del assembly; el correlativo, si existe, va **en el medio**.

Esto se corrigió en dos pasadas el **2026-09-16**, las dos por el mismo motivo: el graph
esperaba el correlativo **después** del assembly y los modeladores lo escriben **antes**.

**Primera pasada — los T.A.** 00 esperaba `T.A._{assembly}_{NN}`, con correlativo obligatorio.
Ningún nivel tenía esa forma, así que **no se creaba ni una sola planta T.A.** Medido sobre
`MODELO DE PRUEBA_KPF (1)` (34 niveles), los 10 T.A. estaban escritos así:

```
T.A._ES-1001     T.A._2_ES-1001   T.A._3_ES-1001   T.A._4_ES-1001
T.A._ES-1002     T.A._2_ES-1002
T.A._ES-1003
T.A._ES-1004                      T.A._3_ES-1004   T.A._4_ES-1004
```

**Segunda pasada — los P.T.** El mismo día apareció `1818-2120-S-MOD-001_detached` con
`P.T._01_ES-1001` y `P.T._02_ES-1001`: los modeladores extendieron el patrón a los puntos de
trabajo, y además **son dos en el mismo assembly** (EL. 69,155 y 70,355). 00 seguía buscando
`P.T._{assembly}` y no salía ninguna planta P.T.

Se cambió **el graph**, no el modelo: es la forma que los modeladores ya usan, y la que se
desprende de la codificación acordada `TIPO DE NIVEL_NOMBRE DEL ASSEMBLY`. `calza_nivel` y
`parte_ta` se eliminaron; quedan `parte_serie(lname, pref, tname)` y `calza_serie()`, los
mismos para NIPB, P.T. y T.A.

> ⚠️ **El N.I.P.B. sigue siendo uno por assembly** — es el nivel inferior de las placas base,
> no hay dos. Acepta el correlativo en el nombre (`N.I.P.B._01_ES-1001` calza), pero si hay
> más de uno se usa el más bajo y se avisa.

> ⚠️ Un efecto lateral de unificar el matcher: el sufijo **detrás** del assembly ya no vale
> para ninguna clase. `N.I.P.B._ES-1001_PL01` antes calzaba y ahora cae en la lista de
> sospechosos. No existe en ninguno de los dos modelos medidos, pero si aparece, se ve en el
> log en vez de desaparecer.

> ⚠️ **El número del nombre NO sigue la altura.** En `ES-1001`, `T.A._2_ES-1001` (EL. 2.304,17)
> está **40 cm por debajo** de `T.A._ES-1001` (EL. 2.304,57); y `ES-1004` tiene `3` y `4` pero
> no `2`. Eso no rompe nada — el correlativo de la **vista** sale de la altura, no del nombre —
> pero 00 avisa en cada desajuste. Ver [§ El número de la vista sale de la altura](#el-número-de-la-vista-sale-de-la-altura-nunca-del-nombre-del-nivel).

#### La regla es «empieza con Y termina con», no «contiene»

El nombre tiene que **empezar** con el prefijo y **terminar** con el nombre del assembly. No
alcanza con mencionarlos. El modelo real está lleno de niveles que hablan de T.A. o N.I.P.B.
sin ser de esta convención, y **ninguno** de ellos genera planta:

| Nivel del modelo | ¿Genera planta? | Por qué |
|---|---|---|
| `N.I.P.B._ES-1001` | ✅ | prefijo + assembly, sin correlativo |
| `P.T._01_ES-1001` | ✅ | correlativo adelante del assembly |
| `T.A._ES-1001` | ✅ | primero de la serie, sin correlativo |
| `T.A._2_ES-1001` | ✅ | ídem, con correlativo |
| `T.A._ES-1001_01` | ❌ | correlativo **atrás**: es la forma vieja, ya no vale |
| `T.A_ES-1001` | ❌ | le falta el punto: es `T.A_`, no `T.A._` |
| `T.A._ES_1001_01` | ❌ | `_` en vez de `-` dentro del nombre del assembly |
| `RLO_T.A. ES-1002` | ❌ | no empieza con el prefijo |
| `AMT_T.A.-01_EST. TOLVA` | ❌ | ídem |

Detalles del match:

- **Ignora mayúsculas/minúsculas y espacios sobrantes** en los extremos.
- Lo que va entre el prefijo y el nombre del assembly tiene que **cerrar en `_`**, o no haber
  nada. Sin esa regla el assembly `ES-100` se robaría `T.A._ES-1001`: ahí el «medio» sería un
  `1` suelto, sin guion bajo.
- Un nivel que **nombra al assembly pero no califica** se lista en el log como sospechoso:
  puede ser un nivel viejo legítimo, un typo, o un nombre en la forma vieja. 00 no adivina
  cuál, pero tampoco lo esconde. Ver [§ Los typos se reportan, no se adivinan](#️-los-typos-se-reportan-no-se-adivinan).
- Si hay **más de un** `N.I.P.B._` para el mismo assembly, se usa el más bajo y se avisa. En
  `P.T._` y `T.A._` varios son lo normal: generan una planta cada uno.

> ⚠️ **Sin nivel no hay planta.** No hay fallback geométrico: si falta `P.T._ES-1002`, ese
> assembly simplemente no tiene planta P.T., y el log lo dice. Es marcado faltante en el
> modelo, no un bug del graph.

#### ⚠️ Los typos se reportan, no se adivinan

**Cambio 2026-09-16.** El detector de sospechosos buscaba el nombre del assembly *literal*
dentro del nombre del nivel. Eso cubría los typos del **prefijo** (`T.A_ES-1001`,
`TA_ES-1001`, `N.I.P.B_ES-1001`), porque ahí el `ES-1001` sigue escrito tal cual. Pero si el
typo caía **dentro del token del assembly** el nivel desaparecía sin dejar rastro: para
`ES-1001`, un `T.A._ES_1001_01` no generaba planta **y tampoco aparecía en el log**.

Ahora la comparación del detector normaliza los dos lados — a minúsculas y sin nada que no
sea letra o número:

```python
def solo_alfanum(s):
    return ''.join(c for c in (s or '').lower() if c.isalnum())
...
elif solo_alfanum(tname) in solo_alfanum(n):
    sospechosos.append(n)
```

`T.A._ES_1001_01` → `taes100101`, que contiene `es1001`, así que cae en la lista de
sospechosos. Lo mismo con espacios (`T.A._ES 1001_01`) y con puntos de más o de menos en
cualquier posición.

> ⚠️ **Se amplió el detector, NO el matcher.** `calza_nivel` sigue siendo estricto a
> propósito: un nivel con typo **nunca** genera planta, solo aparece en el log para que
> alguien lo renombre en Revit. Si el matcher aceptara nombres aproximados, un
> `T.A._ES_1001` podría generar una planta del assembly equivocado a la cota equivocada, en
> silencio, y 01/02/03/06 construyen todos encima de eso. **Un aviso cuesta un renombre; un
> match errado cuesta un plano mal emitido.**

Es el mismo criterio que el parámetro `ASSEMBLY` de los ejes: el modelador declara, el graph
obedece y avisa cuando la declaración no se entiende.

Verificado contra los niveles reales del modelo: de 19 casos de prueba **solo 2 cambian de
clasificación**, los dos de `(ignorado)` a `sospechoso`. Ningún nivel cambia qué planta
genera, y los legacy (`RLO_T.A. ES-1002`, `AMT_T.A.-01_EST. TOLVA`) quedan exactamente como
estaban.

> **Pendiente**: la solución de fondo es sacar la semántica del nombre y ponerla en dos
> parámetros de proyecto sobre la categoría `Levels` — `ASSEMBLY` (texto) y `TIPO_NIVEL`
> (`NIPB` / `PT` / `TA`) — igual que ya se hizo con los ejes. Ahí el nombre del nivel pasa a
> ser cosmético y el match es exacto. Requiere poblar el parámetro en los niveles existentes
> y un fallback por nombre durante la transición.

### A. Planta N.I.P.B. (Nivel Inferior Placa Base)

Exactamente **1 por assembly**, sobre el nivel `N.I.P.B._{assembly}`. El plano de corte va
a **+1.00 m** del nivel (input `10.`, en cm) y el fondo, a `14.` cm por debajo.

00 compara el nivel contra la **cara sólida más baja real** del assembly (`Solid.Volume > 0`,
incluidas familias anidadas) y **avisa si difieren en más de 20 cm** — o el nivel está mal
puesto, o el assembly no es el que se cree. Avisa y sigue: **manda el nivel modelado**, que
es todo el punto del cambio.

- Nombre interno: `{assembly} - NIPB`
- Título en lámina: `{assembly} - PLANTA N.I.P.B.`

#### ⚠️ Qué se oculta en las plantas base (NIPB y P.T.)

**La NIPB y la P.T. comparten criterio.** Las dos miran la misma zona de anclaje, así que
lo que se documenta en ambas son **las placas y las sillas** (`Structural Connections`).
Una viga no aporta a esas cotas, y la escalera tampoco — salvo por las placas con las que
se ancla al piso. Pero **la escalera es una sola familia con todo anidado adentro**, así que
ocultarla entera se lleva puestas esas placas. Medido el **2026-08-11**: el contenedor
`ESCALERA METÁLICA` (id 6950307) va de `z 3,08` a `16,60` pies y sus dos `VM ANGULO`
(6950324 y 6950325) caen **dentro**, en `z 3,30`–`3,83` — o sea al pie de la escalera,
justo a la cota de la NIPB.

Por eso son **tres listas**, en la cabecera del nodo Python de 00 (no son inputs de
Player):

```python
FAMS_OCULTAR_BASE = ('ESCALERA',)                  # familias: esto y todo lo anidado adentro
CATS_OCULTAR_BASE = ('OST_StructuralFraming',)     # categorías: enteras
FAMS_SALVAR_BASE  = ('VM ANGULO', 'PLACA')         # …salvo esto, que gana sobre las dos
```

Se oculta una pieza si **matchea cualquiera de las dos primeras**; `FAMS_SALVAR_BASE` se
evalúa al final y gana sobre ambas.

##### Las placas de trabajo salen siempre

`PLACA` salva las **placas de trabajo** (`S-Placa Vertical` y compañía), que tienen que
verse siempre en la P.T.

> ⚠️ **Hoy esa línea no hace falta para que se vean — y por eso está.** `S-Placa Vertical`
> es `Structural Connections`, o sea justo la categoría que estas plantas existen para
> mostrar, y `CATS_OCULTAR_BASE` sólo tapa framing. La garantía se cumple **por accidente
> de categoría**. Ponerla en la lista la hace explícita: si mañana alguien agrega
> `OST_StructConnections` a `CATS_OCULTAR_BASE`, o si una placa termina anidada dentro de
> una familia que matchee `FAMS_OCULTAR_BASE`, la excepción la sigue salvando en vez de que
> desaparezca en silencio.

El match suelto por `PLACA` es a propósito —el modelador la nombra `S-Placa Vertical` o
algo con «placa»— y es **seguro**: medido el **2026-08-11**, las **387** instancias del
modelo cuya familia contiene `PLACA` son **362 `Structural Connections` + 25
`Generic Model`**, y **ninguna** es `Structural Framing`. O sea que la excepción no le abre
la puerta a ninguna viga.

> Las constantes y la función se llamaban `*_NIPB` / `ocultar_en_nipb()` mientras la regla
> era sólo de esa planta. Al sumarse la P.T. pasaron a `*_BASE` /
> `ocultar_en_planta_base()`: un nombre que dijera NIPB aplicándose también a la P.T. es
> exactamente el tipo de mentira que después cuesta cara.

> ⚠️ **Las dos primeras listas están acopladas por la tercera.** `VM ANGULO` es
> `Structural Framing`, igual que `ESCALERA METÁLICA`, `VM ESCALERAS` y `PILAR ANGULO`.
> Sin la excepción, la regla por categoría se llevaría puestas justo las placas de anclaje
> que la regla por familia se ocupa de salvar. Si algún día se saca `VM ANGULO` de
> `FAMS_SALVAR_BASE`, desaparecen por **dos** caminos distintos.

**Las columnas no se tocan**: son `Structural Columns`, otra categoría, así que se siguen
viendo apoyadas sobre su placa.

Las dos reglas **se componen y ninguna sobra**, aunque casi todo lo que oculta
`FAMS_OCULTAR_BASE` sea también framing: la baranda de la escalera es categoría
`Balusters`, y el grating y los detail items tampoco son framing.

Cómo funciona el match por **familia**:

- **«Contiene», ignorando tildes y mayúsculas.** No es nombre exacto: en el modelo
  conviven `ESCALERA METÁLICA`, `ESCALERA METÁLICA_1 ESC` y `ESCALERA METÁLICA1`, las tres
  con tilde. Con nombre exacto habría que listar las tres y la próxima variante volvería a
  fallar.
- **Se hereda del contenedor.** Una pieza anidada (`VM ESCALERAS`, los peldaños) no lleva
  «ESCALERA» en su propio nombre de familia, así que se sube la cadena de `SuperComponent`
  hasta la raíz. Basta con que **algún** contenedor matchee.
- **La excepción se evalúa sobre la familia de la pieza**, no del contenedor: `VM ANGULO`
  se salva aunque su escalera esté en la lista de ocultar.

> ⚠️ **La excepción es `VM ANGULO` completo, no `ANGULO`.** `PILAR ANGULO` también vive
> anidado en la escalera y también termina en «ANGULO». Hoy queda fuera de la NIPB por
> altura (`z 12,6` pies, muy por encima del corte), pero salvar `ANGULO` a secas lo dejaría
> entrar por la ventana.

**Solo afecta a la NIPB y a la P.T.** En las T.A. y en las elevaciones de eje la escalera y
las vigas se siguen viendo enteras — ahí la trama de vigas es justamente lo que se
documenta.

> Nada de esto rompe los pasos siguientes: 03 dibuja los ejes de viga **solo en las T.A.**
> (en la NIPB los borra a propósito) y en la NIPB rotula únicamente `Structural
> Connections`, mientras que las cadenas de cotas de esa planta referencian **ejes
> estructurales**, no vigas. La P.T. no recibe anotación de 03.

##### Por qué `HideElements` y no dejarla fuera del aislamiento

El aislamiento de `aislar_assembly()` es **de ida**: `ConvertTemporaryHideIsolateToPermanent`
no se revierte volviendo a correr con una lista más grande — hay que *Recrear* la vista (es
lo mismo que ya documenta [§ Qué ejes se ven en cada vista](#qué-ejes-se-ven-en-cada-vista)).
Estas reglas dependen de **nombres que el modelador va a querer corregir**, así
que se implementaron con `HideElements`/`UnhideElements`: cada corrida **desoculta primero los
miembros del assembly** y vuelve a decidir de cero. Cambiar la constante y re-correr
alcanza, sin recrear nada. Es el mismo patrón que usa 03 con los ejes de las elevaciones.

Desocultar **solo miembros** es lo que hace segura esa pasada: los miembros son justamente
lo que el aislamiento dejó visible, así que lo único que se revierte es lo que ocultó una
corrida anterior de esta misma función — nunca algo que haya ocultado el aislamiento.

> El log dice qué se ocultó (desglosado por familia) y cuántas piezas se salvaron. Si dice
> **«nada que ocultar»**, ningún miembro del assembly cayó en las reglas: si aun así ves la
> escalera o las vigas, es que **no son miembros de ese assembly** y quien las está dejando
> ver no es esta regla.

### B. Plantas P.T. (Punto de Trabajo)

**Una por cada nivel `P.T._{assembly}` o `P.T._{N}_{assembly}`**, ordenadas de abajo hacia
arriba por su cota real. Van **entre la NIPB y las T.A.** en la lámina.

Usan el **mismo view template que las T.A.** (input `06.`) y el **mismo offset de corte**
(input `13.`): son plantas de trabajo, no la de placas base, y no tienen inputs propios.

Pero para **qué se ve** comparten criterio con la NIPB, no con las T.A.: las dos miran la
zona de anclaje, así que la P.T. también oculta la escalera y el `Structural Framing`,
salvando las placas. Ver
[§ Qué se oculta en las plantas base](#️-qué-se-oculta-en-las-plantas-base-nipb-y-pt).

- Nombre interno: `{assembly} - P.T. 01`, `02`… (correlativo **ascendente por altura**, igual
  que las T.A.)
- Título en lámina: `{assembly} - PLANTA P.T. (EL. 69,155)` — la cota va en el título porque
  con dos P.T. en la misma lámina un título repetido no distingue una de otra

> ⚠️ **Eran 1 por assembly hasta el 2026-09-16.** La vista se llamaba `{assembly} - P.T.` a
> secas. Si un modelo ya tiene vistas con ese nombre viejo, **quedan huérfanas**: 00 no las
> vuelve a encontrar y crea las nuevas `- P.T. 01`. Hay que borrarlas a mano (o correr 00 con
> «recrear» activo, que ahora sí barre las P.T. previas del assembly).

> Esta planta obligó a tocar **01 y 02**, no sólo 00 — dos veces: al agregarla, y al pasarla a
> N. Los dos arman la lista de vistas de cada assembly a mano (`vistas_pendientes`), así que
> una vista nueva no se cuenta ni se coloca hasta que se la nombra explícitamente. Ahora la
> buscan por **prefijo**, como las T.A., y no por nombre exacto.

### C. Plantas T.A. (Tope de Acero)

Una por cada nivel `T.A._{assembly}` o `T.A._{N}_{assembly}`, **ordenadas de abajo hacia
arriba por su cota real** (`ProjectElevation`).

El correlativo que traiga el nivel es informativo: el número de la vista sale de la **cota**.
Ver [§ El número de la vista sale de la altura](#el-número-de-la-vista-sale-de-la-altura-nunca-del-nombre-del-nivel).

#### ⚠️ Las diagonales se ocultan; la profundidad la decide el log

En una planta T.A. lo que se documenta es la **trama horizontal** de ese nivel. Las
diagonales que cuelgan por debajo de las vigas —arriostramientos— se dibujan cruzadas
sobre la planta y la ensucian. Y aparecen **justo cuando se baja la profundidad de la
vista** para alcanzar las vigas más bajas, que es el problema que las trajo a colación.

Dos constantes en la cabecera del nodo Python de 00:

```python
OCULTAR_DIAGONALES_TA = True
ANG_DIAGONAL_GRADOS   = 15.0   # más inclinada que esto respecto de la horizontal
```

- **El criterio es geométrico**, por la pendiente de la `Location.Curve`, no por nombre de
  familia. Medido el **2026-08-11**: en este modelo **no existe ninguna familia** llamada
  `RIOSTRA`, `DIAGONAL` ni `BRACE` — las diagonales son perfiles comunes colocados
  inclinados, así que no hay ningún nombre al que agarrarse.
- **Una pieza vertical no es una diagonal.** En planta es un punto, y es información
  legítima (montantes, columnas cortas). Solo se ocultan las inclinadas.
- Mismo mecanismo reversible que la NIPB (`HideElements` + un `UnhideElements` previo
  sobre los miembros del assembly), así que cambiar el ángulo y re-correr recalcula desde
  cero, sin *Recrear*.

#### ⚠️ Vigas de dos colores: lo cortado se dibuja como lo proyectado

Revit dibuja con el gráfico de **Cut** lo que el plano de corte rebana, y con el de
**Projection** lo que queda por debajo. Son **dos ajustes distintos de la misma categoría**
en *Object Styles*, y si tienen colores distintos una misma tanda de vigas sale de dos
colores sin que nada del script lo haya pedido.

Medido el **2026-08-11** en `ES-1001 - T.A. 01`: el nivel está en `z 13,06` pies y el plano
de corte en `16,34`; las vigas `C25x17,9` de la banda `15,72`–`16,54` lo cruzan **6 cm por
debajo de su tope**, así que salen con el gráfico de *Cut* mientras el resto de la planta
sale con el de *Projection*.

```
   Cut plane  16,34  ──X───X───X──   ← lo que el plano rebana: gráfico de Cut
                       │   │   │
   nivel      13,06
   ═══════════════════════════════   ← todo lo de abajo: gráfico de Projection
```

**Mover el corte no lo arregla.** Bajarlo dejaría esas vigas *por encima* y desaparecerían
del todo; subirlo funciona pero no es robusto, porque depende de cuánto sobresalga la viga
más alta de cada assembly. Lo que hace 00 es **copiar el grafismo de *Projection* de la
categoría al override de *Cut* de la vista**, que no depende de ningún número:

```python
UNIFICAR_GRAFISMO_VIGAS = True
CATS_GRAFISMO_UNIFORME  = ('OST_StructuralFraming',)
```

> **El color no está escrito en el script.** Se lee de la propia categoría en runtime
> (`Category.LineColor` y `GetLineWeight(Projection)`), así que sigue siendo un estándar del
> proyecto: si mañana el verde cambia en *Object Styles*, las vistas lo siguen solas.

Se aplica **después** de `preparar_vista()`, que es quien pone el view template — a quien
hay que ganarle, igual que con las categorías que esa misma función enciende. Si un template
controla *V/G Overrides*, la API no deja sobrescribir y el log lo dice.

El log deja el color usado, para poder verificarlo de un vistazo:

```
ES-1001: en planta T.A. 01 el corte de OST_StructuralFraming se dibuja como la
proyeccion (RGB 0,128,0, peso 3).
```

#### ⚠️ El `View Depth` baja más que el `Bottom` (franja `<Beyond>`)

El problema de una planta T.A. no era sólo *cuánto* se ve hacia abajo, sino **con qué peso
de línea**. Hasta el 2026-08-11 el script pegaba `Bottom` y `View Depth` en el mismo valor,
así que todo lo que entraba se dibujaba igual. Medido en `ES-1001 - T.A. 01`: la vista
mezclaba las vigas de `AMT_T.A.-01` (`z 10,74`–`11,73` pies) con las de `AMT_T.A.-02`
(`z 12,06`–`13,04`) **sin ninguna diferencia gráfica entre los dos pisos**.

Revit dibuja lo que cae **entre `Bottom` y `View Depth`** con el estilo `<Beyond>` — línea
más fina. Es la diferencia entre *mostrar* y *mostrar como contexto*:

```
   Top          nivel +1100          ─┐
   Cut plane    nivel +1000           │  trama del nivel propio,
   Bottom       nivel  -500          ─┘  grafismo normal

   View Depth   nivel -1000          ─── franja <Beyond>: está, pero no compite
```

```python
PROF_BEYOND = 50.0 / 30.48   # cm → pies. Cuánto baja el View Depth por debajo del Bottom.
```

**Solo aplica a las plantas T.A.** La NIPB y la P.T. siguen con `Bottom == View Depth`: a
esas cotas no hay un “nivel de abajo asomando” que valga la pena mostrar como contexto.

> El orden estricto que exige Revit (`Top ≥ Cut ≥ Bottom ≥ View Depth`) se mantiene **por
> construcción**: `PROF_BEYOND` nunca es negativo, así que `View Depth` sólo puede quedar
> por debajo del `Bottom`, nunca por encima. Ver
> [§ View Range: `Top` nunca puede igualar al `Cut`](#️-view-range-top-nunca-puede-igualar-al-cut)
> para el otro extremo del mismo invariante, que sí llegó a romper vistas.

El log lo dice en la misma línea de siempre:

```
ES-1001: planta T.A. 01 cortada en EL. 4,98, fondo en EL. 3,48 (1,50 m de profundidad,
top +0,10 m, mas 0,50 m como <Beyond> hasta EL. 2,98).
```

##### La profundidad no se adivina: 00 la mide

**El input `14.` ya viene en 50 cm** y fija el `Bottom`; `PROF_BEYOND` agrega la franja de
contexto por debajo. Si una planta T.A. sigue sin mostrar toda su trama, el problema no se
arregla *poniendo* el input en 50 — hay que **bajarlo más**, y cuánto depende del modelo.

Por eso `ocultar_diagonales_ta()` mide de paso: por cada planta T.A. cuenta las piezas
**horizontales** que quedan enteras por debajo del **`View Depth`** —el plano de abajo real,
franja `<Beyond>` incluida, que es hasta donde la vista dibuja— y avisa cuánto haría falta:

```
ES-1001: OJO, 6 pieza(s) HORIZONTAL(es) de planta T.A. 02 quedan fuera del View Range;
la mas alta tiene su tope 0,74 m bajo el nivel. Para verlas hay que subir la
profundidad de las plantas a mas de 74 cm.
```

La medición está **acotada al hueco entre este nivel y el T.A. de abajo** (`z_piso`), para
que no reporte la trama entera de los niveles inferiores del assembly — que están fuera de
la vista a propósito.

> El orden importa: primero se sube la profundidad con ese número, y las diagonales que eso
> hubiera traído ya están filtradas por la regla de arriba.

- Nombre interno: `{assembly} - T.A. 01`, `02`… (correlativo **ascendente por altura**)
- Título en lámina: `{assembly} - PLANTA T.A. (EL. 2.303,37)` — cota real en metros, con
  punto de miles y coma decimal, igual que las vistas que ya existen en el modelo
  (`PLANTA ESTRUCTURA EL. 2.305,25 T.A.`)

#### El número de la vista sale de la altura, nunca del nombre del nivel

El correlativo del nivel es **informativo**. El número de la vista se asigna por posición en la
lista ordenada por cota, de modo que 01, 02 y 06 siempre ven una serie `01..NN` **sin huecos
ni repetidos**, pase lo que pase en el modelo. Ese correlativo es el nombre estable que usan
los pasos siguientes; la cota va sólo en el título mostrado, porque un nombre basado en la
cota rompería las búsquedas por nombre en cuanto alguien moviera el nivel.

#### ⚠️ La cota NO va en el nombre del nivel

El sufijo es **la numeración**, no la altura. La cota ya aparece en el título de la vista
(`PLANTA T.A. (EL. 2.304,18)`), y ponerla también en el nombre del nivel la duplica: dos
lugares que hay que mantener sincronizados a mano y que se contradicen en cuanto alguien
mueva el nivel.

> Esto se decidió el **2026-08-10** después de probar la alternativa. Durante la primera
> corrida con niveles, los `T.A.` se habían nombrado con la cota
> (`T.A._ES-1001_2.304,175`) y 00 llegó a validar ese número contra la `Elevation`
> compartida real. La validación funcionó —cazó un `T.A._ES-1001_2.308,825` que estaba en
> realidad a **2.308,393**, 43 cm de error— pero el problema de fondo era la duplicación,
> no la falta de chequeo. Con la cota fuera del nombre, ese error ya no puede existir.

Lo único que 00 comprueba del correlativo es que **coincida con la posición por altura**. El
T.A. pelado declara un `1` implícito, por ser el primero de la serie:

```
   ES-1001: el nivel T.A._2_ES-1001 dice "2" pero por altura es el 01; la vista se llama T.A. 01.
   ES-1001: el nivel T.A._ES-1001 dice "1" pero por altura es el 02; la vista se llama T.A. 02.
```

Esos dos avisos salen **hoy** con el modelo real: en `ES-1001` el `_2_` está 40 cm **por
debajo** del pelado. El pipeline funciona igual —la vista se llama por altura— pero o el
modelo está mal numerado o la numeración del proyectista no pretende seguir la altura. En
`ES-1004` pasa algo parecido: hay `3` y `4`, pero no `2`.

Un correlativo que no sea un número se acepta sin decir nada: es una etiqueta libre
(`T.A._INFERIOR_ES-1003` es un T.A. válido de `ES-1003`).

#### ⚠️ El corte se recorta contra el nivel vecino

El offset del corte es fijo (1,00 m) pero la separación entre niveles no. En **ES-1002** los
dos T.A. están a **1,05 m**, así que el corte de la planta 01 caía en EL. 7,15 — **por
encima** de las vigas secundarias del T.A. 02, que están en EL. 7,07 — y esa planta dibujaba
los dos niveles superpuestos.

El corte nunca alcanza al vecino: se queda **20 cm** por debajo del nivel de arriba
(`SEP_NIVEL`), y el fondo, 20 cm por encima del de abajo. Ambos recortes se avisan en el log.
El recorte mira **sólo los niveles T.A.**; el P.T. no participa.

#### Qué reemplazó esto (2026-08-10)

Hasta esta versión las plantas T.A. salían de un **clustering geométrico** de los topes de
viga: se extraía `(Z de la cara superior, largo en planta)` de cada viga, se detectaban picos
de densidad en cubetas de 5 mm ponderadas por metros de viga, se descartaban los niveles
minoritarios bajo el 40 % del mayor, y el nivel dominante era la cubeta con más metros.

Funcionaba —resolvía el sub-modelado del grating, el ruido de las barandas, los niveles
partidos al medio—, pero era **el graph adivinando una decisión de proyecto**. Gerencia
resolvió que los niveles se modelen explícitamente para tener el modelo ordenado, así que se
eliminaron `tops_de_vigas()`, `agrupar_por_altura()`, `z_dominante()`,
`filtrar_grupos_minoritarios()`, `cubeta()` y `nivel_ancla()`, junto con sus dos inputs de
Dynamo Player (`11. Tolerancia de agrupación T.A.` y `12. Mínimo de un nivel T.A.`).

> Las etiquetas del Player **saltan de la `10.` a la `13.`**: se prefirió dejar el hueco
> antes que renumerar referencias ya documentadas en este README. Los índices `IN[]` del
> código sí se corrieron, y quedaron 14 inputs.

Si hace falta recuperar el criterio viejo (por ejemplo para un modelo heredado sin niveles
nombrados), está en el historial de git, en el commit anterior a este cambio.

#### ⚠️ Las sillas de anclaje son `Structural Connections`, no `Structural Framing`

Las **sillas de anclaje, placas base y pernos** viven en la categoría
`OST_StructConnections`. El template de disposición general las apaga —a 1:150 serían
ruido— y sin ellas **la NIPB queda vacía**.

Medido el **2026-07-29** consultando la vista `ES-1001 - NIPB` ya creada: contenía 4
columnas, 2 vigas y 5 ejes, y **cero conexiones**, con las 95 sillas/placas/pernos del
assembly invisibles. El *View Range* estaba correcto (esos 6 elementos son exactamente lo
que vive entre EL. 0,67 y EL. 2,17); lo que fallaba era la **visibilidad por categoría**.

El síntoma engaña: se reporta como «la NIPB sale muy arriba y sin profundidad», porque lo
que queda son columnas flotando sin su base. No es el corte ni la profundidad.

Por eso 00 **enciende explícitamente** `Structural Connections` en todas las plantas,
después de aplicar el template (que es a quien hay que ganarle). Si el template controla
*V/G Overrides*, la API no deja sobrescribirlo y el log lo avisa: en ese caso hay que darle
a las plantas un template que muestre esa categoría.

### D. Elevaciones de eje

Una por cada eje que traiga escrito el **nombre del assembly en su parámetro `ASSEMBLY`**
(`ejes_del_assembly`). Ni uno más: es un marcado explícito, no un criterio geométrico.

#### El parámetro `ASSEMBLY` de los ejes

Es **el único** criterio de 00 sobre ejes, y contesta **dos** preguntas con la misma lista:
qué ejes llevan elevación, y qué ejes se ven en las vistas del assembly (los demás se
ocultan — ver [§ Qué ejes se ven en cada vista](#qué-ejes-se-ven-en-cada-vista)).

- **Dónde**: parámetro de **proyecto**, de tipo **texto**, en la categoría **Grids**
  (`Structural Grids`). 00 lo busca primero en la instancia y, si no está, en el tipo.
- **Qué lleva**: el nombre del assembly tal cual (`AssemblyTypeName`, ej. `ES-1001`). **Un
  solo nombre por eje.** La comparación ignora mayúsculas/minúsculas y espacios sobrantes.
- **Vacío o sin el parámetro** → ese eje no lleva elevación **y no se ve en ninguna vista**.
- Un eje que **no sea línea recta** se descarta con aviso: la caja de sección se alinea a la
  dirección del eje y un arco no tiene una.

| Situación | Qué hace 00 |
|---|---|
| Hay ejes con `ASSEMBLY` = el assembly | crea esas elevaciones y solo esas, y son los únicos ejes visibles en sus vistas |
| Ningún eje calza con este assembly | **cero elevaciones y plantas sin ningún eje**, `AVISO` en el log con los valores que sí aparecen en el modelo |
| El parámetro existe pero está vacío en todos | ídem, `ERROR` en el log |
| Ningún eje del modelo tiene el parámetro | ídem, `ERROR` en el log explicando cómo crearlo |

No hay fallback al criterio geométrico: si nadie marcó ejes, el assembly queda sin
elevaciones a propósito, y el log lo dice fuerte.

> ⚠️ **Sin marcado, las plantas salen sin ejes** — y sin ejes visibles, las cadenas de cotas
> de 03 se crean pero no se dibujan (es exactamente el síntoma documentado arriba en
> [§ Por qué las plantas NO son vistas de assembly](#por-qué-las-plantas-no-son-vistas-de-assembly)).
> Marcar los ejes **antes** de correr 00 no es opcional.

#### Por qué se reemplazó el criterio geométrico (2026-08-10)

Hasta esta versión la elevación se hacía si el eje cruzaba la huella (Liang-Barsky del
segmento contra el rectángulo del bbox) **y** tenía una viga o columna a menos de 30 cm
(`ejes_con_marco`, ahora eliminada). **Salían demasiadas elevaciones** y gerencia lo levantó
como observación.

El problema era de raíz: el test mide contra el **rectángulo** del bounding box, bastante
más grande que la huella real — un assembly en L, o uno alargado en diagonal, deja pasar
ejes que no tocan nada. Y no se arreglaba con tolerancia: medido el **2026-08-04**, 10 de
las 24 elevaciones quedaban vacías con la pieza más cercana entre **1,7 y 4,9 m**; subir la
tolerancia a 1 m no rescató ninguna de las diez y arruinó las que sí andaban (`ES-1002 EJE
E` pasó de 8 tags a **51 sobre 66** piezas, ilegible).

El marcado en el eje es explícito y lo controla quien arma el plano, que es exactamente la
decisión que el criterio geométrico estaba tratando de adivinar.

> ⚠️ **El marcado decide dos cosas a la vez**: qué ejes llevan elevación **y qué ejes se
> ven** en las vistas del assembly — el aislamiento de plantas y elevaciones se arma con esa
> misma lista. Un eje de otro sector no entra al aislamiento, o sea que queda **oculto**. Ver
> [§ Qué ejes se ven en cada vista](#qué-ejes-se-ven-en-cada-vista).

Estas **no** son vistas de assembly: `AssemblyViewUtils.CreateDetailSection` solo admite
orientaciones fijas (`HorizontalDetail`, `DetailSection A`–`D`), que no pueden dar «una
elevación por cada eje». Se crean con `ViewSection.CreateSection` y una caja de sección
alineada al eje (BasisX = dirección del eje, BasisY = vertical, BasisZ = hacia el
observador). Como una sección normal muestra todo el modelo dentro de su caja, el
assembly se **aísla explícitamente**: `IsolateElementsTemporary` con todos los miembros
recursivos **más el propio eje** (para que su burbuja siga visible), y luego
`ConvertTemporaryHideIsolateToPermanent` — el aislamiento temporal no sobrevive al cierre
de la vista.

#### Qué ejes se ven en cada vista

**Solo los del assembly.** Plantas y elevaciones se aíslan con la **misma lista** que decide
las elevaciones: los ejes cuyo `ASSEMBLY` calza. Los ejes de otros sectores no entran al
aislamiento y quedan **ocultos** — que era la segunda parte de la observación de gerencia.

En cada vista entra el eje propio (para que su burbuja siga visible) **más los otros ejes
marcados para ese mismo assembly**, que son los transversales que 03 necesita para sus
cadenas de cotas.

> ⚠️ Un eje que no entre en el aislamiento queda oculto **para siempre**: 03 enciende la
> categoría *Grids* pero eso no revierte un `ConvertTemporaryHideIsolateToPermanent`. Si
> falta un eje en una planta, es **marcado faltante en el modelo**, no un problema de 03 —
> se agrega el `ASSEMBLY` al eje y se re-corre 00 con *Recrear*.

Antes del 2026-08-10 el aislamiento usaba un criterio distinto al de las elevaciones (todos
los ejes que **cruzaban** el bbox, vía `ejes_que_cruzan` + `segmento_cruza_rect`, ambas ya
eliminadas). Eran dos criterios conviviendo en el mismo graph; ahora hay uno solo.

- Nombre interno: `{assembly} - EJE {nombre del eje}` (ej. `ES-1001 - EJE 13a`)
- Título en lámina: `{assembly} - ELEVACION EJE 13a`
- Se ocultan los símbolos de sección (`OST_Sections`) para que las elevaciones del mismo
  assembly no se crucen entre sí.

### Profundidad de vista

Plantas y elevaciones tienen **inputs separados**, porque son dos clases de vista con dos
mecanismos y dos necesidades distintas:

- **Plantas** (`ViewPlan`) → *View Range*, input `14. Profundidad de las plantas bajo el
  nivel (cm)`, default **50 cm**. Los cuatro planos se anclan al **nivel modelado de esa
  planta** (por `ProjectElevation`, ver arriba) y se expresan como offset:
  - `Top` = `Cut` + 10 cm (`HOLGURA_TOP`, ver más abajo — nunca puede ser cero).
  - `Cut` = nivel **+** su offset (input `10.` para la NIPB, `13.` para P.T. y T.A.,
    default 1,00 m).
  - `Bottom = View Depth` = nivel **− 0,50 m**.

  El fondo se mide **desde el nivel**, **no** desde el plano de corte: el corte va un offset
  por encima y no tiene por qué arrastrar la profundidad. Así cada planta T.A. muestra su
  nivel y no los de abajo — con los niveles del modelo separados 1,0–2,3 m, una profundidad
  mayor haría que cada planta arrastrara 2 o 3 niveles inferiores.

  > Este input **solo afecta a las plantas**: `prof_planta` se usa únicamente dentro de
  > `fijar_view_range()`, que se llama nada más que para las `ViewPlan`. Las elevaciones de
  > eje no tienen View Range; su profundidad la fija el *Far Clip Offset* de acá abajo.

- **Elevaciones de eje** (`ViewSection`) → *Far Clip Offset*, input **«Far Clip Offset de
  las ELEVACIONES (mm)»**, default **5500 mm**. Se escribe después del view template, así
  que si el template trae su propio valor gana el input. Si el modo de recorte lejano
  estuviera en *No clip*, se activa primero (`VIEWER_BOUND_ACTIVE_FAR`). Aquí un valor
  generoso es inocuo: la vista está aislada y solo muestra el assembly y sus ejes.

### Escala y view template

**Una sola escala para todo** (input, default **1:75**), plantas y elevaciones. Reemplaza
la escala automática 1:25/1:50 de fundaciones, que no aplica a estructuras de este tamaño.

Hay **tres campos de view template independientes** (dos vistas de familias distintas nunca
comparten template real en Revit, y la NIPB necesita uno propio):

- **`06. View template - PLANTAS T.A. y P.T.`**, default `DISP.GRAL_1/150_PLAN`. La planta
  P.T. comparte template con las T.A.: es una planta de trabajo, no la de placas base.
- **`07. View template - NIPB (debe mostrar conexiones)`**, default `ESTRUCTURAS_1/50`. Va
  aparte de las T.A. porque a esa cota lo que hay que ver son las sillas y placas base, que
  son `Structural Connections` — la categoría que el template de disposición general apaga.
- **`08. View template - ELEVACIONES DE EJE`**, default `ESTRUCTURAS_1/50`. Se aplica a las
  elevaciones por eje.

Los tres vienen pre-seteados así que el modelador no tiene que tocar nada para el caso normal,
pero siguen siendo **campos de texto editables** por si hace falta otro template puntual. Si
alguno queda vacío, esa familia de vistas se queda con el default de su *ViewFamilyType* y no
se fuerza ningún template. En cualquier caso la escala se **re-escribe después** del template
(input único de escala, ver más abajo), así que aunque el template fije una escala gana el
input. El log siempre lista los templates cargados en el proyecto — es la forma de descubrir
el nombre exacto, porque sin paquetes no hay dropdown nativo de view templates en Dynamo.

---

## 01 / 02 — Maquetado en láminas

### Regla de aislamiento de assemblies

**Cada lámina contiene vistas de UN SOLO assembly.** Las vistas de un assembly fluyen por
tantas láminas como haga falta, pero el assembly siguiente **siempre** empieza en una
lámina nueva. Nunca se mezclan dos assemblies en la misma lámina.

### Orden y flujo

Por assembly, en este orden: **planta NIPB → plantas P.T. ascendentes → plantas T.A. ascendentes →
elevaciones de eje** (los ejes en orden natural, de modo que `2` va antes que `13` y `13a`
después de `13`). Las vistas fluyen de izquierda a derecha y de arriba hacia abajo; al
llenarse la lámina se sigue en la siguiente del mismo assembly.

> ⚠️ **Este orden vive escrito en `vistas_pendientes()`, en 01 y en 02.** Ninguno de los dos
> descubre vistas nuevas solo: arman la lista a mano, la NIPB y la P.T. por nombre exacto, las
> T.A. y las elevaciones por prefijo. Una vista que 00 cree y que nadie nombre ahí **no se
> cuenta para las láminas ni se coloca en ninguna**. Fue exactamente lo que pasó al agregar la
> P.T.: hubo que tocar los tres graphs, no sólo 00.

Esto reemplaza el modelo de fundaciones de «bloque = planta arriba + 2 cortes debajo,
bloques en grilla», que no escala a un assembly con 1 + 4 + 8 vistas.

### Reserva para la tabla

La tabla de cantidades va **solo en la primera lámina de cada assembly**, así que la
reserva del lado derecho (input, default 150 mm) **se descuenta únicamente en esa
lámina**; las siguientes del mismo assembly aprovechan el ancho completo.

06 deduce esa lámina por su cuenta (la de `SheetNumber` más bajo, en orden natural, que tenga
alguna vista del assembly) — el usuario no elige láminas. Eso hace que la **reserva tenga que
coincidir en 01, 02 y 06**, no solo en 01 y 02.

> **Importante**: el input «Reserva para la tabla, lado derecho (mm)» y «Margen y
> separación (mm)» deben tener **el mismo valor en 01 y en 02**. La función `empaquetar()`
> es idéntica en ambos graphs a propósito: si divergen, 01 calcula una cantidad de láminas
> que no coincide con lo que 02 realmente coloca.

### Numeración de las elevaciones

El *Detail Number* del viewport de una elevación se fija al **nombre del eje**, no a una
letra correlativa. Así la marca de sección que se dibuja en la planta y el título de la
elevación dicen lo mismo (`EJE 13a`). Si ese número ya está usado en la lámina, cae a
`13a(2)`, `13a(3)`… para no romper la unicidad que exige Revit.

### Leyendas

Sin cambios respecto a T001: por cada sheet se buscan vistas de tipo *Legend* cuyo nombre
coincida (sin distinguir mayúsculas) con el nombre de **tipo de familia** (`Symbol.Name`)
de algún miembro del assembly de esa lámina, y se empacan en la **esquina inferior
derecha** del área útil. Cada sheet recibe su **propio duplicado** de la leyenda
(`{tipo} - {sheet}`), porque `Cantidad_Inserto` es un parámetro de la *vista*: con una
leyenda compartida, 07 pisaría la suma de una lámina con la de otra.

---

## 03 — Ejes y cotas entre ejes

### Ejes visibles

Enciende la categoría *Grids* (`SetCategoryHidden(..., False)`) en las plantas NIPB y T.A.
y —si el input lo pide— también en las elevaciones de eje. Es idempotente: si la categoría
ya estaba encendida, no hace nada.

> El input `Mostrar ejes tambien en las elevaciones` es además el interruptor general de
> todo lo que 03 hace en las elevaciones: en False, no las toca.

### Una sola burbuja por eje: arriba y a la izquierda

Por defecto Revit dibuja las dos burbujas de cada eje, así que una planta queda con
burbujas arriba **y** abajo, a la izquierda **y** a la derecha. En las plantas se deja
**una sola** (input `Burbujas de eje: solo arriba y a la izquierda`, default True):

```
        (13a) (13b) (13c)          <- se conservan
   (Ea)   |     |     |    (Ea)
     +----+-----+-----+----+       <- la columna derecha se oculta
   (E)|   |     |     |    |(E)
     +----+-----+-----+----+
          |     |     |
        (13a) (13b) (13c)          <- la fila de abajo se oculta
```

Se usa `ShowBubbleInView` / `HideBubbleInView`, que son **específicos de la vista**: el
resto del proyecto sigue viendo sus ejes como siempre.

Qué extremo se conserva lo decide el eje de la vista que **más separa** a los dos extremos:
en un eje casi vertical manda la altura (se queda el de más arriba) y en uno casi horizontal
manda el ancho (el de más a la izquierda). Así también funciona con ejes oblicuos y con
vistas rotadas, sin depender de que `End0`/`End1` estén dibujados en un orden concreto —
que es cosa del modelador, no de Revit.

> Solo se aplica a las **plantas**. Las elevaciones de eje llevan su propio tratamiento,
> acá abajo.

### En las elevaciones: todos los ejes del assembly, recortados al alto

Una elevación solo dibuja los ejes **perpendiculares al papel** — el eje de la propia
elevación corre a lo largo de la vista y Revit no lo dibuja, así que queda fuera solo. De
los que sí se dibujan **se muestran todos**, con el mismo criterio que en las plantas: los
que pertenecen al assembly, o sea los marcados con el parámetro `ASSEMBLY`.

> ⚠️ **Esto cambió el 2026-08-11 y con él la razón de fondo.** Hasta entonces se conservaban
> sólo **los dos extremos** y se ocultaban los de por medio, con el argumento de que «la
> posición de cada eje ya está acotada en las plantas y repetirla en la elevación sólo tapa
> la estructura con líneas verticales».
>
> El corte pasó a llevar **su propia cadena de cotas entre ejes** (más abajo), y una cota no
> puede referenciar un eje oculto: se crea sin error y no se dibuja, la misma trampa ya
> documentada para las categorías apagadas. Sin todos los ejes a la vista, la cadena no
> tiene contra qué referenciarse.
>
> El input `Elevaciones: dejar solo los dos ejes extremos` **sigue existiendo pero viene en
> `False`**. Si alguna vez se lo vuelve a poner en True, la cadena entre ejes del corte se
> queda con dos referencias y se degrada sola a la cota total — no falla, pero deja de
> haber tramos.

Se usa `View.HideElements` / `UnhideElements` (específico de la vista). **Cada corrida
recupera primero los que ocultó la anterior** y vuelve a decidir cuáles son los extremos:
si el assembly creció o se agregó un eje, los de antes tienen que volver.

**La extensión** de los que quedan pasa a ser específica de la vista
(`SetDatumExtentType(..., ViewSpecific)` + `SetCurveInView`), del alto del assembly más lo
que sobresalga por arriba y por abajo (inputs en mm de papel, defaults **3** y **2**), con
la burbuja arriba. Por defecto el eje cruza la vista entera porque su extensión 3D es la
de toda la planta industrial; esa extensión **no se toca**, el recorte vive solo en esta
vista. Apagar el input devuelve los ejes a `DatumExtentType.Model`, no los deja congelados
con el recorte anterior.

> Cuál de los dos extremos de la curva quedó arriba se **lee de vuelta** con
> `GetCurvesInView` en vez de darlo por supuesto: si Revit invirtiera la curva, la burbuja
> iría al pie del eje.

### En las elevaciones: cotas de altura, marcas de nivel y línea de terreno

Todo esto sale de **desarmar la vista del modelador** `ELEVACION EJE A` (id 6154003,
medida el 2026-08-03), que es exactamente el plano-tipo al que hay que llegar:

| Lo que tiene | Cuánto |
|---|---|
| Ejes | **2** (los extremos) |
| Cotas (`2.5 ROMAND(MILIMETROS)`) | **3**: una cadena de 4 referencias (200/1450/500), una de 2 = 2150, otra de 2 = 1200 |
| Spot Elevations | **3**, un tipo distinto cada una: `T.A. ELEVACION`, `P.T. ELEVACION`, `nipb ELEVACIÓN inf` |
| Multi-Category Tags | 10, de `C-MultiCat : Item_TBL` (los rótulos de perfil — los pone 05) |
| Detail Lines | 1 de 13,94 m con estilo `L-CENTER` (la línea de terreno) |

#### ⚠️ El T.A. se vota por metros de viga, no por la pieza más alta

Medido en `ES-1003 / EJE E` el **2026-08-04**: la pieza más alta de esa elevación es un
marco `OR100x14,4` de 100 mm de canto —el apoyo del `SPE-1004`— **2,4 m por encima de la
plataforma**. Tomando el máximo, el T.A. salía en `EL. 2306,267` en vez de `2303,900`, la
marca de nivel apuntaba a ese marco y la cadena de cotas entera colgaba de ahí (una cota
de «100» flotando sobre la estructura).

Los topes de viga se agrupan en cubetas de 10 mm y cada cubeta acumula **metros de viga**
(el largo en planta de su bbox, así una columna aporta ~0 y no vota). Una cubeta es un
T.A. si llega al **40 %** de los metros de la cubeta dominante. El log lista las cubetas
descartadas con su altura y sus metros.

> Este criterio geométrico es **propio de 03** y sobrevive por su cuenta, pero desde el
> **2026-08-11 ya no alimenta las marcas de nivel** — sólo la cadena de cotas, donde lo que
> se acota es lo que la vista realmente muestra. Las marcas pasaron a leer los `Level`
> modelados (ver abajo), que era el *«candidato a revisar»* que estaba anotado acá.

> **Y no hay un solo T.A.**: si en la elevación hay vigas a alturas distintas que pasan el
> umbral, **cada altura lleva su propia marca** y entra como una referencia más en la
> cadena de cotas.

#### ⚠️ El grating es `Structural Framing`, igual que una viga

La familia `C-GRATING ARRIGONI ARS-5` está en la misma categoría que las vigas y su cara
superior queda **32 mm por encima** del tope de acero. Si entrara en la votación, el T.A.
saldría en la cara del grating. Se excluye por nombre de familia (input, default
`GRATING`), igual que en 05.

#### ⚠️ La cota va contra detail lines, no contra las caras del acero

Una cota contra cara **se crea sin error y después desaparece**. `NewDimension` devuelve
el objeto, el log dice OK y la verificación post-commit la encuentra viva — pero Revit la
descarta cuando Dynamo cierra su propia transacción, con el manejador de fallas ya
desuscrito: ni cota ni mensaje de error. Medido el 2026-08-04: **54 cotas «OK» en el log y
cero en el modelo**, mientras las 5 cadenas de cada planta —que referencian ejes y detail
lines— sobrevivían todas.

Por eso la elevación se acota igual que las plantas: se dibuja una **detail line
`L-CENTER` en cada altura** y la cota referencia esas líneas. La de la base va de lado a
lado (es la línea de terreno del plano-tipo); las otras dos son marcas cortas a la
izquierda, donde corre la cota.

#### Las tres alturas que se acotan

```
   altura maxima  ---+          <- la pieza mas alta que muestra la vista
                     |
   T.A.           ---+          <- tope de acero (votado por metros de viga)
                     |
   base           ===+========  <- lo mas bajo del assembly (= linea de terreno)
```

Salen del **bbox de lo que la vista muestra**, sin depender de encontrar ninguna cara. Si
dos alturas caen a menos de 2 mm se funden en una sola referencia.

#### Los planos de altura (para las marcas de nivel)

```
        tope de baranda   ---+
                             | 1200
   T.A. (tope de acero)   ---+---------  <- Spot Elevation "T.A."
                             | 200
       fondo de viga      ---+
                             |            2150
                             | 1450
   P.T. (piso terminado)  ---+---------  <- Spot Elevation "P.T."
                             | 500
   N.I.P.B.               ---+---------  <- Spot Elevation "N.I.P.B."
   ============================================  linea de terreno
```

> ⚠️ **Desde el 2026-08-11 las tres marcas (`T.A.`, `P.T.`, `N.I.P.B.`) ya no salen de
> acá**, sino de los `Level` modelados — ver
> [§ Las marcas de nivel salen de los `Level` modelados](#️-las-marcas-de-nivel-salen-de-los-level-modelados).
> Lo que sigue describe los planos geométricos, que **siguen vivos** para la cadena de cotas
> y para el `fondo de viga` y el `tope de baranda`, que no tienen `Level` que los declare.

Se sacan de la **geometría que la elevación muestra** y solo sirven para saber **a qué
altura** poner cada anotación — no para anclarla (ver más abajo por qué).

- **T.A.** = tope de las vigas, votado por metros (arriba). Puede haber más de uno.
- **tope de baranda** = lo más alto de las categorías de barandas (`Railings`). Solo
  cuenta si sobresale del T.A. Ojo: un pasamanos de tubo redondo **no tiene cara
  horizontal en el tope**, así que ese plano suele quedar sin resolver y se dice en el log.
- **fondo de viga** = la cara inferior más baja de las vigas que llegan al T.A. más alto
  («llegan» = su tope está a menos de 10 mm de ese T.A.: el acero se modela con
  contraflechas y tolerancias de milímetros).
- **P.T.** = tope de las **sillas de anclaje**. En la vista del modelador no hay
  hormigón: lo único que hay a 500 mm sobre la placa base son los `Structural
  Connections` que arrancan en ella, y su tope es el piso terminado. Si la base es
  placa sola, sin silla, no hay plano P.T.
- **N.I.P.B.** = lo más bajo de la vista, **por debajo de las sillas de anclaje** (cara
  inferior de las placas base).

Cada altura se **confirma contra una cara real** antes de aceptarla (`ComputeReferences`
+ `GetInstanceGeometry()`, probando hasta 8 candidatos ordenados por profundidad y
posición): si nada tiene una cara horizontal genuina ahí, ese plano queda sin resolver y
consta en el log. Pero esa cara **solo sirve para validar la altura** — la anotación en
sí no se apoya en ella, por lo que sigue.

#### ⚠️ Las marcas de nivel salen de los `Level` modelados

Desde el **2026-08-11** las tres marcas (`T.A.`, `P.T.`, `N.I.P.B.`) se colocan a la cota
que declaran los niveles del modelo, **no** a la que vota la geometría:

| Marca | Antes (geométrico) | Ahora |
|---|---|---|
| `T.A.` | votación por metros de viga en cubetas de 10 mm | una por cada `T.A._[{N}_]{assembly}` |
| `P.T.` | tope de las sillas de anclaje | una por cada `P.T._[{N}_]{assembly}` |
| `N.I.P.B.` | lo más bajo del assembly | `N.I.P.B._{assembly}` |

**El modelo declara, el pipeline obedece** — el mismo principio que rige a 00. La razón de
fondo no es de precisión sino de **coherencia**: 00 genera las plantas sobre esos `Level`,
así que mientras 03 votara su propia altura, la marca de una elevación podía contradecir a
la planta del mismo nivel. Ahora los dos graphs leen el mismo contrato de nombres —
`parte_serie()` y `calza_serie()` están **copiadas tal cual de 00**.

> ⚠️ Esa copia es real y hay que mantenerla a mano. El **2026-09-16**, al cambiar el formato
> de los T.A. en 00, 03 se quedó con el matcher viejo y sus marcas de nivel T.A. habrían
> desaparecido igual que las plantas. Se arreglaron los dos a la vez, y lo mismo al pasar los
> P.T. a N. Hay un test que compara las dos implementaciones sobre los niveles reales de los
> dos modelos medidos y exige **cero diferencias**; si algún día se toca una, hay que tocar
> la otra.

> ⚠️ **Siempre `ProjectElevation`, nunca `Elevation`**, por la misma trampa de la cota
> compartida que ya documenta [00 § Cota compartida](#️-cota-compartida-usar-projectelevation-nunca-elevation).
> Con `Elevation` la marca se iría 2300 m fuera de la vista.

**Sin nivel no hay marca.** No hay fallback geométrico: si falta `P.T._ES-1002`, esa marca
no se pone en ninguna elevación del assembly y el log lo dice una vez, al resolver los
niveles.

##### Una marca que la vista no muestra no se dibuja

Una elevación de eje muestra una **tajada** del assembly: un nivel que no llega a *ese* eje
no tiene estructura ahí, y su marca quedaría flotando en el vacío. Por eso las pendientes se
acotan al alto de lo que la vista dibuja, con `MARGEN_MARCA` (300 mm, del orden del canto de
una viga) de holgura para no perder el nivel que coincide justo con el borde. Las que se
descartan constan en el log con su nombre.

> La **cadena de cotas no cambió**: sigue saliendo de `alturas_para_cotar()` sobre la
> geometría, porque ahí lo que se acota es lo que la vista realmente muestra. Lo mismo el
> `fondo de viga` y el `tope de baranda`, que no tienen ningún `Level` que los declare.

#### ⚠️ Una marca de nivel va contra detail lines, igual que la cota

Con la **misma cara**, la cota se dibujaba bien y la marca fallaba con `Spot Dimension
does not lie on its reference` — incluso sobre vigas simples, probando decenas de
candidatos y puntos, `Face.Project` incluido (medido el 2026-08-04). Pero el dato que
resolvió el diagnóstico fue otro: en `ES-1003`, las **7 elevaciones de eje horizontal**
(`B, Ba, Ca, E, Ea, Va, Vb`) sacaban su marca T.A. sin problema, mientras que de las
**8 de eje vertical** (`13a…18a`) solo **una** lo lograba.

La diferencia es la **profundidad de vista**: una elevación de eje vertical mira a lo
**largo** de la plataforma y ve decenas de vigas repetidas apiladas en profundidad; una
de eje horizontal mira a lo **ancho** y ve solo una tanda. Más profundidad, más
candidatos con los que toparse — y el problema de fondo con las referencias de cara
nunca se resolvió del todo; simplemente con pocos candidatos había más chance de
acertar por casualidad.

La solución es la misma que ya funcionó para la cota: **la marca se apoya en una detail
line propia**, no en la cara del acero. Deja de importar cuántas vigas se superpongan en
profundidad, porque la línea es geometría nuestra, no del modelo.

> Cota y marca **comparten la línea** cuando caen a la misma altura (típico de T.A.): se
> arma una sola caché `altura → (Reference, punto)` por vista y ambas la consultan, así
> no se dibuja una línea de más.

#### Dónde va cada cosa

Se reusan los inputs de separación de las plantas, para no alargar el Player:

- **cota de alturas** (base / T.A. / altura máxima), a la izquierda a `separacion al
  borde` (20 mm de papel). La línea de la base va de lado a lado —es también la línea de
  terreno del plano-tipo—, sobresaliendo esa misma separación a cada lado (con los
  defaults, 1:75 y 20 mm, da 1,5 m por lado, similar a los ~14 m del plano-tipo); las
  otras dos son líneas cortas a la izquierda, donde corre la cota.
- **marcas de nivel**, a la derecha con directriz horizontal: codo a media separación y
  texto a una separación y media.

La **sigla** de cada marca (`T.A.`, `P.T.`, `N.I.P.B.`) la pone el **tipo** de Spot
Elevation, no el script: son tres familias distintas ya cargadas en el proyecto. El input
lleva los tres nombres separados por coma **en ese orden**; vacío = no se ponen marcas.
Si un nombre no existe, el log lista los tipos disponibles y sigue con los otros dos.

#### ⚠️ Una categoría apagada no da error: da una anotación invisible

Medido el 2026-08-04: el log reportaba **54 cotas creadas**, la verificación post-commit
no encontraba ninguna borrada, y en la lámina no se veía **ninguna**. Estaban todas en el
documento (ids `765xxxx`, acumulándose corrida tras corrida) pero el view template
`ESTRUCTURAS` trae la categoría *Dimensions* apagada.

Cuando una categoría está oculta en la vista, la anotación se crea sin error pero **no se
dibuja y `FilteredElementCollector(doc, view.Id)` tampoco la devuelve** — desde el script
parece que nunca se creó, así que la idempotencia la vuelve a crear en cada corrida. Es la
misma trampa que ya estaba resuelta para *Lines* con los ejes de viga en planta.

Antes de anotar una elevación se encienden **Dimensions, Spot Elevations y Lines** si
estuvieran apagadas. Si el template no lo permite, el log lo dice con todas las letras en
vez de dejar el misterio.

> **Idempotencia**: si la elevación ya tiene alguna cota (`Dimension`), no se re-cotan las
> alturas ni se redibujan sus líneas de apoyo; si ya tiene alguna marca de nivel
> (`SpotDimension`), no se ponen ni se redibujan las suyas. Son dos compuertas
> independientes: una corrida puede completar la cota y dejar pendientes las marcas (o
> al revés) sin pisar lo que ya está.
>
> Con `Elevaciones: rehacer la anotacion existente` = True se **borra todo lo anotado en
> las elevaciones** (cotas, marcas y línea de terreno) y se rehace. Es seguro barrer con
> todo: 00 crea la vista vacía y lo único que agrega 05 son tags, que no se tocan. Sin
> este input no se puede iterar — la anotación de una corrida anterior bloquea la
> siguiente.

### Rótulos de las conexiones en la planta NIPB

**Solo en la NIPB**, un Multi-Category Tag por cada `Structural Connection` del assembly
que la vista muestre: placas base, sillas de anclaje y pernos — **la familia contenedora,
nunca sus componentes internos** (ver abajo). Lo gobiernan
tres inputs: `NIPB: rotular las conexiones (placas base, sillas)` (default True),
`NIPB: tipo de C-MultiCat del rotulo` (default **`ELEMENTO_TBL`**, el tipo que lee el
parámetro compartido del mismo nombre) y `NIPB: rehacer los rotulos existentes`.

Es lo único que se rotula a esa cota, y a propósito:

| Vista | Quién rotula | Qué |
|---|---|---|
| NIPB | **03** | `Structural Connections` |
| Plantas T.A. | 05 | vigas |
| Elevaciones de eje | 05 | vigas y columnas |

Las columnas quedan fuera: ya salen rotuladas en las T.A. y en las elevaciones, y a la
cota de la placa base no dicen nada. Las vigas ya ni siquiera se ven — desde el
**2026-08-11** 00 oculta `Structural Framing` en esta planta, ver
[00 § Qué se oculta en las plantas base](#️-qué-se-oculta-en-las-plantas-base-nipb-y-pt).

Medido en `ES-1001 - NIPB` el **2026-08-05**, **antes** de ese cambio: 53 elementos del
assembly visibles — **47 conexiones**, 4 columnas y 2 vigas.

#### ⚠️ Solo la familia contenedora, nunca sus componentes internos

Una silla de anclaje **es una familia con piezas anidadas compartidas**: en el modelo de
referencia, `C-SillaAnclaje` con ocho `C-AtiesadorSillaAnclaje` adentro. Las dos están en
`Structural Connections`, y una anidada **compartida** es un elemento independiente para
`FilteredElementCollector` — con su propio Id, su propio bounding box y su propio centro
en planta.

Sin filtro, una sola silla salía con **nueve rótulos** encimados en el mismo punto. Medido
el **2026-08-11** en la vista del modelador `PLANTA PLACAS BASE EL. 2.301,775 N.I.P.B.`:
36 conexiones de silla = **4 contenedores + 32 atiesadores**, o sea 36 tags donde
correspondían 4.

El filtro es por **jerarquía, no por nombre de familia**: se descarta la pieza cuyo
`SuperComponent` sea **otro candidato** de la misma corrida. Da igual cómo termine
llamándose la familia que modele el proyectista (`C-SillaAnclaje`, `SILLAS`, lo que sea).

La condición es «hijo de **otro candidato**», no simplemente «tiene padre», y eso importa:

| Situación | Qué se rotula |
|---|---|
| Contenedor visible + sus anidadas visibles | **solo el contenedor** |
| Silla modelada como atiesadores sueltos, sin contenedor (pasa en `ES-1003`) | cada pieza, como antes |
| Contenedor fuera del View Range o anidado en otra categoría | sus piezas, en vez de perderse en silencio |

El log dice cuántas piezas internas se saltaron en cada vista.

> ⚠️ **Lo que queda encimado.** El filtro mata la mayor parte del problema, pero no todo:
> varias conexiones **distintas** sobre la misma placa base siguen compartiendo el centro
> en planta, y el tag va ahí y sin directriz. Esas hay que separarlas a mano — son piezas
> superpuestas, no repartidas, y no hay un lugar «correcto» que el script pueda calcular.
> (Antes del filtro eran ~12 tags por placa; las 47 conexiones de `ES-1001` vivían sobre
> **4 placas**.)

> ⚠️ **Hoy todos dirían lo mismo.** En el modelo de referencia `ELEMENTO_TBL` (parámetro
> compartido `433e7c25-aa15-4380-9474-c6dffa787fa2`) vale **`ES-1001`** en las 53 piezas
> visibles — es el nombre del assembly, no el identificador de la pieza (`PL-1004`,
> `PL-1005`, `SILLA ANCLAJE ES-1001` están en el *Type Name*). El script no controla qué
> texto muestra un tipo de tag —eso lo define la etiqueta de la familia `C-MultiCat`, igual
> que en 05—, así que esto es dato del modelo. Si hace falta otra cosa, se cambia el input
> de tipo de tag.

Detalles:

- **Se enciende la categoría `Multi-Category Tags`** antes de nada, por la misma trampa ya
  documentada para *Dimensions*, *Spot Elevations* y *Lines*: un tag creado en una vista
  con su categoría apagada existe pero no se dibuja, y `FilteredElementCollector(doc,
  view.Id)` tampoco lo devuelve — con lo que la idempotencia deja de verlo y cada corrida
  apila tags invisibles. A quien hay que ganarle es al view template de la NIPB
  (`07.` en 00, default `ESTRUCTURAS_1/50`).
- **Qué piezas entran**: las que devuelve el colector por vista (respeta el View Range y el
  aislamiento permanente de 00) **filtradas contra los miembros recursivos del assembly**,
  y de ésas, **solo las que no sean componente interno de otra** (arriba). Si la vista no
  muestra ninguna conexión, el log lo dice y apunta a la categoría
  `Structural Connections`, que el template apaga y 00 vuelve a encender.
  > Ojo con `miembros_recursivos()`: entra a propósito en las anidadas vía
  > `GetSubComponentIds()`, así que ese filtro **no** excluye los componentes internos —
  > los incluye. El descarte lo hace el filtro por `SuperComponent`, no éste.
- El tipo de tag se busca **por nombre de tipo** y se activa si hacía falta. Si no existe,
  no se rotula nada y el log lista los disponibles como `Familia : Tipo` — mismo criterio
  que 05: rotular decenas de piezas con el tag equivocado es peor que no rotular.
- **Idempotencia**: si la NIPB ya tiene tags de ese tipo se salta, salvo que `NIPB: rehacer
  los rotulos existentes` esté en True.
- El punto del tag se prueba contra el `Origin.Z` de la vista y contra la
  `ProjectElevation` de su nivel, porque con **cota compartida** no coinciden (ver 00).

### Eje de cada viga en las plantas T.A. y en los cortes (línea roja `L-CENTER`)

En cada planta **T.A.** y en cada **elevación de eje** se dibuja, sobre el eje de cada
viga, una **detail line** con el estilo de línea `L-CENTER` (input; si el estilo no existe
en el proyecto se crea **rojo** y con el primer patrón de línea de eje que encuentre).

**En la NIPB no.** A esa cota hay placas base, pernos y sillas de anclaje —no vigas—, así
que ahí el eje de viga no dice nada. Si una corrida anterior las dibujó, 03 las **borra** al
pasar por esa vista (y Revit se lleva de paso la cadena que las referenciaba); queda
registrado en el log.

#### El criterio es uno solo; lo que cambia es el plano

Desde el **2026-08-11** las elevaciones llevan el mismo eje de viga que las plantas. La
lógica **no se duplicó**: lo único que difiere entre una planta y un corte es cómo se lleva
un punto del modelo al plano de la vista, y eso vive aislado en `proyectores_de_vista()`.

| Vista | Proyección | Qué se descarta |
|---|---|---|
| Planta | fijar la `Z` al plano de trabajo | riostras y montantes **verticales** |
| Elevación de eje | `(distancia sobre `RightDirection`, `Z`)` | vigas que entran **hacia el fondo del papel** |

En los dos casos la regla es la misma: **el largo se mide sobre el plano de la vista**, y
una viga que se proyecta en un punto no tiene eje que dibujar. El log lo reporta como
`N sin largo en el plano de la vista`.

> En las plantas, `proyectores_de_vista()` devuelve **varios** candidatos porque la cota del
> plano de trabajo no siempre es `view.Origin.Z` y hay que probar el nivel como plan B (si
> no, Revit rechaza la curva por no estar en el plano de la vista). En una elevación el
> plano es uno solo.

> ⚠️ **En las elevaciones estas líneas comparten estilo con las de apoyo de las cotas.** Una
> vez dibujadas no se distinguen, así que la idempotencia **no** puede mirarlas después: se
> resuelve en `anotar_altura()`, que captura `ya_lineas` **antes** de dibujar nada. Si la
> elevación ya traía líneas `L-CENTER`, toda su anotación es de una corrida anterior y se
> respeta; con `Elevaciones: rehacer la anotacion existente` ya se borraron más arriba, la
> lista queda vacía y se redibuja todo. Por eso el eje de viga de los cortes se dibuja
> **dentro** del mismo ciclo de anotación y no por su cuenta.

> El interruptor sigue siendo `Eje de vigas` (el mismo de las plantas), y en las elevaciones
> manda además `Mostrar ejes tambien en las elevaciones`, que es el interruptor general de
> todo lo que 03 hace ahí.

**Qué se dibuja**: la *curva de ubicación* (`Location.Curve`) de cada `Structural Framing`,
proyectada al plano de la vista. Es el eje real de la viga, no el centro de su bounding box.
Las vigas curvas se teselan en una poligonal.

**Qué vigas entran en cada vista**: las que devuelve `FilteredElementCollector(doc,
view.Id)` **filtradas contra los miembros recursivos del assembly**. El colector por vista
respeta el *View Range* y el aislamiento permanente que dejó 00, así que cada T.A. recibe
exactamente los ejes de las vigas de **su** nivel. El filtro por miembros es el cinturón de
seguridad por si el aislamiento de una vista se perdiera. En las elevaciones se reusa
`ids_vista`, que ya viene filtrado contra los miembros, en vez de recalcularlo.

**Por qué detail lines y no model lines**: una *model line* aparecería en las tres plantas
del assembly y en las elevaciones, y ensuciaría el modelo para todo el resto del proyecto.
La detail line vive solo en la vista donde se creó.

Detalles:

- Las vigas que se **proyectan en un punto** se saltan: en planta, las verticales (riostras,
  montantes); en un corte, las que entran hacia el fondo del papel. Se cuentan en el log.
- Se enciende la categoría *Lines* en la vista si el view template la tenía apagada — si no,
  las líneas se crean pero no se ven (el mismo mecanismo que con *Structural Connections*
  en 00).
- **Idempotencia**: si la planta ya tiene líneas de ese estilo, se salta y lo dice en el log.
  Con `Eje de vigas: redibujar los existentes` = True las borra y las rehace. El borrado
  está acotado a las líneas **de ese estilo** en esa vista, así que no toca otro dibujo.
- El grafismo de un `L-CENTER` que **ya exista** en el proyecto no se toca: es un estándar
  del modelo, no de este script. El log imprime su RGB para poder verificarlo de un vistazo.
- Estas líneas son además las **referencias de la cadena de cotas entre ejes de viga** (más
  abajo). Borrarlas de una vista se lleva por delante esa cadena.

### Las dos familias de cadenas

Hay **dos familias de cadenas de cotas**, ninguna va por debajo, y no aplican a las mismas
vistas:

| Familia | NIPB | T.A. |
|---|---|---|
| Entre **ejes estructurales** (por fuera) | ✅ | ✅ |
| Entre **ejes de viga** (por dentro) | ➖ | ✅ |

```
              [ cadena ENTRE EJES horizontal ]        <- off       (20 mm)
              [ cadena de VIGAS horizontal   ]        <- off_viga  (12 mm)
   [E]  [V]   +-------------------------------+  [V]
    |    |    |                               |   |
    |    |    |          PLANTA T.A.          |   |
    |    |    |                               |   |
    |    |    +-------------------------------+   |
    ^    ^                                        ^
   off  off_viga                               off_viga
                        (nada abajo)

   la NIPB lleva solo [E]: cadena entre ejes, arriba y a la izquierda
```

`off_viga = off − separación entre cadenas` (inputs en mm de papel, defaults 20 y 8). Si la
separación entre cadenas fuese mayor que la del borde, la de vigas se pone a media distancia
y se avisa. La cadena de ejes **no se movió**: la de vigas se mete por dentro.

### Cadena de cotas entre ejes en las elevaciones

Desde el **2026-08-11** cada elevación de eje lleva, **por encima del assembly**, la misma
pareja que las plantas: **cadena entre ejes + cota total** del primero al último eje que el
corte despliega.

```
   [──────────────── cota TOTAL ────────────────]   <- z del assembly + off + sep
   [ tramo ][  tramo  ][ tramo ][    tramo     ]   <- z del assembly + off
    (13a)     (13b)     (14a)    (16)     (18a)
      │         │         │        │        │
   ┌──────────────────────────────────────────┐
   │                ELEVACION                 │
```

Se acota contra los **propios `Grid`**, igual que en planta: son referencias limpias, a
diferencia de las caras del acero, que Revit descarta al comitear.

**No se duplicó `dibujar_cadena()`.** Lo único que cambia entre una planta y un corte es
cómo se construye un punto sobre el plano de la vista, así que la función acepta un
constructor `punto(pos, perp)`: en planta `perp` es un desplazamiento sobre un eje
horizontal y el plano vive a la cota del origen; en una elevación `perp` **es la cota** y el
punto sale de `punto_en_elevacion()`. Es el mismo patrón que `proyectores_de_vista()` para
el eje de viga.

> **Va después de `anotar_altura()`**, no antes: con `Elevaciones: rehacer la anotacion
> existente` esa función borra **todas** las `Dimension` de la vista, así que dibujarla
> antes sería dibujarla para nada.

> **Idempotencia**: se reutiliza `cadenas_existentes()`, que reconoce la cadena entre ejes
> por que su primera referencia es un `Grid`. Si ya está, se salta.

### Cadena de cotas entre ejes estructurales (en planta)

Por cada planta se arman hasta dos cadenas, **por fuera del borde del assembly** (input de
separación en mm de papel, default 20, escalado por la escala de la vista):

- **Horizontal, por encima**: los ejes paralelos al *arriba* de la vista, medidos a lo
  largo del *derecha*.
- **Vertical, a la izquierda**: los ejes paralelos al *derecha* de la vista, medidos a lo
  largo del *arriba*.

Se acota **eje contra eje**, sin los segmentos borde→eje→borde del proyecto base: el
estándar pide «cotas asociadas entre ejes», y además evita depender de caras de la
geometría, que en acero son malas referencias (pernos, planchas, conexiones dan
referencias que Revit rechaza al comitear).

Los ejes se ordenan por su posición sobre el eje de medición y se descartan los que estén
a menos de ~5 mm de otro (mismo eje dibujado dos veces). Los ejes **oblicuos** a la vista
quedan fuera de la cadena y se avisan en el log.

#### ⚠️ Qué ejes son «del assembly»: el parámetro `ASSEMBLY`, no la geometría

03 lee **el mismo marcado que 00**: el parámetro de proyecto `ASSEMBLY` de cada eje (ver
[00 § El parámetro `ASSEMBLY` de los ejes](#el-parámetro-assembly-de-los-ejes)). Un eje sin
ese marcado no entra en ninguna cadena ni se muestra en ninguna vista del assembly.

> **Esto estuvo roto entre el 2026-08-10 y el 2026-08-11.** 03 elegía los ejes con un test
> **geométrico** (recorte Liang-Barsky del eje contra el **rectángulo** del bounding box en
> planta) y el comentario del código afirmaba que era «el mismo test que usa 00». Dejó de
> serlo el 2026-08-10, cuando 00 pasó al parámetro `ASSEMBLY`, y 03 no se actualizó.
>
> El síntoma no era sólo acotar de más. El rectángulo del bbox es bastante más grande que la
> huella real, así que 03 devolvía **ejes de otros sectores** — y `ejes_de_elevacion()` los
> **desocultaba**, revirtiendo el aislamiento permanente que había dejado 00. Un eje ajeno
> reaparecía en el corte y encima entraba en la cadena de cotas.
>
> `segmento_cruza_rect()` y `ejes_que_cruzan()` se eliminaron. El comentario que afirmaba la
> equivalencia es lo que hizo que el desfase pasara desapercibido: **una afirmación de
> consistencia entre dos graphs no se comenta, se comparte el código**.

#### La cota total, del primer al último eje

Desde el **2026-08-11** cada cadena lleva **además** una cota de punta a punta, por fuera:

```
   (0c)(0b)(0a)  (0)   (1)     (2)      (3)          <- burbujas de eje
   [────────────── 12600 ──────────────]             <- cota TOTAL
   [1200|1050|1200|1100|1100|875|...]                <- cadena de EJES DE VIGA
   [E]  ┌───────────────────────────────┐
    │   │            PLANTA             │
```

Medido en `ES-1001 - T.A. 01`: la vista queda con **6 cotas** — cadena entre ejes y total a
la izquierda, cota total arriba, y las dos cadenas de ejes de viga (arriba y derecha).

Los tramos dan la modulación y el total da la dimensión general del sector, que es como se
acota un plano. Se controla con `COTA_TOTAL_EJES` en la cabecera del nodo Python, y se
separa de la cadena con el mismo input de **separación entre cadenas** (default 8 mm).

##### Arriba va sólo la total; a la izquierda, las dos

```python
SOLO_TOTAL_ARRIBA    = True     # arriba, sólo la cota de punta a punta
SOLO_TOTAL_IZQUIERDA = False    # a la izquierda, tramos + total
```

**Arriba los tramos entre ejes competían con la cadena de ejes de viga**, que corre justo
debajo y en `ES-1001` trae **13 tramos**, algunos de 425 y 575 mm. Dos filas de tramos
apiladas no se leen, y de las dos la que aporta es la de vigas: la modulación entre ejes ya
queda dicha por la cota total más las burbujas.

A la izquierda se dejan las dos porque **ahí no hay nada compitiendo** — la cadena de vigas
vertical va por la **derecha** — y la modulación entre ejes se lee sin problema.

> Cuando un lado va con `solo_total`, la cota total **ocupa el lugar de los tramos** en vez
> de quedar más afuera dejando un hueco vacío adentro. Y ahí se dibuja aunque haya sólo 2
> ejes: es la única cota de ese lado, no una repetición de la cadena.

- **Va como `Dimension` aparte**, no como una referencia más de la cadena: una `Dimension`
  de Revit con **3 o más referencias dibuja los tramos**, así que el total no se puede pedir
  sobre la misma cadena.
- **Con sólo 2 ejes no se dibuja**: ahí la cadena ya *es* el total, y repetirlo serían dos
  cotas idénticas una encima de la otra.
- **Sólo en la cadena exterior** (entre ejes estructurales). La cadena interior de ejes de
  viga no lleva total.

> ⚠️ La separación se pasa **con signo**, no en valor absoluto: la cadena de arriba se
> desplaza con `+off` y la de la izquierda con `−off`, así que «hacia afuera» no es la misma
> dirección en las dos. Con el signo puesto por quien llama, `dibujar_cadena()` sólo tiene
> que sumar.

> Si algún día se quisiera el total **en vez de** los tramos, no es esta constante: sería no
> pasarle a `dibujar_cadena()` la lista completa de ejes.

### Cadena de cotas entre ejes de viga (solo en las plantas T.A.)

Por dentro de la anterior, tres cadenas que miden la distancia **entre los ejes de las
vigas**: una horizontal arriba y una vertical a **cada lado**. La NIPB no lleva ninguna,
porque tampoco lleva los ejes de viga que le servirían de referencia.

**Qué se referencia**: las propias detail lines `L-CENTER` que dibuja el paso anterior.
Son literalmente el eje de la viga, viven en la misma vista y son referencias limpias — al
revés que las caras del acero, que Revit rechaza al comitear. Consecuencia: sin líneas de
eje no hay cadena de vigas. Si el dibujo está apagado pero la cadena encendida, se acota
contra las líneas que hubiera de una corrida anterior; si no hay ninguna, el log lo dice.

#### ⚠️ Las verticales se reparten por mitades

Una planta de vigas es extensa y tiene muchas más vigas que ejes. Acotarlas todas hacia un
solo lado amontona el texto hasta volverlo ilegible. Por eso cada viga paralela al *derecha*
de la vista va a la cadena del lado en cuya **mitad** cae su punto medio: mitad izquierda →
cadena izquierda, mitad derecha → cadena derecha. Cada cadena queda con la mitad de las
cotas.

Consecuencia esperable: cada cadena lateral abarca solo el tramo donde viven **sus** vigas,
no toda la altura de la planta. Si un lado tiene pocas vigas, su cadena sale corta — no es
un fallo, es el reparto.

Las vigas paralelas al *arriba* de la vista van todas a la cadena horizontal de arriba: no
hay cadena abajo donde repartirlas.

#### ⚠️ El paralelismo se mide contra las vigas entre sí, no contra los ejes de la vista

Revit exige que **todas** las referencias de una cadena sean paralelas **entre sí**; una
sola viga fuera de escuadra hace que rechace la cadena **entera** al comitear.

La primera versión comparaba cada viga contra el eje de la vista con una tolerancia fija
(`0.9999`, ~0,8°). Ese criterio falla por los dos lados a la vez, y el log del 2026-08-03
lo mostró:

```
DIAG ES-1003 / T.A. 01: ... | fuera (oblicuos): 46 | total dibujados: 149
AVISO: cadenas eliminadas por referencias invalidas: ES-1003 (T.A. 01 vigas arriba)
```

**46 de 149 vigas descartadas** (cada una es una cota que falta) y, aun así, la cadena de
32 referencias que sí se creó fue **rechazada por Revit**: entre las que pasaron el umbral
había alguna no paralela a las otras. Comparar contra el eje de la vista no responde la
pregunta que importa.

Ahora el reparto es en dos pasos:

1. **Grueso, sin umbral**: cada eje de viga va al grupo del eje de la vista al que más se
   parece (`|d·arriba|` contra `|d·derecha|`). No se descarta nada todavía.
2. **Fino, contra la dirección dominante del grupo**: la dirección que acumula **más metros
   de línea** —mismo criterio de «votar por metros, no por piezas» que usa 00 para el nivel
   dominante de una planta T.A., y por el mismo motivo: dos piezas cortas torcidas no pueden
   decidir por el grupo— y se conservan solo las vigas paralelas a ella dentro de
   `TOL_PARALELO / 2`.

Lo que queda fuera son diagonales de verdad (arriostramientos) o vigas mal modeladas, y se
cuenta en el log con su desvío máximo en grados. La línea de cota se dibuja **perpendicular
a la dirección dominante**, no a los ejes de la vista, así que la cadena también funciona si
la trama del edificio está girada respecto al norte de proyecto.

#### ⚠️ `Application.AngularTolerance` es 12 veces más permisiva de lo que las cotas toleran

`TOL_PARALELO` vale **1·10⁻⁵ rad**, no `Application.AngularTolerance` (1,75·10⁻³ rad = 0,1°),
que es lo que parecería el número correcto. Medido en `ES-1003 - T.A. 01` el 2026-08-03:

| Elemento | Desvío de la vertical |
|---|---|
| Los 64 ejes de viga bien modelados | ~1·10⁻¹⁵ rad (exactos) |
| Detail lines `7647576` y `7647577` (450 mm) | **1,45·10⁻⁴ rad = 0,0083°** |

Esas dos, que pasaban el umbral de 0,1° sin problema, bastaron para que Revit borrara la
cadena entera de 16 referencias al comitear:

```
ERROR: The References of the highlighted Dimension are no longer parallel.
```

Con las vigas sanas coincidiendo a 10⁻¹⁵ rad y la torcida a 10⁻⁴, hay diez órdenes de
magnitud de margen: cortar en 10⁻⁵ rad (0,1 mm sobre 10 m) deja fuera la mala sin rozar
ninguna buena.

Dos detalles de implementación que no son opcionales:

- **El corte es a `TOL_PARALELO / 2`, no a `TOL_PARALELO`.** La condición que le importa a
  Revit es entre **cada par** de referencias, no contra un promedio: dos vigas a `+tol` y
  `−tol` de la dominante pasan las dos y están `2·tol` una de otra. Con medio umbral,
  cualquier par queda dentro.
- **El ángulo se mide con el producto cruz, no con el escalar.** Cerca del paralelismo el
  coseno pierde toda la precisión —`cos(1e-5)` y `1.0` son el mismo `double`— mientras que
  el seno vale directamente el ángulo.

> Cómo llegó a colarse una referencia torcida existiendo una vertical exacta a 100 mm: la
> fusión por cercanía conserva **la primera en orden** sobre el eje de medición, y la
> torcida tenía la coordenada menor. Con el umbral nuevo se descarta antes de llegar ahí.

> La cadena entre **ejes estructurales** no pasa por este filtro (solo clasifica por
> `PARALELO = 0.99`). Los ejes del modelo son ortogonales exactos y esa cadena nunca falló;
> si algún día un modelo trae ejes casi-paralelos, fallará igual y el log lo dirá.

#### ⚠️ Separación mínima entre cotas: en mm de **papel**, no de modelo

Una fila de vigas colineales (tramo A-B, tramo B-C) da **una** referencia, no tres. Pero
con un umbral de 5 mm de **modelo** eso no alcanza ni de lejos. Medido sobre `ES-1003 -
T.A. 01` el 2026-08-03, la cadena lateral salió así:

```
731  144  601  268   ...   835  293  27  808
```

A 1:75, una cota de 27 mm ocupa **0,36 mm de papel** y una de 144 mm ocupa 1,9 mm: el texto
se encima y la cadena no se puede leer. Y probablemente es también lo que hacía que Revit
rechazara la cadena de 32 referencias de esa misma planta — un segmento de milímetros
invalida la cadena **entera**.

Por eso el umbral de la cadena de vigas es un input **en mm de papel** (`Eje de vigas:
separación mínima entre cotas`, default **5**), escalado por la escala de la vista: a 1:75
son 375 mm de modelo. Dos ejes más juntos que eso se fusionan en una sola referencia y la
cota pasa a medir el salto completo. `0` = sin fusión (vuelve al mínimo de 5 mm de modelo).

El log dice cuántas se fusionaron y cuál fue el **menor segmento** de cada cadena, que es
el número a mirar cuando una cadena sale ilegible o Revit la rechaza:

```
OK ES-1003: cadena T.A. 01 vigas izquierda con 8 referencias, largo 6.97 m,
            menor segmento 601 mm, 5 fusionadas por cercania
```

> La cadena entre **ejes estructurales** no usa este umbral: los ejes están lo bastante
> separados y esa cadena ya se veía bien.

### Manejo de fallas

Igual que 04: se engancha un handler a `FailuresProcessing` y se fuerza el commit **dentro
del script** (`ForceCloseTransaction`), porque Dynamo confirma su transacción después de
que el script terminó y el handler ya se des-suscribió. Las cotas que Revit rechaza por
referencias inválidas se borran solas en vez de abrir un diálogo bloqueante, y el script
verifica después del commit cuáles sobrevivieron.

**Idempotencia**: se mira **a qué referencia** la primera cota de cada cadena que ya existe
en la vista — un eje estructural es la cadena entre ejes, una línea `L-CENTER` es la de
vigas. Así se le puede agregar la cadena de vigas a una planta que ya tenía la de ejes, sin
duplicar ni rehacer esa. Para rehacer una cadena hay que borrarla a mano.

> Antes bastaba con que existiera *cualquier* cota de más de un segmento para saltarse la
> planta entera. Con eso, agregar la cadena de vigas a las plantas ya acotadas habría
> exigido borrar a mano todas las cadenas de ejes.

Si se redibujan los ejes de viga (`Eje de vigas: redibujar los existentes`), Revit se lleva
por delante la cadena de vigas que los referenciaba; por eso el chequeo de qué cadenas
existen se hace **después** de dibujarlos, y la cadena se vuelve a crear sobre las líneas
nuevas.

---

## 04 — Grating en plantas

> **Estado: escrito pero no probado en Revit.** La geometría se diseñó midiendo el modelo
> real, pero ninguna de las booleanas se ejecutó todavía. Esperá iterar.

### El grating no se puede simplemente encender

| Dato medido (2026-08-03) | Valor |
|---|---|
| Familia / tipo | `C-GRATING ARRIGONI ARS-5` / `GRATING ARRIGONI` |
| Categoría | `Structural Framing`, `FamilyInstance` |
| Instancias | 101, en 4 niveles |
| Un paño | 2,54 × 0,97 m, **32 mm** de espesor |
| Cota | Z = 3,300 → **3,332** m |
| `Assembly Name` | `ES-1003` (es miembro del assembly) |

La cota es la clave: el T.A. es **tope de acero**, o sea que las vigas terminan en 3,300 y
el grating se apoya **encima**. Encender su subcategoría no sirve: taparía la estructura,
que es justo lo que el plano tiene que mostrar.

Verificado además que en la T.A. generada hay **cero** instancias visibles, y que no es por
el aislamiento de 00 (es miembro) ni por el View Range (3,300–3,332 cae entre el fondo 2,80
y el corte 4,30). Queda como causa la visibilidad de la subcategoría en el view template.

### Lo que hacen los modeladores, y por qué es una convención y no una oclusión

Dibujan una **Filled Region** con trama de rejilla y la recortan alrededor de cada elemento
que la cruza, para que el grating se lea **por debajo** del acero. Como el grating está en
realidad arriba, ese recorte no es geométrico: es una convención de dibujo. Además separan
la trama **35 mm** de cada viga para que no quede sucia contra el ala.

Eso simplifica el problema: **no hay que razonar cotas**. Se resta la huella en planta de
todo lo que pisa el paño, sin importar quién está encima de quién.

### Revit no tiene booleanas 2D: se hacen en 3D

| Paso | Herramienta |
|---|---|
| Sombra en planta de un sólido | `ExtrusionAnalyzer` → `GetExtrusionBase()` → `GetEdgesAsCurveLoops()` |
| Engordar la sombra 35 mm | `CurveLoop.CreateViaOffset` |
| La resta | extruir a prismas y `BooleanOperationsUtils.ExecuteBooleanOperation(Difference)` |
| Recuperar los retazos | caras planas del resultado con normal `+Z` → `GetEdgesAsCurveLoops()` |
| Dibujar | `FilledRegion.Create(doc, tipoId, vistaId, loops)` |

De la sombra de cada obstáculo se toma **solo el contorno exterior**: los huecos de un
perfil no cambian lo que tapa en el dibujo.

#### ⚠️ Una pieza de acero tiene dos largos, y el sólido es el corto

El **eje** (`LocationCurve`) va de nudo a nudo: es el largo teórico, el que traza el
modelador. El **sólido** viene recortado en las puntas para dejar lugar a la unión — el
gusset, la plancha, los pernos. Revit lo expone en parámetros propios: `Start Extension`,
`End Extension`, `Join Cutback`.

Medido en la diagonal `5482395` (`L-Viga / L6,5x4,780`) el 2026-08-03:

| Parámetro | Valor |
|---|---|
| `System Length` (eje) | 1.185,6 mm |
| `Cut Length` (sólido) | 883,4 mm |
| `Start Extension` / `End Extension` | **−178,5** / **−216,5 mm** |

**302 mm de diferencia**, repartidos en las dos puntas; en otra diagonal la diferencia llega
a 411 mm. Restando la sombra del sólido, ese hueco de cada nudo —justo donde va la unión— no
se resta nunca, y la trama del grating se mete ahí. En pantalla se ve como un triángulo de
trama en el vértice de cada arriostramiento; y se confirma a ojo porque las líneas rojas
`L-CENTER` (que salen del eje) llegan al vértice mientras el hueco blanco (que sale del
sólido) se queda corto.

Por eso el obstáculo se reconstruye como un rectángulo con el **ancho de la sombra real** y
el **largo del eje completo**. Si el sólido sobresale del eje (ménsulas, placas de punta) se
respeta lo más largo de los dos. Las piezas sin eje recto —columnas, piezas curvas— caen a la
sombra del sólido tal cual, y el log dice cuántos obstáculos se reconstruyeron desde el eje.

#### ⚠️ El grating no es una plancha, son barras

`NUM BARRAS RECT LONG = 31`, `NUM BARRAS CIRCULARES = 25`: la familia modela las barras una
por una. Su volumen es 0,50 pies³ contra los 2,78 que tendría una plancha maciza — un 18 %
de llenado. Su sombra real es un **peine**, no un rectángulo, así que no sirve como
contorno del paño.

Por eso el contorno se toma como el rectángulo que envuelve todas las barras **en el sistema
local de la instancia** (`FamilyInstance.GetTransform()`), no el bounding box del mundo: así
también vale para paños girados.

#### ⚠️ `CreateViaOffset` no dice hacia qué lado desplaza

Depende de la orientación del contorno, que no se controla. Se prueban los dos signos y gana
el que **aumenta el área** — que es el que separa la trama de la viga en vez de comérsela.

#### Qué paños le tocan a cada planta

El grating **no aparece en el colector por vista** (su subcategoría está apagada), así que no
se puede preguntar «qué se ve acá». Se calcula la franja de altura de la vista con
`GetViewRange()` y se toman los paños cuyo bounding box la cruza. Es el mismo criterio con el
que Revit decide qué dibuja, así que el grating seleccionado es el que corresponde.

#### Qué se resta

Todo lo que la vista dibuja de la estructura del assembly: `Structural Framing` (vigas
**y diagonales**) y `Structural Columns`. Las **conexiones quedan fuera a propósito** —
sillas, placas y pernos no corresponden a una planta de T.A.

> ⚠️ Hoy 00 enciende `Structural Connections` en **todas** las plantas, porque la NIPB las
> necesita. En las T.A. no deberían verse. Pendiente de arreglar en 00.

#### Costos y modos de fallo esperados

- Se pre-filtra por solapamiento de bounding box: sin eso serían ~20 paños × 162 miembros =
  3.240 booleanas por vista.
- Cada booleana va en su propio `try`: una cara coincidente no puede tumbar el paño entero.
  El log cuenta cuántas fallaron.
- Los retazos por debajo del **área mínima** (input, default 100 cm²) se descartan, para que
  la vista no se llene de esquirlas donde dos vigas casi se tocan.

#### ⚠️ Las tiras angostas se consumen enteras

La separación de 35 mm se suma a **medio ancho de ala a cada lado**. En un `C20` eso son
100 mm de semiala, así que cada obstáculo se come 135 mm por lado: **270 mm de ancho útil**.
Medido sobre los 48 paños de ES-1003, cinco no llegan a eso con holgura:

| Id | Medida crítica |
|---|---|
| `5482636`, `5482644` | 299 mm de ancho |
| `5482866` | 314 mm de ancho |
| `5482626` | 415 mm de alto |
| `5482804` | 577 mm de alto |

Una tira de 299 mm deja 29 mm después de la resta, muy por debajo del área mínima, y el paño
desaparece. **Ningún paño puede desaparecer en silencio**: si no deja ninguna región, el log
dice su id y su medida, y el resumen cuenta `panos SIN region`. Si aparece uno, hay que bajar
la separación o el área mínima — o aceptar que ese paño no se dibuja.
- Si el proyecto no tiene ningún `FilledRegionType`, el log lo dice y no se crea nada. El log
  siempre lista los tipos disponibles — es la forma de descubrir el nombre exacto. En este
  modelo el que corresponde es **`HATCH GRATING`**, que es el default del input.

#### ⚠️ Revit valida el contorno recién al comitear

Medido el 2026-08-03: el log decía `Regiones de trama creadas: 40 | panos SIN region: 0`, y
sin embargo faltaba un paño en el dibujo. El motivo estaba al final:

```
ERROR: Can't draw Detail Filled Region. (x1)
```

Las 40 se crearon, pero Revit **valida el contorno al confirmar la transacción**, no al
crear, y el manejador de fallas borró la que rechazó. El resumen contaba una región que ya
no existía. Es el mismo problema que 03 tuvo con las cadenas de cotas, y la solución es la
misma: **verificar después del commit** cuáles sobrevivieron y descontarlas, diciendo qué
paño era.

Sobre la causa, los huecos minúsculos que deja la resta booleana son suficientes para que
Revit rechace la región **entera**. Por eso ahora:

1. Los contornos se ordenan por área — el mayor es el exterior, el resto son huecos.
2. Los huecos por debajo del área mínima se descartan antes de crear nada.
3. Si aun así la rechaza, se reintenta con el **contorno exterior solo**, y el log avisa que
   esa región se dibujó sin sus huecos.

#### Grating que no es miembro de su assembly

`ES-1001` y `ES-1002` reportan `ningun miembro coincide con el filtro de familia`. No es del
script: hay geometría de grating en la zona de ES-1002 (nivel `RLO_T.A. ES-1002`) pero
**ninguna instancia tiene `Assembly Name`**. De las 102 del modelo, solo 48 pertenecen a un
assembly, y las 48 son de ES-1003. Si esas plantas tienen que llevar trama, hay que agregar
el grating al ensamble en Revit.

---

## 05 — Rótulos de perfil

> **Estado (2026-08-04): funcionando en Revit.** Plantas **233** tags (110 con el tipo corto,
> 24 que no entran ni cortos, 13 vigas sin curva recta); elevaciones **110** tags, ya
> visibles.
>
> Costó cuatro corridas y cada una falló por una causa distinta, todas documentadas abajo:
> el plano de referencia (`View.Origin` no está sobre el eje), la categoría apagada por el
> view template, y los cortes que quedaban vacíos — que resultaron ser ejes sin marco y se
> arreglaron en **00**, no acá.
>
> ⚠️ **Al re-correr, `07. Rehacer los rótulos existentes` en True.** Si no, las vistas ya
> rotuladas se saltan.

**Todo son Multi-Category Tags de la familia `C-MultiCat`**, en dos sitios, y en los dos se
elige entre el mismo par de tipos según lo que entre en la pieza:

| Vista | Qué pone | Cómo se orienta |
|---|---|---|
| Plantas T.A. | un tag por **viga** | girado con la viga, centrado en su eje |
| Elevaciones de eje | un tag por **tipo de familia** presente en la vista | horizontal encima de las vigas, girado 90° a la izquierda de las columnas |

En ambos casos: **`Item_TBL` si entra, `Modelo` si no**.

### ⚠️ En las elevaciones: un tag por TIPO, no por pieza

Desde el **2026-08-11** una elevación de eje lleva **un rótulo por cada tipo de familia
distinto** que muestre, en vez de uno por cada pieza. Es la única forma de que quepan
columnas, vigas, escaleras y barandas en la misma vista sin que quede ilegible: con un tag
por tipo, **la cantidad de rótulos deja de depender de cuántas piezas haya**.

El problema que resuelve está medido: una elevación muestra en profundidad todo lo que entre
en el *Far Clip* —en la vista del modelador son **87 vigas**— y el **2026-08-04**, sin
filtro, `ES-1002 EJE E` daba **51 tags sobre 66 piezas**.

**El representante de cada tipo es la pieza más cercana al plano del eje**, que es la que la
elevación está documentando. A igual distancia gana la más larga, porque es a la que mejor
le entra el rótulo largo.

> ⚠️ **El input `09.` dejó de ser un filtro duro.** Antes descartaba toda pieza a más de
> 30 cm del eje; ahora sólo **elige representante**: si de un tipo no hay ninguna pieza sobre
> el eje, se rotula igual la más cercana en vez de perder el tipo entero. Sin ese cambio la
> escalera y las barandas no recibirían tag nunca, porque casi nunca caen sobre el plano del
> eje.

El log lo reporta como `N tag(s) — uno por tipo: M tipo(s) distinto(s) sobre P pieza(s)
visibles`, y cuenta aparte los tipos que no tenían ninguna pieza sobre el eje.

**Las plantas no cambiaron**: ahí sigue habiendo un tag por viga, que es lo que una planta
tiene que decir.

### ⚠️ Qué categorías se rotulan

`CATS_ROTULABLES` pasó de dos categorías a cuatro familias de categoría:

| Categoría | Desde |
|---|---|
| `Structural Framing` (vigas **y la escalera**) | siempre |
| `Structural Columns` | siempre |
| `Stairs Railing` / `Railings` / `Railing System` | **2026-08-11** |
| `Stairs` | **2026-08-11** |

Antes sólo estaban vigas y columnas, así que **una baranda no recibía tag ni aunque cayera
justo sobre el eje**. Medido en `ES-1001 - EJE …` (id 7712664): la vista muestra 65
`Structural Framing`, 4 `Structural Columns` y **6 `Stairs Railing`**, y esas 6 quedaban
fuera por categoría.

> En este modelo la escalera es `Structural Framing` (`ESCALERA METÁLICA`), así que ya estaba
> cubierta. `OST_Stairs` va igual por si algún proyecto la modela como escalera de sistema.
> Los nombres se resuelven con `getattr` porque no todas esas categorías existen en todas las
> versiones de Revit; la que no exista se ignora sin romper nada.

### ⚠️ Cambio 2026-08-04: en planta ya no son TextNotes

Hasta esta versión las plantas se rotulaban con `TextNote`. El motivo era que un
`IndependentTag` muestra lo que dice su etiqueta y **la API no deja sobrescribir ese texto
por instancia**, así que con tags no se podía truncar — y en planta truncar es obligatorio,
porque el nombre completo no entra en el largo de la viga.

**La familia `C-MultiCat` resuelve eso desde el otro lado**: trae un tipo corto y uno largo
(parámetros `TagCorto` / `TagLargo` del propio tipo), así que «acortar» dejó de ser cortar
un string y pasó a ser **elegir el otro tipo de tag**. Con eso desaparece la contrapartida
que se había asumido: los rótulos de planta ahora son tags de verdad, asociativos, que se
actualizan solos si alguien le cambia el perfil a la pieza.

> Verificado en el modelo el 2026-08-04: los **11** tipos de Multi-Category Tag cargados
> son todos de la familia `C-MultiCat`, e incluyen `Modelo` (con `TagCorto = 1`) e
> `Item_TBL` (con `TagLargo = 1`). Como no hay otra familia Multi-Category en el proyecto,
> el script busca el tipo **por nombre de tipo** y no necesita desambiguar por familia; si
> no lo encuentra, el log lista los disponibles como `Familia : Tipo`.

Consecuencia: **el truncado en la `x` ya no se escribe en ningún lado**. La función
`acortar()` sigue en el script, pero solo para reconocer los rótulos de texto de la versión
anterior y barrerlos (ver más abajo).

### En la elevación ya eran tags — pero ahora también eligen

Es lo que usa el modelador en su vista de referencia (`ELEVACION EJE A`, id 6154003: 10 tags
de `C-MultiCat : Item_TBL`, los de las columnas con `Angle = π/2`).

Lo que sí cambió: antes la elevación ponía **siempre** el tipo largo, con el argumento de
que ahí el nombre entraba completo y no había nada que decidir. **Eso era falso**: el tag se
dibuja a lo ancho de la pieza igual que en planta —horizontal sobre la viga, girado sobre la
columna— así que una viga corta o una columna baja desbordan lo mismo. Ahora aplica el mismo
criterio, medido contra la dimensión que la pieza ocupa **en la dirección en que se lee el
tag**:

| Pieza | Orientación del tag | Contra qué se mide |
|---|---|---|
| Viga | horizontal, encima | largo proyectado sobre el `RightDirection` de la vista |
| Columna | girada 90°, a la izquierda | altura proyectada sobre el `UpDirection` de la vista |

Con eso los tres inputs de tipo de tag se consolidaron en dos: **el mismo par corto/largo
sirve para plantas y elevaciones**, y el input que antes elegía el tag de elevación pasó a
ser un booleano `08. Rotular también las elevaciones de eje`.

> ⚠️ **Bug corregido de paso.** La medición leía `Max.X − Min.X` del *bounding box*, lo que
> da el ancho del tag solo si el «a lo ancho» de la vista coincide con el eje X del modelo.
> En plantas suele valer; en una elevación de eje **no**, porque el `RightDirection` apunta
> según el eje del edificio. Ahora se proyecta la diagonal del bbox sobre `RightDirection`,
> que para un tag plano y sin girar da el ancho exacto sea cual sea la orientación de la
> vista.

- **Qué se rotula**: `Structural Framing` y `Structural Columns` que la elevación muestre
  y sean miembros del assembly, con el mismo filtro de familias excluidas que las plantas.
- **⚠️ Solo lo que está sobre el eje.** Una elevación muestra en profundidad todo lo que
  entre en el *Far Clip* —en la vista del modelador son **87** vigas— y rotularlas todas
  sería ilegible. Se rotula solo lo que esté a menos de una distancia **de la línea del
  eje** (input, default **30 cm**), que es el marco que esa elevación documenta. El log
  cuenta cuántas piezas quedaron fuera por ese motivo.

#### ⚠️ El plano de referencia es el eje, no `View.Origin`

Corrido el **2026-08-04**, el filtro descartaba **1762 de 1762** piezas: cero tags en las 24
elevaciones. Un filtro que descarta el 100% no es un filtro.

La causa está en cómo 00 crea la vista. El crop box abarca **todo el ancho del assembly en
profundidad** (de `lzmin` a `lzmax` respecto del plano del eje) y `ViewSection.CreateSection`
deja el plano de la vista en la **cara frontal** de esa caja:

```python
caja.Min = XYZ(lxmin - m, lymin - m, lzmin - m)
caja.Max = XYZ(lxmax + m, lymax + m, lzmax + m)
v = ViewSection.CreateSection(doc, sec_vft.Id, caja)
```

O sea que **`v.Origin` no está sobre el eje**: está a medio ancho del assembly de distancia
—metros, no centímetros— así que ninguna pieza entraba nunca en los 30 cm de tolerancia.

Ahora la referencia es **la línea del `Grid`** que la elevación documenta, que sale del
propio nombre de la vista (`ES-1001 - EJE B` → grid `B`, verificado: los 16 ejes que aparecen
en el log existen con ese nombre). Si el eje no existe en el modelo, la vista **no se
rotula** y el log lo dice — rotular 87 piezas es peor que no rotular.

> `v.Origin` **sí** se sigue usando para proyectar el punto del tag al plano de la vista.
> Son dos planos distintos con dos papeles distintos: el eje contesta *«¿está sobre el
> eje?»*, el plano de la vista decide *dónde se dibuja la anotación*. Si se dibujara en el
> plano del eje, éste puede quedar detrás del *near* y el tag se clipearía.

#### ⚠️ Y encima la categoría estaba apagada

Corregido lo anterior, la corrida siguiente puso **110** tags en las elevaciones… y **no se
veía ninguno**. Es la misma trampa que 03 ya tenía documentada para `Dimensions`, `Spot
Elevations` y `Lines`: el view template `ESTRUCTURAS_1/50` trae **`Multi-Category Tags`
apagada**.

Un tag creado en una vista con su categoría apagada **existe en el modelo pero no se dibuja,
y `FilteredElementCollector(doc, view.Id)` tampoco lo devuelve**. Eso rompe dos cosas a la
vez:

1. desde el script parece que nunca se creó;
2. como `tags_previos()` tampoco lo ve, **la idempotencia deja de funcionar** y cada corrida
   apila tags invisibles encima de los anteriores.

Ahora 05 llama a `encender_tags()` en cada vista —plantas y elevaciones— **antes** de buscar
los tags previos y antes de medir. El orden importa: si se encendiera después, se borraría
sobre una lista vacía y se mediría contra un *bounding box* que no existe.

> Si el view template bloqueara el cambio, `SetCategoryHidden` lanza y el log avisa que hay
> que encenderla a mano en el template. En este modelo no bloquea.

**Por qué 05 nunca lo necesitó antes**: cuando el tag era la única anotación de la elevación,
el modelador ya tenía la categoría encendida a mano en su vista de referencia (`ELEVACION
EJE A`), así que el problema no se veía. Las elevaciones que crea 00 nacen con el template
puesto.

#### Elevaciones que quedan vacías: casi siempre es 00, no 05

Con todo lo anterior corregido pueden seguir apareciendo cortes sin ningún tag. **A veces es
correcto**: desde el 2026-08-10 quién lleva elevación lo decide el parámetro `ASSEMBLY` del
eje, así que un eje marcado a mano lejos de toda pieza produce una elevación legítimamente
vacía. El corte sobra, y eso se arregla **desmarcando el eje**, no en 05.

Como la otra causa —que los 30 cm de tolerancia se queden cortos— produce **el mismo
síntoma**, 05 no adivina: cuando una elevación queda en cero escribe una línea `DIAG` con la
distancia de la pieza más cercana al eje.

| Lo que dice el `DIAG` | Qué significa |
|---|---|
| «la pieza más cercana está a **metros**» | el eje no toca el assembly → el `ASSEMBLY` de ese eje está mal puesto; se arregla en el modelo, no en 05 |
| «a **decenas de cm**» | falta tolerancia → subir el input `09` |

**Historia.** El 2026-08-04 el `DIAG` sobre las 10 elevaciones vacías dio entre 1,7 y 4,9 m
en todas: eran ejes sin marco, no falta de tolerancia. Se resolvió en 00 con el filtro
geométrico `ejes_con_marco()`, que el 2026-08-10 quedó reemplazado por el marcado explícito
(ver [00 § D](#d-elevaciones-de-eje)).
- **Orientación**: `TagOrientation.Horizontal` en las vigas y `Vertical` en las columnas
  (la pieza es vertical si su dirección se parece más al *arriba* de la vista que a su
  *derecha*). El rótulo de una viga va por encima de su eje y el de una columna a la
  izquierda, a una separación en mm de papel (input, default 2).
- El punto del tag se **proyecta al plano de la elevación** antes de colocarlo.
- **Idempotencia**: si la elevación ya tiene tags de cualquiera de los dos tipos se salta,
  salvo que `07. Rehacer los rótulos existentes` esté en True (el mismo input que gobierna
  las plantas).
- Con `08. Rotular también las elevaciones de eje` en False no se toca ninguna elevación y
  las plantas se rotulan igual.

### El «¿cabe?» se mide, no se estima

Nada de calcular anchos de fuente ni contar caracteres: **se mide el tag**. Por cada tipo de
pieza se crea un tag de prueba largo, horizontal y fuera del crop; se regenera **una sola
vez por vista** (lo caro es el `Regenerate`, no el tag) y se lee su *bounding box*. Con eso
el criterio queda en un `if`:

```
disponible = dimensión de la pieza en la dirección de lectura − 2 × margen
                                                            (input, default 2 mm de papel)
si ancho(tag largo) > disponible  →  se coloca el tipo corto
```

La «dirección de lectura» es el largo de la viga en planta, el `RightDirection` de la vista
para una viga en elevación y el `UpDirection` para una columna. **El margen es el mismo
input para los tres casos.**

Las mediciones se **cachean por (tipo de tag, tipo de pieza, escala)**, porque los perfiles
se repiten decenas de veces por vista.

> Esto mide el **tag renderizado**, no el string. Es más fiel que la versión anterior, que
> medía el texto que iba a escribir: ahora entran en la cuenta el recuadro, el relleno y
> cualquier prefijo que traiga la etiqueta de la familia.

Si la medición fallara (bounding box nulo), se deja el **tipo largo**: es el rótulo
completo. **Si ni el corto entra, se pone igual**: vale más una pieza rotulada de más que
una sin identificar. El log cuenta ambos casos por separado para plantas y elevaciones, por
si conviene resolverlos a mano.

### ⚠️ Migración de los rótulos de texto viejos

Las plantas que ya se rotularon con la versión anterior tienen `TextNote` que el nuevo graph
no reconocería como suyos, y los tags nuevos quedarían encima. Al pasar por cada planta se
borran, pero **solo los `TextNote` cuyo texto coincide exactamente con un nombre de perfil
de esa misma vista** — entero o cortado en la `x`, que es como acortaba antes. Un `TextNote`
con cualquier otro texto es una nota que puso alguien a mano y no se toca.

Por eso el script ya no necesita un input de tipo de texto: identifica los rótulos viejos
por su contenido, no por su tipo. El log dice cuántos barrió en cada vista.

> El truncado en la `x` era mecánico —tirar el peso en kg/m y dejar la designación:
> `C20x13,1` → `C20`, `L6,5x4,780` → `L6,5`— e insensible a mayúsculas, porque en el modelo
> conviven `L8x7,07` y `L8X7,07`. Esa regla sobrevive solo dentro de este barrido.

### Orientación en planta

El tag se crea `TagOrientation.Horizontal` en el **punto medio de la viga** y después se
gira con `ElementTransformUtils.RotateElement` sobre su propia cabeza, alrededor del eje Z.
Como la cabeza del tag es su centro, queda centrado sobre el eje de la viga sin el ajuste de
media altura que necesitaba el `TextNote` (cuyo origen era el tope de la línea).

El ángulo sale del eje en planta y se normaliza a (−90°, 90°] para que nunca se lea de
cabeza — misma regla que antes.

### Los dos tipos de tag son inputs

| Input | Default |
|---|---|
| `03. Tipo de C-MultiCat del rotulo CORTO` | `Modelo` |
| `04. Tipo de C-MultiCat del rotulo LARGO` | `Item_TBL` |

Son editables desde Player y valen para plantas **y** elevaciones. **Se exigen los dos**: si
falta cualquiera de ellos no se rotula nada y el log lo dice, junto con la lista de tags
disponibles como `Familia : Tipo`. Deliberadamente no cae a un tipo cualquiera — con un
nombre mal escrito, rotular decenas de piezas con el tag equivocado es peor que no rotular.

### Lo que NO hace

- **No evita colisiones** entre rótulos ni contra otras anotaciones. El modelador los acomoda
  a ojo; replicar eso es un problema aparte.
- **En planta no rotula columnas**, solo `Structural Framing` (en elevación sí, ambas).
- **No toca la NIPB.** Las conexiones de la placa base las rotula **03**
  (ver [03 § Rótulos de las conexiones en la planta NIPB](#rótulos-de-las-conexiones-en-la-planta-nipb)).
- El grating queda fuera por el input `06. Familias a NO rotular` (default `GRATING`).
- **No controla qué texto muestra cada tipo de tag.** Eso lo define la etiqueta de la
  familia `C-MultiCat`; el script solo elige entre el tipo corto y el largo.

### ⚠️ Tipos duplicados en el modelo

Hay tipos que son el mismo perfil con nombres distintos: `L8x7,070`, `L8x7,07` y `L8X7,07`
conviven, igual que `C20x13,1` con `C20x13,10` y `L8x5,960` con `L8x5,96`. Vigas idénticas
van a salir rotuladas distinto según qué tipo les tocó. Antes el truncado en la `x`
disimulaba buena parte; ahora depende de qué lea la etiqueta de cada tipo de `C-MultiCat`.
En cualquier caso es basura del modelo, no del graph.

---

## 06 — Tabla de materiales

Reemplaza a la tabla de fundaciones (que era *una tabla por lámina*, con los assemblies
documentados en ESA lámina). Ahora es **una tabla por assembly**, y va **solo en su primera
lámina** — el resto de las láminas del mismo assembly aprovechan el ancho completo. Por eso
el graph ya **no** tiene input de «sheets destino»: la lámina se deduce sola.

Sigue siendo un **schedule nativo** multi-categoría (no líneas de detalle + textos), colocado
con `ScheduleSheetInstance` en la esquina superior derecha del área útil, dentro de la franja
que 01/02 reservan.

### La tabla objetivo

```
+-------------------------------------------------------------------------------+
|                     LISTA DE MATERIALES TOLVA ES-1004                         |
+-------------+---------+------------+-------------------------------+----------+
|             |  CANT/  | DIMENSIONES|             PESO              |          |
| DESCRIPCION | UNIDAD  |  L (m)m2   | KG/M  | UNIT. KG  | TOTAL KG  |OBSERVACION
+-------------+---------+------------+-------+-----------+-----------+----------+
| ES-1004                                                                       |
| CANT=1                                                                        |
| []15x26,4   |   16    |   68,35    |  26,4 | 1.804,42  | 1.804,42  |ASTM A572 |
| ARS-5       |   53    |   70,21    |    33 | 2.316,88  | 2.316,88  |ASTM A572 |
|    ...                                                                        |
+-------------+---------+------------+-------+-----------+-----------+----------+
|                568    |  567,45    |       | 27.539,16 | 27.539,16 |          |
+-------------------------------------------------------------------------------+
```

| Columna | De dónde sale |
|---|---|
| `DESCRIPCION` | nombre del **tipo** de familia (campo `Type`) |
| `CANT/UNIDAD` | campo nativo `Count`: instancias de ese tipo **en un** assembly |
| `L (m)m2` | metros de eje, o m² si el tipo cae en los prefijos de m² |
| `KG/M KG/M2` | parámetro **de tipo** `PESO_UNIT` (input) — no se calcula |
| `UNIT. KG` | `L × PESO`, sumado sobre las instancias del tipo |
| `TOTAL KG` | `UNIT. KG × CANT` (instancias de ese **tipo de assembly** en el proyecto) |
| `OBSERVACION` | parámetro compartido `MATERIAL`, tal cual |

Las dos filas de encabezado (`ES-1004` y `CANT=1`) son *group headers* del schedule, igual que
en la versión de fundaciones. Las filas se ordenan por **familia** y después por **tipo**, para
que todos los tipos de una familia queden juntos.

### Una fila por tipo, y solo las hojas

Se recorren los miembros del assembly **recursivamente**: familias anidadas dentro de familias
aparecen como filas propias. Un elemento que a su vez contiene sub-componentes es un
envoltorio (la familia contenedora, normalmente sin geometría propia) y **no** genera fila;
solo las hojas cuentan. Se recorre únicamente la **primera instancia** de cada tipo de
assembly, misma convención que el resto del pipeline.

### ⚠️ Las columnas son numéricas, y eso obliga a 3 parámetros de instancia

La fila de totales del pie solo existe si los campos son **numéricos** con
`DisplayType = Totals`. La versión de fundaciones escribía los valores como **texto**
(reusando `Status Vendor`) precisamente porque los campos calculados nativos no eran
confiables — y por eso no podía tener ni sumas por fila ni total general.

`Totals` hace doble trabajo acá: además del total del pie, es lo que permite escribir el valor
**por instancia** y que la fila del tipo muestre la **suma** (68,35 m repartidos entre 16
piezas de largos distintos). Sin eso habría que escribir el total del grupo en cada instancia,
que es justo lo que se rompe cuando dos piezas del mismo tipo miden distinto.

Hacen falta entonces tres parámetros **de instancia** tipo *Number*: `DIMENSIONES`,
`UNIT. KG` y `TOTAL KG`. Solo el primero existe en el esquema de la oficina (`BIDIMENSION`).
Los otros dos, si no existen, **los crea el graph** como parámetros de proyecto
(`PESO_UNIT_TBL`, `PESO_TOTAL_TBL`), instancia, todas las categorías de modelo, con un GUID
derivado por MD5 del nombre — así son el mismo parámetro en todos los modelos sin depender de
un archivo de parámetros compartidos versionado. El archivo `.txt` temporal que Revit exige
para crearlos se escribe en `%TEMP%` y `SharedParametersFilename` se deja como estaba.

#### ⚠️ Renombre del esquema de la oficina (2026-09-16)

La jefa de modeladores mandó el esquema nuevo. Los que tienen «Alternativa Parámetro» se
renombran; los que dicen **ASSEMBLY** en gris se eliminan porque la herramienta *Assembly* de
Revit ya cubre esa necesidad.

| Parámetro viejo | Nuevo | Ámbito | Qué pasa en 06 |
|---|---|---|---|
| `ITEM_TBL` | `DESCRIPCION_ITEM` | Type, Text | **06 no lo usa**: la columna `DESCRIPCION` sale del nombre del `Type` |
| `RECUENTO_TBL` | `BIDIMENSION` | Instance, Number | ✅ renombrado (default del input 10) |
| `PESO_TBL` | `PESO_UNIT` | **Type**, Number | ✅ renombrado (default del input 09) |
| `ELEMENTO_TBL` | *(ASSEMBLY)* | Instance, Text | 06 lo **escribe** hoy vía `GUID_TAG` |
| `ELEMENTO_CANT_STR_TBL` | *(ASSEMBLY)* | Instance, Text | 06 lo **escribe** hoy vía `GUID_PART_NUMBER` |
| `CANT_TBL` | *(ASSEMBLY)* | Instance, Integer | 06 no lo usa |

> ⚠️ **`PESO_UNIT` es el de TIPO, el del catálogo.** Es el peso **por metro** (o por m²), no
> los kilos de la pieza. No confundir con el `PESO_UNIT_TBL` **de instancia** que crea y
> calcula 06 (`largo × PESO_UNIT`). Con el perfil `[]15x26,4`: `PESO_UNIT` = 26,4 y
> `PESO_UNIT_TBL` = 1.804,42. Los nombres quedaron peligrosamente parecidos.

**Lo que este renombre NO resuelve.** El esquema nuevo trae dos parámetros por **fórmula**
que hacen exactamente lo que 06 calcula y escribe a mano:

```
PESO_UNITARIO#  = BIDIMENSION * PESO_UNIT
PESO_TOTAL#     = Cantidad de Assemblies * BIDIMENSION * PESO_UNIT
```

Si esas fórmulas viven en el modelo, `PESO_UNIT_TBL` y `PESO_TOTAL_TBL` **sobran**: 06 no
tendría que crearlos ni escribirlos, solo poner esos campos en el schedule. Lo mismo con
`ELEMENTO_TBL` y `ELEMENTO_CANT_STR_TBL`, que 06 escribe hoy para agrupar por assembly y que
la herramienta *Assembly* ya provee. Eso es una reestructuración de 06, no un renombre, y
está **sin decidir**. Ver [§ Pendientes](#pendientes--próximos-pasos).

> **Si se apunta un input a un parámetro de TIPO, la columna suma mal en silencio**: las N
> instancias comparten un solo valor, la última escritura pisa a las demás y la fila muestra
> ese valor multiplicado por N. El graph lo detecta comparando el dueño del parámetro
> (`Parameter.Element`) contra el elemento, y lo dice en el log como ERROR.

### Qué se mide en metros y qué en m²

Todo es lineal **salvo las pletinas `PL` y el grating `ARS`**, que van en m². La decisión es
por **prefijo del nombre del tipo** (input `08`, default `PL,ARS`), no por categoría ni por
geometría: `PL` atrapa también `PLACA DIAMANTADA` y `PLACAS DE CONEXIONADO`, que en el plano
tipo efectivamente van en m².

- **Lineal**: el largo del **eje** (`LocationCurve`), no el del sólido. Ver *«una pieza de
  acero tiene dos largos»* en la sección de 04: el sólido viene recortado en las puntas por
  `Join Cutback` / `Start-End Extension` y en una diagonal medida daba 302 mm menos. Si la
  pieza no tiene eje (columnas, piezas sin `LocationCurve`) cae a `System Length` → `Length`
  → `Cut Length` → dimensión mayor del bounding box local. El log dice de dónde salió cada
  medida y cuántas cayeron a cada fallback.
- **m²**: las dos dimensiones mayores de la caja que envuelve la geometría **en el sistema
  local de la instancia**, no el bounding box del mundo — así también vale para pletinas y
  paños girados. Es el mismo criterio con el que 04 saca el contorno del grating, que no es
  una plancha sino un peine de barras y por eso no se puede medir cara por cara.

### El filtro es `ASSEMBLY`, no `Comments`

`Assembly Name` sí trae el nombre del assembly (1.247 elementos lo tienen en el modelo real),
pero **no sirve**: Revit sólo admite **parámetros compartidos** como campo de un schedule
*Multi-Category*, y `Assembly Name` es nativo. Tampoco sirve escribir en el esquema `_TBL` de
las familias anidadas más profundas: ahí esos campos quedan **bloqueados por fórmula de
familia**.

Por eso el nombre del assembly se **copia** a un parámetro compartido que sí se puede poner
como columna, y por eso el filtro de cada tabla es `ASSEMBLY = {nombre del assembly}`.
`Part Number` lleva el `CANT=N`.

La versión anterior además escribía el número de sheet en `Comments` y filtraba por ahí. Ya no:
**`Comments` no se toca**. Los valores que haya dejado la versión vieja son inocuos.

#### ⚠️ Hay DOS parámetros llamados `ASSEMBLY`

**Cambio 2026-09-16.** Antes se reusaba el compartido `TAG`
(`7e3a03e3-1771-478b-ad2e-edd00325f602`), que no figuraba en el esquema de la oficina y
significaba otra cosa. Karin creó un compartido propio y 06 pasó a usarlo:

| Parámetro | GUID | Categorías | Quién lo llena |
|---|---|---|---|
| `ASSEMBLY` **compartido** | `92fdc373-364b-4440-8198-f8093299f73a` | categorías de modelo | **06**, copiándolo de `Assembly Name` |
| `ASSEMBLY` **no compartido** | — (sin GUID) | sólo *Grids* | el **modelador**, a mano |

Son **parámetros distintos que comparten nombre**. Hoy conviven sin problema porque están en
categorías disjuntas, y 06 accede por GUID, así que no hay ambigüedad posible de su lado.

> ⚠️ **00 y 03 buscan `ASSEMBLY` por NOMBRE** (`LookupParameter`), no por GUID — por eso
> encuentran el de los ejes. Si alguien vincula el compartido también a *Grids*, esa búsqueda
> se vuelve ambigua y los dos graphs pueden leer el equivocado: el assembly se quedaría sin
> elevaciones y con las plantas sin ejes. **No vincular el compartido a Grids.**

**Limpieza**: al borrar un assembly sus miembros quedan sueltos en el modelo pero conservan el
valor escrito, y reaparecerían como filas fantasma. Cada corrida releva el modelo una vez y
borra el `ASSEMBLY` de lo que ya no sea miembro legítimo — pero **solo cuando su valor es
exactamente el nombre de un tipo de assembly**; cualquier otro valor es del modelador y no se
pisa.

#### Se escribe sólo en la PRIMERA instancia de cada assembly

06 recorre `by_type[tname][0]` — los miembros de **una** unidad — y la tabla multiplica por
`CANT=N`. Por eso `ASSEMBLY` queda vacío en los miembros de las instancias repetidas.

Es deliberado, y confirmado con Javier el **2026-09-16**: la lista de materiales muestra el
contenido de un assembly y el `CANT=N`, que es el formato del plano tipo. Si se llenara
`ASSEMBLY` en todos los elementos, un assembly repetido 3 veces mostraría el triple en
`CANT/UNIDAD` y el nónuplo en `TOTAL KG`.

### ⚠️ Encabezados agrupados: sin verificar

`DIMENSIONES` sobre la columna de largo y `PESO` sobre las tres de peso son la fila de
*grouped headers* que en la UI de Revit hace el botón *Group*. Por API hay que insertar una
fila en la sección de encabezado del `TableData` y fusionar celdas (`TableSectionData.InsertRow`
+ `MergeCells`), y **no se pudo comprobar sin Revit abierto**: no está confirmado en qué
sección (`Header` o `Body`) vive la fila de títulos ni si `InsertRow` es aceptada ahí. El graph
prueba las dos secciones, busca la fila cuyo primer texto sea `DESCRIPCION`, y deja en el log
la geometría real de la sección (`DEBUG ...: fila de titulos en ... (sección de NxM)`). Si
falla, la tabla sale igual pero con los encabezados en **una sola fila** — todo lo demás no
depende de esto.

Es idempotente: si la fila de grupos ya está, no inserta otra.

### ⚠️ Los tipos duplicados salen como filas duplicadas

Lo mismo que se dijo para los rótulos de 05 (`L8x7,070` / `L8x7,07` / `L8X7,07` conviviendo)
pega más fuerte acá: cada nombre es una fila distinta con su propio peso y su propio total. La
tabla no los une — es basura del modelo, y unir por «parecido» sería inventar.

---

## Convenciones (contrato entre graphs)

- **Nombres internos de vista** (no renombrar a mano — 01 y 02 los buscan por nombre):
  - `{assembly} - NIPB`
  - `{assembly} - T.A. NN` (correlativo ascendente por altura)
  - `{assembly} - EJE {eje}`
- El **título en lámina** es siempre distinto del nombre interno y lo escribe 00 vía
  *Title on Sheet* (`VIEW_DESCRIPTION`). 02 ya **no** reescribe títulos.
- Si hay **varias instancias del mismo tipo de assembly**, se documenta la primera.
- Distancias de anotación y márgenes se ingresan en **mm de papel** y se convierten con la
  escala de cada vista.
- Offsets y tolerancias geométricas se ingresan en **cm**.
- **Qué ejes llevan elevación lo decide el parámetro `ASSEMBLY` del eje**, no la geometría
  (desde el 2026-08-10). 00 ya no tiene tolerancia propia para esto: `TOL_MARCO` y
  `ejes_con_marco()` se eliminaron. 05 conserva su input `09` (30 cm) para decidir **qué
  piezas rotula dentro** de la elevación, que es otra pregunta. Si una elevación marcada
  sale sin tags, revisar el marcado del eje antes que la tolerancia de 05.
- **06 se apoya en los nombres de vista de 00** para saber cuál es la primera lámina de cada
  assembly (`{assembly}` o `{assembly} - ...`). Si alguien renombra vistas a mano, 06 deja
  ese assembly sin tabla y lo dice en el log.
- **Parámetros que 06 escribe en los elementos**: `ASSEMBLY` (nombre del assembly), `Part Number`
  (`CANT=N`) y los tres numéricos de las columnas. No toca `Comments` ni `UNIDAD_TBL`, que sí
  usaba la versión de fundaciones.
- **El título bajo cada vista lo controla el tipo de viewport**, no la vista: plantas →
  input «Tipo de viewport para plantas», elevaciones → «Tipo de viewport para elevaciones
  de eje». Si el tipo no existe, 02 lo crea (duplica uno existente, activa *Show Title* y
  asigna la familia de View Title).

## Idempotencia (qué pasa al re-correr)

- `00`: si la vista ya existe **no la recrea, pero sí le vuelve a aplicar** escala, view
  template, profundidad, aislamiento y título en lámina. Antes las elevaciones existentes
  se saltaban enteras y quedaban congeladas con los defaults de la corrida que las creó
  (visto el 2026-07-28: elevaciones con `ESTRUCTURAS_1/50`, escala 1:100 y far clip
  8550 mm bastante después de cambiar los defaults). Con «Recrear vistas existentes» = True la borra y
  rehace — y además **limpia todas las T.A. y elevaciones previas del assembly**, no solo
  las que va a recrear: si alguien borra un nivel `T.A._...` o desmarca el `ASSEMBLY` de un
  eje, esta corrida devuelve menos vistas que la anterior y las sobrantes quedarían
  huérfanas con un correlativo que ya no aplica.
- `01`: si no hay vistas pendientes de colocar, no crea ninguna lámina. Los números de
  sheet ya usados en el proyecto se saltan.
- `02`: vistas ya colocadas se omiten; solo se usan láminas **sin ningún viewport** (un
  assembly no puede compartir lámina, así que no se rellenan láminas a medias).
- `06`: no salta nada — **siempre reconfigura** la tabla de cada assembly (campos,
  agrupación, filtro, anchos, grafismo y título), así que re-correr también retrofitea las
  tablas ya colocadas. Si la tabla quedó en una lámina que ya no es la primera del assembly
  (cambió la cantidad de láminas o el orden), la instancia se borra y se recoloca. El
  schedule en sí se reusa por nombre (`TBL_{assembly}`); solo se borra y rehace si es de
  una versión anterior del graph, porque la categoría de un `ViewSchedule` no se puede
  cambiar una vez creado.
- `05`: una vista que ya tiene tags de alguno de los dos tipos se salta, salvo con
  `07. Rehacer los rótulos existentes` en True. ⚠️ **La idempotencia depende de que la
  categoría `Multi-Category Tags` esté encendida en la vista**: si estuviera apagada, el
  collector no ve los tags previos y cada corrida los apila invisibles. Por eso 05 la
  enciende antes de contarlos — la misma precaución que 03 toma con `Dimensions`.

## Mensajes de log frecuentes

- `no hay ningun Assembly en el modelo` (00) — falta el paso previo: crear los assemblies
  en Revit. Nada de este pipeline funciona sin ellos.
- `no existe la vista ... (corre 00_Vistas de assembly)` (01/02) — falta el paso 0.
- `no existe el nivel "N.I.P.B._X"` (00) — falta modelar ese nivel; el assembly se queda sin
  planta NIPB. No hay cota de reemplazo.
- `no existe ningun nivel "P.T._X" ni "P.T._N_X"` / `"T.A._X" ni "T.A._N_X"` (00) — el
  assembly se queda sin ninguna planta de esa clase. Ojo con el formato viejo `T.A._X_01`,
  con el correlativo **atrás**: ya no vale para ninguna de las tres.
- `N nivel(es) nombran al assembly sin empezar por ningun prefijo` (00) — el log los lista.
  Puede ser un typo del prefijo (`T.A_ES-1001` por `T.A._ES-1001`), un typo dentro del
  nombre del assembly (`T.A._ES_1001_01`), un T.A. en el formato viejo (`T.A._ES-1001_01`), o
  un nivel viejo legítimo que no debe generar planta. Hay que mirarlos: 00 no adivina cuál es
  cuál, y **ninguno de ellos genera planta hasta que se lo renombre en Revit**.
- `el nivel T.A._2_X dice "2" pero por altura es el 01` (00) — el correlativo del nivel no
  coincide con su posición por altura. La vista se nombra por altura igual, así que el
  pipeline sigue funcionando; lo que hay que corregir es el modelo. **Hoy sale dos veces en
  ES-1001**: el `_2_` está 40 cm por debajo del T.A. pelado, que declara un `1` implícito.
- `el nivel N.I.P.B._X esta en EL. ... y la cara mas baja del assembly en EL. ...` (00) — el
  nivel no coincide con la geometría real (más de 20 cm). **Manda el nivel** y la planta se
  crea igual, pero uno de los dos está mal.
- `hay N niveles "N.I.P.B._X..."; el N.I.P.B. es uno solo por assembly` (00) — duplicados;
  se usa el más bajo. En `P.T._` y `T.A._` varios **no** son un problema: son N por assembly.
- `ningun eje del modelo tiene el parametro ASSEMBLY` (00) — hay que crearlo: parámetro de
  proyecto de texto en la categoría *Grids*. Sin él **no se crea ninguna elevación y las
  plantas salen sin ejes**.
- `ningun eje tiene ASSEMBLY = "X"` (00) — nadie marcó ejes para ese assembly. El log lista
  los valores que sí aparecen en el modelo: casi siempre es un typo o un espacio de más.
- `el eje X esta marcado con ASSEMBLY = "Y" pero no es una linea recta` (00) — ejes en arco;
  no se les puede alinear una caja de sección.
- `nada a menos de N cm del eje` (05) — la elevación existe y quedó sin tags. Con el marcado
  explícito esto es marcado de más en el eje, no falta de tolerancia en 05.
- `sin geometria solida valida en ningun miembro` (00) — geometría rota en el modelo
  (elemento con `Volume = 0`); hay que arreglarlo en Revit, no en el script.
- `SIN ESPACIO para N vistas` (02) — correr 01 para agregar las láminas que falten.
- `el tipo seleccionado NO es una viñeta` (01) — elegir del dropdown uno de los tipos que
  el propio log lista.
- `sin lamina todavia (corre 01 y 02)` (06) — el assembly existe pero ninguna lámina tiene
  todavía una vista suya, así que no hay dónde poner la tabla.
- `el parametro compartido ASSEMBLY (...) no esta cargado en el modelo` (06) — sin `ASSEMBLY`,
  `Part Number` o `MATERIAL` la tabla saldría vacía; hay que cargar el esquema de la oficina.
  Ojo que el que busca 06 es el **compartido** (`92fdc373-…`), no el de los ejes.
- `Se creo el parametro de proyecto "PESO_UNIT_TBL"` (06) — **es normal la primera vez** en
  cada modelo. Ver *Las columnas son numéricas*.
- `"X" es un parametro de TIPO, no de instancia` (06) — un input de parámetro numérico
  apunta a un parámetro de tipo; la columna sumaría mal. Cambiarlo o dejar el default.
- `N tipo(s) sin "PESO_UNIT" (peso 0)` (06) — esos tipos salen con 0 en `UNIT. KG` y
  `TOTAL KG`. Es dato faltante del modelo, no del graph.
- `no se pudo insertar la fila de encabezado agrupado` (06) — la tabla sale bien pero con
  los encabezados en una sola fila, sin `DIMENSIONES` / `PESO`. Ver la advertencia de 06.

## Pendientes / próximos pasos

1. **Probar en Revit lo de las elevaciones** (03: ejes recortados, cotas de altura, marcas
   de nivel y línea de terreno; 05: tags de perfil). Lo más frágil es la **referencia a
   caras** de 03: si Revit rechaza alguna, el log lo dice con el texto exacto del error y
   la cota desaparece al comitear. Verificar también contra el plano-tipo si la cadena
   queda bien repartida entre las dos verticales.
2. Arreglar en 00 que `Structural Connections` se enciende en las T.A., donde no
   corresponde.
3. **Tipo de cota**: el modelador usa `2.5 ROMAND(MILIMETROS)` en todas. Hoy ni 03 en
   planta ni 03 en elevación fijan el `DimensionType`: queda el que traiga el proyecto o
   el view template. Si en el plano salen con otra fuente, hace falta un input más.
4. **Probar 06 (tabla) en Revit.** Ya está adaptado (una tabla por assembly en su primera
   lámina, columnas del plano tipo, sin exclusión de categorías). Falta verificar, por orden
   de riesgo:
   - los **encabezados agrupados** (`DIMENSIONES` / `PESO`): es lo único que no se pudo
     razonar hasta el final sin Revit. El log trae el `DEBUG` con la geometría real de la
     sección para corregirlo en una pasada;
   - que `BIDIMENSION` sea de **instancia** y sin unidades (el log lo dice si no);
   - que la creación de `PESO_UNIT_TBL` / `PESO_TOTAL_TBL` funcione — toca
     `SharedParametersFilename` y lo restaura, pero eso no se probó;
   - que los prefijos `PL,ARS` cubran de verdad todo lo que va en m² en el modelo real;
   - que el ancho de columna calculado quepa en los 150 mm reservados sin cortar texto.
5. **Decidir si 06 sigue calculando `UNIT. KG` y `TOTAL KG`, o los deja al modelo.** El
   esquema nuevo de la oficina (2026-09-16) trae `PESO_UNITARIO#` y `PESO_TOTAL#` por
   **fórmula** (`BIDIMENSION * PESO_UNIT` y `Cantidad de Assemblies * BIDIMENSION *
   PESO_UNIT`), que es exactamente lo que 06 calcula y escribe por instancia. Si esas
   fórmulas existen de verdad en el modelo, sobran `PESO_UNIT_TBL` y `PESO_TOTAL_TBL`: 06
   dejaría de crearlos y de escribirlos, y sólo pondría esos campos en el schedule.

   Lo mismo del otro lado: `ELEMENTO_TBL` y `ELEMENTO_CANT_STR_TBL` quedan marcados como
   cubiertos por la herramienta *Assembly*, y 06 los escribe hoy (vía `GUID_TAG` y
   `GUID_PART_NUMBER`) para agrupar las filas por assembly.

   Antes de tocar nada hay que confirmar **dónde** viven esas fórmulas: un parámetro de
   proyecto de Revit **no admite fórmulas**, así que o son *calculated values* del propio
   schedule (otra API: `ScheduleDefinition.AddCalculatedParameter`) o son parámetros de
   familia. Cambia bastante qué hay que escribir. También falta definir de dónde sale
   `Cantidad de Assemblies` y si `TIPO_ACERO` (Type, Text, `PESADO`) entra en la tabla.
6. **`07_Cantidad en leyendas`** fue borrado del working tree (aparece como `D` en git) —
   decidir si se recupera para acero o se descarta.
7. **Verificar en Revit** todo lo de esta iteración: no se pudo probar porque el modelo no
   tiene assemblies. En particular quedan dos supuestos técnicos sin confirmar:
   - que `AssemblyViewUtils.CreateDetailSection` acepte **varias** `HorizontalDetail` para
     el mismo assembly (00 cae a `Duplicate` de la planta NIPB si falla o si devuelve la
     misma vista);
   - que `ConvertTemporaryHideIsolateToPermanent` deje el aislamiento estable en las
     elevaciones de eje;
   - **que los ejes se puedan ver y acotar dentro de una vista de assembly.** Las vistas
     de assembly limitan lo que muestran a los miembros del assembly, y los datums (ejes,
     niveles) son un caso aparte. Si Revit no los deja aparecer, las cadenas de 03 en
     planta no se van a poder crear (el log lo dirá: «cadenas eliminadas por referencias
     inválidas»). Plan B en ese caso: acotar sobre las elevaciones, o generar las plantas
     como secciones normales aisladas, igual que ya se hace con las elevaciones de eje.
   - que el crop ceñido al assembly no deje las **burbujas de los ejes** fuera de la
     vista. El *Annotation Crop* está desactivado, que es la condición necesaria, pero
     habrá que mirarlo en pantalla.
8. **Orden de colocación configurable** en 02 (hoy alfabético por assembly).
