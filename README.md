# Generador de planos de assemblies de ACERO — Revit 2027

Pipeline de graphs de Dynamo (4.0, CPython3, **sin paquetes externos**) para documentar
assemblies de estructura metálica: planta N.I.P.B., plantas T.A. por agrupación
geométrica de vigas, elevaciones por eje, distribución en láminas y tabla de cantidades.

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

**Primera vez en una máquina nueva**: abrir cada .dyn en Dynamo (no en Player), correr y
guardar. Eso registra los inputs para Dynamo Player. Después, todo se opera desde Player.

## Orden de ejecución

| # | Graph | Estado | Qué hace |
|---|-------|--------|----------|
| 0 | `00_Vistas de assembly.dyn` | ✅ acero | Por assembly: 1 planta NIPB + N plantas T.A. + 1 elevación por eje que lo cruza |
| 1 | `01_Calcular y crear laminas.dyn` | ✅ acero | Calcula cuántas láminas hacen falta (1 assembly por lámina) y las crea |
| 2 | `02_Colocar vistas en laminas.dyn` | ✅ acero | Coloca las vistas en flujo, más las leyendas |
| 3 | `03_Ejes y cotas entre ejes.dyn` | ✅ acero | Enciende los ejes (una burbuja por eje), dibuja el eje de cada viga en las T.A. (L-CENTER) y acota entre ejes y entre ejes de viga |
| 4 | `04_Grating en plantas.dyn` | 🧪 acero | Dibuja el grating de las T.A. como Filled Regions recortadas contra las vigas — **sin probar en Revit** |
| 5 | `05_Rotulos de perfil en plantas.dyn` | 🧪 acero | Escribe el nombre de tipo sobre cada viga de las T.A., acortándolo si no entra — **sin probar en Revit** |
| 5b | `05_Tags en plantas.dyn` | ➖ neutro | Multi-Category Tag por selección manual; sigue sirviendo como herramienta suelta |
| 6 | `06_Tabla de assemblies.dyn` | ⚠️ fundaciones | Tabla por sheet — **sin adaptar** (ver *Pendientes*) |

**Flujo**: 00 (crear vistas) → 01 (calcular y crear láminas) → 02 (colocar vistas y
leyendas) → 03 (ejes y cotas entre ejes) → 04 (grating) → 05 (rótulos de perfil). El paso 06
todavía responde al modelo de fundaciones; ver *Pendientes*.

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
**aisladas** a sus miembros + los ejes que lo cruzan + los marcadores de sus elevaciones
(`IsolateElementsTemporary` → `ConvertTemporaryHideIsolateToPermanent`). Es la misma técnica
que ya funcionaba en las elevaciones de eje.

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
- El tipo se **reaplica también a las vistas que ya existían**, así que cambiar el nombre de
  la carpeta y volver a correr las mueve todas — no hace falta recrearlas.
- El grafismo se hereda del tipo base, así que las vistas se ven igual que antes.

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
Por eso las etiquetas van numeradas con dos dígitos (`01.`, `02.`, … `13.`): sin el cero a
la izquierda, `10.` se ordenaría antes que `02.`. El orden resultante va de lo que se toca
siempre (qué procesar, carpeta, tipo de vista) a lo que casi nunca se toca (márgenes y
far clip).

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

### A. Planta N.I.P.B. (Nivel Inferior Placa Base)

Exactamente **1 por assembly**. El plano de corte se sitúa a **+1.00 m** (input, en cm)
por encima de la **cara sólida más baja de todo el assembly** — sea placa base, pletina o
lo que resulte ser el punto más bajo. No se usa ningún `Level` nativo: la cota se mide
sobre geometría real (`Solid.Volume > 0`, incluidas familias anidadas), porque los niveles
declarados por los modeladores no son fiables.

- Nombre interno: `{assembly} - NIPB`
- Título en lámina: `{assembly} - PLANTA N.I.P.B.`

