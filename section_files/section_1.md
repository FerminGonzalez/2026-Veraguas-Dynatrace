> ***Ir a***: [Menú principal](../README.md) ***||*** [Siguiente sección: Infraestructura — Kubernetes Cluster Analysis](section_2.md)

# Contenido

[[_TOC_]]

# 01 Propósito del Reto: Construyendo una plataforma de Observabilidad integral con Dynatrace

En el entorno tecnológico actual, las organizaciones enfrentan presiones crecientes para mantener la disponibilidad, el rendimiento y la seguridad de sus sistemas distribuidos. Los incidentes no detectados a tiempo pueden costar millones de dólares, impactar la reputación y comprometer la experiencia del usuario final. Por lo que la observabilidad se ha convertido en una capacidad estratégica para cualquier equipo de ingeniería moderno.

El #hackathonCopa desafía a los participantes a desarrollar habilidades prácticas en observabilidad utilizando **Dynatrace** como plataforma central. Los equipos trabajarán con datos reales de un entorno Kubernetes monitoreado, analizando métricas de infraestructura, logs estructurados, trazas distribuidas, vulnerabilidades de seguridad y experiencia de usuario digital.

# Descripción general

En este reto, los participantes deberán demostrar dominio de las capacidades de Dynatrace para investigar y responder preguntas concretas sobre el estado operativo de un entorno de producción. El reto, abarca cinco dominios principales de observabilidad que reflejan los desafíos reales de un equipo de SRE (Site Reliability Engineering) o plataforma:

* **Infraestructura**: Análisis del clúster Kubernetes, workloads, namespaces y configuración de pods.
* **Logs**: Investigación de logs de aplicaciones y red usando Dynatrace Log Analytics con comandos DQL.
* **Trazas**: Análisis de trazas distribuidas, servicios, queries de base de datos y excepciones.
* **Seguridad**: Evaluación de vulnerabilidades de software, CVEs, STIGs y posturas de cumplimiento.
* **Experiencia de Usuario**: Análisis de sesiones de usuario, errores de frontend, Apdex y business flows con DEM.

Los participantes deberán demostrar dominio de las herramientas nativas de Dynatrace incluyendo:

* Kubernetes Explorer
* Log & Event Viewer / DQL (Dynatrace Query Language)
* Distributed Tracing & Service Map
* Application Security
* Digital Experience Monitoring (DEM / RUM)
* Live Debugger
* Dashboards y Business Analytics

# Recomendaciones del reto

* Se recomienda exploren la plataforma Dynatrace sistemáticamente antes de responder cada pregunta.
* Utilicen los filtros de tiempo adecuados — muchas preguntas requieren un rango específico.
* Lean con atención el enunciado de cada pregunta: el formato exacto de la respuesta es importante.
* Para las preguntas de logs, usen el template DQL provisto en cada enlace.
* Las pistas (hints) tienen un costo en puntos — úsenlas solo cuando sea necesario.
* Todas las preguntas son de **respuesta abierta**: escriban exactamente el valor encontrado en la plataforma.

---

## Estructura de puntos

Cada pregunta tiene un valor calculado según su complejidad:

| Complejidad | Factor | Valor base | Puntos por pregunta |
|---|---|---|---|
| 0 (Init/Unlock) | 10 | 1 | 10 |
| 1 (Básico) | 15 | 3 | 45 |
| 2 (Intermedio) | 15 | 3–4 | 45–60 |
| 3 (Avanzado) | 20 | 5 | 100 |

> ***Ir a***: [Menú principal](../README.md) ***||*** [Siguiente sección: Infraestructura — Kubernetes Cluster Analysis](section_2.md)
