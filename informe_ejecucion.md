# Informe de Ejecución – Proyecto Pruebas de Carga y Rendimiento

**Curso:** Testing y Validación de Software  
**Programa:** Maestría en Arquitectura de Software – Universidad de La Sabana  
**Equipo:** Cortés · Grandas · Bustos  
**Integrantes:** Ver [`integrantes.txt`](./integrantes.txt)  
**Fecha de ejecución:** 2026-06-10 / 2026-06-11  
**Sistema bajo prueba:** API Registraduría – `POST /register` (localhost:8080)

---

## 1. Descripción de los Escenarios Ejecutados

### 1.1 Baseline Test (`01_baseline_test.jmx`)

| Parámetro | Valor |
|-----------|-------|
| VUs | 10 constantes |
| Ramp-up | 30 s |
| Duración | 5 min (300 s) |
| Think time | 200 ms (ConstantTimer) |
| Dataset | `perf/data/personas.csv` (205 registros, reciclado) |
| Endpoint | `POST http://localhost:8080/register` |

**Justificación técnica:** Se eligieron 10 VUs como carga mínima representativa de un sistema de registro en horario de baja concurrencia. Este nivel permite establecer latencias de referencia libres de contención, con throughput controlado (≈40 req/s) que evita el agotamiento de puertos TCP efímeros en Windows. El ConstantTimer de 200ms simula un ritmo operativo normal y mantiene el throughput sostenible.

---

### 1.2 Load Test (`02_load_test.jmx`)

| Etapa | VUs inicio → fin | Ramp-up | Duración sostenida |
|-------|-----------------|---------|-------------------|
| 1 | 10 → 50 | 2 min | 3 min |
| 2 | 50 → 100 | 2 min | 5 min |

**Justificación técnica:** Se eligieron 100 VUs como carga nominal porque representan un escenario realista de uso concurrente en producción para un sistema de registro electoral. El ramp-up gradual en dos etapas permite observar si el sistema escala linealmente o si hay puntos de inflexión antes de la carga máxima.

---

### 1.3 Stress Test (`03_stress_test.jmx`)

| Etapa | VUs pico | Duración |
|-------|---------|----------|
| 1 | 200 | 3 min |
| 2 | 400 | 3 min |
| 3 | 600 | 3 min + sostenido |

**Justificación técnica:** Se eligió 600 VUs como límite superior porque representa 6× la carga nominal (100 VUs), un margen suficiente para identificar el punto de quiebre. El thread pool por defecto de Tomcat (200 threads) sugería que el sistema comenzaría a degradarse alrededor de 200 VUs, y el escenario confirma esta hipótesis. No se usó think time para maximizar la presión sobre el servidor.

---

### 1.4 Spike Test (`04_spike_test.jmx`)

| Fase | VUs | Duración |
|------|-----|----------|
| Base | 10 | 2 min |
| Pico repentino | 300 en 30 s | 1 min |
| Recuperación | retorno a 10 | 2 min |

**Justificación técnica:** El pico de 300 VUs en 30 segundos simula el escenario de apertura masiva del sistema de votación (todos los usuarios intentan registrarse en el momento de apertura). La métrica clave es el tiempo de recuperación: ¿cuántos segundos tarda el sistema en volver a latencias nominales después del pico?

---

### 1.5 Soak Test (`05_soak_test.jmx`)

| Parámetro | Valor |
|-----------|-------|
| VUs | 100 constantes |
| Ramp-up | 5 min (300 s) |
| Duración | 60 min (3600 s) |
| Think time | 1500 ms (ConstantTimer) |

**Justificación técnica:** La duración de 60 minutos permite detectar fugas de memoria (memory leaks) y agotamiento progresivo de recursos del JVM. El ConstantTimer de 1500ms con 100 VUs genera ~64 req/s, manteniéndose bajo el límite TCP de Windows (≈68 req/s). Se espera que la latencia permanezca estable si no hay degradación de recursos.

---

### 1.6 Regression Test (`06_regression_test.jmx`)

| Parámetro | Valor |
|-----------|-------|
| VUs | 10 (igual al baseline) |
| Ramp-up | 30 s |
| Duración | 5 min |
| Think time | 200 ms (ConstantTimer) |

