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
| 0 | `00_Auditoria de assemblies.dyn` | Diagnóstico: lista assemblies con tamaño, escala que les tocará y origen. Correr primero en cada proyecto nuevo. No modifica nada. |
| 1 | `01_Crear laminas.dyn` | Crea N sheets con la viñeta elegida. El primero con número/nombre manual; el resto incrementa el último bloque numérico conservando ceros (`...-006` → `...-007`). Salta números ya ocupados. |
| 2 | `02_Vistas de assembly.dyn` | Por assembly: planta (`HorizontalDetail`) y **dos cortes perpendiculares que se cruzan al centro** (`DetailSectionA` + `DetailSectionB`, uno por cada eje del assembly), renombrados y con escala automática (ancho o largo > 3 m → 1:50, si no 1:20). El plano de corte de la planta se sube por encima del assembly (input en cm, default 30) para que nada aparezca cortado. En los cortes se ocultan los símbolos de corte (solo la planta los muestra) y se les aplica el tipo de vista del input "Tipo de vista para cortes" (default `02_FORMAS`, que define el símbolo de corte; si no existe en el proyecto, avisa con la lista de tipos disponibles y sigue con el default; vacío = no cambiar). En cada planta escribe el parámetro de vista `Cantidad_Inserto` = número de instancias de ese assembly en el proyecto (lo muestra el título `C_ConTitulo`; si el parámetro no existe en el proyecto, avisa una vez y sigue). Opcionalmente aplica un **view template** a todas las vistas (input con el nombre; vacío = ninguno). No existe un dropdown nativo de Dynamo para view templates sin paquetes, así que **si el campo queda vacío, el log siempre lista los templates cargados en el proyecto** (correr una vez para ver la lista, copiar el nombre exacto y correr de nuevo); si el nombre no coincide con ninguno, también avisa con la lista. El template no debe controlar la escala (rompería la escala automática) ni conviene que fuerce la visibilidad de la categoría Sections en los cortes (los dejaría de mostrar limpios). El **crop de la planta y el ancho del crop de los cortes se ajustan justo al borde del assembly** (input "Margen del crop, planta y cortes", default 5 mm de papel); esto también acorta la línea de sección que se dibuja en la planta, que antes salía mucho más larga que el assembly. El *Annotation Crop* se desactiva en las tres vistas para que cotas, cadenas y tags —que se colocan por fuera del borde— nunca queden cortados por este recorte. Re-correrlo también corrige plantas y cortes existentes sin recrearlos. |
| 3 | `03_Colocar vistas en laminas.dyn` | Coloca las vistas en los sheets destino: por cada assembly, **planta arriba y los dos cortes debajo**, centrados en el mismo eje vertical. Los bloques se reparten en grilla con márgenes respecto a la viñeta; cuando un sheet se llena, sigue en el siguiente. Si un bloque excede el área útil, se coloca apilado igual y el log avisa. |
| 4 | `04_Cotas generales.dyn` | Planta: cotas de ancho y largo total (arriba e izquierda). Cortes A y B: cota de altura total + spot elevations superior e inferior en cada uno, **acotando solo las caras de hormigón** (input "Filtro material cortes", default `hormig`, coincidencia parcial con el nombre del material; vacío = acotar toda la geometría). |
| 5 | `05_Cotas de ejes.dyn` | Planta: cadena borde → eje de cada elemento → borde opuesto (usa los planos de referencia centrales de cada familia). |
| 6 | `06_Titulos de vistas.dyn` | Título bajo cada vista: crea el tipo de viewport con título si no existe (default `C_ConTitulo`), lo aplica a los viewports de los sheets destino y alinea el título justo debajo de cada vista con la línea al ancho de la vista. |
| 7 | `07_Tags en plantas.dyn` | Multi-Category Tag **solo a las familias** (FamilyInstance) del assembly — nunca al assembly ni a elementos de sistema — y **solo en las plantas que el usuario seleccione con click** (viewports en la lámina, vistas o assemblies en el modelo). Se elige el tipo de tag en el dropdown; leader opcional. |

Los graphs 04 y 05 pueden correrse en cualquier orden. Las cotas de la planta van **arriba
y a la izquierda**, en dos líneas: la cadena de ejes pegada al objeto (05, default 10 mm)
y la cota general por fuera (04, default 20 mm). Los defaults ya respetan ese orden; si se
cambian, mantener siempre la separación del 04 mayor que la del 05.

## Convenciones (contrato entre graphs)