### B. Plantas T.A. (Tope de Acero) — agrupación geométrica

Cantidad **variable** por assembly. El criterio es puramente geométrico:

1. De cada viga (`Structural Framing`, recursivo a cualquier profundidad) se extrae un par
   **(Z de la cara superior, largo en planta)**. El largo es el **peso** con el que esa
   viga vota; se mide como la diagonal en planta de su bounding box.
2. **Detección de niveles por picos de densidad** (*mode seeking*): la cubeta de 5 mm que
   concentra **más metros de viga** es el primer nivel, y absorbe todas las vigas a
   **±tolerancia/2** de ese pico. Se repite con las que sobran hasta que no queda ninguna.
3. Se **descartan los niveles minoritarios**: los que no llegan al **40 %** de los metros
   de viga del **nivel más grande** (input `11.`) son ruido, no niveles. Ver abajo.
4. El **nivel dominante** de cada grupo es la cubeta de 5 mm con más metros de viga (no más
   piezas). Empate → gana la cubeta más alta. Ver abajo.
5. Una planta por nivel, con el plano de corte **+1.00 m** (input `12.`) sobre el nivel
   dominante, **recortado** para que nunca alcance el nivel de arriba. Ver abajo.
6. La profundidad la fija el View Range (ver más abajo).

#### ⚠️ Por qué picos de densidad y no encadenamiento

La primera versión recorría las vigas de abajo hacia arriba y cerraba el grupo cuando una
se alejaba más de la tolerancia del **inicio** del grupo. Con una escalera continua de
vigas esa ventana se cierra en un punto **arbitrario** — el que caiga a `tol` del inicio —
y puede **partir al medio el racimo denso que es el T.A. real**. Evidencia en el log de
ES-1003 (2026-07-29):

```
T.A. 02: 112 viga(s) / 151.1 m de viga, dominante EL. 3,30, amplitud 1.00 m
T.A. 03: 106 viga(s) / 179.5 m de viga, dominante EL. 3,30, amplitud 0.13 m
```

**Dos plantas para el mismo nivel 3,30**, porque la ventana se cerró justo en medio de ese
racimo. La `amplitud 1.00 m` — exactamente la tolerancia — es la firma del bug: el corte lo
decidió la ventana, no la geometría.

Buscando el pico primero, el nivel nace **centrado en la densidad** y ningún racimo se
puede partir: o entra entero en el radio, o no entra. La tolerancia deja de ser «el ancho
que se permite acumular» y pasa a ser «qué tan lejos del pico puede estar una viga para
seguir siendo del mismo nivel».

- Nombre interno: `{assembly} - T.A. 01`, `02`… (correlativo **ascendente por altura**)
- Título en lámina: `{assembly} - PLANTA T.A. (EL. 2.303,37)` — cota real en metros, con
  punto de miles y coma decimal, igual que las vistas que ya existen en el modelo
  (`PLANTA ESTRUCTURA EL. 2.305,25 T.A.`)

El correlativo es el nombre estable que usan los pasos siguientes; la cota va solo en el
título mostrado, porque si un modelador mueve una viga y el cluster se desplaza, un
nombre basado en la cota rompería las búsquedas por nombre.

#### ⚠️ El nivel dominante se vota por metros de viga, no por cantidad de piezas

Medido en **ES-1002** el **2026-07-29**. El modelador confirma que ese assembly tiene
**dos** T.A., a **+150** y **+1200** del nivel `RLO_T.A. ES-1003` — que está en **EL. 6,000 m
exacta** de proyecto (verificado con el *Project Base Point*: `Elev = 7547,900 ft`, el offset
de cota compartida). O sea: T.A. en **EL. 6,150** y **EL. 7,200**.

Contando **piezas**, la moda elegía el nivel equivocado en los dos grupos:

