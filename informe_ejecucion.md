# Informe de Ejecución – Proyecto Pruebas de Carga y Rendimiento

**Curso:** Testing y Validación de Software  
**Programa:** Maestría en Ingeniería de Software – Universidad de La Sabana  
**Equipo:** [Nombre del equipo]  
**Integrantes:** Ver [`integrantes.txt`](./integrantes.txt)  
**Fecha de ejecución:** [Fecha]  
**Sistema bajo prueba:** [Nombre de la aplicación] – `POST /register` (localhost:8080)

---

## 1. Descripción de los Escenarios Ejecutados

### 1.1 Baseline Test (`01_baseline_test.jmx`)

| Parámetro | Valor |
|-----------|-------|
| VUs | 10 constantes |
| Ramp-up | 30 s |
| Duración | 5 min |
| Dataset | `perf/data/personas.csv` (205 registros) |
| Endpoint | `POST http://localhost:8080/register` |

**Justificación técnica:** [¿Por qué 10 VUs como baseline? Justificar en función del sistema]

---

### 1.2 Load Test (`02_load_test.jmx`)

| Etapa | VUs inicio → fin | Ramp-up | Duración sostenida |
|-------|-----------------|---------|-------------------|
| 1 | 10 → 50 | 2 min | 3 min |
| 2 | 50 → 100 | 2 min | 5 min |

**Justificación técnica:** [¿Por qué 100 VUs como carga nominal? Justificar con el contexto del sistema]

---

### 1.3 Stress Test (`03_stress_test.jmx`)

| Etapa | VUs pico | Duración |
|-------|---------|----------|
| 1 | 200 | 3 min |
| 2 | 400 | 3 min |
| 3 | 600 | 3 min + sostenido |

**Justificación técnica:** [¿Por qué se eligió 600 VUs como límite de estrés?]

---

### 1.4 Spike Test (`04_spike_test.jmx`)

| Fase | VUs | Duración |
|------|-----|----------|
| Base | 10 | 2 min |
| Pico repentino | 300 en 30 s | 1 min |
| Recuperación | retorno a 10 | 2 min |

**Justificación técnica:** [¿Qué evento o situación real simula este pico?]

---

### 1.5 Soak Test (`05_soak_test.jmx`)

| Parámetro | Valor |
|-----------|-------|
| VUs | 100 constantes |
| Ramp-up | 5 min |
| Duración | 60 min |

**Justificación técnica:** [¿Por qué 60 minutos? ¿Qué tipo de defecto se quería detectar?]

---

### 1.6 Regression Test (`06_regression_test.jmx`)

| Parámetro | Valor |
|-----------|-------|
| VUs | 10 (igual al baseline) |
| Ramp-up | 30 s |
| Duración | 5 min |

**Justificación técnica:** [¿Qué cambio o versión de la aplicación motivó ejecutar la regresión?]

---

## 2. Definición de SLOs

| SLO | Valor objetivo | Justificación técnica |
|-----|---------------|----------------------|
| Latencia p95 | < 300 ms | [Justificar según tipo de sistema y expectativa de usuario] |
| Tasa de errores | < 1 % | [Justificar] |
| [SLO adicional] | | [Justificar] |

---

## 3. Métricas Obtenidas por Escenario

### Tabla comparativa general

| Escenario | VUs pico | Avg (ms) | p95 (ms) | p99 (ms) | Throughput (req/s) | Error % | ¿Cumple SLO? |
|-----------|---------|---------|---------|---------|------------------|---------|-------------|
| Baseline | 10 | | | | | | |
| Load | 100 | | | | | | |
| Stress | 600 | | | | | | |
| Spike | 300 | | | | | | |
| Soak (60 min) | 100 | | | | | | |
| Regresión | 10 | | | | | | |

### 3.1 Baseline

> Adjuntar captura del **Aggregate Report** de JMeter o del reporte HTML en `perf/results/01_baseline-report/index.html`

```
[Pegar aquí la salida de texto del Summary Report de JMeter]
```

### 3.2 Load Test

> Adjuntar captura y pegar salida del Summary Report.

```
[Pegar aquí]
```

### 3.3 Stress Test