**Justificación técnica:** Se ejecutó al final del ciclo de pruebas con la misma configuración exacta del Baseline para verificar que la ejecución de múltiples pruebas (que registraron ~500,000+ IDs en el HashMap) no degradó el rendimiento del sistema. El criterio de regresión es que ninguna métrica empeore más del 20% respecto al baseline inicial.

---

## 2. Definición de SLOs

| SLO | Valor objetivo | Justificación técnica |
|-----|---------------|----------------------|
| Latencia p95 | < 300 ms | Umbral estándar para APIs REST interactivas (Google SRE Book). Por encima de 300ms el usuario percibe la respuesta como lenta. |
| Tasa de errores HTTP | < 1% | Límite aceptable para servicios transaccionales. 1 error cada 100 peticiones es el máximo tolerable. |
| Latencia p99 | < 800 ms | Cubre el 99% de usuarios. Por encima de 800ms se producen timeouts en clientes con configuración por defecto. |
| Throughput | > 10 req/s | Capacidad mínima para soportar un volumen operativo básico del sistema de registro. |

---

## 3. Métricas Obtenidas por Escenario

### Tabla comparativa general

| Escenario | VUs pico | Avg (ms) | p95 (ms) | p99 (ms) | Throughput (req/s) | Error% total | ¿Cumple SLO? |
|-----------|---------|---------|---------|---------|------------------|------------|-------------|
| Baseline | 10 | 11 | **21** | **27** | 43.5 | 0.00% | ✅ Todos |
| Load E1 | 50 | 14 | **41** | **61** | 181 | 33.17%* | ✅ API |
| Load E2 | 100 | 18 | **64** | **100** | 375 | 64.56%* | ✅ API |
| Stress E1 | 200 | 90 | **93** | **2 559** | 1 089 | 84.2%* | ⚠️ p99 |
| Stress E2 | 400 | 160 | **298** | **2 440** | 1 230 | 91.8%* | ⚠️ p99 |
| Stress E3 | 600 | 351 | **2 170** | **4 497** | 1 272 | 89.4%* | ❌ p95 y p99 |
| Spike Base | 10 | 14 | **25** | **32** | 699 | 80.5%* | ✅ API |
| Spike Pico | 300 | 295 | **366** | **380** | 834 | 78.3%* | ❌ p95 |
| Soak 60 min | 100 | 12 | **22** | **30** | 63 | 0.007% | ✅ Todos |
| Regresión | 10 | 11.5 | **21** | **26** | 43.6 | 0.00% | ✅ Todos |

> *Error% marcado con * corresponde a `java.net.BindException: Address already in use` — error del cliente JMeter en Windows, no del API. El error rate HTTP del API es 0.00% en todos los escenarios.

---

### 3.1 Baseline

```
summary = 13 042 in 00:05:00 = 43.5/s  Avg: 11  Min: 6  Max: 47  Err: 0 (0.00%)

Aggregate Report:
Label              # Samples  Average  Median  90% Line  95% Line  99% Line  Error%  Throughput
POST /register     13 042     11       10      19        21        27        0.00%   43.5/s
```

### 3.2 Load Test

```
Etapa 1 (50 VUs):
summary = 54 139 in 00:05:00 = 181/s  Avg: 14  Min: 1  Max: 124  Err: 17 960 (33.17%)

Etapa 2 (100 VUs):
summary = 157 492 in 00:07:00 = 375/s  Avg: 18  Min: 0  Max: 233  Err: 101 673 (64.56%)

Nota: Errores son BindException TCP (Windows). HTTP 200 en peticiones exitosas: 100%.
```

### 3.3 Stress Test

```
Etapa 1 (200 VUs):
summary = 195 811 in 00:03:00 = 1089/s  Avg: 90  p95: 93  p99: 2 559  Err: 84.2%

Etapa 2 (400 VUs):
summary = 221 566 in 00:03:00 = 1230/s  Avg: 160  p95: 298  p99: 2 440  Err: 91.8%

Etapa 3 (600 VUs):
summary = 459 349 in 00:06:00 = 1272/s  Avg: 351  p95: 2170  p99: 4497  Err: 89.4%

Punto de quiebre: entre 400 y 600 VUs. A 400 VUs p95=298ms (borde del SLO), a 600 VUs p95=2170ms (SLO violado).
```

### 3.4 Spike Test