| Grupo | Racimo que ganaba por cantidad | Racimo correcto | Piezas | Metros de viga |
|---|---|---|---|---|
| 1 | EL. 6,550 (8 × `C20x13,1` de 5,5 m) | **EL. 6,150** | 8 vs **7** ❌ | 41,1 vs **63,7** ✅ |
| 2 | EL. 7,070 (6 angulares `L6,5`) | **EL. 7,200** | 6 vs 3 ❌ | 25 vs **38,7** ✅ |

El grupo 1 perdía **por una sola pieza**, aunque el racimo correcto incluye dos vigas
`IN25x46,6` de **17,8 m**. Consecuencia: la planta salía cortada y **rotulada 400 mm por
encima** del T.A. verdadero (`PLANTA T.A. (EL. 6,55)`).

Por eso el voto es el **largo de la viga**, tomado como la diagonal en planta de su bounding
box — contrastado contra el parámetro `Length` del modelo (`IN25x46,6` de 17,811 → 17,826
medidos; `C20x13,1` de 5,479 → 5,484). No se lee `Length` directamente para no depender de
que cada familia lo publique. Si ninguna viga reporta largo útil, se cae al criterio anterior
por cantidad.

> **La tolerancia se queda en 1 m.** Bajarla a 50 cm no reduce vistas, las **aumenta**:
> partiría el grupo 1 de ES-1002 en dos y daría **3 plantas** donde hay 2. Lo que reduce
> vistas es el filtro por proporción de acá abajo, no la tolerancia.

#### ⚠️ `Structural Framing` es un cajón de sastre: filtro por proporción

Medido en el modelo real el **2026-07-29** (assembly **ES-1003**, 240 miembros): el graph
generaba **5 plantas T.A. cuando el T.A. real es uno solo**. La causa no es la subcategoría
(*Girder* vs *Other*) — los 240 miembros reportan la misma categoría `OST_StructuralFraming`
y el filtro los toma a todos por igual. La causa es **qué vive dentro de esa categoría**:

| Familia | Tope (`Max.Z`) | Qué es |
|---|---|---|
| `ESCALERA METÁLICA1` | 4,393 m | una **escalera completa**, una sola pieza que abarca 1,06 → 4,39 m |
| `PELDAÑO METALICO` ×4 | 3,136 / 2,936 / 2,736 / 2,536 m | **peldaños**, separados exactamente 20 cm |
| `OR100x14,4`, `L-Viga` | 5,667 m | 2 piezas sueltas |
| `C10x7,20` | 4,690 / 2,540 / 1,940 m | piezas sueltas |
| `GRATING ARRIGONI` ×5 | 3,332 m | rejilla (cae en el grupo bueno, inofensiva) |
| **vigas reales** ×~218 | **3,300 m** | el T.A. verdadero, todas al mismo Z exacto |

Los peldaños son los peores: forman una **escalera de topes separados 20 cm** y, como el
agrupamiento encadena por cercanía, van sembrando niveles falsos.

La señal que los separa limpiamente es la **proporción de acero**. El filtro es un
porcentaje de **metros de viga medidos contra el nivel más grande** del assembly (input
`11. Mínimo de un nivel T.A. (% del nivel más grande)`, default **40**), no una lista de
nombres de familia a excluir, que dependería de cómo bautice sus familias cada modelador.

Se mide contra el nivel más grande y **no contra el total** a propósito: así el umbral no
depende de cuántos niveles tenga el assembly. Un assembly con 4 niveles legítimos y
parecidos los conserva los 4 (cada uno ~100 % del mayor), cosa que un umbral sobre el total
haría imposible.

- `0` = sin filtro.
- Si **ningún** nivel alcanza el umbral, se conserva el mayor: un assembly nunca se queda
  sin planta T.A.
- Cada nivel descartado se **reporta en el log** con su cota, sus vigas y su porcentaje, así
  que si el filtro se come uno legítimo se ve de inmediato y basta bajar el número.

Resultado con el default de 40 % sobre el modelo real, contrastado contra lo que declara el
modelador:

