# Estado del Proyecto — Guaymallén

> Última actualización: 2026-05-14
> Sistema: Catastro Municipal de Guaymallén (Laravel 5.8 / PHP 7.4)

---

## En progreso

*(features actualmente en desarrollo)*

| Feature | Fase | Inicio | Última actualización |
|---------|------|--------|---------------------|

---

## Completados

*(features terminados y verificados)*

| Feature | Fecha completado | Spec |
|---------|-----------------|------|

---

## Bloqueados

*(features pausados o con dependencias externas)*

| Feature | Motivo | Desde |
|---------|--------|-------|

---

## Decisiones pendientes

*(cosas que requieren input del usuario o stakeholders)*

---

## Deuda técnica identificada

*(issues encontrados en revisiones que no son bloqueantes pero deben atenderse)*

| Issue | Origen | Severidad | Fecha |
|-------|--------|-----------|-------|

---

## Arquitectura — Decisiones clave

Ver `.opencode/memory/decisions/` para el log detallado.

| Decisión | Fecha | Archivo |
|----------|-------|---------|

---

## Stack técnico

- **Backend**: Laravel 5.8, PHP 7.4
- **Bases de datos**: MySQL (SGC, RUD, PAD) + PostgreSQL/PostGIS (cartografía estática/dinámica)
- **Frontend**: Laravel Mix + Webpack, OpenLayers (cartografía)
- **PDF**: Snappy/wkhtmltopdf, DomPDF
- **Excel**: Maatwebsite/Laravel-Excel 3.x
- **Testing**: PHPUnit 7.5

## Conexiones a BD (crítico)

| Conexión | Key | Engine | Base |
|----------|-----|--------|------|
| SGC (default) | `mysql` | MySQL | `catastro_guaymallen` |
| RUD | `mysql2` | MySQL | `gestion_direcciones_gllen` |
| PAD | `core/` PDO | MySQL | Padrones |
| PostGIS estático | `pgsql` | PostgreSQL | Cartografía estática |
| PostGIS dinámico | `pgsql2` | PostgreSQL | Cartografía dinámica |

> **Regla**: Siempre verificar la conexión correcta antes de tocar la DB.
> `DB::connection('mysql2')` para RUD, `DB::connection('pgsql')` para PostGIS.
