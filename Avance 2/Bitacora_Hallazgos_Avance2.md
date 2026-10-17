# Bitácora de hallazgos — Avance 2 (IF-6201, Process Intelligence Challenge)

> Propósito: ir guardando hallazgos, fórmulas, capturas y textos ya validados para generar al final el Word del Avance 2, continuando `EntregaAvance1.pdf`.
> Etiquetas de trazabilidad: **[ENUNCIADO]** texto oficial · **[CELONIS]** / **[EXCEL]** calculado por el equipo en esa herramienta · **[VERIF-PY]** verificación cruzada hecha por Claude en Python (no se cita como fuente, solo valida) · **[EQUIPO]** interpretación o decisión del equipo.
> Regla: `Spend_CO2e` nunca se suma con `Hotel_CO2e` + `Transportation_CO2e`.

---

## 0. Guía de formato del documento final (tomada de EntregaAvance1.pdf)

- Portada centrada: Universidad de Costa Rica / Carrera de Informática Empresarial, Sede de Occidente, Recinto de Tacares / **Avance 2: Análisis exploratorio y hallazgos encontrados** / 5 nombres / IF6201 – Informática Aplicada a los Negocios / Prof. Iyubanit Rodríguez Ramírez / fecha (26 de octubre de 2026).
- Times New Roman 12, interlineado doble, sangría de primera línea en párrafos.
- Títulos numerados en negrita ("1. Introducción…", "2.1 …"); preguntas como título en negrita ("Q1. ¿…?").
- Tablas con bordes simples, encabezado en negrita, texto centrado, números con coma de miles y punto decimal (ej. 233,211,160.73).
- Pie de figura: "Figura N. Descripción. Fuente: Celonis, sobre Trip_Event_Log." (numeración continúa después de la Figura 7 del Avance 1).
- Redacción impersonal/sobria ("En estos datos…", "Se observa…"), sin adornos.
- Avance 1 terminó con "TODO: aquí continúan los análisis exploratorios de las demás preguntas." → el Avance 2 continúa desde ahí.

## 1. Estructura propuesta del Word (a confirmar al generarlo)

1–4. Se conservan del Avance 1 (introducción, datasets, herramientas, proceso as-is), resumidos o actualizados donde haga falta.
5. Ajustes a las preguntas (tabla Q1–Q8 actualizada + justificación de cambios).
6. Hipótesis por pregunta (tabla).
7. Análisis y respuestas parciales Q1–Q8 (hipótesis → análisis → figura → respuesta → pilar P1/P2/P3).
8. Análisis original/novedoso.
9. Dashboards (capturas de las hojas de Celonis).
10. Conclusiones ligadas al desafío (P1/P2/P3) y limitaciones.
11. Declaración de uso de IA.

---

## 2. Hallazgos por pregunta

### Q6 — ¿Qué variantes del proceso se asocian a mayores tiempos de ciclo, cancelaciones o cambios, y qué relación tienen con emisiones y costos? — ✅ CERRADA (16-oct-2026)

**Hipótesis H6 [EQUIPO]:** los viajes con desviaciones (cambios, cancelaciones, retrasos, extensiones) emiten y cuestan más que los que siguen el camino estándar.

**Herramienta:** Celonis — Analysis del paquete `ProyectoInfoAplicada_1`, hoja "Q6-ProcessCode vs CO2", dos OLAP Tables.

**Fórmulas PQL usadas [CELONIS]:**
- Casos: `COUNT_TABLE("trip_event_attributes")`
- CO2 promedio (kg/viaje): `AVG("trip_data_with_emissions"."TRANSPORTATION_CO2E" + "trip_data_with_emissions"."HOTEL_CO2E")`
- Costo promedio (USD): `AVG("trip_data_with_emissions"."NETCOSTS")`
- % fuera de política: `AVG(CASE WHEN "trip_data_with_emissions"."OUTOFPOLICY" = 'Yes' THEN 1.0 ELSE 0.0 END)`
- Tiempo de ciclo (días), primer a último evento del caso: `AVG(DAYS_BETWEEN(PU_MIN("trip_data_with_emissions", "trip_event_log"."EVENTTIMESTAMP"), PU_MAX("trip_data_with_emissions", "trip_event_log"."EVENTTIMESTAMP")))`
  - Nota técnica: `CALC_THROUGHPUT(CASE_START TO CASE_END…)` dio error "No common table" porque `trip_event_log_CASES` y `trip_event_attributes` no tienen tabla común; se resolvió con PU_MIN/PU_MAX hacia `trip_data_with_emissions`.
  - Diferencia con el Avance 1: allá el tiempo de ciclo fue de la primera Book Mode of Transportation a la última Expense Reimbursement (83,749 casos, prom. 22.9 d). Aquí es primer→último evento de cada caso (87,053 casos). Declararlo.
- Dimensión "Tipo de viaje" [EQUIPO — definición propia]: "Con desviación" si el caso tiene al menos una de 19 actividades de excepción: Flight Cancellation, Flight Change, Flight Delay, Missed Flight, Hotel Change, Mode of Transportation Change, Ticket Reissued, Vehicle Change, Travel Delay, Rental Cancellation, Expense Request Edit, Missed Pickup, Expense Request Denied, Train Change, Train Delay, Train Cancellation, Missed Train, Trip Extension, Itinerary Edit. **Manager Preapproved NO se cuenta** (es aprobación previa, no excepción).
  ```
  CASE WHEN PU_COUNT("trip_data_with_emissions", "trip_event_log"."EVENTNAME",
    "trip_event_log"."EVENTNAME" IN (…19 actividades…)) > 0
  THEN 'Con desviación' ELSE 'Sin desviación' END
  ```
- Advertencia amarilla de Celonis (AVG sobre tabla 1:N respecto a la common table `trip_event_attributes`): no afecta, porque `trip_event_attributes` tiene exactamente 1 fila por viaje (87,053 = 87,053). Mencionarlo en una línea.

**Resultados [CELONIS], verificados [VERIF-PY] — coinciden exactos:**

Tabla "Q6 · Viajes con desviación vs. sin desviación":

| Tipo de viaje | Casos | CO2 promedio (kg/viaje) | Costo promedio (USD) | % fuera de política | Tiempo de ciclo (días) |
|---|---|---|---|---|---|
| Con desviación | 33,399 (38.4%) | 2,661 | 2,761 | 18.41% | 21.47 |
| Sin desviación | 53,654 (61.6%) | 2,690 | 2,776 | 18.44% | 22.76 |

Tabla "Q6 · Emisión, costo y tiempo de ciclo por ProcessCode" (top 13 por casos; % fuera de política ya visible en Celonis con 2 decimales):