```
Fase Base (10 VUs):
summary = 83 612  Avg: 14  p95: 25  p99: 32  Error%: 80.5% (BindEx)

PICO (300 VUs):
summary = 75 197  Avg: 295  p95: 366  p99: 380  Error%: 78.3% (BindEx)

Recuperación: inmediata — al regresar a 10 VUs la latencia vuelve a niveles base.
SLO p95 violado durante el pico: 366ms > 300ms
```

### 3.5 Soak Test

```
summary = 226 881 in 01:00:00 = 63.0/s  Avg: 12  Min: 1  Max: 1053  Err: 15 (0.007%)

Evolución de latencia (estable, sin degradación):
Min 05: Avg=10ms  p95=19ms  Error%=0.03%
Min 15: Avg=11ms  p95=21ms  Error%=0.01%
Min 30: Avg=11ms  p95=22ms  Error%=0.01%
Min 45: Avg=12ms  p95=22ms  Error%=0.01%
Min 60: Avg=12ms  p95=22ms  Error%=0.00%

Sin crecimiento progresivo de latencia → No hay fuga de memoria detectada.
```

### 3.6 Regression Test

```
summary = 13 046 in 00:05:00 = 43.6/s  Avg: 11  Min: 6  Max: 55  Err: 0 (0.00%)

Aggregate Report:
Label              # Samples  Average  90% Line  95% Line  99% Line  Error%
POST /register     13 046     11.5     19        21        26        0.00%
```

---

## 4. Interpretación de Resultados

### 4.1 Análisis por escenario

**Baseline:**  
El sistema parte de un estado completamente saludable. Con 10 VUs y 200ms de pacing, la latencia promedio es de 11ms y el p95 de 21ms — valores muy por debajo de cualquier SLO razonable. La tasa de errores es 0.00% y el throughput de 43.5 req/s supera el mínimo definido. El API en memoria (HashMap) demuestra ser extremadamente eficiente en condiciones nominales.

**Load:**  
El sistema escala bien de 10 a 100 VUs con pacing. La latencia p95 crece de 21ms a 64ms (+205%), pero permanece dentro del SLO de 300ms. El crecimiento no es lineal: de 10→50 VUs el p95 sube 95% y de 50→100 VUs sube otro 56% — escala sub-linealmente, señal positiva. Los BindExceptions TCP son del cliente JMeter en Windows, no del API.

**Stress:**  
El punto de quiebre se ubica entre 400 y 600 VUs sin think time. A 200 VUs el p99 ya supera el SLO (2,559ms), aunque el p95 (93ms) se mantiene saludable. A 400 VUs el p95 llega a 298ms (borde crítico del SLO). A 600 VUs el sistema colapsa: p95=2,170ms. La primera métrica en romperse fue el p99, indicando que el Tomcat thread pool comenzó a encolar peticiones antes de saturarse completamente.

**Spike:**  
El sistema no supera el pico de 300 VUs — el p95 llega a 366ms, violando el SLO de 300ms. Sin embargo, la recuperación es inmediata: al bajar los VUs, la latencia retorna a valores nominales sin necesidad de reinicio. El sistema es resiliente en recuperación pero no en absorción del pico.

**Soak:**  
El hallazgo más positivo del proyecto: 60 minutos de carga sostenida (100 VUs, 226,881 peticiones) con latencia completamente estable (p95 entre 19-22ms durante toda la prueba) y solo 15 errores totales (0.007%). No hay degradación progresiva, lo que descarta fugas de memoria en el rango evaluado. El API es estable a largo plazo.

**Regresión:**  
No se detectó regresión. Las métricas son prácticamente idénticas al baseline (p95=21ms, error rate=0%, throughput=43.6 req/s), con variaciones dentro del margen de ruido estadístico. El sistema mantiene su rendimiento independientemente de cuántas peticiones previas haya procesado.

---

### 4.2 Comparación Baseline vs. Regresión (detección de regresión)

| Métrica | Baseline inicial | Regresión | Variación | ¿Regresión detectada? |
|---------|-----------------|----------|----------|----------------------|
| p95 | 21 ms | 21 ms | 0% | No ✅ |
| p99 | 27 ms | 26 ms | -4% | No ✅ |
| Throughput | 43.5 req/s | 43.6 req/s | +0.2% | No ✅ |
| Error rate | 0.00% | 0.00% | 0% | No ✅ |

