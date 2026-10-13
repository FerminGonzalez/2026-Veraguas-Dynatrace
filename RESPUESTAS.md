# Reto 2026-Veraguas-Dynatrace — RESPUESTAS (64 preguntas)

Guía de respuestas para el reto de observabilidad. Cada pregunta incluye:
- **Respuesta concreta** cuando es derivable de los `lookups/` locales (✅ verificable aquí).
- **Consulta DQL exacta** lista para pegar en Dynatrace cuando requiere el tenant en vivo.
- **Ruta en la UI** cuando se resuelve navegando (Kubernetes, Security, DEM).

> ⚠️ **Requisito**: las respuestas marcadas 🔒 necesitan el tenant Dynatrace
> (`daa00609.apps.dynatrace.com`) con credenciales del evento y el **unique id**
> del equipo. Reemplazar `<UID>` en cada DQL por ese valor (está en el notebook:
> `log.source == "<paste-your-unique-id-here>"`).

> **Registros esperados (validación de datos)**:
> PaymentService **2.439** · Nginx **1.000** · Gigamon **7.425**.
> Si el conteo no coincide → mal `log.source`/timeframe.

---

## Datos derivables de los `lookups/` locales (ya verificados)

| Dato | Valor | Fuente |
|---|---|---|
| Total de usuarios | **400** | `payment-service-user-lookup.csv` |
| Margaret Wilson → user id | **u530064** (tier: **diamond**) | idem |
| Miembros Diamond | **50** | idem |
| Distribución tiers | silver 75 · gold 100 · platinum 50 · diamond 50 · none 125 | idem |
| Catálogo códigos de error | **17** códigos (1xxx negocio, 5xxx técnica) | `payment-service-error-codes.csv` |
| CVE-2019-11043 → ruta | **`/info.php`** | `web_path_vulnerabilities.jsonl` (✅ verificado) |
| IPs Rusia (para 2215) | **11** (Moscow, Saint Petersburg, Novosibirsk, …) | `ip_geo_locations.csv` |
| Suites TLS clave < 256 bits | **138** (p.ej. `TLS_RSA_WITH_RC4_128_MD5`, export 40-bit) | `tls_cipher_suites_table.csv` |
| User-agents Chrome+Android | **4** variantes (Android 2, 5, 8, 11) | `user-agent-parsed.jsonl` |
| Chrome en Android **< 10** | Android **2, 5, 8** (no el 11) → excluir UA con `Android 11` | idem |

> Códigos de error clave (para leer resultados de logs):
> **1010 / 1020** = *invalid transaction id* · **1011** = *payment declined* ·
> **5010** = *bank system error* · **5011** = *payment network error* ·
> **5012** = *payment maintenance error* · **5001** = *database connection error*.

---

## 2100 — Infraestructura (Kubernetes Explorer) — 10 preguntas 🔒

*Resueltas navegando en `Dynatrace > Infrastructure > Kubernetes` (clúster del reto).*

| # | Pregunta | Dónde está la respuesta |
|---|---|---|
| 2101 | Versión del clúster | Kubernetes Explorer → cabecera del clúster → *Kubernetes version* |
| 2102 | Namespace con eventos de backoff | *Events* → filtrar `Back-off`/`CrashLoopBackOff` → leer `k8s.namespace.name` |
| 2103 | Pods en `kube-system` | Namespace `kube-system` → contador de pods |
| 2104 | Contenedor con reinicios en `unguard` | Namespace `unguard` → columna *Restarts* → contenedor |
| 2105 | CPU limit de `broker-service` (easytrade) | Deployment `broker-service` → Pod spec → *Limits → cpu* |
| 2106 | ContainerPort de `unguard-frontend` | Deployment `unguard-frontend` → Pod spec → *containerPort* |
| 2107 | Servicio con fallas en `credit-card-order-service` | Pod `credit-card-order-service` → pestaña *Problems/Services* |
| 2108 | DynaKube mode | K8s cluster → *Properties/Settings* → `dynakube.mode` |
| 2109 | SO de los nodos | *Nodes* → columna *OS* |
| 2110 | Tipo de instancia de nodos | *Nodes* → *Instance type* (p.ej. `m5.large`) |

