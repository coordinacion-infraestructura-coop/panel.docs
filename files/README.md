# Sistema Integral de Gestión — Secretaría General de Gobierno

## Documentación

| Archivo | Descripción |
|---------|-------------|
| `.claude/CLAUDE.md` | **Leer primero.** Contexto completo para Claude Code |
| `docs/context/organigrama.md` | Estructura orgánica y mapeo a servicios |
| `docs/context/arquitectura.md` | Decisiones técnicas y patrones |
| `docs/context/roadmap.md` | Fases de implementación |

## Specs por servicio

| Spec | Servicio | Estado |
|------|----------|--------|
| `docs/specs/services/spec-svc-vivienda.md` | Secretaría de Vivienda | draft |
| `docs/specs/services/spec-svc-infraestructura.md` | Secretaría de Infraestructura | draft |
| `docs/specs/services/spec-svc-territorial-y-desarrollo.md` | Territorial + Desarrollo | draft |
| `docs/specs/services/spec-svc-gasifera.md` | Secretaría Gasífera | draft |
| `docs/specs/infrastructure/spec-infraestructura-gcp.md` | GCP, API Gateway, Cloud Run | draft |
| `docs/specs/frontend/spec-frontend.md` | Portal React | draft |

## Inicio rápido para Claude Code

```bash
# 1. Leer el contexto principal
cat .claude/CLAUDE.md

# 2. Ver el organigrama
cat docs/context/organigrama.md

# 3. Elegir el servicio a implementar y leer su spec
cat docs/specs/services/spec-svc-vivienda.md

# 4. Implementar siguiendo exactamente el spec
```

## Convención de estados de specs

- `draft` — en elaboración, no implementar aún
- `review` — listo para revisión del equipo
- `approved` — aprobado, se puede implementar
- `implemented` — implementado y testeado
- `deprecated` — reemplazado por nueva versión