| ProcessCode | Casos | CO2 prom. | Costo prom. | % fuera pol. | Días |
|---|---|---|---|---|---|
| V1 | 46,042 | 2,686 | 2,776 | 18.5% | 22.91 |
| V18 | 2,774 | 2,721 | 2,803 | 18.1% | 4.79 |
| V17 | 2,688 | 2,581 | 2,690 | 18.6% | 23.24 |
| V19 | 2,569 | 2,690 | 2,823 | 21.2% | 22.32 |
| V20 | 2,531 | 2,750 | 2,808 | 18.5% | 22.85 |
| V16 | 2,504 | 2,628 | 2,696 | 17.2% | 23.04 |
| V21 | 2,495 | 2,766 | 2,837 | 18.6% | 23.25 |
| V15 | 2,419 | 2,625 | 2,746 | 17.7% | 22.96 |
| V14 | 2,168 | 2,606 | 2,750 | 17.7% | 22.89 |
| V22 | 2,123 | 2,707 | 2,760 | 18.2% | 22.63 |
| V23 | 1,942 | 2,705 | 2,743 | 18.4% | 23.01 |
| V13 | 1,903 | 2,629 | 2,716 | 19.0% | 22.85 |
| V12 | 1,667 | 2,618 | 2,698 | 18.4% | 22.53 |

Rango completo de los 35 ProcessCode [VERIF-PY]: CO2 prom. ~2,356 (V34, 76 casos) a ~3,219 (V3, 122 casos); los extremos son códigos con pocos casos. V2 (217) y V3 (122) tienen tiempo ≈ 0 días.

**Respuesta parcial [EQUIPO]:** H6 se rechaza. Las desviaciones del proceso no se asocian a mayores emisiones ni costos (diferencia < 1.1% en CO2 y < 0.6% en costo), y el % fuera de política es prácticamente igual (~18%) en todas las variantes. Las emisiones se determinan en la decisión inicial de modo y clase de transporte, no en lo que ocurre después de la reserva.

**Vínculo con el reto [EQUIPO]:** P1 — la métrica de carbono debe integrarse en el momento de reserva y aprobación (que es donde se define la emisión), no en etapas posteriores.

**Limitación [EQUIPO]:** V18 (2,774 casos, 4.79 días) = variante #3 del Avance 1, que termina en Submit Travel Request sin aprobación. Probablemente casos que seguían en trámite al momento de la extracción (log truncado). Verificar fechas si se quiere afirmar con más fuerza.

**Captura FINAL (usar en el Word):** `capturas/Q6_FINAL_tablas_ProcessCode_y_desviacion.png` — ambas tablas con título y % con 2 decimales. Decisión del equipo: se deja así (la primera tabla conserva encabezados "CO2 Promedio" y "TIempo en dias"; en el pie de figura del Word aclarar que es kg CO2e por viaje y tiempo de ciclo en días).
Capturas `Q6_01_…_provisional.png` y `Q6_02_…_provisional.png`: obsoletas, no usar (se pueden borrar).
---

### Q8 (propia) — ¿La aprobación de viajes filtra los viajes fuera de política y qué validaciones podrían automatizarse? — ✅ CERRADA (16-oct-2026)

**Hipótesis H8 [EQUIPO]:** el paso de aprobación (Travel Request Approved) funciona como control: los viajes fuera de política deberían aprobarse en menor proporción o tardar más (revisión adicional) que los que cumplen la política.

**Herramienta:** Celonis — hoja "Q8 – Aprobación vs política".

**Parte 1 — Tabla "Q8 · Aprobación de solicitudes según política" [CELONIS], verificada [VERIF-PY] exacta:**
- Dimensión: `"trip_data_with_emissions"."OUTOFPOLICY"` (renombrada "Fuera de política").
- Casos: `COUNT_TABLE("trip_data_with_emissions")`
- % aprobados: `AVG(CASE WHEN PU_COUNT("trip_data_with_emissions", "trip_event_log"."EVENTNAME", "trip_event_log"."EVENTNAME" = 'Travel Request Approved') > 0 THEN 1.0 ELSE 0.0 END)`
- Mediana horas hasta aprobación: `MEDIAN(HOURS_BETWEEN(PU_FIRST("trip_data_with_emissions", "trip_event_log"."EVENTTIMESTAMP", "trip_event_log"."EVENTNAME" = 'Submit Travel Request'), PU_FIRST("trip_data_with_emissions", "trip_event_log"."EVENTTIMESTAMP", "trip_event_log"."EVENTNAME" = 'Travel Request Approved')))`

| Fuera de política | Casos | % aprobados | Mediana horas hasta aprobación |
|---|---|---|---|
| No | 71,010 | 96.42% | 7.25 |
| Yes | 16,043 | 96.43% | 0.50 |

**Lectura parcial [EQUIPO]:** H8 se rechaza en esta primera parte. La aprobación no filtra los viajes fuera de política: se aprueban en la misma proporción (96.43% vs 96.42%) y además más rápido (mediana 0.50 h vs 7.25 h, ~14.5 veces más rápido). Es decir, el control existe en el flujo pero no actúa sobre la política. Vínculo: P2 (automatizar la validación de política, que hoy no ocurre en la aprobación) y P1 (mostrar la emisión al aprobador).
Pregunta abierta para interpretar: ¿por qué los fuera de política se aprueban más rápido? Posibles explicaciones [EQUIPO, no verificadas]: son reservas de último momento que se aprueban con urgencia; o el aprobador no ve el estado de política.

**Captura:** `capturas/Q8_FINAL_01_aprobacion_segun_politica.png`

**Parte 2 — Tabla "Q8 · Anticipación de la reserva vs. política y emisiones" [CELONIS], verificada [VERIF-PY] exacta:**
- Dimensión "Anticipación" [EQUIPO — definición propia]: días entre la PRIMERA Book Mode of Transportation y la PRIMERA salida (Take Departure Flight, Take Departure Train o Pickup Rental). Cortes 3/7/14/30 días alineados con la política del diccionario (trenes 3, autos 7, vuelos 14). Casos sin salida se separan en "6. Sin salida registrada" (si no, caerían en el ELSE).
  ```
  CASE
   WHEN PU_FIRST("trip_data_with_emissions", "trip_event_log"."EVENTTIMESTAMP", "trip_event_log"."EVENTNAME" IN ('Take Departure Flight','Take Departure Train','Pickup Rental')) IS NULL THEN '6. Sin salida registrada'
   WHEN DAYS_BETWEEN(PU_FIRST(... 'Book Mode of Transportation'), PU_FIRST(... IN ('Take Departure Flight','Take Departure Train','Pickup Rental'))) <= 3 THEN '1. 3 días o menos'
   ... <= 7 '2. 3 a 7 días' ... <= 14 '3. 7 a 14 días' ... <= 30 '4. 14 a 30 días'
   ELSE '5. Más de 30 días' END
  ```
- KPIs: Casos, % aprobados, % fuera de política, CO2 promedio (kg/viaje), Costo promedio (USD) (mismas fórmulas que arriba / Q6).

| Anticipación | Casos | % aprobados | % fuera de política | CO2 prom. (kg/viaje) | Costo prom. (USD) |
|---|---|---|---|---|---|
| 1. 3 días o menos | 7,191 | 100.00% | 100.00% | 2,218 | 2,671 |
| 2. 3 a 7 días | 5,911 | 100.00% | 56.12% | 2,998 | 3,485 |
| 3. 7 a 14 días | 11,067 | 100.00% | 44.84% | 2,800 | 3,294 |
| 4. 14 a 30 días | 56,290 | 100.00% | 0.00% | 2,677 | 2,612 |
| 5. Más de 30 días | 3,481 | 100.00% | 0.00% | 2,675 | 2,590 |
| 6. Sin salida registrada | 3,113 | 0.00% | 18.41% | 2,753 | 2,821 |

