# 📋 Spec: Seleccionar parcelas sin nomenclatura desde el mapa en Alta Pura

## Objetivo

Permitir que los usuarios seleccionen parcelas con `nomenc21 = null` desde el mapa (click en OpenLayers) dentro del modal "Alta Pura". Actualmente, al hacer click en una parcela sin nomenclatura, el sistema muestra un alert bloqueante "NO POSEE NOMENCLATURA" y no permite continuar con el alta. El cambio debe habilitar la selección completa de estas parcelas, configurando automáticamente el modal como parcela informal con nomenclatura provisoria, mientras que la búsqueda por texto (`#nomenclatura_busqueda_alta`) sigue sin encontrar parcelas sin nomenclatura (comportamiento natural, ya que no hay nomenclatura para buscar).

---

## Análisis del codebase existente

### Archivos relevantes

| Archivo | Rol | Se modifica? |
|---------|-----|-------------|
| `public/js/cartografia_js/single_click.js` | Contiene `getParcelaAltaPura(evt)` — función que maneja el click en mapa cuando `moduloActivado == "ALTA_PURA"` | **SÍ** (solo líneas 128-146, bloque `else`) |
| `resources/views/gestion/padron/index.blade.php` | Contiene el modal Alta Pura, funciones JS: `configurarOpcionesAlta()`, `setParcelaSeleccionadaAlta()`, `sincronizarSeccionesNomenclatura()`, handler de `tipo_nomenclatura`, `generarAlta()`, etc. | NO |
| `public/js/cartografia_js/funciones_generales.js` | `dibujarPoligono()`, `SwalAlertHtml()`, `getParcelaPorParametro()` | NO |
| `app/Http/Controllers/ParcelaController.php` | `altapura()` — ya soporta `parcela_gid` y actualiza `nomenc21` en `parcelas_pos` | NO |

### Flujo actual — click en parcela CON nomenclatura (`nomenc21 != null`)

1. Click en mapa → `getParcelaAltaPura(evt)`
2. `setParcelaSeleccionadaAlta("")` + `generarHabilitado = false` (reset)
3. GetFeatureInfo → obtiene `gid` y `nomenc21`
4. `setParcelaSeleccionadaAlta(gid)` + `generarHabilitado = true`
5. Detecta tipo de nomenclatura (provisoria/posicional/antigua) → setea `tipo_nomenclatura`
6. Setea `#nomenclatura` y `#nomenclatura_busqueda_alta` con la nomenclatura formateada
7. `sincronizarSeccionesNomenclatura()` → parsea distrito/sección/etc.
8. Muestra `btn-generar`, dibuja polígono

### Flujo actual — click en parcela SIN nomenclatura (`nomenc21 == null`) — BLOQUE A MODIFICAR

1. Click en mapa → `getParcelaAltaPura(evt)`
2. `setParcelaSeleccionadaAlta("")` + `generarHabilitado = false` (reset)
3. GetFeatureInfo → obtiene `gid` pero `nomenc21 == null`
4. **PROBLEMA**: Entra al `else` (línea 128) que:
   - Muestra `SwalAlertHtml` bloqueante con "NO POSEE NOMENCLATURA"
   - Setea `datosPoligono = data` (formato incorrecto para `dibujarPoligono`)
   - Limpia `#nomenclatura_busqueda_alta` y `#nomenclatura`
   - Llama a `sincronizarSeccionesNomenclatura()`
   - Muestra `btn-generar` (pero `generarHabilitado` sigue `false`)
   - Llama a `dibujarPoligono(datosPoligono)` con formato incorrecto

### Flujo existente para parcelas informales (vía `verificarParcela`)

Cuando `verificarParcela()` detecta `datos.informal && datos.wkt`:
1. Llama `configurarOpcionesAlta(true)` → tipo_parcela=6, tipo_nomenclatura=2 (provisoria, readonly)
2. El handler de `tipo_nomenclatura` change se dispara → como `parcelaSeleccionada` es false (gid aún no seteado), genera nomenclatura provisoria con `arregloNomenclaProvisoria`
3. Luego `setParcelaSeleccionadaAlta(datos.gid)` marca la parcela como seleccionada

### Handler de `tipo_nomenclatura` change (línea 753) — comportamiento clave

