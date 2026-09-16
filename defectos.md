# Registro de Defectos

Curso: Testing y Validación de Software
Proyecto: Pruebas de Carga y Rendimiento — Registraduría
Equipo: Sofy Alejandra Prada Murillo y Juan Camilo Estévez Otalora
Fecha: 16 de septiembre de 2026

---

## Introducción

Este documento recopila los defectos y hallazgos identificados durante la ejecución
de pruebas de rendimiento (Baseline, Load y Stress) sobre el endpoint `POST /register`
del sistema Registraduría, usando k6 contra una instancia local (`http://localhost:8080`).

---

## Formato 1: Lista detallada

## Defecto PERF-01 — Tasa de duplicados elevada bajo Stress (datos de prueba no idempotentes)

- Capa afectada: Datos de prueba / Persistencia (H2 en memoria)
- Escenario: Stress Test (hasta 600 VUs)
- SLO definido: `register_failed` (rate) < 1%
- Resultado esperado: Cumplimiento del umbral bajo carga alta
- Resultado obtenido: **rate = 24.73%** (1.481.811 de 5.992.940 iteraciones)

### Evidencia

register_failed:
  rate = 0.2473 (umbral rate<0.01 → FAIL)
  passes = 1,481,811
  fails = 4,511,129

http_req_failed (nivel HTTP):
  rate = 0.0000372 (umbral rate<0.01 → OK)
  passes = 223
  fails = 5,992,717

checks:
  status 200 → 5,992,717 passes / 223 fails
  body VALID → 4,511,129 passes / 1,481,811 fails

### Impacto

El sistema respondió HTTP 200 de forma consistente (99.996% de éxito a nivel de
transporte), pero el cuerpo de respuesta reportó `DUPLICATED` en ~24.7% de los
casos. Esto es un falso positivo de rendimiento: el escenario stress reutiliza un
rango de identificadores que ya habían sido registrados en corridas previas del
mismo escenario (la base H2 vive en memoria y no se reinicia entre corridas a
menos que se reinicie el servicio manualmente).

### Causa probable

- El script de carga no genera identificadores únicos por ejecución (ej. timestamp
  o UUID como semilla), por lo que corridas repetidas del mismo escenario colisionan
  con datos ya insertados.
- No hay limpieza automática de estado entre corridas de k6.

### Estado

Abierto

### Prioridad

Media (defecto de diseño de la prueba, no del sistema bajo prueba)

---

## Defecto PERF-02 — Degradación progresiva de latencia con el aumento de carga

- Capa afectada: Aplicación / Acceso a datos (`RegistryRepository`)
- Escenario: Comparación Baseline (20 VUs) → Load (200 VUs) → Stress (600 VUs)
- SLO definido: p95 < 300 ms, p99 < 800 ms
- Resultado esperado: Latencia estable o con crecimiento marginal al escalar carga
- Resultado obtenido: Crecimiento sostenido y proporcional al número de VUs

### Evidencia

| Escenario | VUs máx. | p95 (ms) | p90 (ms) | avg (ms) | máx (ms) |
|---|---|---|---|---|---|
| Baseline | 20 | 2.79 | 2.09 | 1.38 | 100.40 |
| Load | 200 | 29.79 | 24.00 | 12.99 | 747.41 |
| Stress | 600 | 78.73 | 61.99 | 31.01 | 1449.28 |

El p95 se multiplicó por ~10.7x entre baseline y load, y por ~2.6x adicional
entre load y stress (~28.2x acumulado desde baseline).

### Impacto

Aunque **ningún umbral de latencia fue incumplido** en estas corridas (p95 se
mantuvo muy por debajo de 300 ms incluso a 600 VUs), la tendencia de crecimiento
no lineal sugiere que el sistema se acerca a un cuello de botella de recursos
(el código fuente de `RegistryRepository.getConnection()` abre una conexión JDBC
nueva por cada operación en lugar de usar un pool). Con una carga sostenida mayor
o un `soak test` prolongado, es previsible que el umbral sí se incumpla.

### Causa probable

- Ausencia de connection pooling en el acceso a la base de datos.
- Cada request incurre en el costo de apertura/cierre de conexión.

### Estado

Abierto

### Prioridad

Alta (riesgo de incumplimiento de SLO en producción bajo carga sostenida)

---

## Formato 2: Tabla de seguimiento

| ID | Escenario | Resultado esperado | Resultado obtenido | Estado | Prioridad |
|----|-----------|--------------------|--------------------|--------|-----------|
| PERF-01 | Stress | register_failed < 1% | 24.73% | Abierto | Media |
| PERF-02 | Baseline→Load→Stress | p95 estable | Crecimiento ~28x (2.79ms→78.73ms) | Abierto | Alta |

---

## Convenciones de Estado

Abierto: Defecto identificado sin corrección aplicada.
En progreso: En proceso de corrección.
Resuelto: Corregido y validado con nuevas pruebas.

---

Universidad de La Sabana — Facultad de Ingeniería
Curso: Testing y Validación de Software