Nota: "% aprobados = 100%" en filas 1–5 es en parte tautológico (si el viaje salió, fue aprobado), pero sí sostiene una afirmación: los 7,191 viajes reservados con 3 días o menos (100% fuera de política) fueron TODOS aprobados.

**Lectura parcial [EQUIPO]:** las reservas tardías (≤14 días) concentran el 100% de los viajes fuera de política y son las más caras (US$3,294–3,485 en 3–14 días vs US$2,612 en 14–30). Las de 3–14 días son además las de mayor CO2 (2,800–2,998 kg). Contraintuitivo: las de ≤3 días emiten MENOS (2,218 kg).

**Parte 3 — Tabla "Q8 · Clase de vuelo según anticipación" [CELONIS], verificada [VERIF-PY]** (tabla aparte porque los KPIs son por tramo, common table = transportations):
- % tramos Business Class: `AVG(CASE WHEN "transportations_data_with_emissions"."SHIPPINGTYPEDESCRIPTION" = 'Business Class Flight' THEN 1.0 ELSE 0.0 END)`
- CO2 promedio tramo Economy (kg): `AVG(CASE WHEN "transportations_data_with_emissions"."SHIPPINGTYPEDESCRIPTION" = 'Economy Flight' THEN "transportations_data_with_emissions"."DEPARTURE_CO2E" ELSE NULL END)`

| Anticipación | % tramos Business Class | CO2 prom. tramo Economy (kg) |
|---|---|---|
| 1. 3 días o menos | 39.88% | 348 |
| 2. 3 a 7 días | 54.70% | 380 |
| 3. 7 a 14 días | 51.23% | 372 |
| 4. 14 a 30 días | 19.84% | 889 |
| 5. Más de 30 días | 19.64% | 884 |
| 6. Sin salida registrada | 27.89% | 818 |

**Explicación del contraintuitivo (% Business y CO2 Economy ya en CELONIS; Tren/Auto y HotelNights solo VERIF-PY):**
- Mezcla de clase por tramo: Business Class 39.9% (≤3 d), 54.7% (3–7 d), 51.2% (7–14 d) vs 19.8% (14–30 d) y 19.6% (30+). Las reservas tardías duplican o casi triplican el uso de Business.
- Las reservas tardías son rutas más cortas: CO2 promedio de un tramo Economy = 348 kg (≤3 d), 380 (3–7), 372 (7–14) vs 889 (14–30) y 884 (30+). Tren/Auto = 12.2% de los tramos en ≤3 d vs 4.2% en 14–30.
- HotelNights igual en todos los grupos (~2.9).
- Interpretación [EQUIPO]: los viajes largos se planifican con anticipación; los de último momento son regionales/cortos y aun así se reservan en Business. → rutas cortas de último momento = candidatas a tren (conecta con Q3, quick wins) y a bloqueo automático de clase Business fuera de plazo (P2).

**Captura FINAL (usar en el Word):** `capturas/Q8_FINAL_hoja_completa_3tablas.png` (hoja Q8 completa, 3 tablas, 6 filas). Las capturas `Q8_02_…PARCIAL` y `Q8_03_…PARCIAL` quedan obsoletas.

**Respuesta parcial Q8 [EQUIPO, basada en CELONIS]:** la aprobación NO filtra los viajes fuera de política: se aprueban en igual proporción (96.43% vs 96.42%) y más rápido (mediana 0.50 h vs 7.25 h). Todos los viajes que salieron con reserva de 14 días o menos (24,169 viajes = 7,191 + 5,911 + 11,067) fueron aprobados, aunque 100% / 56.12% / 44.84% estaban fuera de política. Las reservas tardías son más caras y usan 2 a 2.8 veces más Business Class, aun cuando sus rutas son más cortas.

**Validaciones automatizables propuestas [EQUIPO — propuesta, no hallazgo]:**
1. Regla automática de anticipación por modo (vuelo 14 d, auto 7 d, tren 3 d) en el momento de Submit Travel Request: si no cumple, no se aprueba automáticamente; requiere justificación (P2).
2. Bloqueo/escalamiento de Business y First Class cuando la reserva es fuera de plazo o la ruta es corta (P2).
3. Mostrar al aprobador el CO2e estimado del viaje y el estado de política (hoy la aprobación es casi inmediata para los fuera de política) (P1).
4. Sugerir tren o Economy para rutas cortas de último momento (conecta con Q3) (P1).
5. Registrar el motivo de excepción en un campo obligatorio para el reporte de carbono (P3).


---

### Q4 — ¿Existe relación entre viajes OutOfPolicy y mayores emisiones? — ✅ CERRADA (16-oct-2026)

**Hipótesis H4 [EQUIPO]:** los viajes fuera de política emiten y cuestan más por viaje que los que cumplen la política, porque al reservar tarde se eligen opciones de mayor emisión.

**Herramienta:** Celonis — hoja "Q4 – Política vs emisiones", dos OLAP Tables.

**Tabla 1 — "Q4 · Peso de los viajes fuera de política" [CELONIS], verificada [VERIF-PY] exacta:**
- Dimensión: `"trip_data_with_emissions"."OUTOFPOLICY"` (renombrada "Fuera de política").
- % de viajes: `COUNT_TABLE("trip_data_with_emissions") / GLOBAL(COUNT_TABLE("trip_data_with_emissions"))`
- CO2 total (kg): `SUM("trip_data_with_emissions"."TRANSPORTATION_CO2E" + "trip_data_with_emissions"."HOTEL_CO2E")`
- % del CO2 total: `SUM(...CO2...) / GLOBAL(SUM(...CO2...))`
- % del costo total: `SUM("trip_data_with_emissions"."NETCOSTS") / GLOBAL(SUM("trip_data_with_emissions"."NETCOSTS"))`
- CO2 promedio y Costo promedio: mismas fórmulas que Q6.

| Fuera de política | Casos | % de viajes | CO2 total (kg) | % del CO2 total | % del costo total | CO2 prom. (kg/viaje) | Costo prom. (USD) |
|---|---|---|---|---|---|---|---|
| No | 71,010 | 81.57% | 175,983,841 | 75.46% | 72.97% | 2,478 | 2,478 |
| Yes | 16,043 | 18.43% | 57,227,320 | 24.54% | 27.03% | 3,567 | 4,063 |

Control: 175,983,841 + 57,227,320 = 233,211,161 kg = total de Q1 (233,211,160.73) ✔.

**Tabla 2 — "Q4 · Clase de transporte según política" [CELONIS], verificada [VERIF-PY]:** (KPIs por tramo, tabla aparte; % = proporción de tramos)
| Fuera de política | % tramos Business Class | % tramos First Class | % tramos Economy |
|---|---|---|---|
| No | 19.59% | 1.14% | 73.56% |
| Yes | 65.49% | 3.99% | 24.08% |
Número de tramos [VERIF-PY]: No 82,291; Yes 17,634 (total 99,925). (Nota: en el chat se dio por error 84,436/15,489; el dato correcto es 82,291/17,634.)

