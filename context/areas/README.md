# Áreas del Ministerio — Contexto de desarrollo

Cada carpeta corresponde a una secretaría. El desarrollo de su módulo backend y frontend
**no comienza hasta tener el archivo `contexto_detallado.md` completo**, surgido de la reunión con el área.

## Flujo por área

```
Reunión con el área
       ↓
Crear contexto_detallado.md en esta carpeta
       ↓
Validar spec en docs/files/spec-svc-{area}.md → cambiar estado a "approved"
       ↓
Implementar services/svc-{area}/
       ↓
Implementar frontend/src/modules/{area}/
       ↓
Deploy + UAT con operador del área
```

## Estado actual de cada área

| Carpeta | Área | Reunión | Desarrollo |
|---------|------|---------|------------|
| `Privada Ministro/` | Secretaría Privada del Ministro | ✅ hecha | ✅ sistema productivo existente |
| `secretaria_Vivienda/` | Secretaría de Vivienda | ✅ hecha | ✅ svc-vivienda implementado |
| `secretaria_Gasifera/` | Secretaría de Infraestructura Gasífera | 🟡 primera reunión hecha (2026-09-21), parcial — ver `contexto_detallado.md` | ⏳ sync Fase 0 implementado y desplegado (svc-gasifera) + panel preliminar de solo lectura (excepción documentada, ver `spec-sync-gasifera-pit.md §12`); panel de negocio completo pendiente |
| `secretaria_Gral_Gobierno/` | Secretaría General de Gobierno (programa ATP) | ⏳ pendiente | ⏳ sync Fase 0 en desarrollo (svc-gralgob) — no estaba en el plan original, agregado 2026-09-21 |
| `secretaria_Infraestructura/` | Sec. de Gestión y Vinculación de Infraestructura (Luis Molinari) | ⏳ pendiente | ⏳ pendiente |
| `secretaria_Territorial/` | Sec. de Planificación y Articulación Territorial (Gabriel Fizza) | ⏳ pendiente | ⏳ pendiente |
| `secretaria_Desarrollo/` | Secretaría de Desarrollo (Domingo Benso) — requiere contrato API UTN | ⏳ pendiente | ⏳ pendiente |

## Contenido esperado de `contexto_detallado.md`

Tras cada reunión, el archivo debe cubrir:

1. **Responsable del área** — nombre, cargo, contacto
2. **Procesos actuales** — cómo trabajan hoy (planillas, sistemas, papel)
3. **Funcionalidades requeridas** — qué debe hacer el sistema (ABM, reportes, estados)
4. **Campos y datos** — qué información manejan (validar o ajustar el spec draft)
5. **Estados y flujos** — ciclo de vida de los registros principales
6. **Usuarios del sistema** — quién usa qué (roles operativos)
7. **Integraciones** — si se conecta con otros sistemas o datos externos
8. **Prioridades** — qué es urgente vs. nice-to-have