| Assembly | Niveles detectados | Sobreviven | Cotas |
|---|---|---|---|
| ES-1001 | 2 | **1** | EL. 4,650 (el de EL. 3,450 queda en 29 %) |
| ES-1002 | 3 | **2** | EL. 6,150 y EL. 7,200 |
| ES-1003 | 5 | **1** | EL. 3,300 (los otros, entre 0 % y 1 %) |

#### ⚠️ El corte se recorta contra el nivel de arriba

El offset del corte es fijo (1,00 m) pero la separación entre niveles no. En **ES-1002** los
dos T.A. están a **1,05 m**, así que el corte de la planta 01 caía en EL. 7,15 — **por
encima** de las vigas secundarias del T.A. 02, que están en EL. 7,07 — y esa planta dibujaba
los dos niveles superpuestos.

Ahora el corte nunca alcanza al vecino: se queda **20 cm** por debajo del nivel de arriba
(`SEP_NIVEL`), y el fondo, 20 cm por encima del de abajo. Ambos recortes se avisan en el log.

> Validación con el modelo real: los niveles nativos del proyecto van de 7545.3 a 7579.8
> pies (≈ 2300.4 a 2310.9 m — el sitio está en el salar, a 2300 m). Varios están a menos
> de 0.5 m entre sí (`AMT_T.A. COLUMNAS` 2303.37 m, `AMT_T.A. PLATAFORMA` 2303.59 m,
> `RLO_T.A.2 ES-1001` 2303.71 m, `RLO_T.A. ES-1002` 2303.86 m): con tolerancia de 1 m
> colapsan en **una sola** planta T.A., que es exactamente el comportamiento buscado.

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

### C. Elevaciones de eje

Tantas como **ejes estructurales crucen la huella en planta** del assembly (test de
intersección segmento-rectángulo Liang-Barsky contra el bbox sólido del assembly).

Estas **no** son vistas de assembly: `AssemblyViewUtils.CreateDetailSection` solo admite
orientaciones fijas (`HorizontalDetail`, `DetailSection A`–`D`), que no pueden dar «una
elevación por cada eje». Se crean con `ViewSection.CreateSection` y una caja de sección
alineada al eje (BasisX = dirección del eje, BasisY = vertical, BasisZ = hacia el
observador). Como una sección normal muestra todo el modelo dentro de su caja, el
assembly se **aísla explícitamente**: `IsolateElementsTemporary` con todos los miembros
recursivos **más el propio eje** (para que su burbuja siga visible), y luego
`ConvertTemporaryHideIsolateToPermanent` — el aislamiento temporal no sobrevive al cierre
de la vista.

El aislamiento incluye **todos los ejes que cruzan el assembly**, no solo el propio: 03
enciende la categoría *Grids* en estas vistas y los ejes transversales tienen que poder
verse. Un eje que no entre en el aislamiento queda oculto para siempre y 03 no puede
recuperarlo encendiendo la categoría.

- Nombre interno: `{assembly} - EJE {nombre del eje}` (ej. `ES-1001 - EJE 13a`)
- Título en lámina: `{assembly} - ELEVACION EJE 13a`
- Se ocultan los símbolos de sección (`OST_Sections`) para que las elevaciones del mismo
  assembly no se crucen entre sí.

### Profundidad de vista

Plantas y elevaciones tienen **inputs separados**, porque son dos clases de vista con dos
mecanismos y dos necesidades distintas:

- **Plantas** (`ViewPlan`) → *View Range*, input `13. Profundidad de las plantas bajo el
  nivel (cm)`, default **50 cm**. Los cuatro planos se anclan al mismo nivel (por
  `ProjectElevation`, ver arriba) y se expresan como offset:
  - `Top` = `Cut` + 10 cm (`HOLGURA_TOP`, ver más abajo — nunca puede ser cero).
  - `Cut` = nivel definido **+** su offset (input `12.`, default 1,00 m).
  - `Bottom = View Depth` = nivel definido **− 0,50 m**.

  El fondo se mide **desde el nivel definido** (la cara más baja en la NIPB, el nivel
  dominante en las T.A.), **no** desde el plano de corte: el corte va un offset por encima
  del nivel y no tiene por qué arrastrar la profundidad. Así cada planta T.A. muestra su
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