<details><summary>DQL de apoyo (eventos backoff — 2102)</summary>

```dql
fetch events
| filter k8s.namespace.name == "<ns>"
| filter matchesPhrase(content, "Back-off") or event.type == "K8S_POD_BACKOFF"
| summarize c = count(), by:{k8s.namespace.name}
```
</details>

---

## 2200 — Logs (DQL) — 26 preguntas 🔒

### Sub-categoría 1 — PaymentService (2201–2210)

**Formato del `content` (XML):**
```
2025-07-24T00:39:27Z INFO [PaymentServiceThread] '<paymentservice ...><transaction type="payment" stage="execution" id="..."><card .../><user id="u552293" /><amount value="328.13" currency="USD" tax="16.41" /><result status="FAIL"><error code="5012" /><error code="5010" /><error code="1011" /></result></transaction></paymentservice>'
```

**2201 — Total de logs con `FAIL`** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| search "[PaymentServiceThread]"
| filter matchesPhrase(content, "status=\"FAIL\"")
| summarize logs = count()
```

**2202 — Código de error más frecuente** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| search "[PaymentServiceThread]"
| filter matchesPhrase(content, "status=\"FAIL\"")
| parse content, "LD '<error code=\"' INT:code"
| expand code
| summarize c = count(), by:{code}
| sort c desc
| limit 5
```
> Cruzar el `code` ganador con `payment-service-error-codes.csv` → devolver `name`.
> (Patrón `parse` correcto del enunciado: `ISO8601:original_ts SPACE WORD:loglevel SPACE '[PaymentServiceThread]' SPACE PUNCT{1} XML:message`.)

**2203 — Devoluciones validadas exitosamente** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter matchesPhrase(content, "type=\"refund\"")
| filter matchesPhrase(content, "status=\"SUCCESS\"")
| summarize refunds = count()
```

**2204 — Importe medio reembolsado** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter matchesPhrase(content, "type=\"refund\"")
| filter matchesPhrase(content, "status=\"SUCCESS\"")
| parse content, "LD '<amount value=\"' STRING:amt"
| summarize avg_refund = avg(toDouble(amt))
```

**2205 — Revenue de pagos validados** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter matchesPhrase(content, "type=\"payment\"")
| filter matchesPhrase(content, "status=\"SUCCESS\"")
| parse content, "LD '<amount value=\"' STRING:amt"
| summarize revenue = sum(toDouble(amt))
```

**2206 — Total de usuarios** ✅
> **Respuesta: `400`** (derivada de `payment-service-user-lookup.csv`, 400 filas).
```dql
// Alternativa en Dynatrace (si el lookup está cargado):
| lookup "payment-service-user-lookup"
| summarize users = countDistinct(id)
```

**2207 — Diagnóstico de Margaret Wilson** 🔒 (id local ✅)
> **Su user id: `u530064`** (tier diamond). Diagnosticar su error:
```dql
fetch logs
| filter log.source == "<UID>"
| filter matchesPhrase(content, "u530064")
| parse content, "LD '<error code=\"' INT:code"
| expand code
| summarize c = count(), by:{code}
| sort c desc
```
> Leer los códigos resultantes y traducirlos con el catálogo (p.ej. `5010` bank system
> error / `1011` declined / `5001` db connection) → esa es la causa del fallo de compra.

**2208 — Transacciones con ID inválido (¿10 incl. `fa28b512-…`?)** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter matchesPhrase(content, "stage=\"validation\"")
| filter matchesPhrase(content, "code=\"1010\"") or matchesPhrase(content, "code=\"1020\"")
| parse content, "LD 'id=\"' STRING:txid"
| summarize c = count(), by:{txid}
| sort c desc
```
> Verificar: total = **10** y que `fa28b512-54ff-418e-8d9e-1c2fa6465bc7` esté en la lista → **Verdadero**.

