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
| 0 | `00_Vistas de assembly.dyn` | Por assembly: planta (`HorizontalDetail`) y **dos cortes perpendiculares que se cruzan al centro** (`DetailSectionA` + `DetailSectionB`, uno por cada eje del assembly), renombrados y con escala automática (ancho o largo > 3 m → 1:50, si no 1:20). El plano de corte de la planta se sube por encima del assembly (input en cm, default 30) para que nada aparezca cortado. En los cortes se ocultan los símbolos de corte (solo la planta los muestra) y se les aplica el tipo de vista del input "Tipo de vista para cortes" (default `02_FORMAS`, que define el símbolo de corte; si no existe en el proyecto, avisa con la lista de tipos disponibles y sigue con el default; vacío = no cambiar). En cada planta escribe el parámetro de vista `Cantidad_Inserto` = número de instancias de ese assembly en el proyecto (lo muestra el título `C_ConTitulo`; si el parámetro no existe en el proyecto, avisa una vez y sigue). Opcionalmente aplica un **view template** a todas las vistas (input con el nombre; vacío = ninguno). No existe un dropdown nativo de Dynamo para view templates sin paquetes, así que **si el campo queda vacío, el log siempre lista los templates cargados en el proyecto** (correr una vez para ver la lista, copiar el nombre exacto y correr de nuevo); si el nombre no coincide con ninguno, también avisa con la lista. El template no debe controlar la escala (rompería la escala automática) ni conviene que fuerce la visibilidad de la categoría Sections en los cortes (los dejaría de mostrar limpios). El **crop de la planta y el ancho del crop de los cortes se ajustan justo al borde del assembly** (input "Margen del crop, planta y cortes", default 5 mm de papel); esto también acorta la línea de sección que se dibuja en la planta, que antes salía mucho más larga que el assembly. El *Annotation Crop* se desactiva en las tres vistas para que cotas, cadenas y tags —que se colocan por fuera del borde— nunca queden cortados por este recorte. Re-correrlo también corrige plantas y cortes existentes sin recrearlos. |
| 1 | `01_Calcular y crear laminas.dyn` | **Automatiza la cantidad de láminas**: reúne las vistas de assembly aún no colocadas en ningún sheet, mide su tamaño real (ya recortado por el 00) y simula el mismo empaquetado en grilla que usa el 02 con una lámina de prueba (se crea, se mide su área útil con la viñeta elegida, y se borra) para calcular exactamente cuántas láminas hacen falta. Crea esa cantidad con la viñeta/numeración elegidas (dropdown de viñeta, número/nombre del primer sheet, incremento automático del resto). Reserva en el cálculo el mismo margen derecho que el 02 (ver nota abajo). |
| 2 | `02_Colocar vistas en laminas.dyn` | Coloca las vistas en los sheets destino: por cada assembly, **planta arriba y los dos cortes debajo**, centrados en el mismo eje vertical. Los bloques se reparten en grilla con márgenes respecto a la viñeta; cuando un sheet se llena, sigue en el siguiente. Si un bloque excede el área útil, se coloca apilado igual y el log avisa. Aplica el tipo de viewport con título correspondiente (plantas → `C_ConTitulo`, cortes → `C_Titulo_Seccion`) **creándolo si todavía no existe en el proyecto** (duplica uno existente, activa *Show Title* y asigna la familia de View Title). El título bajo cada vista lleva más separación en los **cortes** (24 mm) que en la **planta** (5 mm), porque los cortes muestran spot elevations por debajo del borde (con Annotation Crop desactivado) que de otro modo quedarían tapadas por el título. Re-correrlo corrige también el espaciado de cortes ya colocados. |
| 3 | `03_Cotas generales.dyn` | Planta: cotas de ancho y largo total (arriba e izquierda). Cortes A y B: cota de altura total + spot elevations superior e inferior en cada uno, **acotando solo las caras de hormigón** (input "Filtro material cortes", default `hormig`, coincidencia parcial con el nombre del material; vacío = acotar toda la geometría). |
| 4 | `04_Cotas de ejes.dyn` | Planta: cadena borde → eje de cada elemento → borde opuesto (usa los planos de referencia centrales de cada familia). |
| 5 | `05_Tags en plantas.dyn` | Multi-Category Tag **solo a las familias** (FamilyInstance) del assembly — nunca al assembly ni a elementos de sistema — y **solo en las plantas que el usuario seleccione con click** (viewports en la lámina, vistas o assemblies en el modelo). Se elige el tipo de tag en el dropdown; leader opcional. |

**Flujo**: 00 (crear vistas) → 01 (calcular y crear láminas) → 02 (colocar vistas) →
03/04 (cotas) → 05 (tags).

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

## Ajustes pendientes / ideas

- Tipo de cota y de spot elevation específicos del proyecto (hoy usa los tipos por defecto).
- Acotado en cadena también en el corte (hoy solo planta).
- Fallback geométrico en 04 para familias sin planos de referencia centrales.
- Orden de colocación en 02 configurable (hoy alfabético por assembly).
- **Tabla de cantidades en la lámina** (150x150mm, esquina superior derecha): el espacio ya
  está reservado desde el 01/02 (input "Reserva lado derecho", default 150mm), pero la tabla
  en sí todavía no se construye.
- Si se acumula desperdicio de espacio bajo la reserva del lado derecho (hoy es una franja
  completa a lo alto de la lámina, no solo la esquina), evaluar un recorte más preciso solo
  en la zona de la tabla.
