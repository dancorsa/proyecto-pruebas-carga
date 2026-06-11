# Registro de Defectos de Rendimiento

**Curso:** Testing y Validación de Software  
**Proyecto:** Pruebas de Carga y Rendimiento – JMeter  
**Equipo:** [Nombre del equipo]  
**Fecha de ejecución:** [Fecha]  
**Sistema bajo prueba:** API Registraduría – POST /register (localhost:8080)

---

## Introducción

Este documento recopila los defectos de rendimiento identificados durante la ejecución de los seis escenarios de prueba (Baseline, Load, Stress, Spike, Soak y Regresión). Cada defecto incluye evidencia técnica, análisis de impacto y propuesta de mejora.

---

## Defecto PERF-01 — Incumplimiento de SLO de latencia bajo Load Test

- **Identificador:** PERF-01
- **Capa afectada:** Aplicación / Base de datos
- **Escenario:** Load Test (100 VUs, Etapa 2)
- **SLO violado:** p95 < 300 ms
- **Resultado esperado:** p95 < 300 ms bajo carga nominal de 100 VUs
- **Resultado obtenido:** p95 = 612 ms

### Evidencia

```
Aggregate Report – Load Test (Etapa 2, 100 VUs):
Label             # Samples  Average  90% Line  95% Line  99% Line  Error%  Throughput
POST /register    18,420     402      548       612       890       0.12%   60.1/s
```

### Impacto

Incumplimiento del objetivo de nivel de servicio bajo carga nominal. Los usuarios finales experimentan tiempos de respuesta superiores al umbral aceptable (300ms) bajo condiciones normales de operación. Afecta la experiencia de usuario en producción.

### Causa probable

- Saturación del pool de conexiones HikariCP (configurado con `maximumPoolSize=10`).
- Ausencia de índice en la columna `id` de la tabla `persons`, forzando full table scan.

### Propuesta de mejora

- Incrementar `spring.datasource.hikari.maximum-pool-size=30` en `application.properties`.
- Crear índice: `CREATE INDEX idx_persons_id ON persons(id);`
- Considerar caché de primer nivel con `@Cacheable` para consultas repetidas.

### Estado

Abierto

### Prioridad

Alta

---

## Defecto PERF-02 — Error rate crítico bajo Stress Test

- **Identificador:** PERF-02
- **Capa afectada:** Servidor de aplicación (Spring Boot / Tomcat embebido)
- **Escenario:** Stress Test Etapa 3 (600 VUs)
- **SLO violado:** Error rate < 1%
- **Resultado esperado:** Error rate < 1% incluso bajo carga extrema hasta el punto de quiebre
- **Resultado obtenido:** Error rate = 4.8%

### Evidencia

```
Summary Report – Stress Test (Etapa 3, 600 VUs):
summary = 32,000 in 00:06:00 = 88.9/s  Avg: 1,240  Min: 43  Max: 9,800  Err: 1,536 (4.80%)

Errores encontrados:
- java.net.SocketTimeoutException: Read timed out (82% de errores)
- org.apache.catalina.connector.ClientAbortException (18% de errores)
```

### Impacto

Sistema inasequible para el 4.8% de usuarios bajo carga de 600 VUs. Afecta directamente la disponibilidad del servicio y viola el SLO de error rate. Bajo estos niveles de carga, ~1 de cada 20 requests falla.

### Causa probable

- Agotamiento del pool de threads de Tomcat (configuración por defecto `maxThreads=200`).
- Timeout de respuesta del servidor inferior al tiempo de procesamiento bajo alta concurrencia.
- Posible deadlock en transacciones de base de datos concurrentes.

### Propuesta de mejora

- Incrementar `server.tomcat.max-threads=400` y `server.tomcat.accept-count=200`.
- Implementar circuit breaker con Resilience4j para degradación controlada.
- Aumentar timeout de respuesta JMeter a 20,000ms para distinguir errores reales de timeouts de cliente.

### Estado

En progreso

### Prioridad

Crítica

---

## Defecto PERF-03 — Degradación progresiva de latencia en Soak Test

- **Identificador:** PERF-03
- **Capa afectada:** JVM / Memoria del servidor
- **Escenario:** Soak Test (100 VUs, 60 minutos)
- **SLO violado:** Latencia p95 debe mantenerse estable (variación < 20%)
- **Resultado esperado:** p95 constante entre 180-250 ms durante toda la prueba
- **Resultado obtenido:** p95 creció de 210 ms (min. 5) a 680 ms (min. 60)

### Evidencia

```
Evolución de latencia promedio durante el Soak Test:
Minuto  5:  Avg=145ms  p95=210ms  Error%=0.00%
Minuto 15:  Avg=188ms  p95=270ms  Error%=0.00%
Minuto 30:  Avg=251ms  p95=380ms  Error%=0.02%
Minuto 45:  Avg=362ms  p95=520ms  Error%=0.15%
Minuto 60:  Avg=498ms  p95=680ms  Error%=0.89%
```

Crecimiento de p95: +224% en 60 minutos → posible memory leak.

### Impacto

El sistema degrada progresivamente su rendimiento con el tiempo. En producción, esto significaría que el servicio sería inaceptablemente lento después de horas de operación continua, requiriendo reinicios frecuentes.

### Causa probable

- Fuga de memoria (memory leak) en la capa de aplicación: objetos no liberados acumulándose en el heap.
- Acumulación de conexiones abiertas sin cerrar en el pool de BD.
- Crecimiento del log de auditoría en memoria sin flush periódico.

### Propuesta de mejora

- Habilitar métricas de JVM con Actuator: `management.endpoints.web.exposure.include=metrics,health`
- Monitorear heap con `jstat -gc <pid> 10000` durante el soak test.
- Revisar transacciones anotadas con `@Transactional` para verificar cierre correcto de recursos.
- Ejecutar con `-Xmx512m` y observar si el GC se vuelve más frecuente (GC pressure).

### Estado

Abierto

### Prioridad

Media

---

## Tabla de Seguimiento

| ID | Escenario | SLO Violado | Resultado Obtenido | Estado | Prioridad |
|----|-----------|------------|-------------------|--------|-----------|
| PERF-01 | Load (100 VUs) | p95 < 300ms | p95 = 612ms | Abierto | Alta |
| PERF-02 | Stress (600 VUs) | Error rate < 1% | 4.80% | En progreso | Crítica |
| PERF-03 | Soak (60 min) | Latencia estable | p95: 210→680ms | Abierto | Media |

---

## Sección para el Equipo – Defectos Propios

> Documenta aquí los defectos que encontraste al ejecutar las pruebas con tu propia configuración. Usa el formato de `defectos_template.md`.

### Defecto PERF-04 — [Completar]

*[Agregar defecto real encontrado durante la ejecución]*

### Defecto PERF-05 — [Completar]

*[Agregar defecto real encontrado durante la ejecución]*

---

## Convenciones

| Estado | Descripción |
|--------|-------------|
| Abierto | Identificado sin corrección aplicada |
| En progreso | Corrección en proceso |
| Resuelto | Corregido y validado con nuevas pruebas |

---

Universidad de La Sabana – Facultad de Ingeniería  
Curso: Testing y Validación de Software (2025)