> Adjuntar captura. Anotar en qué etapa empezaron a subir los errores.

```
[Pegar aquí]
```

### 3.4 Spike Test

> Adjuntar captura. Anotar tiempo de recuperación tras el pico.

```
[Pegar aquí]
```

### 3.5 Soak Test

> Adjuntar gráfica de latencia en el tiempo (del reporte HTML). Anotar si la latencia creció progresivamente.

```
[Pegar aquí — incluir tabla de evolución de latencia por intervalo de tiempo si está disponible]
```

### 3.6 Regression Test

> Comparar directamente con los valores del Baseline.

```
[Pegar aquí]
```

---

## 4. Interpretación de Resultados

### 4.1 Análisis por escenario

**Baseline:**  
[¿Las métricas base son buenas? ¿El sistema ya parte de un estado saludable o tiene problemas incluso sin carga?]

**Load:**  
[¿Escala bien de 10 a 100 VUs? ¿La latencia crece linealmente o hay degradación no lineal?]

**Stress:**  
[¿En qué etapa se detectó el punto de quiebre? ¿Qué métrica rompió primero: latencia o error rate?]

**Spike:**  
[¿El sistema se recuperó tras el pico? ¿Cuánto tiempo tardó en volver a los valores base?]

**Soak:**  
[¿Hubo crecimiento progresivo de latencia? ¿Es indicio de memory leak o agotamiento de recursos?]

**Regresión:**  
[¿Mejoró, empeoró o se mantuvo igual respecto al baseline inicial? ¿Hay regresión de rendimiento?]

---

### 4.2 Comparación Baseline vs. Regresión (detección de regresión)

| Métrica | Baseline inicial | Regresión | Variación | ¿Regresión detectada? |
|---------|-----------------|----------|----------|----------------------|
| p95 | ms | ms | % | Sí / No |
| Throughput | req/s | req/s | % | Sí / No |
| Error rate | % | % | % | Sí / No |

> Se considera regresión de rendimiento cuando cualquier métrica empeora más del **10%** respecto al baseline.

---

## 5. Identificación de Cuellos de Botella

| Capa | Señal observada | Causa probable |
|------|----------------|---------------|
| Base de datos | | |
| Servidor de aplicación | | |
| JVM / Memoria | | |
| Red | | |

**Análisis técnico principal:**  
[Describe el cuello de botella más crítico encontrado, con evidencia de las métricas]

---

## 6. Defectos de Rendimiento

> Ver el documento completo en [`perf/defectos_rendimiento.md`](./perf/defectos_rendimiento.md)

| ID | Escenario | SLO violado | Resultado | Estado | Prioridad |
|----|-----------|------------|-----------|--------|-----------|
| PERF-01 | | | | | |
| PERF-02 | | | | | |
| PERF-03 | | | | | |

---

## 7. Propuestas de Mejora

### Mejora 1: [Título]
- **Problema:** 
- **Acción técnica:** 
- **Impacto esperado:** 

### Mejora 2: [Título]
- **Problema:** 
- **Acción técnica:** 
- **Impacto esperado:** 

### Mejora 3 (opcional): [Título]
- **Problema:** 
- **Acción técnica:** 
- **Impacto esperado:** 

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

> Los reportes HTML se generan con:
> ```bash
> jmeter -n -t perf/scripts/01_baseline_test.jmx \
>   -l perf/results/01_baseline.jtl \
>   -e -o perf/results/01_baseline-report/
> ```

---

## 9. Reflexión Técnica

[Responde las siguientes preguntas en forma de párrafo:]

- ¿Qué aprendió el equipo sobre el comportamiento del sistema bajo carga?
- ¿Qué fue lo más sorprendente de los resultados?
- ¿Qué harían diferente si tuvieran que repetir las pruebas?
- ¿Cómo integrarían estas pruebas en el ciclo de desarrollo continuo (CI/CD)?

---

## 10. Conclusiones

[Párrafo final que sintetice si el sistema es apto para producción según los SLOs definidos, cuáles son los riesgos identificados y cuál es la recomendación del equipo]

---

*Proyecto Final – Testing y Validación de Software*  
*Universidad de La Sabana – Maestría en Ingeniería de Software – 2025*