Lección técnica (para el documento, sección de metodología): al mezclar KPIs de nivel viaje (trip_data) con KPIs de nivel tramo (transportations) en una misma OLAP Table, Celonis cambia la common table a transportations y repite los valores del viaje por cada tramo (se obtuvieron valores imposibles como 101.91% del CO2). Por eso los KPIs por tramo van en tablas separadas.

**Respuesta parcial [EQUIPO, basada en CELONIS]:** H4 se confirma. Los viajes fuera de política son 18.43% de los viajes pero generan 24.54% del CO2 y 27.03% del costo. Por viaje emiten 43.9% más (3,567 vs 2,478 kg) y cuestan 64.0% más (US$4,063 vs US$2,478). El mecanismo principal es la clase: 65.49% de sus tramos son Business Class vs 19.59% (3.3 veces más), y solo 24.08% Economy vs 73.56%.

**Escenario de impacto [EQUIPO — cálculo propio sobre cifras de Celonis, supuesto explícito]:** si los 16,043 viajes fuera de política tuvieran el CO2 y costo promedio de los que cumplen: (3,567 − 2,478) × 16,043 ≈ 17.47 M kg CO2e menos (≈7.49% del total) y (4,063 − 2,478) × 16,043 ≈ US$25.4 M menos. Es un techo teórico (supone que el viaje se mantiene, solo cambia la forma de reservarlo).

**Vínculo:** P2 (validación automática de política y clase), P1 (mostrar el CO2 extra del viaje fuera de política al reservar/aprobar), P3 (el % del CO2 fuera de política es un KPI natural de monitoreo).

**Captura FINAL:** `capturas/Q4_FINAL_hoja_2tablas.png`

---

### Q5 — ¿Qué unidades de negocio y propósitos concentran mayores emisiones? — ✅ CERRADA (16-oct-2026)

**Ajuste respecto al Avance 1 [EQUIPO]:** en el Avance 1 los totales por BU y Purpose se presentaron dentro de Q1. En el Avance 2, Q5 se diferencia porque no solo mide cuánto emite cada grupo, sino si emite de forma DESPROPORCIONADA (comparando % del CO2 vs % de viajes) y si su comportamiento es distinto (CO2 por viaje, costo por viaje, % fuera de política). Esto responde directamente a la pregunta del enunciado "Hotspot Identification: Is there a particular region, Business Unit, or travel purpose with disproportionately high output?" [ENUNCIADO].

**Hipótesis H5 [EQUIPO]:** las unidades con más contacto con clientes (Sales, Marketing) y los viajes de Customer Visit no solo emiten más en total, sino que tienen un comportamiento de viaje más intensivo en carbono (más CO2 por viaje y más viajes fuera de política).

**Herramienta:** Celonis — hoja "Q5 – Unidades y propósitos", dos OLAP Tables con los mismos KPIs de Q4 + % fuera de política. Dimensiones: `"trip_data_with_emissions"."BUSINESSUNIT"` ("Unidad de negocio") y `"trip_data_with_emissions"."PURPOSE"` ("Propósito del viaje").

**Tabla "Q5 · Emisiones por unidad de negocio" [CELONIS], verificada [VERIF-PY]:**
| Unidad de negocio | Casos | % de viajes | CO2 total (kg) | % del CO2 total | % del costo total | CO2 prom. (kg/viaje) | Costo prom. (USD) | % fuera de política |
|---|---|---|---|---|---|---|---|---|
| Sales | 29,105 | 33.43% | 78,412,703 | 33.62% | 33.48% | 2,694 | 2,774 | 18.52% |
| Marketing | 16,311 | 18.74% | 43,526,827 | 18.66% | 18.76% | 2,669 | 2,774 | 18.43% |
| Value Engineering | 8,587 | 9.86% | 22,565,611 | 9.68% | 9.69% | 2,628 | 2,722 | 17.31% |
| Customer Support | 7,122 | 8.18% | 19,282,888 | 8.27% | 8.32% | 2,708 | 2,816 | 19.36% |
| Executive Management | 6,270 | 7.20% | 16,736,255 | 7.18% | 7.16% | 2,669 | 2,752 | 18.45% |
| Ecosystem | 4,665 | 5.36% | 13,022,159 | 5.58% | 5.55% | 2,791 | 2,867 | 18.86% |
| Services | 4,438 | 5.10% | 11,588,893 | 4.97% | 5.04% | 2,611 | 2,738 | 17.87% |
| Operations | 3,623 | 4.16% | 9,531,566 | 4.09% | 4.05% | 2,631 | 2,697 | 18.49% |
| Legal | 2,125 | 2.44% | 5,763,074 | 2.47% | 2.47% | 2,712 | 2,799 | 18.40% |
| Office Management | 1,878 | 2.16% | 4,991,423 | 2.14% | 2.13% | 2,658 | 2,731 | 19.01% |
| Engineering | 1,188 | 1.36% | 3,272,933 | 1.40% | 1.42% | 2,755 | 2,890 | 19.61% |
| Finance | 1,002 | 1.15% | 2,561,495 | 1.10% | 1.10% | 2,556 | 2,647 | 17.86% |
| Product Management | 644 | 0.74% | 1,698,200 | 0.73% | 0.73% | 2,637 | 2,743 | 16.15% |
| Sustainability | 95 | 0.11% | 257,132 | 0.11% | 0.10% | 2,707 | 2,618 | 17.89% [VERIF-PY; tapado por íconos en la captura] |

**Tabla "Q5 · Emisiones por propósito del viaje" [CELONIS], verificada [VERIF-PY]:**
| Propósito del viaje | Casos | % de viajes | CO2 total (kg) | % del CO2 total | % del costo total | CO2 prom. | Costo prom. | % fuera de política |
|---|---|---|---|---|---|---|---|---|
| Customer Visit | 37,621 | 43.22% | 100,842,939 | 43.24% | 43.07% | 2,680 | 2,761 | 18.58% |
| Conference/Exhibition | 30,976 | 35.58% | 82,532,345 | 35.39% | 35.55% | 2,664 | 2,767 | 18.26% |
| Internal Business Trip | 9,822 | 11.28% | 26,503,192 | 11.36% | 11.44% | 2,698 | 2,807 | 18.45% |
| Enablement | 6,148 | 7.06% | 16,631,453 | 7.13% | 7.08% | 2,705 | 2,779 | 18.30% |
| Partner Visit | 2,486 | 2.86% | 6,701,232 | 2.87% | 2.86% | 2,696 | 2,770 | 18.50% |