```javascript
$("#tipo_nomenclatura").on("change", function() {
    const esProvisoria = $(this).val() == 2;
    const parcelaSeleccionada = ($("#parcela_gid").val() || "").trim() !== "";

    $nomenclatura.prop("readonly", esProvisoria);

    if (parcelaSeleccionada) {
        // Solo sincroniza, NO regenera la nomenclatura
        sincronizarSeccionesNomenclatura();
        return;
    }

    // Si NO hay parcela seleccionada, genera nomenclatura según tipo
    if ($(this).val() == 2) {
        $nomenclatura.val(arregloNomenclaProvisoria.join("-"));
    }
    // ... limpia campos, sincroniza
});
```

**Orden crítico**: Si se setea el gid ANTES de llamar a `configurarOpcionesAlta(true)`, el handler verá `parcelaSeleccionada = true` y NO generará la nomenclatura provisoria. Por lo tanto, `configurarOpcionesAlta(true)` debe ejecutarse ANTES de `setParcelaSeleccionadaAlta(gid)`.

### Backend `altapura()` — ya soporta `parcela_gid`

Líneas 1205-1217: Valida que `parcela_gid` sea un GID válido en `parcelas_pos`.
Líneas 1309-1313: Si `parcela_gid` existe, actualiza `nomenc21` en `parcelas_pos` con la nomenclatura generada.

---

## Decisiones de diseño

### Decisión 1: Modificar solo el bloque `else` en `single_click.js`

**Enfoque elegido**: Reemplazar el bloque `else` (líneas 128-146) de `getParcelaAltaPura()` con lógica que habilite la selección de parcelas sin nomenclatura.

**Alternativas descartadas**:
- ❌ Crear una función nueva en `single_click.js`: Innecesario, el cambio es localizado y las funciones auxiliares ya existen en el blade.
- ❌ Modificar el backend: Ya soporta `parcela_gid`, no se necesita cambio.
- ❌ Modificar `verificarParcela()`: No se llama desde el flujo de mapa cuando gid ya está seteado.

### Decisión 2: Usar `configurarOpcionesAlta(true)` para configurar el modal como informal

**Justificación**: Esta función ya existe y hace exactamente lo necesario: oculta las opciones de tipo_parcela excepto la 6 (informal), oculta las opciones de tipo_nomenclatura excepto la 2 (provisoria), y dispara el handler de cambio que genera la nomenclatura provisoria.

### Decisión 3: Usar toast no bloqueante en lugar de `SwalAlertHtml`

**Justificación**: `SwalAlertHtml` es un modal bloqueante que requiere interacción del usuario para cerrarlo. Un toast (`Swal.mixin({toast: true})`) es consistente con otros mensajes informativos en el sistema y no interrumpe el flujo.

### Decisión 4: Formato de `datosPoligono` para `dibujarPoligono()`

**Justificación**: En el flujo de `nomenc21 != null`, `datosPoligono` se setea como `{ "features": data.features }`. En el `else` actual se setea como `datosPoligono = data` (formato incorrecto). El nuevo código usará el formato correcto: `{ "features": data.features }`.

### Decisión 5: No modificar `ejecutarBusquedaNomenclatura()`

**Justificación**: La búsqueda por texto busca nomenclaturas en cartografía. Una parcela con `nomenc21 = null` no puede encontrarse por texto (no hay nomenclatura para buscar). Este comportamiento es correcto y no necesita cambios.

---

## Contratos e Interfaces

### Funciones existentes invocadas (sin cambios)

| Función | Ubicación | Firma | Comportamiento |
|---------|-----------|-------|----------------|
| `configurarOpcionesAlta(esParcelaInformal)` | `index.blade.php` línea 924 | `(boolean) → void` | Si `true`: tipo_parcela=6, tipo_nomenclatura=2 (provisoria, readonly). Dispara `trigger("change")` en tipo_nomenclatura. |
| `setParcelaSeleccionadaAlta(gid)` | `index.blade.php` línea 825 | `(string) → void` | Setea `#parcela_gid` y actualiza indicador visual (check verde o X roja). |
| `sincronizarSeccionesNomenclatura()` | `index.blade.php` línea 874 | `() → string` | Parsea `#nomenclatura` en campos hidden (distrito, sección, etc.). |
| `dibujarPoligono(elem)` | `funciones_generales.js` línea 463 | `(Object) → void` | Espera `{ features: [...] }` con geometría GeoJSON. |

### Variables globales existentes (sin cambios)

| Variable | Tipo | Uso |
|----------|------|-----|
| `generarHabilitado` | `boolean` | Controla si el botón "Generar" puede proceder |
| `datosPoligono` | `Object` | Datos GeoJSON para dibujar el polígono |
| `esInformal` | `boolean` | Flag que indica si la parcela es informal |
| `moduloActivado` | `string` | Indica el módulo activo ("ALTA_PURA", "NINGUNO", etc.) |

