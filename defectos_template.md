# Plantilla: Registro de Defecto de Rendimiento

Usa esta plantilla para reportar cada defecto de rendimiento identificado durante las pruebas. El equipo debe documentar **mínimo 3 defectos** en `perf/defectos_rendimiento.md`.

---

## Defecto PERF-XX — [Título descriptivo del defecto]

- **Identificador:** PERF-XX
- **Capa afectada:** [Aplicación / Base de datos / Red / Servidor / JVM]
- **Escenario donde ocurre:** [Baseline / Load / Stress / Spike / Soak / Regresión] ([N] VUs)
- **SLO violado:** [Ej: p95 < 300 ms]
- **Resultado esperado:** [Descripción del comportamiento que debería tener el sistema]
- **Resultado obtenido:** [Valor real medido]

### Evidencia

```
[Pegar la salida del Aggregate Report o Summary Report de JMeter]
[Puedes incluir también fragmentos de logs del servidor si los tienes]

Ejemplo:
Label           # Samples  Average  95% Line  Error%  Throughput
POST /register  5000        412      638       2.30%   28.4/s
```

### Impacto en el sistema

[Describir concretamente cómo este defecto afecta al usuario o al servicio. 
¿Es perceptible? ¿Afecta la disponibilidad? ¿Causa pérdida de datos?]

### Causa probable

- [Primera hipótesis técnica con justificación]
- [Segunda hipótesis técnica si aplica]

### Propuesta de mejora

- [Acción técnica concreta y específica]
- [Ajuste de configuración, optimización de código, cambio de infraestructura]

### Estado

[Abierto / En progreso / Resuelto]

### Prioridad

[Crítica / Alta / Media / Baja]

---

## Tabla de Seguimiento (completar con todos los defectos)

| ID | Escenario | SLO Violado | Resultado Obtenido | Estado | Prioridad |
|----|-----------|------------|-------------------|--------|-----------|
| PERF-01 | | | | | |
| PERF-02 | | | | | |
| PERF-03 | | | | | |

---

## Guía de Prioridades

| Prioridad | Cuándo asignarla |
|-----------|-----------------|
| **Crítica** | Sistema inaccesible o error rate > 5% bajo carga esperada |
| **Alta** | Incumplimiento de SLO de latencia bajo carga nominal |
| **Media** | Degradación leve observable solo bajo carga alta |
| **Baja** | Impacto mínimo en usuarios, solo bajo condiciones extremas |

## Guía de Estados

| Estado | Significado |
|--------|-------------|
| **Abierto** | Defecto identificado, sin corrección aplicada |
| **En progreso** | Se está trabajando en la corrección |
| **Resuelto** | Corrección aplicada y validada con nuevas pruebas |