**Respuesta parcial [EQUIPO, basada en CELONIS]:** H5 se rechaza en su segunda parte. Sales y Marketing concentran 52.28% del CO2 y Customer Visit + Conference/Exhibition 78.63%, pero ninguna unidad ni propósito emite de forma desproporcionada: el % del CO2 es prácticamente igual al % de viajes en todos los grupos (diferencia máxima ≈ 0.2 puntos: Ecosystem 5.36% de viajes → 5.58% del CO2). El CO2 por viaje varía solo entre 2,556 y 2,791 kg, y el % fuera de política entre 16.15% y 19.61% por unidad (18.26%–18.58% por propósito). Conclusión: las emisiones dependen del VOLUMEN de viajes de cada unidad, no de un comportamiento distinto. Por lo tanto, la palanca no es una política dirigida a una unidad, sino controles transversales en el proceso de reserva/aprobación (P1, P2), y presupuestos o metas de carbono por unidad según su volumen (P3).

**Captura FINAL:** `capturas/Q5_FINAL_hoja_2tablas.png` (el % fuera de política de Sustainability quedó tapado por los íconos de la barra flotante de Celonis; si se quiere, recapturar sin el mouse encima).

---

### Q7 (propia) — ¿Qué tan confiable es Spend_CO2e frente al cálculo real? — ✅ CERRADA (16-oct-2026) — CANDIDATA PRINCIPAL A "ANÁLISIS ORIGINAL"

**Contexto [EQUIPO]:** Spend_CO2e estima el CO2 a partir del gasto (NetCosts) — "spend-based method", común cuando una empresa no tiene datos de actividad por viaje. Hotel_CO2e + Transportation_CO2e es el cálculo basado en actividad. NO se suman: son dos estimaciones del mismo impacto. Q7 compara ambas para evaluar si la empresa podría confiar en el método por gasto para su contabilidad de carbono (P3).

**Hipótesis H7 [EQUIPO]:** el método por gasto aproxima bien el total, pero distorsiona las comparaciones entre grupos, porque el precio no es proporcional a la emisión (un tramo corto caro o un vuelo largo barato).

**Herramienta:** Celonis — hoja "Q7 – Spend_CO2e vs cálculo real", dos OLAP Tables.
- CO2 calculado (kg): `SUM("trip_data_with_emissions"."TRANSPORTATION_CO2E" + "trip_data_with_emissions"."HOTEL_CO2E")`
- CO2 por gasto (kg): `SUM("trip_data_with_emissions"."SPEND_CO2E")`
- Razón gasto / calculado: `SUM("trip_data_with_emissions"."SPEND_CO2E") / SUM("trip_data_with_emissions"."TRANSPORTATION_CO2E" + "trip_data_with_emissions"."HOTEL_CO2E")` (1.00 = coinciden; <1 subestima; >1 sobreestima)
- Lección técnica: un primer intento dio totales inflados ×10.3 (= eventos por caso, 895,366/87,053) porque un KPI hacía que la common table fuera trip_event_log; se corrigió dejando todos los KPIs sobre trip_data_with_emissions.

**Tabla "Q7 · Método por gasto vs. cálculo real, por región" [CELONIS], verificada [VERIF-PY]:**
| Región de salida | Casos | CO2 calculado (kg) | CO2 por gasto (kg) | Razón gasto/calculado |
|---|---|---|---|---|
| Asia | 12,512 | 38,242,382.21 | 46,363,897.75 | 1.21 |
| Europa | 12,063 | 5,263,797.20 | 1,846,241.51 | 0.35 |
| Latinoamérica | 19,533 | 52,662,072.64 | 31,449,573.79 | 0.60 |
| Norteamérica | 12,636 | 18,366,297.30 | 17,384,555.79 | 0.95 |
| Oceanía | 28,395 | 117,598,380.59 | 97,647,663.81 | 0.83 |
| África | 1,914 | 1,078,230.80 | 685,740.34 | 0.64 |

**Tabla "Q7 · Método por gasto vs. cálculo real, según política" [CELONIS], verificada [VERIF-PY]:**
| Fuera de política | Casos | CO2 calculado (kg) | CO2 por gasto (kg) | Razón |
|---|---|---|---|---|
| No | 71,010 | 175,983,840.81 | 137,176,060.44 | 0.78 |
| Yes | 16,043 | 57,227,319.92 | 58,201,612.55 | 1.02 |

Totales (suma de filas): calculado 233,211,160.73 kg vs por gasto 195,377,673 kg → razón 0.84 (el método por gasto subestima el total en ~16.2%).
Nivel viaje [VERIF-PY, no en Celonis]: correlación Spend_CO2e vs CO2 calculado = 0.89; correlación Spend_CO2e vs NetCosts = 0.95; mediana de la diferencia por viaje = −37.8%; solo 17.9% de los viajes tiene Spend_CO2e > calculado.

**Respuesta parcial [EQUIPO, basada en CELONIS]:** H7 se confirma, y más fuerte de lo esperado. El método por gasto no solo subestima el total (−16%), sino que invierte el ranking regional: según el gasto, Asia (46.4 M kg) superaría a Latinoamérica (31.4 M kg) — en el cálculo real es al revés (38.2 M vs 52.7 M); y Europa aparecería casi 3 veces más eficiente de lo que es (razón 0.35). Además, el método por gasto "castiga" los viajes caros (fuera de política, razón 1.02) y deja pasar los baratos de alta emisión (dentro de política, 0.78). Conclusión: si la empresa reportara con el método por gasto, focalizaría mal sus esfuerzos.

**Vínculo:** P3 directamente — el sistema de contabilidad de carbono debe usar datos de actividad (modo, clase, distancia, noches de hotel) por viaje, que el proceso de reserva ya captura; el método por gasto solo como respaldo. También P1: si la métrica de carbono se integra en la reserva, debe ser la de actividad, no la de gasto.

**Por qué es "original" [EQUIPO]:** no es una pregunta del enunciado; cuestiona la calidad del propio dato de emisiones (data quality / metodología de medición), conecta con cómo las empresas reales reportan Scope 3 (categoría 6, viajes de negocio) y cambia las conclusiones de Q2.

**Captura FINAL:** `capturas/Q7_FINAL_hoja_2tablas.png` (encabezado "Region de salida" sin tilde; corregir en el pie de figura del Word).

---

## 2b. Dashboard — "Dashboard – Huella de carbono de viajes" (Celonis) — ✅ CERRADO (17-oct-2026)

**Fila 1 — 4 Single KPI [CELONIS], verificados [VERIF-PY]:**
| Título | Fórmula | Valor |
|---|---|---|
| CO2 total (t CO2e) | `SUM(T+H)/1000` | 233,211 |
| CO2 promedio por viaje (kg) | `AVG(T+H)` | 2,679 |
| % de viajes fuera de política | `AVG(CASE WHEN OUTOFPOLICY='Yes' THEN 1.0 ELSE 0.0 END)` | 18.43% |
| % del CO2 generado fuera de política | `SUM(CASE WHEN OUTOFPOLICY='Yes' THEN T+H ELSE 0.0 END)/SUM(T+H)` | 24.54% |
(T+H = "trip_data_with_emissions"."TRANSPORTATION_CO2E" + "trip_data_with_emissions"."HOTEL_CO2E")

