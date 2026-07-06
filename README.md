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
| 2 | `02_Vistas de assembly.dyn` | Por assembly: planta (`HorizontalDetail`) y corte transversal (`DetailSectionA`), renombrados y con escala automática (ancho o largo > 3 m → 1:50, si no 1:20). El plano de corte de la planta se sube por encima del assembly (input en cm, default 30) para que nada aparezca cortado — vista en proyección como una planta real. Re-correrlo también corrige plantas existentes sin recrearlas. |
| 3 | `03_Colocar vistas en laminas.dyn` | Coloca las vistas en los sheets destino: por cada assembly, **planta arriba y corte abajo**, centrados en el mismo eje vertical. Los bloques se reparten en grilla con márgenes respecto a la viñeta; cuando un sheet se llena, sigue en el siguiente. Si un bloque excede el área útil, se coloca apilado igual y el log avisa. |
| 4 | `04_Cotas generales.dyn` | Planta: cotas de ancho y largo total. Corte: cota de altura total + spot elevations superior e inferior. |
| 5 | `05_Cotas de ejes.dyn` | Planta: cadena borde → eje de cada elemento → borde opuesto (usa los planos de referencia centrales de cada familia). |
| 6 | `06_Titulos de vistas.dyn` | Título bajo cada vista: crea el tipo de viewport con título si no existe (default `C_ConTitulo`), lo aplica a los viewports de los sheets destino y alinea el título justo debajo de cada vista con la línea al ancho de la vista. |
| 7 | `07_Tags en plantas.dyn` | Multi-Category Tag **solo a las familias** (FamilyInstance) del assembly — nunca al assembly ni a elementos de sistema — y **solo en las plantas que el usuario seleccione con click** (viewports en la lámina, vistas o assemblies en el modelo). Se elige el tipo de tag en el dropdown; leader opcional. |

Los graphs 04 y 05 pueden correrse en cualquier orden. Para el layout clásico
(cadena pegada al assembly, total por fuera): 05 con 10 mm y 04 con 20 mm de separación.

## Convenciones (contrato entre graphs)

- **Nombre de la planta = nombre del assembly**. Nombre del corte = `{assembly} - CORTE A`.
  No renombrar vistas a mano entre pasos: 03, 04 y 05 buscan las vistas por estos nombres.
- Si hay **varias instancias del mismo tipo de assembly**, se documenta la primera.
- Distancias de anotación y márgenes se ingresan en **mm de papel** y se convierten con la
  escala de cada vista (mismo aspecto en 1:20 y 1:50).
- **Título del corte en lámina** = `{assembly} - {número de detalle}` (el mismo número que
  muestra el símbolo de corte en la planta). El 03 lo escribe en *Title on Sheet* al colocar
  el viewport; el nombre interno de la vista sigue siendo `{assembly} - CORTE A` (no
  renombrar: es lo que usan 02/04/05 para encontrarla). Si se renumeran los detalles a mano,
  re-correr el 03 actualiza los títulos de los cortes ya colocados.
- **El título bajo cada vista lo controla el tipo de viewport**, no la vista. El 06 crea
  el tipo `C_ConTitulo` si no existe (con *Show Title = Yes*), lo aplica y alinea el título
  bajo el borde inferior izquierdo de cada vista. El 03 usa ese mismo tipo (su input
  "Tipo de viewport con titulo") para que los viewports nuevos salgan con título de fábrica.
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