Hay **dos campos de view template independientes**, uno por familia de vista (una planta y
una elevación de eje nunca comparten template real en Revit):

- **`06. View template - PLANTAS`**, default `DISP.GRAL_1/150_PLAN`. Se aplica a la NIPB y a
  todas las T.A.
- **`07. View template - ELEVACIONES DE EJE`**, default `ESTRUCTURAS_1/50`. Se aplica a las
  elevaciones por eje.

Ambos vienen pre-seteados así que el modelador no tiene que tocar nada para el caso normal,
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

Por assembly, en este orden: **planta NIPB → plantas T.A. ascendentes → elevaciones de
eje** (los ejes en orden natural, de modo que `2` va antes que `13` y `13a` después de
`13`). Las vistas fluyen de izquierda a derecha y de arriba hacia abajo; al llenarse la
lámina se sigue en la siguiente del mismo assembly.

Esto reemplaza el modelo de fundaciones de «bloque = planta arriba + 2 cortes debajo,
bloques en grilla», que no escala a un assembly con 1 + 4 + 8 vistas.

### Reserva para la tabla

La tabla de cantidades va **solo en la primera lámina de cada assembly**, así que la
reserva del lado derecho (input, default 150 mm) **se descuenta únicamente en esa
lámina**; las siguientes del mismo assembly aprovechan el ancho completo.

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

### En las elevaciones: solo los dos ejes extremos, recortados al alto del assembly

Una elevación solo dibuja los ejes **perpendiculares al papel** — el eje de la propia
elevación corre a lo largo de la vista y Revit no lo dibuja, así que queda fuera solo. De
los que sí se dibujan se conservan **los dos extremos** y se ocultan los de por medio
(input `Elevaciones: dejar solo los dos ejes extremos`, default True):

```
   (13a) (13b) (13c) (14a) (16) (18a)        <- lo que sale por defecto
     |     |     |     |    |     |
                                              (13a)              (18a)
   +---------------------------+      -->       |                  |
   |         ELEVACION         |                +------------------+
   +---------------------------+                |    ELEVACION     |
                                                +------------------+
```

La posición de cada eje ya está acotada en las plantas; repetirla en la elevación solo
tapa la estructura con líneas verticales. Los extremos se quedan porque son la referencia
del ancho de la estructura.

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
| Multi-Category Tags | 10, familia `Item_TBL` (los rótulos de perfil — los pone 05) |
| Detail Lines | 1 de 13,94 m con estilo `L-CENTER` (la línea de terreno) |

#### Los cinco planos de altura

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

Se sacan de la **geometría que la elevación muestra**, no de los niveles del modelo:
de los cinco, solo T.A. y N.I.P.B. tienen nivel, los otros tres son geometría y nada más.

- **tope de baranda** = lo más alto de la vista. Si no sobresale nada del T.A., no se
  dibuja esa cota.
- **T.A.** = lo más alto de los `Structural Framing`.
- **fondo de viga** = la cara inferior más baja de las vigas que llegan al T.A.
  («llegan» = su tope está a menos de 10 mm del T.A.: el acero se modela con
  contraflechas y tolerancias de milímetros).
- **P.T.** = tope de las **sillas de anclaje**. En la vista del modelador no hay
  hormigón: lo único que hay a 500 mm sobre la placa base son los `Structural
  Connections` que arrancan en ella, y su tope es el piso terminado. Si la base es
  placa sola, sin silla, no hay plano P.T. y la cadena queda de dos tramos.
- **N.I.P.B.** = lo más bajo de la vista (cara inferior de las placas base).

