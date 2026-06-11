# Registro de Defectos de Rendimiento

**Curso:** Testing y Validación de Software  
**Proyecto:** Pruebas de Carga y Rendimiento – JMeter  
**Equipo:** Cortés · Grandas · Bustos  
**Fecha de ejecución:** 2026-06-10 / 2026-06-11  
**Sistema bajo prueba:** API Registraduría – POST /register (localhost:8080)

---

## Introducción

Este documento recopila los defectos de rendimiento identificados durante la ejecución de los seis escenarios de prueba (Baseline, Load, Stress, Spike, Soak y Regresión). Cada defecto incluye evidencia técnica real, análisis de impacto y propuesta de mejora.

---

## Defecto PERF-01 — TCP BindException: agotamiento de puertos efímeros en Windows

- **Identificador:** PERF-01
- **Capa afectada:** Infraestructura de prueba (cliente JMeter en Windows)
- **Escenario:** Load (>50 VUs), Stress, Spike
- **SLO violado:** Error rate < 1%
- **Resultado esperado:** Error rate < 1% bajo cualquier nivel de carga
- **Resultado obtenido:** 33% (Load 50 VUs) → 65% (Load 100 VUs) → 89% (Stress 600 VUs)

### Evidencia

```
JTL – responseCode: "Non HTTP response code: java.net.BindException"
JTL – responseMessage: "Address already in use: connect"

Load Etapa 1 (50 VUs):  17 960 errores de 54 139 total = 33.17% BindException
Load Etapa 2 (100 VUs): 101 673 errores de 157 492 total = 64.56% BindException
Stress E3 (600 VUs):    410 715 errores de 459 349 total = 89.41% BindException
```

### Causa raíz

Windows reserva puertos efímeros en el rango 49152–65535 (16 383 puertos) con periodo TIME_WAIT de 240 s. Cuando JMeter genera peticiones sin think time, cada request consume un puerto que no se libera por 240 s. El throughput máximo sostenible es:

```
16 383 puertos / 240 s ≈ 68 req/s
```

Con 100 VUs sin timer: 100 × 5 req/s = 500 req/s >> 68 req/s → agotamiento en segundos.

### Impacto

Las peticiones que sí llegan al servidor responden correctamente (p95=64ms, 0% errores HTTP). El impacto real es sobre la capacidad de prueba, no sobre el sistema evaluado. Sin embargo, impide medir el rendimiento del API bajo alta carga desde un entorno Windows.

### Propuesta de mejora

1. Ejecutar JMeter desde Linux (sin limitación agresiva de TIME_WAIT)
2. Reducir TIME_WAIT en Windows: `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters` → `TcpTimedWaitDelay = 30`
3. Activar HTTP Keep-Alive en JMX: `<boolProp name="HTTPSampler.use_keepalive">true</boolProp>`

### Estado

Abierto (limitación de infraestructura de prueba)

### Prioridad

Alta

---

## Defecto PERF-02 — HTTP 406 por header `Accept: application/json` incompatible con `text/plain`

- **Identificador:** PERF-02
- **Capa afectada:** Configuración del cliente JMeter (HTTP Header Manager)
- **Escenario:** Todos los escenarios (detectado en primera ejecución)
- **SLO violado:** Error rate < 1%
- **Resultado esperado:** HTTP 200 con cuerpo `VALID`, `DUPLICATED` o `DEAD`
- **Resultado obtenido:** 100% HTTP 406 Not Acceptable

### Evidencia

```
JTL – responseCode: 406
JTL – responseMessage: Not Acceptable

Todas las peticiones del plan original fallaban:
summary = 15 000 in 00:05:00 = 50.0/s  Avg: 12  Err: 15 000 (100.00%)
```

### Causa raíz

El `RegistryController` de Spring Boot está anotado con `produces = MediaType.TEXT_PLAIN_VALUE`, lo que significa que solo puede responder con `text/plain`. El HTTP Header Manager de JMeter tenía configurado `Accept: application/json`. Spring aplica content negotiation y rechaza la petición con HTTP 406 porque no puede satisfacer el header `Accept`.

```java
// RegistryController.java
@PostMapping(consumes = APPLICATION_JSON_VALUE, produces = TEXT_PLAIN_VALUE)
public String register(@RequestBody Person person) { ... }
```

### Solución aplicada

Cambiar el header `Accept` en todos los JMX de `application/json` a `*/*`:

```xml
<!-- Antes (causaba HTTP 406) -->
<stringProp name="Header.name">Accept</stringProp>
<stringProp name="Header.value">application/json</stringProp>

<!-- Después (correcto) -->
<stringProp name="Header.name">Accept</stringProp>
<stringProp name="Header.value">*/*</stringProp>
```

### Estado

Resuelto ✅

### Prioridad

Crítica

---

## Defecto PERF-03 — HTTP 400 por caracteres acentuados en nombres del CSV

- **Identificador:** PERF-03
- **Capa afectada:** Dataset de prueba (`perf/data/personas.csv`) / Codificación JMeter Windows
- **Escenario:** Todos los escenarios (detectado en segunda ejecución)
- **SLO violado:** Error rate < 1%
- **Resultado esperado:** HTTP 200 para todos los registros del CSV
- **Resultado obtenido:** ~75% HTTP 400 Bad Request para nombres con tildes

### Evidencia

```
JTL – responseCode: 400
JTL – responseMessage: Bad Request

Peticiones con "Ana García", "José López" → HTTP 400
Peticiones con "Maria Perez", "Ana Torres" → HTTP 200 VALID

# Distribución de errores:
Registros con acentos (á,é,í,ó,ú,ñ): 100% HTTP 400
Registros sin acentos (ASCII puro):   100% HTTP 200
```

