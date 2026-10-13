# 2026-Veraguas-Dynatrace — Development Log (Observabilidad)

Historial de trabajo del reto **Observabilidad Dynatrace** (#hackathonCopa 2026 Veraguas).

---

## 2026-10-12 — Auditoría, versionado y reparación de datos

### Naturaleza del reto
Reto 100% de **observabilidad** sobre el tenant Dynatrace
(`daa00609.apps.dynatrace.com`). **No hay despliegue propio**: se responden
**64 preguntas abiertas** en 6 categorías usando las apps nativas de Dynatrace
(Kubernetes Explorer, Log Analytics, Distributed Tracing, Security, DEM).

### Hallazgo principal: `CLAUDE.md` y `SOLUTION.md` sin versionar
Ambos archivos existían en disco pero **nunca se habían pusheado**. Se versionaron.

### Categorías y puntos
| Código | Categoría | Preguntas | Puntos |
|--------|-----------|-----------|--------|
| 2100 | Infraestructura (Kubernetes Explorer) | 10 | 450 |
| 2200 | Logs (Log Analytics) | 26 | 2600 |
| 2300 | Trazas (Distributed Tracing) | 9 | 405 |
| 2400 | Seguridad | — | — |
| 2500 | Experiencia de Usuario (DEM) | — | — |

### Datos de apoyo (`hackathon_files/lookups/`)
| Archivo | Contenido | Registros |
|---------|-----------|-----------|
| `payment-service-error-codes.csv` | Catálogo de códigos de error | 17 |
| `payment-service-user-lookup.csv` | Usuarios (id, tier Diamond) | 400 |
| `ip_geo_locations.csv` | IP → ciudad/país | 110 |
| `user-agent-parsed.jsonl` | User-agents parseados (browser/OS) | 103 |
| `web_path_vulnerabilities.jsonl` | Rutas maliciosas → CVE | 10 |
| `gigamon_ip_enriched_table.csv` | Enriquecimiento de red | 494 |
| `tls_cipher_suites_table.csv` | Suites TLS | 367 |

### 🐛 Bug de datos encontrado y reparado
**`lookups/web_path_vulnerabilities.jsonl` estaba malformado**: envuelto con llaves
dobles (`{{...}}`) como un array incorrecto, por lo que **no era JSONL válido**
(`json.loads` fallaba con *Expecting property name enclosed in double quotes*).
Esto impediría su ingesta como lookup JSONL en Dynatrace.

- **Fix:** se removió el envoltorio sobrante y se reescribió como **JSONL válido**
  (1 objeto JSON por línea). 10 registros.
- **Verificado:** la ruta clave referenciada en `SOLUTION.md`
  (`/info.php → CVE-2019-11043`) se conserva intacta.

### Verificaciones realizadas
| Verificación | Resultado |
|--------------|-----------|
| Notebooks `*.json` (DQL) JSON válido | ✅ 3 válidos |
| `user-agent-parsed.jsonl` JSONL válido | ✅ 103 líneas |
| `web_path_vulnerabilities.jsonl` | ✅ **Reparado** (10 líneas JSONL válidas) |
| Lookups `*.csv` columnas consistentes | ✅ 5 válidos |
| Contenido DQL (filter/summarize/parse) | ✅ Presente en notebooks |

### Mejoras aplicadas
1. **Versionado** — `CLAUDE.md` y `SOLUTION.md` pasaron a estar en GitHub.
2. **Reparación de datos** — `web_path_vulnerabilities.jsonl` corregido a JSONL válido.
3. **`.gitignore`** — creado (sistema/editores; no hay build ni dependencias).
4. **`DEVELOPMENT-LOG.md`** — este archivo.

### Nota operativa clave
En cada notebook DQL reemplazar `log.source == "<paste-your-unique-id-here>"`
por el **unique id del equipo** — es la llave para aislar los datos propios del tenant.

### Pendiente / futuro
- ⬜ Responder las 64 preguntas en el tenant Dynatrace (requiere acceso al evento).
- ⬜ Exportar respuestas/evidencias por categoría al repo.