**2209 — Transacciones Diamond con fallos técnicos (`50xxx`)** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter matchesPhrase(content, "status=\"FAIL\"")
| filter matchesPhrase(content, "code=\"50")
| parse content, "LD '<user id=\"' STRING:uid"
| lookup "payment-service-user-lookup", matchKey:id, lookupKey:uid
| filter tier == "diamond"
| summarize failed = count()
```
> Miembros Diamond en el lookup: **50** (para validar el join).

**2210 — Revenue perdido Diamond** 🔒
```dql
// sobre el mismo filtro de 2209:
| parse content, "LD '<amount value=\"' STRING:amt"
| summarize lost = sum(toDouble(amt))
```

### Sub-categoría 2 — Nginx access logs (2211–2217)

**Formato:**
```
3.91.157.123 - [24/Jul/2025:10:32:19 +0000] "POST /broker-service/v1/balance/35/deposit HTTP/1.1" 200 56 "ref" "UA"
```

**2211 — Registros con método POST** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| search "\"POST "
| summarize posts = count()
```

**2212 — Tamaño medio de respuesta** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| parse content, "LD '-' LD '[' LD '\"' WORD:method SPACE STRING:proto '\"' SPACE INT:status SPACE INT:size"
| summarize avg_size = avg(size)
```

**2213 — Chrome en Android < 10 a `broker-service`** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter matchesPhrase(content, "broker-service")
| parse content, "LD '\"' STRING:ua"
| lookup "user-agent-parsed", matchKey:user_agent, lookupKey:ua
| filter browser_family == "Google Chrome"
  and os_family == "Android"
  and toInt(os_version_major) < 10
| summarize c = count()
```
> En el lookup, Chrome+Android existe en versiones 2, 5, 8 y 11 → **excluir Android 11**.

**2214 — Solicitudes desde Estados Unidos** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| parse content, "IP:client_ip"
| lookup "ip_geo_locations", matchKey:IP, lookupKey:client_ip
| filter Country == "United States"
| summarize c = count()
```

**2215 — Ciudad del tráfico CVE-2019-11043** 🔒 (ruta ✅)
> **Ruta sospechosa: `/info.php`** (confirmada en `web_path_vulnerabilities.jsonl` → CVE-2019-11043).
```dql
fetch logs
| filter log.source == "<UID>"
| filter matchesPhrase(content, "/info.php")
| parse content, "IP:client_ip"
| lookup "ip_geo_locations", matchKey:IP, lookupKey:client_ip
| fields client_ip, Country, City
```
> Devolver el `City` (entre las IPs rusas del lookup: Moscow, Saint Petersburg, …).

**2216 — Accesos a la raíz `/`** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| parse content, "LD '\"' WORD:method SPACE STRING:path SPACE STRING:proto"
| filter path == "/"
| summarize c = count()
```

**2217 — Tercera IP con más tráfico** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| parse content, "IP:client_ip"
| summarize c = count(), by:{client_ip}
| sort c desc
| limit 3
```
> Tomar la **3ª** fila.

### Sub-categoría 3 — Network / Gigamon (2218–2226)

**Formato JSON** (Dynatrace auto-parsea los campos: `src_ip`, `dst_ip`, `app_name`, `src_bytes`, `dst_bytes`, `tcp_rtt`, `asset_id`, …).

**2218 — Aplicaciones distintas** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter vendor == "Gigamon"
| summarize apps = countDistinct(app_name)
```

**2219 — Orígenes distintos** 🔒
```dql
| summarize origins = countDistinct(src_ip)
```

**2220 — Destinos distintos** 🔒
```dql
| summarize dests = countDistinct(dst_ip)
```

