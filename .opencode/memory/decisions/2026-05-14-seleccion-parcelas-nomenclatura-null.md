# Decisión: Permitir selección de parcelas sin nomenclatura desde el mapa

**Fecha**: 2026-05-14
**Feature**: Seleccionar parcelas con `nomenc21 = null` desde el mapa en Alta Pura

## Decisión 1: Modificar solo el bloque `else` en `single_click.js`

**Contexto**: El bloque `else` (líneas 128-146) de `getParcelaAltaPura()` actualmente muestra un alert bloqueante y no permite continuar.

**Opciones evaluadas**:
1. ✅ **Modificar solo el bloque `else`** — Cambio mínimo, reutiliza funciones existentes (`configurarOpcionesAlta`, `setParcelaSeleccionadaAlta`, `sincronizarSeccionesNomenclatura`)
2. ❌ Crear función nueva en `single_click.js` — Innecesario, las funciones auxiliares ya existen
3. ❌ Modificar el backend — Ya soporta `parcela_gid`, no se necesita cambio

**Resultado**: Opción 1. Cambio localizado en un solo archivo, una sola función, un solo bloque.

## Decisión 2: Orden de operaciones — `configurarOpcionesAlta(true)` ANTES de `setParcelaSeleccionadaAlta(gid)`

**Contexto**: El handler de `tipo_nomenclatura` change (línea 753) verifica si `parcelaSeleccionada` es true para decidir si regenera la nomenclatura o solo sincroniza.

**Riesgo**: Si se setea el gid ANTES de llamar a `configurarOpcionesAlta(true)`, el handler verá `parcelaSeleccionada = true` y NO generará la nomenclatura provisoria (solo sincronizaría campos vacíos).

**Resultado**: Orden estricto: `configurarOpcionesAlta(true)` → `setParcelaSeleccionadaAlta(gid)`.

## Decisión 3: Toast no bloqueante en lugar de SwalAlertHtml

**Contexto**: `SwalAlertHtml` es un modal bloqueante. El nuevo flujo debe permitir al usuario continuar sin interacción adicional.

**Resultado**: Se usa `Swal.mixin({toast: true})` con timer de 5 segundos, consistente con otros toasts del sistema.

## Decisión 4: Formato de `datosPoligono`

**Contexto**: El código actual en el `else` hace `datosPoligono = data` (objeto completo), mientras que `dibujarPoligono()` espera `{ features: [...] }`.

**Resultado**: Se usa `{ "features": data.features }`, consistente con el bloque `if` (línea 95).