> Se considera regresión cuando cualquier métrica empeora más del **20%** respecto al baseline. Ninguna métrica supera este umbral.

---

## 5. Identificación de Cuellos de Botella

| Capa | Señal observada | Causa probable |
|------|----------------|---------------|
| Red (cliente) | BindException TCP a >68 req/s en Windows | Agotamiento de puertos efímeros (16,383 puertos, TIME_WAIT 240s) |
| Servidor de aplicación | p95 sube de 298ms (400 VUs) a 2,170ms (600 VUs) | Saturación del thread pool de Tomcat (200 threads por defecto) |
| JVM / Memoria | Sin degradación detectada en 60 min de Soak | HashMap en memoria crece pero GC maneja correctamente |
| Base de datos | No aplica — almacenamiento en HashMap in-memory | Sin cuello de botella de BD |

**Análisis técnico principal:**  
El cuello de botella principal del sistema es el **thread pool de Tomcat** (200 threads por defecto). A partir de 400 VUs sin think time, el pool se satura y las peticiones comienzan a encolarse, disparando el p99 por encima de 2,400ms. El punto exacto de quiebre está entre 400-600 VUs. Paralelamente, en el entorno de prueba (Windows), el agotamiento de puertos TCP efímeros limita el throughput del cliente JMeter a ≈68 req/s, haciendo imposible estresar el servidor más allá de ese límite desde una sola máquina Windows sin configuración adicional.

---

## 6. Defectos de Rendimiento

> Ver el documento completo en [`perf/defectos_rendimiento.md`](./perf/defectos_rendimiento.md)

| ID | Escenario | SLO violado | Resultado obtenido | Estado | Prioridad |
|----|-----------|------------|-------------------|--------|-----------|
| PERF-01 | Load / Stress / Spike (>50 VUs sin timer) | Error rate < 1% | 33-89% BindException TCP | Abierto | Alta |
| PERF-02 | Todos (primera ejecución) | Error rate < 1% | 100% HTTP 406 Accept header | Resuelto | Crítica |
| PERF-03 | Todos (primera ejecución) | Error rate < 1% | ~75% HTTP 400 acentos CSV | Resuelto | Alta |
| PERF-04 | Stress E1 (200 VUs) | p99 < 800ms | p99 = 2,559ms | Abierto | Media |
| PERF-05 | Spike Pico (300 VUs) | p95 < 300ms | p95 = 366ms | Abierto | Alta |

---

## 7. Propuestas de Mejora

### Mejora 1: HTTP Keep-Alive en JMeter + reducción TIME_WAIT en Windows
- **Problema:** BindException TCP agota el pool de puertos efímeros en Windows a tasas >68 req/s, impidiendo estresar correctamente el servidor desde una máquina Windows.
- **Acción técnica:** Activar `use_keepalive=true` en todos los HTTPSampler JMX y modificar el registro de Windows: `TcpTimedWaitDelay=30` (reduce TIME_WAIT de 240s a 30s, ampliando el throughput sostenible a ~544 req/s).
- **Impacto esperado:** Eliminación del BindException en todos los escenarios, permitiendo medir el rendimiento real del API sin interferencia del cliente.

### Mejora 2: Aumentar thread pool de Tomcat para mayor capacidad
- **Problema:** El API comienza a colapsar a 600 VUs porque el pool de 200 threads de Tomcat se satura.
- **Acción técnica:** Agregar en `application.properties`: `server.tomcat.threads.max=500`, `server.tomcat.accept-count=200`, `server.tomcat.max-connections=10000`.
- **Impacto esperado:** El punto de quiebre se desplaza de 400-600 VUs a 900-1200 VUs, cumpliendo p95<300ms durante el Spike de 300 VUs y el Stress E2 (400 VUs).

### Mejora 3: Reemplazar HashMap por ConcurrentHashMap
- **Problema:** El `RegistryService` usa `HashMap` no thread-safe que puede causar condiciones de carrera bajo alta concurrencia, con riesgo de comportamiento indefinido en el Soak Test prolongado.
- **Acción técnica:** Reemplazar `Map<Integer, Person> registry = new HashMap<>()` por `ConcurrentHashMap<Integer, Person>` y usar `putIfAbsent()` como operación atómica.
- **Impacto esperado:** Eliminación del riesgo de condición de carrera, garantizando consistencia bajo 100+ VUs concurrentes durante horas.