#### ⚠️ Se acota contra caras, y eso obliga a `ComputeReferences`

Una cota necesita un `Reference`, y las caras solo lo traen si la geometría se pide con
`Options.ComputeReferences = True`. Además hay que entrar a las instancias por
**`GetInstanceGeometry()`**, no por `GetSymbolGeometry()`: las referencias de la
geometría de símbolo son del *tipo*, no de la pieza, y Revit rechaza la cota.

De cada elemento candidato se toma la **cara horizontal más grande** a esa cota, con la
normal hacia el lado correcto (arriba para T.A./P.T., abajo para el fondo de viga y la
N.I.P.B.). Si el primer candidato no ofrece cara utilizable —una cartela, una plancha de
conexión o un perfil con corte en ángulo tienen el bbox a esa cota pero no la cara— se
prueba el siguiente. El log dice qué planos se resolvieron y cuáles quedaron sin cara.

Esto contradice a propósito la regla de «no depender de caras del acero» que rige para
las cadenas en planta: ahí había una alternativa (los ejes de viga `L-CENTER`), acá no
existe ninguna. Un plano que no se pueda resolver simplemente no se acota y queda en el
log; nunca se inventa una cota suelta.

#### Dónde va cada cosa

Se reusan los inputs de separación de las plantas, para no alargar el Player:

- **cadena 200/1450/500 y el tramo de baranda**, a la izquierda, a `separacion al borde
  − separacion entre cadenas` (12 mm de papel con los defaults). Van en la misma
  vertical, como en el plano-tipo.
- **total 2150**, a la izquierda del todo, a `separacion al borde` (20 mm).
- **marcas de nivel**, a la derecha con directriz horizontal: codo a media separación y
  texto a una separación y media.
- **línea de terreno**, a la cota N.I.P.B., sobresaliendo una separación a cada lado.
  Con los defaults (1:75, 20 mm) da 1,5 m por lado — los mismos ~14 m del plano-tipo.

La **sigla** de cada marca (`T.A.`, `P.T.`, `N.I.P.B.`) la pone el **tipo** de Spot
Elevation, no el script: son tres familias distintas ya cargadas en el proyecto. El input
lleva los tres nombres separados por coma **en ese orden**; vacío = no se ponen marcas.
Si un nombre no existe, el log lista los tipos disponibles y sigue con los otros dos.

> **Idempotencia**: si la elevación ya tiene alguna cota, no se re-cotan las alturas; si
> ya tiene alguna marca de nivel, no se ponen; si ya tiene una línea del estilo
> `L-CENTER`, no se redibuja la de terreno.

### Eje de cada viga en las plantas T.A. (línea roja `L-CENTER`)

En cada planta **T.A.** se dibuja, sobre el eje de cada viga, una **detail line** con el
estilo de línea `L-CENTER` (input; si el estilo no existe en el proyecto se crea **rojo** y
con el primer patrón de línea de eje que encuentre).

**Ni en la NIPB ni en las elevaciones de eje.** A la cota de la NIPB hay placas base,
pernos y sillas de anclaje —no vigas—, así que ahí el eje de viga no dice nada. Si una
corrida anterior las dibujó, 03 las **borra** al pasar por esa vista (y Revit se lleva de
paso la cadena que las referenciaba); queda registrado en el log.

**Qué se dibuja**: la *curva de ubicación* (`Location.Curve`) de cada `Structural Framing`,
proyectada al plano de la vista. Es el eje real de la viga, no el centro de su bounding box.
Las vigas curvas se teselan en una poligonal.

**Qué vigas entran en cada planta**: las que devuelve `FilteredElementCollector(doc,
view.Id)` **filtradas contra los miembros recursivos del assembly**. El colector por vista
respeta el *View Range* y el aislamiento permanente que dejó 00, así que cada T.A. recibe
exactamente los ejes de las vigas de **su** nivel. El filtro por miembros es el cinturón de
seguridad por si el aislamiento de una vista se perdiera.

