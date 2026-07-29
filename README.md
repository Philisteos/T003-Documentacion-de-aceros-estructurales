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
| 3 | `03_Ejes y cotas entre ejes.dyn` | ✅ acero | Enciende los ejes en plantas y elevaciones, y acota entre ejes en planta |
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

1. Se extrae la elevación Z de la **cara superior** de cada viga (`Structural Framing`,
   recursivo a cualquier profundidad de anidamiento) del assembly.
2. Se ordenan ascendente y se agrupan con **tolerancia de ±1.00 m** (input, en cm).
3. Por grupo se calcula el **nivel dominante**: la elevación con más vigas (moda en
   cubetas de 5 mm). Empate → gana la cubeta más alta.
4. Se genera una planta por grupo, con el plano de corte **+1.00 m** (input) por encima
   del nivel dominante, para capturar las vigas desfasadas dentro del rango.
5. La profundidad la fija el View Range (ver más abajo).

**Detalle importante del agrupamiento**: la amplitud se mide contra el **inicio** del
grupo, no contra la viga anterior. Encadenar por la viga anterior permitiría que una
escalera de vigas separadas 0.9 m cada una terminara en un solo grupo de decenas de
metros; con este criterio cada grupo mide como máximo la tolerancia.

- Nombre interno: `{assembly} - T.A. 01`, `02`… (correlativo **ascendente por altura**)
- Título en lámina: `{assembly} - PLANTA T.A. (EL. 2.303,37)` — cota real en metros, con
  punto de miles y coma decimal, igual que las vistas que ya existen en el modelo
  (`PLANTA ESTRUCTURA EL. 2.305,25 T.A.`)

El correlativo es el nombre estable que usan los pasos siguientes; la cota va solo en el
título mostrado, porque si un modelador mueve una viga y el cluster se desplaza, un
nombre basado en la cota rompería las búsquedas por nombre.

> Validación con el modelo real: los niveles nativos del proyecto van de 7545.3 a 7579.8
> pies (≈ 2300.4 a 2310.9 m — el sitio está en el salar, a 2300 m). Varios están a menos
> de 0.5 m entre sí (`AMT_T.A. COLUMNAS` 2303.37 m, `AMT_T.A. PLATAFORMA` 2303.59 m,
> `RLO_T.A.2 ES-1001` 2303.71 m, `RLO_T.A. ES-1002` 2303.86 m): con tolerancia de 1 m
> colapsan en **una sola** planta T.A., que es exactamente el comportamiento buscado.

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

- **Plantas** (`ViewPlan`) → *View Range*, input **«Profundidad de las PLANTAS bajo el
  nivel (cm)»**, default **100 cm**. Los cuatro planos se anclan al mismo nivel (por
  `ProjectElevation`, ver arriba) y se expresan como offset:
  - `Top = Cut` = nivel definido **+** su offset (1,00 m) → no se ve nada por encima.
  - `Bottom = View Depth` = nivel definido **−** 1,00 m.

  El fondo se mide **desde el nivel definido** (la cara más baja en la NIPB, el nivel
  dominante en las T.A.), **no** desde el plano de corte: el corte va un offset por encima
  del nivel y no tiene por qué arrastrar la profundidad. Así cada planta T.A. muestra su
  nivel y no los de abajo — con los niveles del modelo separados 1,0–2,3 m, una profundidad
  mayor haría que cada planta arrastrara 2 o 3 niveles inferiores.

- **Elevaciones de eje** (`ViewSection`) → *Far Clip Offset*, input **«Far Clip Offset de
  las ELEVACIONES (mm)»**, default **5500 mm**. Se escribe después del view template, así
  que si el template trae su propio valor gana el input. Si el modo de recorte lejano
  estuviera en *No clip*, se activa primero (`VIEWER_BOUND_ACTIVE_FAR`). Aquí un valor
  generoso es inocuo: la vista está aislada y solo muestra el assembly y sus ejes.

### Escala y view template

**Una sola escala para todo** (input, default **1:75**), plantas y elevaciones. Reemplaza
la escala automática 1:25/1:50 de fundaciones, que no aplica a estructuras de este tamaño.

El input de view template va **vacío por defecto**: 00 no fuerza ningún template y cada
vista se queda con el que traiga por defecto su *ViewFamilyType* (así es como las vistas
creadas con el tipo `02_ESTRUCTURAS` reciben `ESTRUCTURAS_1/50`). Si se escribe un nombre,
ese template se aplica a todas las vistas y la escala se **re-escribe después**, así que
aunque el template fije una escala gana el input. Con el campo vacío el log lista todos los
templates cargados — es la forma de descubrir el nombre exacto, porque sin paquetes no hay
dropdown nativo de view templates en Dynamo.

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
