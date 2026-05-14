# Project State

## En progreso

*Ninguna tarea activa.*

## Completados

*Sin tareas completadas.*

## Deuda tecnica

*Pendiente de identificar.*

## Decisiones tomadas

| Fecha | Decision | Motivo |
|---|---|---|
| 2026-05-14 | Arquitectura SSD con 6 agentes (orchestrator, router, architect, coder, reviewer, tester) | Simplificar desarrollo iterativo con pipelines clasificados por complejidad |
| 2026-05-14 | Pipelines: FASTPATCH (router->coder), PATCH (router->coder->reviewer), FEATURE (router->architect->coder->reviewer->tester) | Optimizar overhead segun complejidad de la tarea |
| 2026-05-14 | Specs solo para FEATURE en `.opencode/memory/specs/` | Evitar burocracia en tareas simples |
| 2026-05-14 | Orchestrator sin permisos de escritura de codigo | Separacion estricta de responsabilidades |

---

Ultima actualizacion: 2026-05-14