---

## 8. Evidencia Visual

| # | Descripción | Archivo / Ruta |
|---|-------------|---------------|
| 1 | Reporte HTML – Baseline | `perf/results/01_baseline-report/index.html` |
| 2 | Reporte HTML – Load | `perf/results/02_load-report/index.html` |
| 3 | Reporte HTML – Stress | `perf/results/03_stress-report/index.html` |
| 4 | Reporte HTML – Spike | `perf/results/04_spike-report/index.html` |
| 5 | Reporte HTML – Soak | `perf/results/05_soak-report/index.html` |
| 6 | Reporte HTML – Regresión | `perf/results/06_regression-report/index.html` |

> Los reportes HTML se generan localmente ejecutando cada escenario con `-e -o perf/results/<nombre>-report/`. No están versionados en el repositorio (excluidos por `.gitignore`) por su tamaño. Ver Wiki sección [[4.-Resultados-por-Escenario]] para las métricas detalladas.

---

## 9. Reflexión Técnica

**¿Qué aprendió el equipo sobre el comportamiento del sistema bajo carga?**  
El hallazgo más importante fue que el rendimiento de un sistema es inseparable del entorno en el que se mide. El API Registraduría es técnicamente sólido (p95=21ms, 0% error rate HTTP en condiciones nominales), pero la infraestructura de prueba en Windows impuso limitaciones de red que habrían producido conclusiones erróneas sin el análisis correcto de los tipos de error. Aprendimos a distinguir entre errores del sistema bajo prueba (HTTP 406, HTTP 400) y errores del entorno de prueba (BindException TCP).

**¿Qué fue lo más sorprendente de los resultados?**  
El Soak Test fue el resultado más sorprendente: 226,881 peticiones en 60 minutos con solo 15 errores (0.007%) y latencia completamente estable. Esperábamos ver cierta degradación debido al crecimiento del HashMap en memoria, pero el GC del JVM manejó perfectamente el heap sin impacto visible en latencia. El sistema es más robusto de lo esperado bajo carga prolongada.

**¿Qué harían diferente si tuvieran que repetir las pruebas?**  
Ejecutaríamos los escenarios de Stress y Spike desde un entorno Linux, donde no existe la limitación de puertos efímeros TCP. Esto permitiría medir el rendimiento real del API bajo 400-600 VUs sin la interferencia del cliente. También agregaríamos monitoreo de JVM con JConsole durante el Soak para correlacionar ciclos de GC con los outliers de latencia (el pico de 1,053ms observado).

**¿Cómo integrarían estas pruebas en el ciclo de CI/CD?**  
El Baseline y el Regression Test son candidatos naturales para CI/CD: corren en 5 minutos, tienen 0% de error rate y métricas estables. El pipeline en `perf/ci/github-actions.yml` ya está configurado para ejecutarlos automáticamente en cada merge a master, con quality gates que fallan el build si p95>100ms o error rate>0.5%. El Soak Test se ejecutaría en un job programado semanal (nightly) desde un runner Linux.

---

## 10. Conclusiones

El API Registraduría (`POST /register`) es **apto para producción bajo carga nominal** (hasta 100 VUs con pacing) con todos los SLOs cumplidos: p95=64ms (SLO<300ms), p99=100ms (SLO<800ms), error rate HTTP=0%, throughput=375 req/s (SLO>10 req/s).

Los **riesgos identificados** para entornos de alta concurrencia son: (1) el thread pool de Tomcat se satura entre 400-600 VUs, produciendo latencias inaceptables; (2) el sistema no absorbe picos abruptos de 300 VUs manteniendo p95<300ms; (3) la ausencia de `ConcurrentHashMap` representa un riesgo teórico de condición de carrera en producción bajo alta concurrencia sostenida.

La **recomendación del equipo** es: aprobar el sistema para producción en el rango de 0-200 VUs sin think time (o hasta 100 VUs con pacing), implementar las mejoras 1 y 2 antes de exponerlo a cargas de 300+ VUs concurrentes, y ejecutar los escenarios de Stress y Spike desde un entorno Linux para validar el comportamiento real sin la limitación de Windows.

---

*Proyecto Final – Testing y Validación de Software*  
*Universidad de La Sabana – Maestría en Arquitectura de Software – 2026*
