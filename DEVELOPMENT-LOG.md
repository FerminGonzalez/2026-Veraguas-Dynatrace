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

---

## 2026-10-12 — Compilación del backend + generación de respuestas

### 1. Backend .NET (`2026-Veraguas-Microsoft`) — compilado y probado
- SDK .NET **10.0.401** instalado localmente (`~/.dotnet`, sin sudo).
- `dotnet build -c Release` → **Build succeeded, 0 errores**.
- Warnings CS8620 (nullability en `Program.cs` DeepHealthCheck) **corregidos** con cast
  a `IReadOnlyDictionary<string, object?>` → bajó de 4 a 2 warnings.
- Queda 1 warning **NU1903** (AutoMapper 13.0.1, advisory GHSA-rvv3-g6hj-g44x): no
  forzar upgrade (AutoMapper 14+ cambia licencia) → decisión del equipo, documentado.
- API ejecutada en local (`http://127.0.0.1:5099`, EF Core InMemory) y endpoints probados:
  | Endpoint | Resultado |
  |---|---|
  | `GET /health` | 200 `{"status":"Healthy"}` |
  | `GET /health/deep` | 503 Degraded (E4 — handshake frontend pendiente, esperado) |
  | `GET /api/status` | 503 (E5) |
  | `GET /api/services` | 200 |
  | `GET /api/flights/list` | 200 |
  | `POST /api/flights/create` | **201 Created** (E6 ✅) |
  | `POST` payload inválido | **400** con errores E6 estructurados |
  | `X-Correlation-Id` | ecoado en respuesta ✅ |
- Hallazgo: `FrontendHandshake` no bloquea GET con Origin no permitido en frío
  (coherente con diseño E5 — requiere handshake previo). Documentado, no es bug.

### 2. Respuestas del reto Dynatrace — `RESPUESTAS.md`
- Extraídas las **64 preguntas** (10+26+9+6+13) de `section_files/`.
- Extraídos los **DQL base** de los 3 notebooks (PaymentService, Nginx, Gigamon).
- Registros esperados confirmados: PaymentService **2.439**, Nginx **1.000**, Gigamon **7.425**.
- **Respuestas derivables de `lookups/` locales** (verificadas con script):
  - 2206 total usuarios = **400**
  - 2207 Margaret Wilson → id **u530064** (tier diamond)
  - Miembros Diamond = **50**; tiers: silver 75 / gold 100 / platinum 50 / diamond 50 / none 125
  - 2215 ruta CVE-2019-11043 = **`/info.php`**; 11 IPs en Rusia
  - 2224 suites TLS < 256 bits = **138**; 4 UA Chrome+Android (2,5,8,11 → <10 excluye 11)
- **63 preguntas restantes requieren el tenant en vivo** (credenciales + `<UID>` del
  equipo): `RESPUESTAS.md` entrega el **DQL exacto listo para pegar** por cada una y la
  ruta UI para infra/trazas/seguridad/DEM.
- Limitación documentada: los `lookups/` son tablas de referencia, **no los logs** → no
  es posible calcular los conteos sin el tenant. Honestidad > inventar valores.

