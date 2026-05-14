# Tarea: Selección de parcelas con nomenclatura null en mapa (Alta Pura)

- **Feature**: Permitir seleccionar parcelas con `nomenc21 = null` desde el mapa en el modal de Alta Pura
- **Fecha de inicio**: 2026-05-14
- **Estado**: `completada`
- **Fecha de finalización**: 2026-05-14
- **Módulo afectado**: `resources/views/gestion/padron/index.blade.php` + `public/js/cartografia_js/single_click.js`

## Requerimiento

En el modal de Alta Pura (`resources\views\gestion\padron\index.blade.php`):
1. **Selección por mapa (PERMITIR)**: Al hacer click sobre una parcela que tenga `nomenc21 == null` en la capa PARCELARIO, el sistema debe permitir la selección configurando el formulario como "parcela informal" (tipo_parcela=6, tipo_nomenclatura=2 provisoria), guardar el GID, y habilitar el botón Generar.
2. **Búsqueda por texto (NO PERMITIR)**: El campo "Buscar nomenclatura" (`#nomenclatura_busqueda_alta`) NO debe permitir encontrar/seleccionar parcelas con nomenclatura null. El código actual ya lo impide naturalmente porque `buscarPorNomenclatura()` consulta por nomenclatura.

## Archivos clave

- `resources/views/gestion/padron/index.blade.php` — Modal Alta Pura, funciones JS (`configurarOpcionesAlta`, `setParcelaSeleccionadaAlta`, `generarHabilitado`, `esInformal`, etc.)
- `public/js/cartografia_js/single_click.js` — `getParcelaAltaPura()` maneja el click en el mapa dentro del modal
- `public/js/cartografia_js/funciones_generales.js` — Funciones compartidas (`dibujarPoligono`, `SwalAlertHtml`, etc.)
- `app/Http/Controllers/ParcelaController.php` — `altapura()` procesa el alta (ya soporta parcela_gid)

## Estado del pipeline

- [x] Spec en progreso
- [ ] Spec completada
- [ ] Implementación en progreso
- [ ] Implementación completada
- [ ] Revisión aprobada
- [ ] Tests pasando
