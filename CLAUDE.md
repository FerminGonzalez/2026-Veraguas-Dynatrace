# 2026-Veraguas-Dynatrace - Observability Challenge

## Propósito
Desafío hackathon 2026 Veraguas: Monitoreo y observabilidad con Dynatrace en clúster Kubernetes.

## Categorías y puntos
| Código | Categoría | Preguntas | Puntos |
|--------|-----------|-----------|--------|
| 2100 | Infraestructura | 10 | 450 |
| 2200 | Logs | 26 | 2,600 |
| 2300 | Trazas | 9 | 405 |
| Otros | Seguridad, UX, etc. | - | - |

## Módulos de análisis

### Módulo 1: Infraestructura (2100)
- Archivo: `section_2/` o `section_files/section_2.md`
- Actividad: Análisis del clúster monitoreado con Dynatrace
- Enfocarse en: health del cluster, nodos, resources

### Módulo 2: Logs (2200)
- Archivo: `section_3/` o `section_files/section_3.md`
- Actividad: Logs de PaymentService, Nginx Access, Network
- Herramientas: Dynatrace Log Analytics

### Módulo 3: Trazas (2300)
- Archivo: `section_4/` o `section_files/section_4.md`
- Actividad: Trazas distribuidas y servicios
- Enfocarse: distributed tracing, service map

### Módulo 4: Seguridad (2400)
- Archivo: `section_5/` o `section_files/section_5.md`
- Actividad: Vulnerabilidades y cumplimiento de seguridad

### Módulo 5: Experiencia de Usuario (2500)
- Archivo: `section_6/` o `section_files/section_6.md`
- Actividad: Digital Experience Monitoring, sesiones de usuario

## Comandos habituales
```bash
# Revisar estructura
ls -la
ls section_files/
ls assets/

# Si hay scripts de análisis
# (depende de la implementación específica del hackathon)
```

## Formato de commits (Convencionales)
```
git commit -m "modulo: categoría - descripción breve"

tipos sugeridos:
- logs: análisis de logs de servicio
- traces: nueva traza o servicio identificado
- security: vulnerabilidad o hallazgo
- infra: configuración de cluster
- docs: resumen de hallazgos por categoría
```

## Estado actual
- ✅ Estructura de menú definida en README
- ✅ 6 secciones principales identificadas (`section_files/section_1..6.md`)
- ✅ `section_files/` y `assets/` disponibles
- ✅ `CLAUDE.md` y `SOLUTION.md` **versionados en GitHub** (antes solo en disco)
- ✅ `hackathon_files/`: 3 notebooks DQL (JSON válido) + 7 lookups (csv/jsonl)
- ✅ `web_path_vulnerabilities.jsonl` **reparado** (era JSON malformado `{{...}}`; ahora JSONL válido, 10 registros)
- ⬜ Responder las 64 preguntas en el tenant Dynatrace (requiere acceso al evento)
