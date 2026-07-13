# Generador de planos de assemblies — Revit 2027

Pipeline de graphs de Dynamo (4.0, CPython3, **sin paquetes externos**) para documentar
assemblies de Revit: sheets, vistas de planta y corte, distribución en láminas y acotado,
según el estándar de referencia LDS25-099058-PL-330-006/007.

## Requisitos

- Revit 2027 (los .dyn están guardados en formato Dynamo 4.0).
- Assemblies ya creados en el proyecto (ej. `FN-1001`, `PD-1002`).
- Viñetas (title blocks) cargadas en el proyecto.
- Ningún paquete de Dynamo: todo es out-of-the-box + Python embebido.

**Primera vez en una máquina nueva**: abrir cada .dyn en Dynamo (no en Player), correr y
guardar. Eso registra los inputs para Dynamo Player. Después, todo se opera desde Player.

## Orden de ejecución

| # | Graph | Qué hace |
|---|-------|----------|
| 0 | `00_Vistas de assembly.dyn` | Por assembly: planta (`HorizontalDetail`) y **dos cortes perpendiculares que se cruzan al centro** (`DetailSectionA` + `DetailSectionB`, uno por cada eje del assembly), renombrados, con escala automática (ancho o largo > 3 m → 1:50, si no 1:20) y **Detail Level = Fine** al crearse (no se reaplica a vistas ya existentes). El plano de corte de la planta se sube por encima del assembly (input en cm, default 30) para que nada aparezca cortado. En los cortes se ocultan los símbolos de corte (solo la planta los muestra) y se les aplica el tipo de vista del input "Tipo de vista para cortes" (default `02_FORMAS`, que define el símbolo de corte; si no existe en el proyecto, avisa con la lista de tipos disponibles y sigue con el default; vacío = no cambiar). En cada planta escribe el parámetro de vista `Cantidad_Inserto` = número de instancias de ese assembly en el proyecto (lo muestra el título `C_ConTitulo`; si el parámetro no existe en el proyecto, avisa una vez y sigue). Opcionalmente aplica un **view template** a todas las vistas (input con el nombre; vacío = ninguno). No existe un dropdown nativo de Dynamo para view templates sin paquetes, así que **si el campo queda vacío, el log siempre lista los templates cargados en el proyecto** (correr una vez para ver la lista, copiar el nombre exacto y correr de nuevo); si el nombre no coincide con ninguno, también avisa con la lista. El template no debe controlar la escala (rompería la escala automática) ni conviene que fuerce la visibilidad de la categoría Sections en los cortes (los dejaría de mostrar limpios). **"Crop View" (geometría) se activa siempre** (nuevas vistas y ya existentes; distinto del *Annotation Crop*, que se desactiva a propósito más abajo) — sin esto la vista muestra su extensión completa sin recortar, mucho más grande que el `CropBox` calculado, desperdiciando espacio en la grilla de 01/02 aunque el crop esté bien ajustado. El rectángulo del crop **no se dibuja** (`Crop Region Visible` = False) para que no aparezca como línea en la lámina, pero el recorte de geometría sigue activo. El **crop de la planta (ancho y alto) y el crop de los cortes (ancho y alto) se ajustan justo al borde del assembly** (input "Margen del crop, planta y cortes", default 5 mm de papel); esto también acorta la línea de sección que se dibuja en la planta, que antes salía mucho más larga que el assembly. El *Annotation Crop* se desactiva en las tres vistas para que cotas, cadenas y tags —que se colocan por fuera del borde— nunca queden cortados por este recorte (por eso es seguro ceñir también el alto del crop de los cortes: la cota de altura y la spot elevation superior de 03 quedan igual de visibles). Re-correrlo también corrige plantas y cortes existentes sin recrearlos. **Importante (2026-07-13)**: hasta esta fecha el alto del crop de los cortes quedaba en el default de Revit (mucho mayor que la geometría real); eso inflaba la estimación de `View.CropBox` que usan 01 y 02 para armar la grilla de láminas, haciendo que el cálculo creyera que cada bloque ocupaba casi una lámina entera — la mayoría de las láminas quedaban casi vacías y el orden de colocación salía errático. Proyectos con sheets ya colocados **antes** de este fix deben limpiarlos (borrar los viewports de esos sheets, o los sheets mismos si 01 puede recrearlos) y volver a correr 00 → 01 → 02 desde cero, porque 02 omite los sheets que ya tienen algún viewport y no reposiciona lo ya colocado. **Además (mismo día)**: `local_size`, `world_z_range`, `ajustar_crop_planta` y `ajustar_crop_corte` ya no usan `Element.get_BoundingBox(None)` — ese método puede devolver un bounding box "fantasma" (basado en referencias/work planes) para un miembro con geometría vacía o degenerada (`Volume = 0` en el elemento; visto en un `C-Zapata` cuyo bbox fantasma daba 3.7 m de alto para una zapata de 0.4×0.4 m, dejando además la sección vacía en pantalla porque no hay sólido que dibujar). Ahora usan `member_solid_bbox()`, que mide solo sólidos reales (`Volume > 0`, incluidos símbolos anidados — mismo patrón que `collect_faces()` de 03). Si un miembro no tiene ningún sólido válido, se excluye del cálculo (no se inventa un tamaño) y el log avisa `sin geometria para ajustar...` con el nombre del assembly — es una señal de que ese elemento tiene geometría rota en el modelo y hay que revisarlo ahí, no en el script. |
| 1 | `01_Calcular y crear laminas.dyn` | **Automatiza la cantidad de láminas**: reúne las vistas de assembly aún no colocadas en ningún sheet, mide su tamaño real (ya recortado por el 00) y simula el mismo empaquetado en grilla que usa el 02 con una lámina de prueba (se crea, se mide su área útil con la viñeta elegida, y se borra) para calcular exactamente cuántas láminas hacen falta. Crea esa cantidad con la viñeta/numeración elegidas (dropdown de viñeta, número/nombre del primer sheet, incremento automático del resto). Reserva en el cálculo el mismo margen derecho que el 02 (ver nota abajo). La simulación de alto de bloque usa `gap_bloque` (margen general + 30 mm fijos), el mismo valor que aplica el 02 al colocar de verdad — si alguno de los dos graphs cambia ese extra fijo sin cambiar el otro, 01 calculará una cantidad de láminas que no alcanza (o sobra) para lo que 02 realmente coloca. |
| 2 | `02_Colocar vistas en laminas.dyn` | Coloca las vistas en los sheets destino: por cada assembly, **planta arriba y los dos cortes debajo**, centrados en el mismo eje vertical. Los bloques se reparten en grilla con márgenes respecto a la viñeta; cuando un sheet se llena, sigue en el siguiente. Si un bloque excede el área útil, se coloca apilado igual y el log avisa. Aplica el tipo de viewport con título correspondiente (plantas → `C_ConTitulo`, cortes → `C_Titulo_Seccion`) **creándolo si todavía no existe en el proyecto** (duplica uno existente, activa *Show Title* y asigna la familia de View Title). El título bajo cada vista lleva más separación en los **cortes** (24 mm) que en la **planta** (5 mm), porque los cortes muestran spot elevations por debajo del borde (con Annotation Crop desactivado) que de otro modo quedarían tapadas por el título. Re-correrlo corrige también el espaciado de cortes ya colocados. **La separación vertical DENTRO de cada bloque (planta → corte A → corte B) se ajusta con el tamaño real ya colocado** (`Viewport.GetBoxOutline()`, que sí incluye anotaciones que sobresalen del crop — cotas, spot elevations, el multitag de capa de cimiento de 03) en vez de `View.CropBox` (que solo mide la geometría recortada y subestimaba el alto real cuando había overhang, causando espaciados inconsistentes entre assemblies con distinta cantidad de anotaciones fuera del crop); cada vista se reposiciona con `SetBoxCenter` una vez medida. Esa separación interna usa el margen general del input (IN[3]) **más un extra fijo de 30 mm** (`gap_bloque`) — el margen general por sí solo (default 5 mm) queda muy angosto entre planta y cortes porque ambas vistas tienen el crop ajustado al borde del assembly, sin aire de por medio; este extra no afecta los márgenes del sheet ni el espacio entre bloques/leyendas en la grilla. **Nota**: el cálculo de grilla (ancho/alto de bloque para decidir filas y cuándo un sheet se llena) todavía usa la estimación de `View.CropBox`, no la medida real — un bloque puede seguir siendo más alto de lo estimado si tiene mucho overhang, con riesgo de invadir la fila siguiente o el borde del sheet; pendiente evaluar si hace falta. Además coloca **leyendas**: por cada sheet destino, busca vistas de tipo *Legend* cuyo nombre coincida (sin distinguir mayúsculas) con el nombre de **tipo de familia** (`Symbol.Name`, ej. "G35" — no el nombre de la familia, ej. "C-CapaCimiento") de algún miembro —a cualquier profundidad de anidamiento, cualquier categoría— de los assemblies ya colocados en ese sheet (de esta corrida o de una anterior), y las empaca en la **esquina inferior derecha** del área útil (de derecha a izquierda y de abajo hacia arriba, dirección opuesta a la grilla de planta+cortes). Si el mismo tipo aparece en varios assemblies del mismo sheet, su leyenda se coloca una sola vez. Como una *Legend* es UNA vista con UN solo `Cantidad_Inserto` (parámetro de vista — el número que muestra el título), colocar la misma vista en varios sheets hacía que 07 pisara la suma de una lámina con la de otra (ganaba la última procesada y todas mostraban ese número); por eso **cada sheet recibe su propio duplicado de la leyenda** (`{tipo} - {sheet}`, vía `Duplicate with Detailing`), con *Title on Sheet* = nombre del tipo para que el título mostrado no cambie, y con `Cantidad_Inserto` reseteado a 0 al crearse (el duplicado hereda el valor de la original — posiblemente viejo — y ese número colgado confunde; escribir el valor real es responsabilidad exclusiva de 07). Idempotencia por sheet: se salta si el sheet ya tiene su duplicado; si el sheet tiene colocada la leyenda ORIGINAL (de una corrida anterior a este esquema), se reemplaza por el duplicado conservando la posición. Aplica el tipo de viewport **`C_Titulo_General`** a las leyendas (estándar de oficina, fijo — no es input de Player, reutiliza la misma búsqueda/bootstrap que plantas y cortes: si no existe lo crea duplicando uno existente). Requiere un `doc.Regenerate()` antes de matchear (los viewports recién colocados en la misma corrida no aparecen en `GetAllViewports()` sin esto). Si ninguna leyenda coincide, el log lista las leyendas cargadas y los nombres de tipo encontrados, para comparar. El tamaño de cada leyenda **no** se mide con el crop de la vista (`View.CropBox` no es confiable para *Legend*, puede devolver un valor por defecto enorme) — se coloca primero en un punto provisorio, se le aplica el tipo de viewport, se mide el tamaño real ya colocado (`Viewport.GetBoxOutline()`, después de aplicar el tipo por si agrega un título) y recién ahí se reposiciona a su lugar final (`Viewport.SetBoxCenter`). |
| 3 | `03_Cotas generales.dyn` | Planta: cotas de ancho y largo total (arriba e izquierda). Cortes A y B: cota de altura **en cadena, un segmento por cada elemento Structural Foundation** (del borde inferior absoluto a la cara propia más alta de cada elemento, en orden ascendente; con un solo elemento da una sola cota, igual que antes) + **spot elevation superior** (la inferior ya no se coloca), **acotando solo los miembros de categoría Structural Foundation** (filtro por categoría del elemento dueño de la cara; si un assembly no tiene miembros de esa categoría con caras referenciables, se **omite** la cota de altura y la spot elevation de ese corte y el log avisa — no se cae a usar toda la geometría, porque pernos/planchas/conexiones dan referencias poco confiables que Revit rechaza para Spot Elevation, o que fallan recién al comitear la transacción con el diálogo "Deleted element: id = -1"). El input "Filtro material cortes" (IN[3]) quedó sin uso — el filtro por nombre de material no era confiable porque los materiales reales no siempre contienen texto reconocible (ej. "G35" en vez de "hormigón"). Además, en **ambos cortes**, etiqueta con un **Multi-Category Tag** (familia `C-MultiCat`, tipo `Descripcion+Comentario`) el elemento **real** más bajo de todo el assembly (recursivo, a cualquier profundidad de anidamiento) **solo si ese elemento pertenece a la familia `C-CapaCimiento`** — no todos los assemblies tienen esa capa, y en ese caso simplemente no se taguea nada. Este tag se evalúa independiente de si el corte ya tenía cotas de una corrida anterior (no se salta junto con la cota), y no se duplica si el corte ya tiene un Multi-Category Tag puesto. |
| 4 | `04_Cotas de ejes.dyn` | Planta: cadena borde → eje de cada elemento → borde opuesto (usa los planos de referencia centrales de cada familia). |
| 5 | `05_Tags en plantas.dyn` | Multi-Category Tag **solo a las familias** (FamilyInstance) del assembly — nunca al assembly ni a elementos de sistema — y **solo en las plantas que el usuario seleccione con click** (viewports en la lámina, vistas o assemblies en el modelo). Se elige el tipo de tag en el dropdown; leader opcional. |
| 6 | `06_Tabla de assemblies.dyn` | Crea, en cada sheet destino, la **tabla de cantidades** de los assemblies documentados en ESE sheet: schedule nativo **Multi-Categoría** (no *Assemblies*: esa categoría no permite desglosar familias miembro), con el **view template** del input "View template para las tablas" (default `TABLA_TIPO`; si no existe, avisa con la lista de templates de schedule disponibles y sigue sin template). Agrupado por assembly (encabezado con su nombre y su "CANT=N") y, debajo, una fila por cada **tipo de familia miembro** (recursivo, incluye anidadas; se excluyen *Structural Framing* y *Generic Models*, este último por filtro de trabajo de la oficina) con columnas **DESCRIPCION / UNIDAD / UNITARIO / TOTAL / MATERIAL / COMENTARIO**. Reglas: *Structural Foundation* → UNIDAD=m3, UNITARIO=suma del **volumen real de la geometría** del elemento (sólidos, incluidas familias anidadas — ya no depende del parámetro `VOLUMEN_TBL` que llenaban a mano los modeladores); cualquier otra categoría → UNIDAD=un, UNITARIO=cantidad de instancias. TOTAL = UNITARIO × CANT, precalculado en Python y escrito como texto **en el parámetro compartido `Status Vendor`** (columna TOTAL, y también la fuente que lee el graph 07 para las leyendas). MATERIAL se lee del parámetro compartido `MATERIAL` (por GUID, para no confundirlo con el `Material` nativo); COMENTARIO lee el `Type Comments` de la familia (nunca se escribe). El input "Título de la tabla" se aplica a todas las tablas creadas (vacío = título en blanco). Para el nombre del assembly y el "CANT=N" se reutilizan los parámetros `TAG` y `Part Number` (fuera del esquema `_TBL`, sin fórmula asociada) en vez de `Mark`/`ELEMENTO_CANT_STR_TBL`, porque en varias familias esos campos quedan bloqueados por fórmula con valores viejos; `Comments` guarda el número de sheet (filtro), con una limpieza global que también alcanza miembros huérfanos de assemblies ya borrados. Solo se recorren los miembros de la primera instancia de cada tipo. Se coloca en la esquina superior derecha, en la franja reservada por 01/02. |
| 7 | `07_Cantidad en leyendas.dyn` | **Debe correrse DESPUÉS de 06.** Por cada leyenda ya colocada (02) en un sheet destino, escribe en su parámetro de vista `Cantidad_Inserto` la **suma del TOTAL** (la misma columna TOTAL de 06, leída del parámetro compartido `Status Vendor`, texto con coma decimal) de ese **tipo de familia**, sumada **solo entre los assemblies colocados en ese mismo sheet** (no todo el proyecto) — mismo criterio de match que usa 02 para decidir qué leyenda corresponde a qué sheet (tipo de familia, `Symbol.Name`, recursivo, cualquier categoría excepto *Structural Framing*/*Generic Models*, mismo filtro que 06). Las leyendas colocadas por 02 son duplicados por lámina (`{tipo} - {sheet}`): 07 les quita el sufijo para matchear el tipo y escribe la suma de ESA lámina en SU duplicado, así cada lámina muestra su propia cantidad (coincidente con la tabla de 06 de ese sheet). Si una leyenda no matchea ningún tipo ya procesado por 06 (porque 06 todavía no corrió, o el sheet no tiene ese tipo), avisa en el log y no le toca el parámetro. Re-correrlo siempre actualiza el valor (no se salta si ya está puesto). Si `Cantidad_Inserto` es un parámetro de texto (String), el valor se formatea con la misma convención que la columna TOTAL de 06 (coma decimal, enteros sin decimales) — antes se escribía con `str()` de Python tal cual, mostrando por ejemplo "12.0" en vez de "12" (visto en la cantidad de pernos). |

**Flujo**: 00 (crear vistas) → 01 (calcular y crear láminas) → 02 (colocar vistas, incluidas leyendas) →
03/04 (cotas) → 05 (tags) → 06 (tabla de assemblies por sheet) → 07 (cantidad en leyendas, después de 06).

Los graphs 03 y 04 pueden correrse en cualquier orden. Las cotas de la planta van **arriba
y a la izquierda**, en dos líneas: la cadena de ejes pegada al objeto (04, default 10 mm)
y la cota general por fuera (03, default 20 mm). Los defaults ya respetan ese orden; si se
cambian, mantener siempre la separación del 03 mayor que la del 04.

**Importante**: el input "Reserva lado derecho (mm)" del 01 (calculador) y del 02 (placer)
debe ser el mismo valor en ambos (default 150 mm — ya reserva espacio para la futura tabla
de cantidades de 150x150mm en la esquina superior derecha, aunque esa tabla todavía no se
construye). Si cambias ese valor en uno, cámbialo también en el otro; si no coinciden, el
número de láminas que calculó el 01 puede no alcanzar para lo que realmente coloque el 02.

## Convenciones (contrato entre graphs)

- **Nombre de la planta = nombre del assembly**. Nombres de los cortes = `{assembly} - CORTE A`
  y `{assembly} - CORTE B` (perpendiculares entre sí, cruzados al centro del assembly).
  No renombrar vistas a mano entre pasos: 01, 02, 03 y 04 buscan las vistas por estos nombres.
- Si hay **varias instancias del mismo tipo de assembly**, se documenta la primera.
- Distancias de anotación y márgenes se ingresan en **mm de papel** y se convierten con la
  escala de cada vista (mismo aspecto en 1:20 y 1:50).
- **Los cortes se identifican con letras** (A, B, C… y tras la Z: AA, AB…), únicas por
  lámina y asignadas en orden visual. El 02 escribe la letra en el *Detail Number* del
  viewport del corte — es lo que muestra el símbolo de corte en la planta — y el título en
  lámina queda `{assembly} - CORTE {letra}` vía *Title on Sheet*. Las plantas conservan
  números. El nombre interno de la vista sigue siendo `{assembly} - CORTE A` (no renombrar:
  es lo que usan 00/03/04 para encontrarla). Re-correr el 02 re-letra y re-titula los cortes
  ya colocados (los que ya tienen letra la conservan).
- **El título bajo cada vista lo controla el tipo de viewport**, no la vista. El 02 aplica
  tipos de viewport distintos por clase de vista: plantas → input "Tipo de viewport para
  plantas" (default `C_ConTitulo`), cortes → "Tipo de viewport para cortes" (default
  `C_Titulo_Seccion`); si el tipo indicado no existe todavía en el proyecto, lo crea
  (duplica uno existente, activa *Show Title = Yes* y asigna la familia de View Title —
  si no hay ninguna familia de ese tipo cargada, avisa: la línea aparece pero no el texto).
  Re-correrlo retrofitea el tipo y el espaciado de los cortes ya colocados. El título
  mostrado es el nombre de la vista (salvo que la vista tenga *Title on Sheet* definido).

## Idempotencia (qué pasa al re-correr)

- `00`: si la vista ya existe, la salta. Con "Recrear vistas existentes" = True la borra y
  rehace (esto también elimina su viewport si estaba colocada).
- `01`: si no hay vistas pendientes de colocar, no crea ninguna lámina. Los números de sheet
  ya usados en el proyecto se saltan.
- `02`: vistas ya colocadas y sheets que ya tienen viewports se omiten.
- `03`: salta vistas que ya tienen cualquier cota.
- `04`: salta plantas que ya tienen una cota de más de un segmento (huella de la cadena).
- `06`: salta sheets que ya tienen la tabla colocada. El `Comments` de los assemblies se
  vuelve a escribir en cada corrida (barato, mantiene la asignación al día si algo se
  movió de sheet).

## Mensajes de log frecuentes

- `no existe la vista ... (corre 00_Vistas de assembly)` — falta el paso 0 para ese assembly.
- `N elementos sin plano de referencia central en la familia` (04) — abrir la familia y
  marcar sus planos centrales como *Is Reference*: `Center (Left/Right)` /
  `Center (Front/Back)`. Se corrige una vez por familia.
- `sin caras perpendiculares para la cota de ...` (03) — los elementos están rotados
  respecto a los ejes del assembly o no exponen caras planas en esa dirección.
- `SIN ESPACIO para N vistas` (02) — correr 01 para agregar las láminas que falten y volver
  a correr 02 (lo ya colocado se respeta).
- `el tipo seleccionado NO es una viñeta` (01) — elegir en el dropdown uno de los tipos
  que el mismo log lista como válidos.
- `el tipo seleccionado NO es un Multi-Category Tag` (05) — elegir en el dropdown uno de
  los tags multicategoría que el log lista; si no hay ninguno, cargar la familia al proyecto.
- `N elementos no admiten este tag` (05) — categorías que el Multi-Category Tag no
  etiqueta (p. ej. subcomponentes sin categoría taggeable); es informativo, no un error.
- `Tipo de viewport {nombre} creado (duplicado de ...)` (02) — bootstrap: el tipo de
  viewport con título no existía en el proyecto, se creó automáticamente. Revisar su
  grafismo (fuente, tamaño de texto) en las propiedades de tipo si no coincide con el
  estándar del proyecto.
- `no se encontro el estilo de linea / tipo de texto ...` (06) — el nombre del input no
  coincide con ningún estilo/tipo de texto cargado en el proyecto; el schedule se crea
  igual con el default de Revit. Corrige el nombre del input y vuelve a correr (re-corre
  también actualiza el grafismo de las tablas ya creadas).
- `no se encontraron los campos [...] en la categoria Assemblies` (06) — el nombre exacto
  de un campo del schedule (por defecto se busca "Assembly Name" y "Count") no existe en
  este Revit; el log lista los campos disponibles para la categoría *Assemblies* para
  elegir el nombre correcto.

## Ajustes pendientes / ideas

- Tipo de cota y de spot elevation específicos del proyecto (hoy usa los tipos por defecto).
- Acotado en cadena también en el corte (hoy solo planta).
- Fallback geométrico en 04 para familias sin planos de referencia centrales.
- Orden de colocación en 02 configurable (hoy alfabético por assembly).
- **Tabla de assemblies por sheet** (06): construida como lista simple (assembly + cantidad).
  Si más adelante se necesita el desglose de materiales/conectores (DESCRIPCION/UNIDAD/
  UNITARIO/TOTAL/MATERIAL por elemento) que se había explorado antes, es una tabla aparte.
- Los nombres de campo del schedule ("Assembly Name", "Count") y de los estilos gráficos
  ("Líneas Tablas Finas", "C_TXT_RomanD2.3mm...") son los inputs por defecto del 06; si el
  Revit del usuario los expone con otro nombre exacto, el log lo avisa y basta con corregir
  el input (sin tocar el graph).
- Si se acumula desperdicio de espacio bajo la reserva del lado derecho (hoy es una franja
  completa a lo alto de la lámina, no solo la esquina), evaluar un recorte más preciso solo
  en la zona de la tabla.
