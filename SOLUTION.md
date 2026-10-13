# Reto 2026-Veraguas-Dynatrace — Solución (metodología + DQL)

Reto de observabilidad 100% sobre el **tenant Dynatrace** (`daa00609.apps.dynatrace.com`).
No hay despliegue propio: se responden 64 preguntas abiertas usando las apps nativas de Dynatrace.
Este documento mapea cada categoría a la herramienta y a la consulta DQL/lookup de apoyo.

> **Datos de apoyo en `hackathon_files/`**
> - `lookups/payment-service-error-codes.csv` — catálogo de códigos de error de PaymentService.
> - `lookups/payment-service-user-lookup.csv` — usuarios (id, tier Diamond, etc.).
> - `lookups/ip_geo_locations.csv` — IP -> ciudad/país.
> - `lookups/user-agent-parsed.jsonl` — user-agents parseados (browser/OS/versión).
> - `lookups/web_path_vulnerabilities.jsonl` — rutas maliciosas + CVE (incluye `/info.php` -> CVE-2019-11043).
> - `lookups/gigamon_ip_enriched_table.csv` y `tls_cipher_suites_table.csv` — enriquecimiento de red.
> - `notebooks/*.json` — plantillas DQL oficiales (PaymentService 2.439 logs, Nginx 1.000, Gigamon 7.425).

> **Importante**: en cada notebook reemplazar `log.source == "<paste-your-unique-id-here>"` por el
> **unique id del equipo**. Es la llave para aislar los datos propios.

---

## 2100 — Infraestructura (Kubernetes Explorer) — 10 preguntas

`Dynatrace > Infrastructure > Kubernetes`, clúster del reto.

| Pregunta | Dónde obtenerla |
|---|---|
| 2101 Versión del clúster | Kubernetes Explorer -> cabecera del clúster |
| 2102 Namespace con backoff | Events -> filtrar por `BackOff` / `Kubernetes events` |
| 2103 nº pods en `kube-system` | Namespace `kube-system` -> contador de pods |
| 2104 Contenedor con reinicios en `unguard` | Namespace `unguard` -> `Restarts` |
| 2105 CPU limit de `broker-service` | Deployment -> Resource requests/limits (`Limits cpu`) |
| 2106 containerPort de `unguard-frontend` | Deployment -> Pod spec -> `containerPort` |
| 2107 Servicio con fallas en `credit-card-order-service` | Pod -> Problems/services |
| 2108 DynaKube mode | K8s cluster -> properties -> `dynakube.mode` |
| 2109 SO de los nodos | Nodos -> `OS` |
| 2110 Tipo de instancia de nodos | Nodos -> `instance type` (p.ej. `m5.large`) |

Consulta DQL útil (eventos):
```
fetch events
| filter k8s.namespace.name == "<ns>"
| filter event.type == "K8S_POD_BACKOFF" or matchesPhrase(content, "Back-off")
| summarize count(), by:{k8s.namespace.name}
```

## 2200 — Logs (DQL) — 26 preguntas

### Sub-categoría 1 — PaymentService (2201–2210)

Estructura del `content` (XML):
```
INFO [PaymentServiceThread] '<paymentservice ...><transaction type="payment" stage="execution" id="...">
<card lastFour="5642" zip="97504" exp="0829" /><user id="u552293" />
<amount value="328.13" currency="USD" tax="16.41" />
<result status="FAIL"><error code="5012" /><error code="5010" /><error code="1011" /></result></transaction>...'
```

- **2201** total de logs con `FAIL`:
  ```
  fetch logs | filter log.source == "<id>"
  | search "[PaymentServiceThread]"
  | filter matchesPhrase(content, "status=\"FAIL\"")
  | summarize logs = count()
  ```
- **2202** código de error más frecuente:
  ```
  fetch logs | filter log.source == "<id>"
  | parse content, "LD '<error code=\"' INT'"  as code"        // repetir/expand
  | expand code
  | summarize c = count(), by:{code}
  | sort c desc | limit 5
  ```
  (cruzar con `payment-service-error-codes.csv` para el nombre).
- **2203** devoluciones validadas: filtrar `<transaction type="refund" ... <result status="SUCCESS">`.
- **2204** importe medio reembolsado: `parse amount value` + `summarize avg(amount)` sobre refunds.
- **2205** revenue de pagos validados: `sum(value)` de pagos con `status="SUCCESS"`.
- **2206** total de usuarios: `lookup` sobre `payment-service-user-lookup` -> `count()`.
- **2207** Margaret Wilson: obtener su `user id` en el user-lookup, filtrar los logs por
  `matchesPhrase(content, "<user-id>")` y leer el `<error code=...>` resultante.
- **2208** afirmación "invalid transaction id": contar transacciones con error `1010`/`1020` en
  `stage="validation"` y verificar el ID indicado.
- **2209** miembros **Diamond** con fallas técnicas: join user-lookup (tier Diamond) + errores `50x`.
- **2210** revenue perdido Diamond: `sum(amount)` de esas transacciones `50x` fallidas.

### Sub-categoría 2 — Nginx access logs (2211–2217)