**Fila 2 — 2 Bar charts:**
- "CO2 de transporte por modo": dimensión SHIPPINGTYPEDESCRIPTION ("Modo de transporte"), KPI `SUM(DEPARTURE_CO2E + RETURN_CO2E) / GLOBAL(SUM(DEPARTURE_CO2E + RETURN_CO2E))`. Business 49.06%, Economy 46.07%, First 4.19%, resto <1% (coincide con Q3 del Avance 1). [VERIF-PY: la suma Departure+Return de los 99,925 tramos = 226,715,208.63 = Transportation_CO2e total.]
- "% de viajes vs. % del CO2 por región": dimensión "Región de salida", KPIs % del CO2 total y % de viajes (barras agrupadas, no apiladas). Mensaje: Oceanía 32.62% de viajes → 50.43% del CO2; Europa 13.86% → 2.26%; Norteamérica 14.52% → 7.88%.

**Fila 3 — Column chart a todo el ancho "CO2 promedio y % fuera de política según anticipación":** dimensión "Anticipación" (Q8); KPIs CO2 promedio (kg/viaje) y % fuera de política. Valores CO2: 2,218 / 2,998 / 2,800 / 2,677 / 2,675 / 2,753 (coinciden con Q8).
✅ Corregido (17-oct): "% fuera de política" ahora en eje secundario (0–100%) a la derecha.
Opcionales no hechos: Dropdown de BUSINESSUNIT (interactividad, P3); "% de viajes" en gris en el gráfico de regiones.

**Captura FINAL:** `capturas/Dashboard_FINAL.png` (versión corregida, sin filtros). `Dashboard_01_filas1y2_provisional.png` queda obsoleta.
**Captura de interactividad:** `capturas/Dashboard_ejemplo_filtro_7a14dias.png` — dashboard filtrado por Anticipación = "3. 7 a 14 días" (13% de los viajes): 30,986 t CO2e, 2,800 kg/viaje, 44.84% fuera de política y 78.32% del CO2 de esa franja generado por viajes fuera de política [CELONIS]. Control: 11,067 × 2,800 ≈ 30.99 M kg ✔. Útil para la presentación (P3, monitoreo).
---

### Conformidad del proceso (nuevo, process mining) — ✅ CERRADA en Celonis (17-oct-2026); se integra a Q8 / P2

**Regla de conformidad [ENUNCIADO / diccionario]:** "Submit Request" debe ocurrir después de "Book Mode of Transportation".
**Hipótesis HC [EQUIPO]:** el proceso respeta el orden reserva → solicitud → aprobación, y cualquier cambio de transporte posterior a la aprobación vuelve a pasar por aprobación.

**Herramienta:** Celonis — hoja "Conformidad del proceso", dos OLAP Tables con los KPIs: Casos, % de viajes, % fuera de política, CO2 promedio, Costo promedio, Mediana horas hasta aprobación.
- Dimensión "Regla del diccionario": `CASE WHEN PU_FIRST(trip, EVENTTIMESTAMP, EVENTNAME = 'Submit Travel Request') < PU_FIRST(trip, EVENTTIMESTAMP, EVENTNAME = 'Book Mode of Transportation') THEN 'Viola: solicitud antes de reservar' ELSE 'Cumple' END`
- Dimensión "Re-reserva": `CASE WHEN PU_LAST(trip, EVENTTIMESTAMP, EVENTNAME = 'Book Mode of Transportation') > PU_FIRST(trip, EVENTTIMESTAMP, EVENTNAME = 'Travel Request Approved') THEN 'Re-reserva sin nueva aprobación' ELSE 'Sin re-reserva posterior' END`

**Resultados [CELONIS], verificados [VERIF-PY]:**
| Grupo | Casos | % de viajes | % fuera de política | CO2 prom. (kg/viaje) | Costo prom. (USD) | Mediana horas hasta aprobación |
|---|---|---|---|---|---|---|
| Cumple | 86,582 | 99.46% | 17.99% | 2,683 | 2,772 | 3.53 |
| Viola: solicitud antes de reservar | 471 | 0.54% | 100.00% | 1,946 | 2,414 | 0.58 |
| Re-reserva sin nueva aprobación | 3,504 | 4.03% | 28.45% | 2,537 | 2,664 | 1.22 |
| Sin re-reserva posterior | 83,549 | 95.97% | 18.01% | 2,685 | 2,774 | 3.52 |

Contexto [VERIF-PY, no en Celonis]: en los 471 casos que violan la regla, la aprobación ocurrió en mediana ~22.5 h ANTES de la reserva del transporte. Cada caso tiene exactamente 1 Submit Travel Request y como máximo 1 Travel Request Approved → ninguna re-reserva tuvo segunda aprobación. Las re-reservas posteriores a la aprobación vienen sobre todo de Flight Cancellation (1,070 casos), Missed Flight (947), Rental Cancellation (494), Missed Pickup (387).

**Respuesta [EQUIPO, basada en CELONIS]:** HC se rechaza. (1) 471 viajes (0.54%) violan la regla del diccionario: la solicitud se envió y aprobó antes de reservar el transporte; el 100% de estos viajes está fuera de política. (2) 3,504 viajes (4.03%) cambiaron su reserva después de aprobados sin pasar por una nueva aprobación; en este grupo el % fuera de política sube a 28.45% (vs 18.01%). Ambos son huecos de control: la aprobación no valida lo que finalmente se reserva. Vínculo: P2 (reglas automáticas: bloquear aprobación sin reserva asociada; re-validar política y CO2 al cambiar la reserva) y P3 (registrar estas excepciones como indicadores de conformidad).

**Captura FINAL:** `capturas/Conformidad_FINAL_hoja_2tablas.png`. `Conformidad_provisional_titulo_repetido.png` queda obsoleta.

---

### Q1 — ✅ Resuelta en Avance 1 (se reutiliza; ver EntregaAvance1.pdf pp. 14–15)
### Q3 — ¿Qué rutas y modos son quick wins frente a inversiones a largo plazo? — ✅ Avance 1 (modos) + ✅ Celonis rutas (17-oct-2026)

Base del Avance 1 (Excel): vuelos 94,087 tramos (94.16%) y 99.33% del CO2 de transporte; Business + First 29.33% de tramos y 53.25% del CO2 (Business 49.06%, First 4.19%). Corrección: First Class = 1,639 tramos = 1.64% (en la Figura 7 del Avance 1 decía 1.84%).

**Nuevo en Celonis — hoja "Q3 – Rutas con alternativa terrestre", tabla "Q3 · Vuelo vs. tren en la misma ruta" [CELONIS], verificada [VERIF-PY]:**
- Dimensión "Ruta": `CONCAT("transportations_data_with_emissions"."DEPARTURELOCATIONCITY", ' → ', "transportations_data_with_emissions"."ARRIVALLOCATIONCITY")` (60 rutas)
- KPIs (todos por tramo): Tramos en vuelo `SUM(CASE WHEN ...SHIPPINGTYPEDESCRIPTION LIKE '%Flight' THEN 1 ELSE 0 END)`; Tramos en tren (= 'Train'); CO2 prom. vuelo Economy (kg, DEPARTURE_CO2E); CO2 prom. tren (kg); % vuelos en Business (Business / vuelos); CO2 total en vuelos (kg, DEPARTURE+RETURN).