### Causa raíz

JMeter en Windows con configuración UTF-8 incorrecta enviaba los caracteres acentuados como bytes malformados en el JSON body. Spring's `ObjectMapper` no podía parsear el JSON y retornaba HTTP 400 Bad Request. El problema es específico de la combinación JMeter + Windows + nombres en español con tildes.

### Solución aplicada

Reescribir el archivo `perf/data/personas.csv` con nombres ASCII-only, eliminando todas las tildes y caracteres especiales:

```
# Antes (causaba HTTP 400 en Windows)
1,Ana García,30,FEMALE,true
2,José López,25,MALE,true

# Después (correcto, solo ASCII)
1,Ana Garcia,30,FEMALE,true
2,Jose Lopez,25,MALE,true
```

### Estado

Resuelto ✅

### Prioridad

Alta

---

## Defecto PERF-04 — p99 supera SLO a partir de 200 VUs en Stress Test

- **Identificador:** PERF-04
- **Capa afectada:** Servidor de aplicación (Spring Boot / Tomcat embebido)
- **Escenario:** Stress Test Etapa 1 (200 VUs), Etapa 2 (400 VUs), Etapa 3 (600 VUs)
- **SLO violado:** p99 < 800 ms
- **Resultado esperado:** p99 < 800ms bajo carga de estrés
- **Resultado obtenido:** p99 = 2,559ms (200 VUs) / 2,440ms (400 VUs) / 4,497ms (600 VUs)

### Evidencia

```
Stress Etapa 1 (200 VUs):
  p90=70ms  p95=93ms  p99=2 559ms  ← SLO p99 violado (800ms)

Stress Etapa 2 (400 VUs):
  p90=274ms  p95=298ms  p99=2 440ms  ← SLO p99 violado

Stress Etapa 3 (600 VUs):
  p90=2 091ms  p95=2 170ms  p99=4 497ms  ← SLO p95 y p99 violados
```

### Causa probable

El thread pool de Tomcat (200 threads por defecto) comienza a saturarse cuando los VUs superan el número de threads disponibles. Las peticiones que exceden la capacidad quedan encoladas (`accept-count=100`). El tiempo de cola dispara el p99 porque el 1% más lento representa precisamente las peticiones que esperaron en cola más tiempo.

### Propuesta de mejora

```properties
# application.properties
server.tomcat.threads.max=500
server.tomcat.threads.min-spare=50
server.tomcat.max-connections=10000
server.tomcat.accept-count=200
```

### Estado

Abierto

### Prioridad

Media (solo se manifiesta bajo carga 20x superior a la nominal)

---

## Defecto PERF-05 — p95 supera SLO durante Spike de 300 VUs

- **Identificador:** PERF-05
- **Capa afectada:** Servidor de aplicación (absorción de picos)
- **Escenario:** Spike Test – Fase de pico (300 VUs en 30 s)
- **SLO violado:** p95 < 300 ms
- **Resultado esperado:** p95 < 300ms incluso durante el pico de tráfico
- **Resultado obtenido:** p95 = 366ms durante el pico (22% sobre el SLO)

### Evidencia

```
Spike – Fase Base (10 VUs):
  Avg=14ms  p95=25ms  p99=32ms  ← Dentro del SLO

Spike – Fase Pico (300 VUs):
  Avg=295ms  p95=366ms  p99=380ms  ← p95 viola SLO (300ms)

Recuperación: inmediata al bajar los VUs — p95 vuelve a <30ms
```

### Causa probable

El pool de threads de Tomcat no puede absorber un aumento instantáneo de 10 a 300 VUs en 30 segundos. El thread pool crece gradualmente (JVM overhead de creación de threads), generando una cola de peticiones temporalmente alta que dispara la latencia p95 por encima del SLO.

### Propuesta de mejora

1. Aumentar `server.tomcat.threads.min-spare=100` para mantener threads pre-calentados
2. Implementar circuit breaker con Resilience4j para retornar respuesta de degradación controlada antes de que la latencia supere 300ms
3. Configurar auto-scaling (si se despliega en contenedor) que reaccione en <30s al incremento de tráfico

### Estado

Abierto

### Prioridad

Alta (el escenario de apertura masiva es un caso de uso real del sistema electoral)

---

## Tabla de Seguimiento

| ID | Escenario | SLO Violado | Resultado Obtenido | Estado | Prioridad |
|----|-----------|------------|-------------------|--------|-----------|
| PERF-01 | Load/Stress/Spike (>50 VUs) | Error rate < 1% | 33-89% BindException TCP | Abierto | Alta |
| PERF-02 | Todos (1ª ejecución) | Error rate < 1% | 100% HTTP 406 | Resuelto | Crítica |
| PERF-03 | Todos (1ª ejecución) | Error rate < 1% | ~75% HTTP 400 | Resuelto | Alta |
| PERF-04 | Stress (200-600 VUs) | p99 < 800ms | 2,440-4,497ms | Abierto | Media |
| PERF-05 | Spike (300 VUs pico) | p95 < 300ms | p95 = 366ms | Abierto | Alta |

---

## Convenciones

| Estado | Descripción |
|--------|-------------|
| Abierto | Identificado sin corrección aplicada |
| En progreso | Corrección en proceso |
| Resuelto | Corregido y validado con nuevas pruebas |

---

Universidad de La Sabana – Facultad de Ingeniería  
Curso: Testing y Validación de Software – 2026