### Contrato del bloque `else` nuevo

**Input**: `gidAltaPura` (string|null), `data.features` (array GeoJSON)

**Output esperado**:
1. Modal configurado como parcela informal (tipo_parcela=6, tipo_nomenclatura=2)
2. Nomenclatura provisoria generada automáticamente en `#nomenclatura`
3. `#parcela_gid` seteado con el GID de la parcela
4. Indicador visual mostrando "Seleccionada" con check verde
5. `generarHabilitado = true`
6. Polígono dibujado en el mapa
7. Toast informativo no bloqueante
8. `#nomenclatura_busqueda_alta` vacío

---

## Código actual vs código nuevo

### Archivo: `public/js/cartografia_js/single_click.js`

#### Bloque actual (líneas 128-146):

```javascript
                } else {
                    html = "";
                    html = html + '<div class="col-sm-12 bg-dark text-light ">\
                                                                <label><h6 class="mt-2 font-weight-bold">Nomenclatura</h6></label>\
                                                            </div>\
                                                            <div class="col-sm-12 bg-dark text-light ">\
                                                                <label><h6 class="mt-2 font-weight-bold"><u>NO POSEE NOMENCLATURA</u></h6></label>\
                                                            </div>';
                    html = html + '</div>';
                    SwalAlertHtml(data.mensaje, html, "top-end");
                    datosPoligono = data;
                    $("#nomenclatura_busqueda_alta").val("");
                    $("#nomenclatura").val("");
                    if (typeof sincronizarSeccionesNomenclatura == "function") {
                        sincronizarSeccionesNomenclatura();
                    }
                    $(".btn-generar").removeClass("d-none");
                    dibujarPoligono(datosPoligono);
                }
```

#### Bloque nuevo (reemplazo completo):

```javascript
                } else {
                    // 1. Configurar modal como parcela informal (ANTES de setear gid)
                    //    Esto dispara tipo_nomenclatura=2 (provisoria) y genera
                    //    la nomenclatura provisoria porque parcela_gid aún está vacío.
                    if (typeof configurarOpcionesAlta == "function") {
                        configurarOpcionesAlta(true);
                    }

                    // 2. Marcar parcela como seleccionada (DESPUÉS de configurarOpcionesAlta)
                    if (typeof setParcelaSeleccionadaAlta == "function") {
                        setParcelaSeleccionadaAlta(gidAltaPura);
                    } else {
                        $("#parcela_gid").val(gidAltaPura || "");
                    }

                    // 3. Habilitar generación
                    generarHabilitado = true;

                    // 4. Guardar datos del polígono en formato correcto
                    datosPoligono = { "features": data.features };

                    // 5. Limpiar campo de búsqueda por texto
                    $("#nomenclatura_busqueda_alta").val("");

                    // 6. Sincronizar secciones de nomenclatura
                    if (typeof sincronizarSeccionesNomenclatura == "function") {
                        sincronizarSeccionesNomenclatura();
                    }

                    // 7. Mostrar botón Generar
                    $(".btn-generar").removeClass("d-none");

                    // 8. Dibujar polígono en el mapa
                    dibujarPoligono(datosPoligono);

                    // 9. Toast informativo no bloqueante
                    const Toast = Swal.mixin({
                        toast: true,
                        position: 'top-end',
                        showConfirmButton: false,
                        timer: 5000,
                        timerProgressBar: true,
                        didOpen: (toast) => {
                            toast.addEventListener('mouseenter', Swal.stopTimer);
                            toast.addEventListener('mouseleave', Swal.resumeTimer);
                        }
                    });
                    Toast.fire({
                        type: 'warning',
                        title: 'Parcela sin nomenclatura',
                        html: 'Se asignará nomenclatura provisoria automáticamente.'
                    });
                }
```

---

## Checklist de implementación