| Ruta | Tramos vuelo | Tramos tren | CO2 prom. Economy (kg) | CO2 prom. tren (kg) | % vuelos Business | CO2 total vuelos (kg) |
|---|---|---|---|---|---|---|
| Berlin → Amsterdam | 814 | 1,007 | 88 | 27 | 17.69% | 155,852 |
| Berlin → Paris | 1,323 | 474 | 131 | 28 | 43.84% | 426,295 |
| Berlin → Madrid | 3,830 | 336 | 274 | 67 | 37.00% | 2,516,685 |
| Hamburg → Paris | 451 | 256 | 113 | 25 | 21.73% | 112,691 |
| Hamburg → Berlin | 0 | 161 | – | 8 | – | 0 |
| Frankfurt → Berlin | 49 | 84 | 113 | 14 | 0.00% | 11,094 |
| London → Paris | 0 | 73 | – | 20 | – | 0 |
| Fukuoka → Tokyo | 298 | 65 | 131 | 22 | 13.42% | 83,097 |
| Berlin → London | 1,483 | 60 | 143 | 50 | 66.89% | 574,831 |
| Frankfurt → Paris | 0 | 47 | – | 18 | – | 0 |
| New York → Toronto | 2,320 | 27 | 83 | 68 | 78.53% | 542,660 |
| Madrid → Lisbon | 340 | 14 | 76 | 17 | 17.65% | 56,265 |
| Sydney → Perth | 1,663 | 3 | 486 | 163 | 21.77% | 1,794,181 |
(El resto de las 60 rutas tiene 0 tramos en tren; p. ej. Beijing → Moscow 4,476 vuelos y 12,387,541 kg; Beijing → New York 95.43% Business.)

**Lectura [EQUIPO]:**
- En 13 rutas la empresa ya usa tren; en 3 (Hamburg → Berlin, London → Paris, Frankfurt → Paris) se viaja SOLO en tren → prueba interna de que la alternativa es viable.
- En la misma ruta, el tren emite 3 a 8 veces menos que un vuelo Economy en Europa y Japón (Berlin → Paris 28 vs 131 kg; Frankfurt → Berlin 14 vs 113; Fukuoka → Tokyo 22 vs 131); en New York → Toronto solo 68 vs 83 kg.
- Rutas cortas con mucho Business: Berlin → London 66.89%, New York → Toronto 78.53% → quick win de CLASE además de modo.
- Escenario [EQUIPO — cálculo propio sobre cifras de Celonis, supuesto: todo vuelo de la ruta pasa a tren con el CO2 promedio del tren, ida y vuelta]: rutas cortas con tren ya usado (Berlin → Amsterdam, Berlin → Paris, Hamburg → Paris, Frankfurt → Berlin, Fukuoka → Tokyo, Berlin → London, Madrid → Lisbon) → ahorro ≈ 1.1 M kg CO2e ≈ 0.47% del total. Se excluyen Berlin → Madrid, New York → Toronto y Sydney → Perth por distancia (tren de muchas horas / días).
- Conclusión de priorización: el tren es un quick win visible y fácil, pero pequeño (~0.5%); la palanca grande es la clase del vuelo (techo de 7.49% en Q4). Largo plazo: rutas intercontinentales (Oceanía, Beijing).

**Captura FINAL:** `capturas/Q3_FINAL_vuelo_vs_tren_por_ruta.png`
### Q2 — ¿Cómo se comparan las regiones en eficiencia de carbono por viaje y cuál ofrece mayor potencial de reducción? — ✅ CERRADA en Celonis (16-oct-2026); pendiente benchmarking externo

**Ajuste respecto al Avance 1 [EQUIPO]:** en el Avance 1 se contaron tramos por país de LLEGADA (Figura 6, solo conteos). En el Avance 2 la región se asigna por país de SALIDA, porque: (1) en los datos solo hay 9 países y 10 ciudades de salida (Sydney, São Paulo, Beijing, Berlín, Hamburgo, Nueva York, Ciudad de México, Ciudad del Cabo, Madrid, Fukuoka) [VERIF-PY], que representan las sedes desde donde se reserva y aprueba el viaje; (2) el enunciado pregunta por "our North American processes compare to Europe and Asia" [ENUNCIADO], es decir, por el proceso de cada sede; (3) por llegada (20 países) la región refleja el destino (clientes/conferencias), que la empresa no controla. Además se mide CO2e real, no solo conteos.
Mapeo de regiones [EQUIPO]: Norteamérica = US, MX (México se incluye en Norteamérica por criterio geográfico; declararlo); Europa = DE, ES; Asia = CN, JP; Oceanía = AU; Latinoamérica = BR; África = ZA.

**Hipótesis H2 [EQUIPO]:** Norteamérica tiene peor eficiencia de carbono por viaje que Europa (donde hay más alternativas terrestres), y la región con mayor volumen de viajes es la de mayor potencial de reducción.

**Herramienta:** Celonis — hoja "Q2 – Regiones", dos OLAP Tables. Dimensión "Región de salida":
```
CASE
 WHEN "trip_data_with_emissions"."DEPARTURELOCATIONCOUNTRY" IN ('US','MX') THEN 'Norteamérica'
 WHEN "trip_data_with_emissions"."DEPARTURELOCATIONCOUNTRY" IN ('DE','ES') THEN 'Europa'
 WHEN "trip_data_with_emissions"."DEPARTURELOCATIONCOUNTRY" IN ('CN','JP') THEN 'Asia'
 WHEN "trip_data_with_emissions"."DEPARTURELOCATIONCOUNTRY" = 'AU' THEN 'Oceanía'
 WHEN "trip_data_with_emissions"."DEPARTURELOCATIONCOUNTRY" = 'BR' THEN 'Latinoamérica'
 WHEN "trip_data_with_emissions"."DEPARTURELOCATIONCOUNTRY" = 'ZA' THEN 'África'
 ELSE 'Otro' END
```
(No apareció fila "Otro" → el mapeo cubre los 87,053 viajes.)
KPIs tabla 1: los 8 de Q5. KPIs tabla 2 (por tramo): % tramos Business Class; % tramos Tren `AVG(CASE WHEN ..."SHIPPINGTYPEDESCRIPTION" = 'Train' THEN 1.0 ELSE 0.0 END)`; % tramos Auto `AVG(CASE WHEN ..."SHIPPINGTYPEDESCRIPTION" IN ('BMW 3 diesel','BMW 3 plugin hybrid','Fiat 500 electric','Volkswagen Golf diesel','Volkswagen Golf petrol') THEN 1.0 ELSE 0.0 END)`.