- **Nombre de la planta = nombre del assembly**. Nombres de los cortes = `{assembly} - CORTE A`
  y `{assembly} - CORTE B` (perpendiculares entre sí, cruzados al centro del assembly).
  No renombrar vistas a mano entre pasos: 03, 04 y 05 buscan las vistas por estos nombres.
- Si hay **varias instancias del mismo tipo de assembly**, se documenta la primera.
- Distancias de anotación y márgenes se ingresan en **mm de papel** y se convierten con la
  escala de cada vista (mismo aspecto en 1:20 y 1:50).
- **Los cortes se identifican con letras** (A, B, C… y tras la Z: AA, AB…), únicas por
  lámina y asignadas en orden visual. El 03 escribe la letra en el *Detail Number* del
  viewport del corte — es lo que muestra el símbolo de corte en la planta — y el título en
  lámina queda `{assembly} - CORTE {letra}` vía *Title on Sheet*. Las plantas conservan
  números. El nombre interno de la vista sigue siendo `{assembly} - CORTE A` (no renombrar:
  es lo que usan 02/04/05 para encontrarla). Re-correr el 03 re-letra y re-titula los cortes
  ya colocados (los que ya tienen letra la conservan).
- **El título bajo cada vista lo controla el tipo de viewport**, no la vista. El 06 crea
  el tipo `C_ConTitulo` si no existe (con *Show Title = Yes*), lo aplica y alinea el título
  bajo el borde inferior izquierdo de cada vista. El 03 aplica tipos de viewport distintos
  por clase de vista: plantas → input "Tipo de viewport para plantas" (default `C_ConTitulo`),
  cortes → "Tipo de viewport para cortes" (default `C_Titulo_Seccion`); re-correrlo retrofitea
  el tipo de los cortes ya colocados. Ojo: el 06 aplica UN solo tipo a todos los viewports
  del sheet — usarlo solo como corrección gruesa, el 03 es quien diferencia por tipo.
  El título mostrado es el nombre de la vista (salvo que la vista tenga *Title on Sheet*
  definido).

## Idempotencia (qué pasa al re-correr)

- `01`: no repite números de sheet existentes (los salta y avisa).
- `02`: si la vista ya existe, la salta. Con "Recrear vistas existentes" = True la borra y
  rehace (esto también elimina su viewport si estaba colocada).
- `03`: vistas ya colocadas y sheets que ya tienen viewports se omiten.
- `04`: salta vistas que ya tienen cualquier cota.
- `05`: salta plantas que ya tienen una cota de más de un segmento (huella de la cadena).

## Mensajes de log frecuentes

- `no existe la vista ... (corre 02_Vistas de assembly)` — falta el paso 2 para ese assembly.
- `N elementos sin plano de referencia central en la familia` (05) — abrir la familia y
  marcar sus planos centrales como *Is Reference*: `Center (Left/Right)` /
  `Center (Front/Back)`. Se corrige una vez por familia.
- `sin caras perpendiculares para la cota de ...` (04) — los elementos están rotados
  respecto a los ejes del assembly o no exponen caras planas en esa dirección.
- `SIN ESPACIO para N vistas` (03) — crear más sheets con 01 y volver a correr 03
  (lo ya colocado se respeta).
- `el tipo seleccionado NO es una viñeta` (01) — elegir en el dropdown uno de los tipos
  que el mismo log lista como válidos.
- `el tipo seleccionado NO es un Multi-Category Tag` (07) — elegir en el dropdown uno de
  los tags multicategoría que el log lista; si no hay ninguno, cargar la familia al proyecto.
- `N elementos no admiten este tag` (07) — categorías que el Multi-Category Tag no
  etiqueta (p. ej. subcomponentes sin categoría taggeable); es informativo, no un error.

## Ajustes pendientes / ideas

- Tipo de cota y de spot elevation específicos del proyecto (hoy usa los tipos por defecto).
- Acotado en cadena también en el corte (hoy solo planta).
- Fallback geométrico en 05 para familias sin planos de referencia centrales.
- Orden de colocación en 03 configurable (hoy alfabético por assembly).
- **Automatizar el cálculo de cuántas láminas hacen falta** (iniciativa en curso, jul 2026):
  hoy 01 crea N sheets a mano y 03 reparte lo que quepa. La meta es invertir el orden —
  generar primero todas las vistas (02), medir cuánto espacio ocupan y recién ahí calcular
  cuántos sheets se necesitan para completar todo. Primer paso ya resuelto: el crop de la
  planta ajustado al borde del assembly (02), que es el dato de tamaño real a considerar.