- [ ] Reemplazar el bloque `else` (líneas 128-146) en `public/js/cartografia_js/single_click.js` con el código nuevo
- [ ] Verificar que el orden de llamadas es: `configurarOpcionesAlta(true)` → `setParcelaSeleccionadaAlta(gid)` (NO invertir)
- [ ] Verificar que `datosPoligono` se setea como `{ "features": data.features }` (NO como `data` directamente)
- [ ] Verificar que el toast usa `Swal.mixin({toast: true})` y no `SwalAlertHtml`
- [ ] Verificar que `generarHabilitado` se setea a `true`
- [ ] Verificar que `#nomenclatura_busqueda_alta` se limpia
- [ ] Verificar que `dibujarPoligono(datosPoligono)` se llama con el formato correcto
- [ ] NO modificar el bloque `if (propiedadesAltaPura.nomenc21 != null)` (líneas 92-127)
- [ ] NO modificar `ejecutarBusquedaNomenclatura()`
- [ ] NO modificar `verificarParcela()`
- [ ] NO modificar el backend `altapura()`

---

## Casos de prueba

### CP-1: Click en parcela sin nomenclatura desde el mapa

**Precondiciones**: Modal Alta Pura abierto, mapa visible.

1. Hacer click en una parcela del mapa que tenga `nomenc21 = null` en la capa PARCELARIO
2. **Esperado**:
   - Se dibuja el polígono de la parcela en el mapa
   - El indicador muestra "✓ Seleccionada" con el GID
   - Tipo Parcela se setea en 6 (informal)
   - Tipo Nomenclatura se setea en 2 (provisoria) y el campo queda readonly
   - La nomenclatura se llena automáticamente con formato provisorio (ej: `07-00-00-0000-XXXXXX-0000`)
   - El campo "Buscar nomenclatura" queda vacío
   - Aparece un toast: "Parcela sin nomenclatura — Se asignará nomenclatura provisoria automáticamente."
   - El botón "Generar" está visible
3. Click en "Generar"
4. **Esperado**: Como `parcelaGid` existe y `tipo_parcela = 6`, se muestra el modal de vinculaciones
5. Confirmar vinculación
6. **Esperado**: Se crea la parcela y se actualiza `nomenc21` en `parcelas_pos`

### CP-2: Click en parcela CON nomenclatura (regresión)

**Precondiciones**: Modal Alta Pura abierto.

1. Hacer click en una parcela con `nomenc21 != null`
2. **Esperado**: Flujo normal sin cambios — se setea nomenclatura, tipo_nomenclatura según corresponda, indicador "Seleccionada", botón "Generar" visible
3. Click en "Generar"
4. **Esperado**: Alta normal o modal de vinculaciones si es informal

### CP-3: Búsqueda por texto de nomenclatura null (no debe encontrar)

**Precondiciones**: Modal Alta Pura abierto.

1. Escribir una nomenclatura en "Buscar nomenclatura" y click en buscar
2. **Esperado**: Solo encuentra parcelas con nomenclatura existente. No hay forma de buscar parcelas con `nomenc21 = null` por texto (comportamiento correcto, no se modifica)

### CP-4: Click en parcela sin nomenclatura y luego click en parcela con nomenclatura

1. Click en parcela sin nomenclatura → se configura como informal
2. Click en parcela con nomenclatura → **Esperado**: Se resetea el flujo (líneas 57-62 limpian gid y generarHabilitado), luego el bloque `if` setea la nomenclatura normalmente, `configurarOpcionesAlta` no se llama (o se llama con false si el tipo de nomenclatura no es provisoria)

### CP-5: Click en parcela sin nomenclatura y luego click en "Generar"

1. Click en parcela sin nomenclatura → se configura como informal con gid
2. Click en "Generar"
3. **Esperado**: En el handler de `btn-generar` (línea 1544), `parcelaGid` tiene valor, `generarHabilitado = true`, `tipo_parcela = 6` → se muestra modal de vinculaciones

### CP-6: Click en parcela sin nomenclatura cuando gidAltaPura es null

1. Click en una zona del mapa donde GetFeatureInfo no devuelve un feature con `gid` válido
2. **Esperado**: `gidAltaPura` será null, `setParcelaSeleccionadaAlta("")` se ejecutará en la línea 58, `generarHabilitado` será false. El bloque `else` no se alcanzará porque `data.features[0]` no existirá o `propiedadesAltaPura.gid` será null. Si se alcanza el else con gid null, `setParcelaSeleccionadaAlta("")` se llama y el flujo no debería permitir generar (verificar).

### CP-7: Parcela sin nomenclatura — nomenclatura provisoria se genera correctamente

1. Click en parcela sin nomenclatura
2. **Esperado**: `configurarOpcionesAlta(true)` dispara `tipo_nomenclatura.val(2).trigger("change")`
3. El handler de change detecta `esProvisoria = true` y `parcelaSeleccionada = false` (gid aún no seteado)
4. Setea `#nomenclatura` con `arregloNomenclaProvisoria.join("-")`
5. Luego `setParcelaSeleccionadaAlta(gid)` marca la parcela como seleccionada
6. **Verificar**: El campo `#nomenclatura` tiene valor y es readonly