**Tabla "Q2 · Emisiones por región de salida" [CELONIS], verificada [VERIF-PY]:**
| Región de salida | Casos | % de viajes | CO2 total (kg) | % del CO2 total | % del costo total | CO2 prom. (kg/viaje) | Costo prom. (USD) | % fuera de política |
|---|---|---|---|---|---|---|---|---|
| Oceanía | 28,395 | 32.62% | 117,598,381 | 50.43% | 41.24% | 4,142 | 3,502 | 13.64% |
| Latinoamérica | 19,533 | 22.44% | 52,662,073 | 22.58% | 23.94% | 2,696 | 2,955 | 11.68% |
| Norteamérica | 12,636 | 14.52% | 18,366,297 | 7.88% | 12.97% | 1,453 | 2,476 | 30.14% |
| Asia | 12,512 | 14.37% | 38,242,382 | 16.40% | 15.75% | 3,056 | 3,036 | 18.85% |
| Europa | 12,063 | 13.86% | 5,263,797 | 2.26% | 5.65% | 436 | 1,128 | 27.84% |
| África | 1,914 | 2.20% | 1,078,231 | 0.46% | 0.45% | 563 | 569 | 18.97% |

**Tabla "Q2 · Modo de transporte por región" [CELONIS], verificada [VERIF-PY]:**
| Región de salida | % tramos Business Class | % tramos Tren | % tramos Auto |
|---|---|---|---|
| Oceanía | 21.69% | 0.01% | 0.91% |
| Latinoamérica | 20.25% | 0.00% | 0.01% |
| Norteamérica | 49.60% | 1.78% | 14.74% |
| Asia | 35.01% | 0.47% | 0.01% |
| Europa | 28.87% | 18.25% | 7.48% |
| África | 16.93% | 0.00% | 1.46% |

Contexto de destinos [VERIF-PY, no está en Celonis; reproducir si se va a citar]: Europa viaja 100% dentro de Europa (ES, FR, NL, GB, IT, DE, PT); África 100% dentro de ZA; Oceanía viaja sobre todo a US (7,373), DE (7,165) y SG (6,528) — ~80% intercontinental; Norteamérica ~87% dentro de Norteamérica (MX, CA, US).

**Respuesta parcial [EQUIPO, basada en CELONIS]:** H2 se confirma solo en parte.
- Norteamérica vs Europa vs Asia: por viaje, Norteamérica emite 1,453 kg, 3.3 veces más que Europa (436 kg), y Asia 3,056 kg (7 veces Europa). Europa es el referente interno de eficiencia: 18.25% de sus tramos son en tren y 7.48% en auto.
- Norteamérica es la región con PEOR COMPORTAMIENTO: mayor % fuera de política (30.14%) y casi la mitad de sus tramos en Business Class (49.60%); su costo (12.97% del total) pesa más que su CO2 (7.88%). → potencial de quick wins por política y validación automática (P2), no por cambio de destino.
- Oceanía es la región con MAYOR POTENCIAL ABSOLUTO: 32.62% de los viajes pero 50.43% del CO2 (4,142 kg/viaje) por la distancia (viajes intercontinentales). Ahí la palanca es de largo plazo: reducir/consolidar viajes, reuniones virtuales, priorizar Economy en vuelos largos (P1, P3).
- La hipótesis de "mayor volumen = mayor potencial" se cumple en Oceanía, pero no es lineal: Latinoamérica (22.44% de viajes) emite proporcional a su volumen.

**Pendiente (asignado a un compañero del equipo, no a Kendal):** benchmarking con la industria tech [ENUNCIADO: "How does this compare to others in the tech industry"] → buscar reportes públicos de emisiones por viajes de negocio de empresas tech (fuente externa, citar).

**Captura FINAL:** `capturas/Q2_FINAL_hoja_2tablas.png` (encabezado corregido; tabla 1 ordenada por Casos; tabla 2 en orden alfabético). `Q2_01_regiones_provisional.png` queda obsoleta.

---

## 2c. Punto de partida para el benchmarking (lo hace un compañero) — fuentes públicas encontradas el 17-oct, SIN revisar a fondo
- Salesforce, Scope 3 cat. 6 "Business travel": 148 mil t CO2e (FY2025), 145 mil (FY2024), 129 mil (FY2019 base). https://www.salesforce.com/en-us/wp-content/uploads/sites/4/documents/white-papers/fy25-schedules-of-environmental-and-employee-metrics.pdf
- SAP, "Category 6 - Business Travel": 108,100 t CO2e (2024), 82,600 (2023); "Business flights" 101,200 (2024). https://www.sap.com/integrated-reports/2024/en/datahub/environmental-data.html
- Microsoft, "Category 6 – Business Travel": 260,000 t CO2e (FY24), 133,000 (FY23), 329,356 (FY20). https://cdn-dynmedia-1.microsoft.com/is/content/microsoftcorp/microsoft/msc/documents/presentations/CSR/2025-Microsoft-Environmental-Data-Fact-Sheet-PDF.pdf
- Nuestro caso: 233,211 t CO2e en el período del log (dic-2023 a jun-2025, ~19 meses) → para comparar hay que anualizar y, idealmente, normalizar por empleado o por ingresos (los reportes no siempre dan headcount). Ojo: el caso es adaptado/ficticio.
- Celonis tiene metas aprobadas por SBTi (ver celonis.com/news/article/science-based-targets-initiative-approves-celonis-near-term-emission-reduction-targets) — no se llegó a leer.

## 3. Pistas para próximos bloques — [VERIF-PY], NO citar hasta reproducir en Celonis/Excel

- **Q8 parte 1:** ya reproducida en Celonis (ver Q8 arriba). Mediana general Submit→Approved 3.44 h [VERIF-PY].
- **Q4 (ya reproducida en Celonis, ver arriba):** OutOfPolicy=Yes 16,043 viajes (18.43%), prom. 3,567 kg y US$4,063, suma 57.23 M kg = 24.54% del CO2 y 27.03% del costo; No 71,010, prom. 2,478 kg y US$2,478, 175.98 M kg (75.46%). Tramos fuera de política: Business 65.49% vs 19.59%; First 3.99% vs 1.14%; Economy 24.08% vs 73.56%. Escenario [EQUIPO, supuesto]: si los viajes fuera de política emitieran/costaran como los dentro de política → −17.47 M kg (7.49% del total) y −US$25.43 M. Economy fuera de política emite 411 kg/tramo vs 834 (rutas más cortas).
- **Anticipación** (Book Mode of Transportation → primera salida: vuelo, tren o pickup de auto): <3 días 7,191 viajes (100% fuera de política); 3–7 d 5,911 (56%); 7–14 d 11,067 (45%); 14–30 d 56,290 (0%); 30+ d 3,481 (0%).

- **Q5 (ya reproducida en Celonis, ver arriba) [VERIF-PY]:** emisión por viaje, costo, % fuera de política y % Business son casi iguales en las 14 BU y los 5 propósitos (CO2/viaje 2,556–2,791; % fuera de política 16.15%–19.61% por BU y 18.26%–18.58% por propósito; % tramos Business 26.09%–30.43%). Las diferencias en CO2 total vienen del VOLUMEN de viajes, no del comportamiento. Sales+Marketing = 52.28% del CO2.

## 4. Pendientes de cierre
- Q8 → Q4 → Q2 → Q5 → Q7, análisis original, dashboards, storytelling, documento, presentación.
