# Registro de Defectos

Curso: Testing y Validación de Software
Proyecto: Pruebas de Carga y Rendimiento — Registraduría
Equipo: Sofy Alejandra Prada Murillo y Juan Camilo Estévez
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

## Defecto PERF-03 — Reutilización del dataset de votantes en CI (duplicados por diseño de prueba)

- Capa afectada: Diseño de la prueba / pipeline CI (`.github/workflows/perf.yml`)
- Escenario: Verificación de resultado de negocio (`register_voter_k6.js`), ejecutado en GitHub Actions con `--duration 60s --vus 20`
- SLO definido: `register_failed` (rate) < 1%
- Resultado esperado: Cumplimiento del umbral
- Resultado obtenido: **65.48%** de resultados de negocio incorrectos (`resultado_negocio_incorrecto: 0.6548`)

### Evidencia

Log de CI (GitHub Actions, ejecución #2):

```
{
  "escenario": "baseline",
  "peticiones": 11780,
  "p95_ms": 2,
  "resultado_negocio_incorrecto": 0.6548387096774193
}
level=error msg="thresholds on metrics 'register_failed' have been crossed"
```

### Impacto

`perf/data/voters.csv` tiene 512 filas. Al forzar `--duration 60s --vus 20` (en vez
de dejar que el escenario `baseline` del script controle su propia duración/VUs),
el job completó 11,780 iteraciones — más de 23 vueltas completas al dataset.
Cada vez que una fila se reutiliza, el sistema responde `DUPLICATED` (correctamente,
por la regla de unicidad), pero el script la marca como resultado de negocio
incorrecto porque esperaba `VALID`/`UNDERAGE`/`DEAD`/`INVALID_AGE` según la fila.
No es un defecto del sistema bajo prueba: es un defecto de diseño de la prueba
en el pipeline de CI.


### Causa probable

- Causa inicial identificada (parcial): el workflow sobreescribía la
  configuración de VUs/duración del script con `--duration`/`--vus`, generando
  más iteraciones que filas tiene el dataset (512).
- Causa raíz real: el paso "Verificación de resultado de negocio" corre sobre
  el MISMO servicio, sin reiniciarlo, que ya fue usado por el paso anterior
  ("Baseline de rendimiento", con `register_person_k6.js`). Los ids generados
  por ese paso colisionan con el rango de ids fijos de `voters.csv`, así que
  incluso con una sola vuelta al dataset (`--iterations 480`, menor que las
  512 filas) seguían apareciendo duplicados (63.75%).

### Estado

Resuelto en dos pasos: (1) se cambió `--duration 60s --vus 20` por
`--vus 20 --iterations 480` para no exceder el tamaño del dataset; (2) se
agregó `--env ID_BASE=700000000` (tal como sugiere el README del taller) para
desplazar el rango de ids del script de votantes y evitar la colisión con los
ids ya registrados por el paso de baseline. Verificado: pipeline en verde
(GitHub Actions, ejecución #4, `Success`, 1m 40s).


### Prioridad

Media

---

## Formato 2: Tabla de seguimiento

| ID | Escenario | Resultado esperado | Resultado obtenido | Estado | Prioridad |
|----|-----------|--------------------|--------------------|--------|-----------|
| PERF-01 | Stress | register_failed < 1% | 24.73% | Abierto | Media |
| PERF-02 | Baseline→Load→Stress | p95 estable | Crecimiento ~28x (2.79ms→78.73ms) | Abierto | Alta |
| PERF-03 | CI (verificación de negocio) | register_failed < 1% | 63.75% → 0% tras fix | Resuelto | Media |
---

## Convenciones de Estado

Abierto: Defecto identificado sin corrección aplicada.
En progreso: En proceso de corrección.
Resuelto: Corregido y validado con nuevas pruebas.

---

Universidad de La Sabana — Facultad de Ingeniería
Curso: Testing y Validación de Software