---

## Casos edge y restricciones

### Edge case 1: `gidAltaPura` es null en el bloque else

Si `propiedadesAltaPura.gid` es null y `featureAltaPura.id` tampoco existe, `gidAltaPura` será null. En este caso:
- `setParcelaSeleccionadaAlta("")` se ejecutará (línea 58, antes del AJAX)
- En el bloque else, `setParcelaSeleccionadaAlta(null)` → normaliza a `""` → indicador muestra "Sin seleccion"
- `generarHabilitado = !!null = false` → el botón "Generar" no procederá
- **Mitigación**: Este caso es raro (un feature sin gid en PARCELARIO), pero el flujo es seguro porque `generarHabilitado` será false.

### Edge case 2: `configurarOpcionesAlta` no existe (script no cargado)

Se usa `typeof configurarOpcionesAlta == "function"` como guard. Si la función no existe, no se configura el modal como informal, pero el resto del flujo continúa. El usuario vería tipo_parcela y tipo_nomenclatura sin configurar.

### Edge case 3: Orden de operaciones — gid seteado antes de configurarOpcionesAlta

Si se invierte el orden y se llama `setParcelaSeleccionadaAlta(gid)` ANTES de `configurarOpcionesAlta(true)`:
- El handler de `tipo_nomenclatura` change verá `parcelaSeleccionada = true`
- Solo llamará `sincronizarSeccionesNomenclatura()` SIN generar la nomenclatura provisoria
- El campo `#nomenclatura` quedaría vacío
- **Esto es un bug crítico** → el orden es: `configurarOpcionesAlta(true)` PRIMERO, `setParcelaSeleccionadaAlta(gid)` DESPUÉS.

### Edge case 4: `datosPoligono` con formato incorrecto

El código actual en el `else` hace `datosPoligono = data` (el objeto completo de respuesta AJAX), mientras que `dibujarPoligono()` espera `{ features: [...] }`. El nuevo código usa `{ "features": data.features }` que es consistente con el bloque `if`.

### Edge case 5: Usuario hace click en otra parcela después de seleccionar una sin nomenclatura

Las líneas 57-62 del inicio de `getParcelaAltaPura()` resetean el estado: `setParcelaSeleccionadaAlta("")` y `generarHabilitado = false`. Esto funciona correctamente para ambos flujos (con y sin nomenclatura).

### Restricción: No modificar la búsqueda por texto

La función `ejecutarBusquedaNomenclatura()` busca nomenclaturas en cartografía. Una parcela con `nomenc21 = null` no puede encontrarse por texto porque no tiene nomenclatura para buscar. Este comportamiento es correcto y **no debe modificarse**.

---

## Criterios de aceptación

- [ ] Al hacer click en una parcela con `nomenc21 = null` en el mapa, se dibuja el polígono correctamente
- [ ] El indicador visual muestra "✓ Seleccionada" con el GID de la parcela
- [ ] Tipo Parcela se setea automáticamente en 6 (informal)
- [ ] Tipo Nomenclatura se setea automáticamente en 2 (provisoria) y el campo queda readonly
- [ ] La nomenclatura provisoria se genera automáticamente (formato `FIJO_DEPARTAMENTO_PROVISORIO-00-00-0000-XXXXXX-0000`)
- [ ] El campo "Buscar nomenclatura" queda vacío
- [ ] Aparece un toast no bloqueante con el mensaje "Parcela sin nomenclatura. Se asignará nomenclatura provisoria automáticamente."
- [ ] El botón "Generar" está visible y funcional
- [ ] Al click en "Generar" con `tipo_parcela = 6`, se muestra el modal de vinculaciones
- [ ] El flujo de parcelas CON nomenclatura (`nomenc21 != null`) funciona sin cambios (regresión)
- [ ] La búsqueda por texto NO encuentra parcelas sin nomenclatura (comportamiento inalterado)
- [ ] El backend `altapura()` recibe `parcela_gid` y actualiza `nomenc21` en `parcelas_pos` correctamente
- [ ] No se muestra el `SwalAlertHtml` bloqueante con "NO POSEE NOMENCLATURA"
- [ ] `datosPoligono` se pasa a `dibujarPoligono()` en formato `{ "features": data.features }`