**Por qué detail lines y no model lines**: una *model line* aparecería en las tres plantas
del assembly y en las elevaciones, y ensuciaría el modelo para todo el resto del proyecto.
La detail line vive solo en la vista donde se creó.

Detalles:

- Las vigas **verticales** (riostras, montantes) se saltan: en planta su eje es un punto.
  Se cuentan en el log.
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

### Cadena de cotas entre ejes estructurales (solo en planta)

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
quedan fuera de la cadena y se avisan en el log. El conjunto de ejes es exactamente el
mismo que usa 00 para decidir qué elevaciones crear (mismo test Liang-Barsky), así que
cadena y elevaciones nunca se contradicen.

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

## 05 — Rótulos de perfil en plantas

> **Estado: escrito pero no probado en Revit.**

Sobre cada viga de las plantas T.A. se escribe el **nombre de su tipo** (`C20x13,1`,
`IN20x35,2`, `L6,5x4,780`…), alineado con la viga y centrado en su eje. Es lo que el
modelador hace a mano con el tag `C-Multicat / Modelo`.

### ⚠️ Son TextNotes, no tags

Un `IndependentTag` muestra el *Type Name* y **la API no deja sobrescribir su texto por
instancia**, así que con tags de verdad no se puede truncar. La alternativa era editar la
familia `C-Multicat` para que su etiqueta leyera un parámetro compartido y escribirlo desde
el script; se descartó para no tocar familias.

Contrapartida asumida: un `TextNote` es texto tonto. No queda asociado a la viga y no se
actualiza si alguien le cambia el perfil. Hay que re-correr el graph.

### El acortado no es arbitrario: se corta en la `x`

Medido sobre el modelo el 2026-08-03: **no existe ningún tipo llamado `C20`, `IN20` ni
`L6,5`**. Los 14 tipos que empiezan así llevan todos el sufijo de peso. O sea que los
nombres cortos del plano son truncados hechos en la anotación, no perfiles distintos.

La regla que aplica el modelador es mecánica — tira el peso en kg/m y deja la designación:

| Completo | Corto |
|---|---|
| `C20x13,1` | `C20` |
| `IN20x35,2` | `IN20` |
| `L6,5x4,780` | `L6,5` |
| `L8x7,07` | `L8` |

> El corte es **insensible a mayúsculas**: en el modelo conviven `L8x7,07` y `L8X7,07`.

### El «¿cabe?» se mide, no se estima

Nada de calcular anchos de fuente. Se crea un `TextNote` de prueba por cada texto distinto
fuera del crop, se regenera, se lee su *bounding box* y se borra. Con eso el criterio queda
en un `if`:

```
disponible = largo de la viga en planta − 2 × margen        (input, default 2 mm de papel)
si ancho(nombre completo) > disponible  →  se usa el corto
```

Las mediciones se **cachean por (texto, escala)**, porque los nombres de tipo se repiten
decenas de veces por vista. Si la medición fallara, se cae a estimar `nº caracteres ×
altura × 0,6`.

**Si ni el corto entra, se pone igual**: vale más una viga rotulada de más que una sin
identificar. El log cuenta cuántas quedaron así, por si conviene resolverlas a mano.

### Tipo de texto

Input `03. Tipo de texto del rotulo`, default **`C_TBL_RomanD2.2mm`** (verificado que existe
en el proyecto, id 504711). Es editable desde Player, así que el modelador puede cambiarlo
sin abrir Dynamo.

> Si el nombre no coincide con ningún tipo, el graph **no escribe nada** y lo dice en el log
> junto con la lista de tipos disponibles. Deliberadamente no cae al primero de la lista:
> con un nombre mal escrito, rotular decenas de vigas en `C_TIT_RomanD5mm` (fuente de
> títulos) es peor que no rotular.

