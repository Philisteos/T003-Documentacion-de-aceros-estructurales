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
| 3 | `03_Ejes y cotas entre ejes.dyn` | ✅ acero | Enciende los ejes, dibuja el eje de cada viga en planta (L-CENTER) y acota entre ejes |
| 4 | `04_Cotas de ejes.dyn` | ⚠️ fundaciones | Cadena borde → eje de **elemento** → borde — **sin adaptar** |
| 5 | `05_Tags en plantas.dyn` | ➖ neutro | Multi-Category Tag por selección manual; sirve igual en acero |
| 6 | `06_Tabla de assemblies.dyn` | ⚠️ fundaciones | Tabla por sheet — **sin adaptar** (ver *Pendientes*) |

**Flujo**: 00 (crear vistas) → 01 (calcular y crear láminas) → 02 (colocar vistas y
leyendas) → 03 (ejes y cotas entre ejes). Los pasos 04 y 06 todavía responden al modelo de
fundaciones; ver *Pendientes*.

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

### Eje de cada viga en planta (línea roja `L-CENTER`)

En **cada planta** (NIPB y T.A.) se dibuja, sobre el eje de cada viga, una **detail line**
con el estilo de línea `L-CENTER` (input; si el estilo no existe en el proyecto se crea
**rojo** y con el primer patrón de línea de eje que encuentre). Solo en plantas: las
elevaciones de eje no reciben nada.

**Qué se dibuja**: la *curva de ubicación* (`Location.Curve`) de cada `Structural Framing`,
proyectada al plano de la vista. Es el eje real de la viga, no el centro de su bounding box.
Las vigas curvas se teselan en una poligonal.

**Qué vigas entran en cada planta**: las que devuelve `FilteredElementCollector(doc,
view.Id)` **filtradas contra los miembros recursivos del assembly**. El colector por vista
respeta el *View Range* y el aislamiento permanente que dejó 00, así que cada planta recibe
exactamente los ejes de las vigas de **su** nivel — la NIPB normalmente no dibuja ninguno
(a esa cota hay placas base y pernos, no vigas) y cada T.A. dibuja solo las suyas. El filtro
por miembros es el cinturón de seguridad por si el aislamiento de una vista se perdiera.

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

### Cadena de cotas entre ejes (solo en planta)

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

### Manejo de fallas

Igual que 04: se engancha un handler a `FailuresProcessing` y se fuerza el commit **dentro
del script** (`ForceCloseTransaction`), porque Dynamo confirma su transacción después de
que el script terminó y el handler ya se des-suscribió. Las cotas que Revit rechaza por
referencias inválidas se borran solas en vez de abrir un diálogo bloqueante, y el script
verifica después del commit cuáles sobrevivieron.

**Idempotencia**: si una planta ya tiene una cota de más de un segmento, se salta entera
(las dos cadenas). Para rehacerlas hay que borrar las cotas a mano.

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

1. **Decidir qué hacer con 04.** Hoy acota el eje de cada *elemento* usando los planos de
   referencia centrales de la familia (lógica de fundaciones). En acero puede que no haga
   falta, o que deba pasar a acotar vigas/columnas contra los ejes estructurales.
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