Formato: `$client_ip - [$ts] "$request" $status $size "$referrer" "$user_agent"`.
- **2211** POST: `| parse content, "LD 'POST' LD"` o `matchesPhrase(content, "\"POST ")`.
- **2212** tamaño medio de respuesta: parse del `$size` + `summarize avg(size)`.
- **2213** Chrome en Android < 10 a `broker-service`: parse del user-agent + `lookup user-agent-parsed.jsonl`
  (browser=Chrome, OS=Android, os_major < 10) y filtrar request conteniendo `broker-service`.
- **2214** solicitudes de EE.UU.: extraer IP cliente + `lookup ip_geo_locations.csv` (country = United States).
- **2215** CVE-2019-11043: la ruta es `/info.php` (`web_path_vulnerabilities.jsonl`); filtrar esa ruta,
  geolocalizar la IP y devolver la **ciudad**.
- **2216** accesos a `/`: request path == `/`.
- **2217** 3ª IP con más tráfico: `summarize c=count(), by:{client_ip} | sort c desc | limit 3` (tomar la 3ª).

### Sub-categoría 3 — Network / Gigamon (2218–2226)

- **2218** apps distintas: `summarize countDistinct(app_name)`. (Los registros traen `app_name`.)
- **2219** orígenes distintos: `countDistinct(src_ip)`.
- **2220** destinos distintos: `countDistinct(dst_ip)`.
- **2221** top-3 apps por TCP RTT: `summarize avg(tcp_rtt), by:{app_name} | sort desc | limit 3`.
- **2222** apps personales de 172.16.133.66: filtrar por `src_ip`/`dst_ip` == esa IP y contar `app_name` de uso personal.
- **2223** bytes de 172.16.133.66: `sum(src_bytes)+sum(dst_bytes)`.
- **2224** servidores con TLS débil (<256 bits): join `tls_cipher_suites_table.csv` por cipher,
  filtrar `key_size < 256` y quedarse solo con **IPs privadas**; responder separadas por comas.
- **2225** ubicación con más tráfico P2P: filtrar apps `BitTorrent, eDonkey, Kazaa, Gnutella, Ares`,
  geolocalizar el dispositivo interno y sumar bytes enviados+recibidos.
- **2226** asset_id de minería: filtrar apps `bitcoin, ethereum, ethereum-nd, minexmr-com, monero`
  (entrante y saliente) agrupando por `asset_id`; devolver el más frecuente.

## 2300 — Trazas (Distributed Traces) — 9 preguntas

`Applications & Microservices > Distributed Traces` / Service Map.
DQL sobre spans cuando aplique:
```
fetch spans | filter service.name == "AccountControllerV2"
| summarize avg(duration)                        // 2301 (ms)
```
- **2301** latencia media AccountControllerV2 (7d, ms).
- **2302** request rate AdService (`summarize count()/...` -> req/min).
- **2303** query con mayor duración acumulada en BrokerService (7d) -> copiar query completa.
- **2304** query más rápida en promedio en OrderController (10d) -> copiar query completa.
- **2305/2306** excepción de span en OrderController (5d) y su *exception class* (traza asociada).
- **2307** stack trace de `java.lang.ArithmeticException` -> `Clase:Línea`.
- **2308** namespace de la base de datos en la traza.
- **2309** `Db operation name` usado en la interacción.

## 2400 — Seguridad (Application Security) — 6 preguntas

`Application Security > Vulnerabilities` y `Security Posture`.
- **2401** componente a corregir en vulnerabilidad **S-99**.
- **2402** entidad afectada por **CVE-2024-47764**.
- **2403** versión del componente afectado por **S-246**.
- **2404** STIG Vulnerability ID de *"Kubernetes must remove old components after updated versions have been installed"*.
- **2405** único namespace **no relevante** en *"Ensure that all Namespaces have Network Policies defined"*.
- **2406** namespace en estado **Manual** en la regla *"Minimize the admission of containers wishing to share the host process ID namespace"*.

## 2500 — Experiencia de Usuario (DEM/RUM) — 13 preguntas

`Digital Experience > Web Applications` (app **easytrade**), Session Segmentation, Error Inspector,
Business Analytics.
- **2501** browser más usado. **2502** país con mejor Apdex (30d). **2503** país con más errores (30d).
- **2504** excepción frontend más frecuente (Error Inspector). **2505** ISP con más sesiones.
- **2506** `Visually Complete` de `loading of page/login` (segundos).
- **2507** acción con fallas del usuario *Jessica*. **2508** ubicación de conexión de *Catherine*.
- **2509/2510** paso con fallas del business flow *Airline Ticket Booking Flow* y su `event.type`.
- **2511** ruta con más bookings (dashboard, 7d) — formato `MEX-PTY`.
- **2512** pasarela de pago con más fallas. **2513** aerolínea con más tickets vendidos.

---

## Puntaje objetivo

| Categoría | Preguntas | Puntos |
|---|---|---|
| Infraestructura (2100) | 10 | 450 |
| Logs (2200) | 26 | 2,600 |
| Trazas (2300) | 9 | 405 |
| Seguridad (2400) | 6 | 270 |
| Experiencia de usuario (2500) | 13 | 780 |
| **Total** | **64** | **4,505** |