**2221 — Top 3 apps por TCP RTT medio (mayor)** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| summarize rtt = avg(tcp_rtt), by:{app_name}
| sort rtt desc
| limit 3
```
> Responder las 3 apps separadas por comas.

**2222 — Apps de uso personal de `172.16.133.66`** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter src_ip == "172.16.133.66" or dst_ip == "172.16.133.66"
| filter app_name != "icmp" and app_name not in ("aws","https","dns")
| summarize c = count(), by:{app_name}
| sort c desc
```
> Contar las apps de uso personal (no corporativas).

**2223 — Bytes de `172.16.133.66`** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter src_ip == "172.16.133.66" or dst_ip == "172.16.133.66"
| summarize bytes = sum(toLong(src_bytes)) + sum(toLong(dst_bytes))
```

**2224 — IPs privadas con TLS < 256 bits** 🔒 (suites ✅)
```dql
fetch logs
| filter log.source == "<UID>"
| fields dst_ip as server_ip, tls_cipher as cipher
| filter matchesPattern(server_ip, "^10\\..*") or matchesPattern(server_ip, "^172\\.(1[6-9]|2[0-9]|3[01])\\..*") or matchesPattern(server_ip, "^192\\.168\\..*")
| lookup "tls-cipher-suites-table", matchKey:Description, lookupKey:cipher
| filter toInt(KeySize) < 256
| summarize by:{server_ip}
```
> **138 suites** del catálogo tienen clave < 256 bits → filtrar tráfico real a IPs privadas y devolver IPs separadas por comas.

**2225 — Ubicación con más tráfico P2P** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter lower(app_name) in ("bittorrent","edonkey","kazaa","gnutella","ares")
| fieldsAdd total = toLong(src_bytes) + toLong(dst_bytes)
| lookup "gigamon-ip-enriched-table", matchKey:ip, lookupKey:src_ip
| summarize traffic = sum(total), by:{location}
| sort traffic desc
| limit 1
```

**2226 — asset_id de minería (más frecuente)** 🔒
```dql
fetch logs
| filter log.source == "<UID>"
| filter lower(app_name) in ("bitcoin","ethereum","ethereum-nd","minexmr-com","monero")
| summarize c = count(), by:{asset_id}
| sort c desc
| limit 1
```

---

## 2300 — Trazas (Distributed Traces) — 9 preguntas 🔒

*`Applications & Microservices > Distributed Traces` / Service Map / Live Debugger.*

| # | Pregunta | Dónde / DQL |
|---|---|---|
| 2301 | Latencia media `AccountControllerV2` (7d, **ms**) | Service Map → servicio → *Response time*; o `fetch spans \| filter service.name=="AccountControllerV2" \| summarize avg(duration)` |
| 2302 | Requests/min de `AdService` | *Request rate* del servicio → **req/min** (sin unidades) |
| 2303 | Query con **mayor duración acumulada** en `BrokerService` (7d) | Traces → filtro DB → *Duration sum* por query → copiar query completa |
| 2304 | Query **más rápida** en promedio en `OrderController` (10d) | igual, ordenar por *avg duration* asc → copiar query |
| 2305 | Excepción a nivel de span en `OrderController` (5d) | Traces → filtrar errores → *Exception* del span |
| 2306 | `exception class` de esa traza | pestaña *Exception* de la traza asociada |
| 2307 | `Clase:Línea` del `java.lang.ArithmeticException` | Stack trace → primera frame de la app → formato **`Clase:Línea`** |
| 2308 | Namespace de la base de datos en la traza | span DB → *DB namespace / database* |
| 2309 | `Db operation name` de la interacción | span DB → *Db operation name* |

<details><summary>DQL de apoyo</summary>

```dql
// 2301
fetch spans
| filter service.name == "AccountControllerV2"
| summarize avg_ms = avg(duration)
```
</details>

---

## 2400 — Seguridad (Application Security) — 6 preguntas 🔒

*`Application Security > Vulnerabilities` y `Security Posture / Secure Posture Management`.*