El proyecto tiene además `C_TBL_RomanD2.2mm_Negrita`, `C_TBL_RomanD3mm`,
`C_TXT_RomanD2.3mm`, `C_TXT_RomanD2.5mm` y `C_TIT_RomanD5mm`. La vista de referencia del
modelador usa `C_TXT_RomanD2.5mm` para el texto de "GRATING ARS-5".

### Orientación

El texto se rota con el eje de la viga y el ángulo se normaliza a (−90°, 90°] para que nunca
se lea de cabeza. Como el origen de un `TextNote` es el **tope** de la línea, se sube media
altura de texto para que quede centrado sobre el eje.

### Lo que NO hace

- **No evita colisiones** entre rótulos ni contra otras anotaciones. El modelador los acomoda
  a ojo; replicar eso es un problema aparte.
- **No rotula columnas**, solo `Structural Framing`.
- El grating queda fuera por el input `05. Familias a NO rotular` (default `GRATING`).

### ⚠️ Tipos duplicados en el modelo

Hay tipos que son el mismo perfil con nombres distintos: `L8x7,070`, `L8x7,07` y `L8X7,07`
conviven, igual que `C20x13,1` con `C20x13,10` y `L8x5,960` con `L8x5,96`. Vigas idénticas
van a salir rotuladas distinto según qué tipo les tocó. El truncado en la `x` disimula buena
parte, pero no todo — es basura del modelo, no del graph.

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
  las que va a recrear: el clustering puede devolver menos grupos que antes (o cruzar menos
  ejes), y las vistas sobrantes quedarían huérfanas con un correlativo que ya no aplica.
- `01`: si no hay vistas pendientes de colocar, no crea ninguna lámina. Los números de
  sheet ya usados en el proyecto se saltan.
- `02`: vistas ya colocadas se omiten; solo se usan láminas **sin ningún viewport** (un
  assembly no puede compartir lámina, así que no se rellenan láminas a medias).

## Mensajes de log frecuentes

- `no hay ningun Assembly en el modelo` (00) — falta el paso previo: crear los assemblies
  en Revit. Nada de este pipeline funciona sin ellos.
- `no existe la vista ... (corre 00_Vistas de assembly)` (01/02) — falta el paso 0.
- `no hay vigas (Structural Framing) con geometria` (00) — el assembly no tiene vigas
  identificables; se genera la NIPB y las elevaciones, pero ninguna planta T.A.
- `ningun eje cruza la huella del assembly` (00) — revisar que los ejes estén dibujados
  con extensión suficiente sobre la zona del assembly.
- `el eje X no es una linea recta` (00) — los ejes en arco no se soportan para elevaciones.
- `sin geometria solida valida en ningun miembro` (00) — geometría rota en el modelo
  (elemento con `Volume = 0`); hay que arreglarlo en Revit, no en el script.
- `SIN ESPACIO para N vistas` (02) — correr 01 para agregar las láminas que falten.
- `el tipo seleccionado NO es una viñeta` (01) — elegir del dropdown uno de los tipos que
  el propio log lista.

## Pendientes / próximos pasos

1. **Probar 05 en Revit** (rótulos de perfil), y decidir qué se hace con el viejo
   `05_Tags en plantas.dyn`: hoy hay dos graphs numerados 05 y en Player eso confunde.
   Además hay que arreglar en 00 que `Structural Connections` se enciende en las T.A.,
   donde no corresponde.
2. **Adaptar 06 (tabla).** Hoy **excluye** `Structural Framing` y `Generic Models` — justo
   la categoría que manda en acero. Además debe pasar de «una tabla por sheet» a **una
   tabla única por assembly, en su primera lámina** (regla del brief).
3. **`07_Cantidad en leyendas`** fue borrado del working tree (aparece como `D` en git) —
   decidir si se recupera para acero o se descarta.
4. **Verificar en Revit** todo lo de esta iteración: no se pudo probar porque el modelo no
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
5. **Orden de colocación configurable** en 02 (hoy alfabético por assembly).
