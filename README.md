# Proyecto Final – Pruebas de Carga y Rendimiento con JMeter

**Curso:** Testing y Validación de Software  
**Programa:** Maestría en Ingeniería de Software – Universidad de La Sabana  
**Basado en:** [TYVS-Proyecto_Pruebas_carga_y_rendimiento](https://github.com/CesarAVegaF312/TYVS-Proyecto_Pruebas_carga_y_rendimiento)

---

## 1. Descripción y Propósito

Este proyecto final tiene como propósito diseñar, ejecutar y analizar pruebas de rendimiento sobre un servicio REST desarrollado durante el curso, utilizando **Apache JMeter**. El equipo deberá aplicar técnicas de pruebas de carga, definir Objetivos de Nivel de Servicio (SLO), interpretar métricas de rendimiento y documentar defectos relacionados con desempeño.

El enfoque no es únicamente ejecutar pruebas, sino **analizar técnicamente el comportamiento del sistema bajo distintas condiciones de carga** y proponer mejoras fundamentadas.

---

## 2. Objetivos Académicos

- Diseñar escenarios de pruebas de rendimiento progresivos y justificarlos técnicamente.
- Ejecutar pruebas de carga, estrés, pico, resistencia y regresión con JMeter.
- Definir SLO medibles y evaluar su cumplimiento.
- Analizar métricas: latencia promedio, p95, p99, throughput y tasa de errores.
- Identificar cuellos de botella y documentar defectos de rendimiento.
- Integrar pruebas de rendimiento en un flujo reproducible (CI/CD opcional).

---

## 3. Alcance Técnico

El equipo puede elegir uno de los siguientes enfoques:

1. Realizar pruebas sobre el endpoint `/register` de la aplicación **Registraduría** (Spring Boot, localhost:8080).
2. Implementar y probar su propio endpoint REST desarrollado durante el curso.
3. Aplicar pruebas sobre un microservicio desarrollado previamente.

El sistema debe ejecutarse localmente o mediante contenedor Docker.

### Levantar la aplicación de referencia (Registraduría)

```bash
git clone https://github.com/CesarAVegaF312/TYVS-Taller_Pruebas_de_carga.git
cd TYVS-Taller_Pruebas_de_carga/registraduria
mvn clean spring-boot:run
```

Verificar: `curl -X POST http://localhost:8080/register -H "Content-Type: application/json" -d '{"name":"Test","id":999,"age":30,"gender":"MALE","alive":true}'`

---

## 4. Tipos de Prueba Obligatorias

El proyecto debe incluir los siguientes 6 escenarios, cada uno correctamente configurado y documentado:

| # | Tipo | Archivo JMX | Descripción |
|---|------|-------------|-------------|
| 1 | **Baseline** | `perf/scripts/01_baseline_test.jmx` | Carga mínima para establecer métricas de referencia |
| 2 | **Load** | `perf/scripts/02_load_test.jmx` | Carga nominal esperada en producción |
| 3 | **Stress** | `perf/scripts/03_stress_test.jmx` | Carga más allá de la capacidad para encontrar el punto de quiebre |
| 4 | **Spike** | `perf/scripts/04_spike_test.jmx` | Pico repentino de tráfico para evaluar resiliencia |
| 5 | **Soak** | `perf/scripts/05_soak_test.jmx` | Carga sostenida para detectar fugas de memoria |
| 6 | **Regresión** | `perf/scripts/06_regression_test.jmx` | Comparar rendimiento con el baseline inicial |

---

## 5. Estructura del Repositorio

```
proyecto-pruebas-carga/
├── README.md
├── defectos_template.md
├── integrantes.txt
└── perf/
    ├── scripts/
    │   ├── 01_baseline_test.jmx
    │   ├── 02_load_test.jmx
    │   ├── 03_stress_test.jmx
    │   ├── 04_spike_test.jmx
    │   ├── 05_soak_test.jmx
    │   └── 06_regression_test.jmx
    ├── data/
    │   └── personas.csv               # 200+ registros de prueba
    ├── results/                        # Resultados JTL y reportes HTML
    ├── defectos_rendimiento.md         # Registro de defectos encontrados
    └── ci/
        └── github-actions.yml          # Pipeline CI/CD automatizado
```

---

## 6. Ejecución Reproducible

El proyecto debe ejecutarse completamente desde consola, sin intervención manual.

### Prerrequisitos

```bash
java -version   # Java 11+
mvn -version    # Maven 3.8+
jmeter --version # JMeter 5.6+
```

### Levantar la aplicación

```bash
cd registraduria
mvn clean spring-boot:run
```

### Ejecutar cada escenario

```bash
# Crear directorio de resultados si no existe
mkdir -p perf/results

# 1. Baseline
jmeter -n -t perf/scripts/01_baseline_test.jmx \
  -l perf/results/01_baseline.jtl \
  -e -o perf/results/01_baseline-report/

# 2. Load
jmeter -n -t perf/scripts/02_load_test.jmx \
  -l perf/results/02_load.jtl \
  -e -o perf/results/02_load-report/

# 3. Stress
jmeter -n -t perf/scripts/03_stress_test.jmx \
  -l perf/results/03_stress.jtl \
  -e -o perf/results/03_stress-report/

# 4. Spike
jmeter -n -t perf/scripts/04_spike_test.jmx \
  -l perf/results/04_spike.jtl \
  -e -o perf/results/04_spike-report/

# 5. Soak (tarda 60 minutos)
jmeter -n -t perf/scripts/05_soak_test.jmx \
  -l perf/results/05_soak.jtl \
  -e -o perf/results/05_soak-report/

# 6. Regresión
jmeter -n -t perf/scripts/06_regression_test.jmx \
  -l perf/results/06_regression.jtl \
  -e -o perf/results/06_regression-report/
```

### Sobrescribir parámetros desde CLI

```bash
# Cambiar host y puerto
jmeter -n -t perf/scripts/01_baseline_test.jmx \
  -JBASE_URL=mi-servidor.local \
  -JPORT=9090 \
  -l perf/results/01_baseline.jtl
```

---

## 7. Definición de SLO

El equipo debe definir al menos **dos SLO claros y medibles**, justificados técnicamente.

### SLO propuestos para este proyecto

| SLO | Valor objetivo | Justificación |
|-----|---------------|---------------|
| Latencia p95 | < 300 ms | Threshold estándar para APIs REST interactivas (Google SRE Book) |
| Tasa de errores | < 1% | Límite aceptable para disponibilidad de servicios transaccionales |
| Latencia p99 | < 800 ms | Cubre el 99% de usuarios incluyendo los de conexión lenta |
| Disponibilidad | ≥ 99% | Requisito mínimo para servicios de producción |

> **Nota:** El equipo debe justificar sus SLO en la Wiki considerando el tipo de sistema evaluado, la criticidad del servicio y las expectativas de los usuarios.

---

## 8. Métricas Obligatorias por Escenario

En cada escenario se debe reportar y analizar:

| Métrica | Descripción | Dónde leerla en JMeter |
|---------|-------------|----------------------|
| **Latencia promedio** | Tiempo medio de respuesta | Aggregate Report → Average |
| **p95** | 95% de requests respondieron en ≤ este tiempo | Aggregate Report → 95% Line |
| **p99** | 99% de requests respondieron en ≤ este tiempo | Aggregate Report → 99% Line |
| **Throughput** | Solicitudes procesadas por segundo | Summary Report → Throughput |
| **Tasa de errores** | Porcentaje de requests fallidos | Summary Report → Error% |

### Tabla comparativa de resultados (completar por el equipo)

| Escenario | VUs pico | p95 (ms) | p99 (ms) | Throughput (req/s) | Error% HTTP | ¿Cumple SLO? |
|-----------|---------|---------|---------|------------------|------------|-------------|
| Baseline | 10 | 22 | 139 | 43.0 | 0.00% | ✅ Todos |
| Load E2 | 100 | 101 | 134 | 372 | 0.00% | ✅ Todos |
| Stress E3 | 600 | 2 170 | 4 497 | 1 272 | 0.00% | ❌ p95 y p99 |
| Spike pico | 300 | 366 | 380 | 834 | 0.00% | ❌ p95 |
| Soak (60 min) | 100 | 22 | 30 | 63 | 0.007% | ✅ Todos |
| Regresión | 10 | 21 | 26 | 43.6 | 0.00% | ✅ Todos |

---

## 9. Registro de Defectos de Rendimiento

Completar el archivo `perf/defectos_rendimiento.md` basándose en el template `defectos_template.md`.

**Requisitos mínimos:**
- Mínimo **3 defectos documentados**
- Cada defecto debe incluir: escenario, evidencia con métricas reales, impacto, causa probable y propuesta de mejora
- Al menos 1 defecto debe ser de severidad Crítica o Alta

---

## 10. Wiki del Proyecto (Documento Oficial)

La evaluación se realizará principalmente sobre la **Wiki del repositorio**. Estructura sugerida:

1. **Introducción y arquitectura del sistema** – Descripción de la app, endpoints evaluados, tecnologías
2. **Definición de SLO** – SLOs elegidos con justificación técnica
3. **Configuración de escenarios** – Parámetros usados en cada JMX
4. **Resultados detallados por escenario** – Capturas del Aggregate Report y gráficas
5. **Comparación entre escenarios** – Tabla comparativa completada
6. **Identificación de cuellos de botella** – Análisis técnico con evidencia
7. **Registro de defectos** – Enlace a `perf/defectos_rendimiento.md`
8. **Propuestas de mejora** – Al menos 2 mejoras técnicas fundamentadas
9. **Reflexión técnica** – Aprendizajes del equipo y limitaciones encontradas

---

## 11. Diferenciación con Otros Proyectos del Curso

| Proyecto | Enfoque principal |
|----------|-----------------|
| Pruebas Unitarias | Validación de lógica interna de componentes aislados |
| Pruebas de Integración | Validación de colaboración entre capas y servicios |
| **Pruebas de Rendimiento** | Validación del comportamiento bajo carga y condiciones extremas |

Este proyecto introduce la **dimensión temporal y de resiliencia** del sistema: no es solo "¿funciona?", sino "¿cuánto aguanta y cómo se comporta cuando falla?".

---

## 12. Rúbrica de Evaluación (50 puntos)

| Criterio | Excelente (5 pts) | Bueno (4 pts) | Necesita mejorar (3.5 pts) | Deficiente (2.5 pts) | No cumple (0 pts) |
|----------|------------------|--------------|--------------------------|---------------------|------------------|
| **Diseño de escenarios** | 6 escenarios progresivos, justificados y reproducibles | Completos con leves inconsistencias | Faltan 1-2 escenarios o sin justificación | Configuración incorrecta | No implementa |
| **Implementación JMX** | Planes JMeter parametrizados, organizados y funcionales | Funcionales con leves errores | Funcionan parcialmente o con valores hardcodeados | Errores de ejecución | No entrega |
| **Definición de SLO** | SLO bien definidos, medidos y analizados en todos los escenarios | SLO definidos pero análisis parcial | SLO poco claros o sin validación | Sin evidencia | No define SLO |
| **Análisis de métricas** | Análisis comparativo profundo entre escenarios | Correcto pero superficial | Reporta métricas sin interpretación | Solo copia resultados | No presenta análisis |
| **Gestión de defectos** | Registro completo con evidencia, impacto y mejora | Registro con evidencia parcial | Lista defectos sin análisis profundo | Mínimo o incompleto | No presenta |
| **Observabilidad y diagnóstico** | Correlaciona métricas de app con infraestructura | Identifica causas sin evidencia suficiente | Menciona problemas sin diagnóstico | Análisis superficial | No identifica causas |
| **Reproducibilidad** | Proyecto ejecuta desde consola con documentación clara | Requiere pequeños ajustes manuales | Ejecución parcial o poco clara | Difícil de ejecutar | No ejecuta |
| **Documentación Wiki** | Completa, organizada y con análisis detallado | Bien estructurada con faltantes menores | Parcial o poco clara | Incompleta sin evidencias | No hay Wiki |
| **Reflexión técnica** | Reflexión crítica con propuestas técnicas fundamentadas | Reflexión adecuada pero general | Superficial | Mínima | No presenta |
| **Calidad general** | Integración sólida entre pruebas, análisis y documentación | Buen nivel general con leves inconsistencias | Parcialmente funcional | Fallos técnicos importantes | Proyecto incompleto |

### Escala de desempeño

| Rango | Desempeño |
|-------|-----------|
| 45 – 50 pts | Excelente manejo de pruebas de rendimiento y análisis técnico avanzado |
| 35 – 44 pts | Buen trabajo, ejecución completa con fallas menores |
| 30 – 34 pts | Cumple lo básico, faltan evidencias o profundidad técnica |
| < 30 pts | No cumple los criterios mínimos |

---

## 13. Referencias

- [Apache JMeter Documentation](https://jmeter.apache.org/usermanual/index.html)
- [Google SRE Book – Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [ISO/IEC 25010 – Performance Efficiency](https://iso25000.com/index.php/en/iso-25000-standards/iso-25010)
- [JMeter Best Practices](https://jmeter.apache.org/usermanual/best-practices.html)

---

## Créditos y Uso Académico

**Autor:** César Augusto Vega Fernández  
**Curso:** Testing y Validación de Software  
**Programa:** Maestría en Ingeniería de Software – Universidad de La Sabana  
**Año:** 2025

Material de uso **exclusivamente académico**. Este material se distribuye bajo la licencia [Creative Commons Atribución-NoComercial-CompartirIgual 4.0 Internacional (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.es).

---

© Universidad de La Sabana – Facultad de Ingeniería  
Maestría en Ingeniería de Software – 2025