| # | Pregunta | Dónde está la respuesta |
|---|---|---|
| 2401 | Componente a corregir en **S-99** | Vulnerabilities → buscar `S-99` → campo *Component / affected component* |
| 2402 | Entidad afectada por **CVE-2024-47764** | buscar la CVE → *Affected entity* (nota: el enunciado escribe 47554 en el título pero 47764 en el cuerpo → usar **CVE-2024-47764**) |
| 2403 | Versión del componente en **S-246** | Vulnerability `S-246` → *Component version* |
| 2404 | **STIG Vulnerability ID** de *"Kubernetes must remove old components after updated versions have been installed"* | Secure Posture Mgmt → regla → campo *STIG Vulnerability ID* |
| 2405 | Únnamespace **no relevante** en *"Ensure that all Namespaces have Network Policies defined"* | Secure Posture Mgmt → regla → lista de namespaces → el marcado **Not relevant** |
| 2406 | Namespace en estado **Manual** en *"Minimize the admission of containers wishing to share the host process ID namespace"* | Secure Posture Mgmt → regla → namespace con estado **Manual** |

---

## 2500 — Experiencia de Usuario (DEM/RUM) — 13 preguntas 🔒

*`Digital Experience > Web Applications` (app **easytrade**), Session Segmentation, Error Inspector, Business Analytics.*

| # | Pregunta | Dónde está la respuesta |
|---|---|---|
| 2501 | Browser más usado | Web app **easytrade** → *User experience / Breakdown → Browser* |
| 2502 | País con **mejor Apdex** (30d) | *Apdex by Country* (30d) → mayor valor |
| 2503 | País con **más errores** (30d) | *Errors by Country* (30d) → mayor conteo |
| 2504 | Excepción frontend más frecuente | **Error Inspector** → *Exceptions* → mayor frecuencia |
| 2505 | ISP con más sesiones | Session Segmentation → *ISP / Provider* → más sesiones |
| 2506 | **Visually Complete** de `loading of page/login` (**s**, sin unidades) | RUM → Action performance → acción `loading of page/login` → *Visually complete* |
| 2507 | Acción con fallas de usuario **Jessica** | Session Segmentation → filtro usuario `Jessica` → sesión con error → *Action* |
| 2508 | Ubicación de conexión de **Catherine** | Session Segmentation → usuario `Catherine` → *Location/City* |
| 2509 | Paso con fallas de **Airline Ticket Booking Flow** | Business Analytics → business flow → paso en rojo |
| 2510 | `event.type` de esa excepción de negocio | paso fallido → *Event type / exception* |
| 2511 | Ruta con más **bookings** (7d, formato `MEX-PTY`) | Dashboard compartido → *Bookings by route* (7d) → mayor |
| 2512 | Pasarela de pago con más fallas | Business Analytics / tabla de pagos → más errores |
| 2513 | Aerolínea con más tickets vendidos | tabla de tickets → *Airline* con mayor conteo |

---

## Resumen de entregables

| Categoría | Preguntas | Respuestas derivables aquí | Requieren tenant |
|---|---|---|---|
| 2100 Infraestructura | 10 | 0 | 10 (UI) |
| 2200 Logs | 26 | **1** (2206 ✅) + parciales (2207 id, 2215 ruta) | 25 (DQL) |
| 2300 Trazas | 9 | 0 | 9 (UI/DQL) |
| 2400 Seguridad | 6 | 0 | 6 (UI) |
| 2500 Experiencia de usuario | 13 | 0 | 13 (UI) |
| **Total** | **64** | **1 + 2 parciales** | **63** |

> **Nota de honestidad**: las respuestas numéricas de logs/trazas **no se pueden
> calcular sin el tenant Dynatrace en vivo** (los `lookups/` locales son tablas de
> referencia, no los logs). Este documento entrega el **DQL exacto** para cada una
> y las respuestas que sí son derivables localmente. Con credenciales del evento +
> el `<UID>` del equipo, cada consulta devuelve el valor en un solo paso.
>
> **Puntaje objetivo**: 4.505 pts (450 + 2.600 + 405 + 270 + 780).
