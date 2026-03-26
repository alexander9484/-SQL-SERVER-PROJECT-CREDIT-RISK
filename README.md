# Proyecto SQL: Análisis de Riesgo Crediticio y Perfil del Solicitante
_El equipo de Riesgos de la institución financiera necesita mitigar la exposición a impagos y optimizar la asignación de tasas de interés. Actualmente, carecen de una visión integral sobre qué perfiles sociodemográficos y financieros presentan mayor probabilidad de default. Mi objetivo es utilizar **SQL** para realizar un Análisis Exploratorio de Datos (EDA) exhaustivo, descubriendo patrones de comportamiento que no solo brinden recomendaciones inmediatas al comité de crédito, sino que sirvan como base para el desarrollo de futuros modelos predictivos de Machine Learning._

## Estructura del Proyecto

- [Sobre los Datos](#sobre-los-datos)
- [Tareas (Task)](#tareas-task)
- [Limpieza de Datos](#limpieza-de-datos)
- [Análisis Exploratorio de Datos (EDA) e Insights](#análisis-exploratorio-de-datos-eda-e-insights)
- [Conclusión](#conclusión)

## Sobre los Datos

El conjunto de datos original contiene registros históricos de solicitudes de crédito, abarcando información demográfica, financiera y la decisión/resultado final del préstamo.

Los datos originales se pueden encontrar [aquí]
(https://www.kaggle.com/datasets/laotse/credit-risk-dataset)

**Diccionario de variables clave:**
* `person_age` / `person_income`: Edad e ingresos anuales del solicitante.
* `person_home_ownership`: Situación de vivienda (RENT, OWN, MORTGAGE).
* `person_emp_length`: Años de antigüedad laboral.
* `loan_intent`: Propósito del préstamo (EDUCATION, MEDICAL, VENTURE, etc.).
* `loan_grade`: Calificación de riesgo asignada (A - G).
* `loan_amnt` / `loan_int_rate`: Monto solicitado y tasa de interés asignada.
* `loan_status`: Variable objetivo (0 = Pagado, 1 = Default/Mora).
* `loan_percent_income`: Ratio Deuda-Ingreso (DTI).
* `cb_person_default_on_file`: Historial previo de impago (Y/N).

## Tareas (Task)

En este análisis, resolvemos 10 preguntas de negocio de dificultad progresiva:

1. **Distribución:** Volumen y ticket promedio por propósito del préstamo.
2. **Morosidad por Grado:** Tasa de impago según la calificación de riesgo.
3. **Solvencia y Vivienda:** Ingreso y monto promedio según situación de vivienda.
4. **Alerta DTI:** Tasa de mora en clientes sobreendeudados con historial corto.
5. **Impacto de Tasas:** Comportamiento de pago según bandas de tasas de interés.
6. **Estabilidad Laboral:** Riesgo de impago vs. años en el empleo actual.
7. **Jóvenes de Riesgo:** Perfilamiento de menores de 25 años con ingresos bajos y manchas previas.
8. **Top 3 Riesgosos:** Identificación de los peores créditos por categoría.
9. **Detección de Anomalías:** Solicitudes atípicas que superan el promedio de su segmento.
10. **Rendimiento por Cuartiles:** Análisis de impago dividiendo a la población en cuartiles de ingresos equitativos.

## Limpieza de Datos

Antes de extraer insights, validamos la integridad estructural de la tabla `credit_risk_data`. Nos enfocamos en buscar valores nulos en variables críticas como antigüedad laboral y tasa de interés, además de verificar registros duplicados.

```sql
-- Verificar valores nulos en columnas críticas --
SELECT 
    SUM(CASE WHEN person_emp_length IS NULL THEN 1 ELSE 0 END) AS Null_Emp_Length,
    SUM(CASE WHEN loan_int_rate IS NULL THEN 1 ELSE 0 END) AS Null_Int_Rate
FROM credit_risk_dataset;

-- Detección de filas completamente duplicadas (asumiendo un pseudo-ID si existiera, o agrupando por todas las columnas) --
SELECT person_age, person_income, loan_amnt, loan_int_rate, COUNT(*) as Duplicates
FROM credit_risk_dataset
GROUP BY person_age, person_income, loan_amnt, loan_int_rate
HAVING COUNT(*) > 1;
```
## Análisis Exploratorio de Datos (EDA) e Insights

### Pregunta #1: ¿Cuál es la distribución de préstamos por propósito (volumen y ticket promedio)?

Para entender dónde se concentra el capital, calculé el volumen total de operaciones y el monto promedio de los préstamos utilizando las funciones de agregación COUNT, SUM y AVG. Agrupé los resultados por la intención del préstamo (loan_intent) para obtener una vista clara de la demanda.

```sql
-- Distribución y monto promedio por propósito de préstamo --

SELECT loan_intent, 
    COUNT(*) AS Total_Loans, 
    SUM(loan_amnt) AS Total_Amount_Lent, 
    ROUND(AVG(loan_amnt), 2) AS Avg_Ticket
FROM credit_risk_dataset
GROUP BY loan_intent
ORDER BY Total_Loans DESC;
```

![image](https://raw.githubusercontent.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/1205fc6e5f9af72623b5415dceafc2936ba63208/Picture/P1.png)

_Volumen de préstamos y montos promedio agrupados por propósito_

Esto permite al negocio entender qué necesidades fundamentales están financiando (ej. educación, salud) y en qué categorías se encuentra el mayor volumen de capital en riesgo.

### Pregunta #2: ¿Cuál es la tasa de morosidad según la calificación de riesgo asignada?
Calculé el porcentaje de préstamos en mora agrupados por su grado de riesgo (loan_grade). Utilicé la función CAST para convertir el estado del préstamo a FLOAT y así obtener un promedio decimal exacto con la función AVG, que luego multipliqué por 100.

```sql
-- Tasa de morosidad por grado de riesgo --

SELECT loan_grade, 
    COUNT(*) AS Total_Loans, 
    SUM(loan_status) AS Defaulted_Loans, 
    ROUND(AVG(CAST(loan_status AS FLOAT)) * 100, 2) AS Default_Rate_Pct
FROM credit_risk_dataset
GROUP BY loan_grade
ORDER BY loan_grade;
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P2.png?raw=true)

_Porcentaje de impagos clasificado por grado de riesgo (A-G)_

Esto valida si el sistema de scoring inicial del banco funciona correctamente: lógicamente, los grados de riesgo más bajos (como F o G) deberían mostrar mayores tasas de impago que los grados A o B.

### Pregunta #3: ¿Cómo varían el ingreso promedio y el monto solicitado según la situación de vivienda del cliente?

Agrupé a los clientes según su tipo de vivienda (person_home_ownership) empleando la cláusula GROUP BY. Luego, con la función AVG redondeada con ROUND, obtuve tanto el ingreso medio como el ticket promedio de préstamo para cada grupo.

```sql
-- Ingreso y préstamo promedio por situación de vivienda --

SELECT person_home_ownership, 
    COUNT(*) AS Total_Applicants,
    ROUND(AVG(person_income), 2) AS Avg_Income, 
    ROUND(AVG(loan_amnt), 2) AS Avg_Loan_Amount
FROM credit_risk_dataset
GROUP BY person_home_ownership
ORDER BY Avg_Income DESC;
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P3.png?raw=true)

_Ingreso y montos solicitados promedio por propiedad de vivienda_

Generalmente, los propietarios y quienes pagan hipoteca suelen tener un perfil financiero más sólido y solicitan montos más altos en comparación con los clientes que alquilan vivienda.

### Pregunta #4: ¿Cuál es la tasa de mora en clientes sobreendeudados con un historial crediticio corto?

Aislé un segmento de alto riesgo filtrando con la cláusula WHERE a los clientes cuyo préstamo supera el 40% de sus ingresos (loan_percent_income > 0.40) y tienen menos de 3 años de historial crediticio. Luego, calculé su tasa de impago general.

```sql
-- Tasa de impago en micro-segmento de alto riesgo (DTI > 40% y perfil nuevo) --

SELECT COUNT(*) AS High_Risk_Profile_Count,
    ROUND(AVG(CAST(loan_status AS FLOAT)) * 100, 2) AS Default_Rate_Pct
FROM credit_risk_data
WHERE loan_percent_income > 0.40 
    AND cb_person_cred_hist_length < 3;
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P4.png?raw=true)

_Métricas de riesgo para clientes sobreendeudados con historial reciente_

Esta consulta identifica un micro-segmento altamente tóxico. Si la morosidad aquí es alarmante, el comité de crédito debería endurecer o crear reglas de rechazo automático para este perfil específico.

### Pregunta #5: ¿Cómo afecta el nivel de la tasa de interés asignada al cumplimiento de pago?

Utilicé una Expresión de Tabla Común (CTE usando WITH) y una sentencia CASE WHEN para clasificar las tasas de interés en tres bandas dinámicas: Baja, Media y Alta. Posteriormente, agrupé los datos por estas nuevas categorías en la consulta principal para evaluar la morosidad.

```sql
-- Morosidad agrupada por bandas de tasas de interés --

WITH Interest_Bands AS (
    SELECT 
        CASE 
            WHEN loan_int_rate < 10 THEN '1. Baja (<10%)' 
            WHEN loan_int_rate BETWEEN 10 AND 15 THEN '2. Media (10%-15%)' 
            ELSE '3. Alta (>15%)' 
        END AS Interest_Band,
        loan_status
    FROM credit_risk_data
    WHERE loan_int_rate IS NOT NULL
)
SELECT Interest_Band, 
    COUNT(*) AS Total_Loans,
    ROUND(AVG(CAST(loan_status AS FLOAT)) * 100, 2) AS Default_Rate_Pct
FROM Interest_Bands
GROUP BY Interest_Band
ORDER BY Interest_Band;
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P5.png?raw=true)

_Tasa de incumplimiento clasificada por niveles de interés_

Esto demuestra el efecto de selección adversa: las tasas excesivamente altas, pensadas originalmente para mitigar el riesgo, a menudo terminan ahogando la capacidad de pago del cliente y provocando el impago.

### Pregunta #6: ¿Existe una relación directa entre los años de estabilidad laboral y el riesgo de impago?

Segmenté a los solicitantes en rangos de antigüedad laboral usando la función condicional CASE WHEN dentro de una CTE. Filtré los valores faltantes con IS NOT NULL y calculé la tasa de mora para cada nivel de experiencia (Junior, Mid, Senior, Estable).

```sql
-- Tasa de mora según rangos de antigüedad laboral --

WITH Employment_Bands AS (
    SELECT 
        CASE 
            WHEN person_emp_length <= 2 THEN '0-2 años (Junior)'
            WHEN person_emp_length BETWEEN 3 AND 5 THEN '3-5 años (Mid)'
            WHEN person_emp_length BETWEEN 6 AND 10 THEN '6-10 años (Senior)'
            ELSE '+10 años (Estable)' 
        END AS Emp_Length_Band,
        loan_status
    FROM credit_risk_data
    WHERE person_emp_length IS NOT NULL
)
SELECT Emp_Length_Band,
    COUNT(*) AS Total_Loans,
    ROUND(AVG(CAST(loan_status AS FLOAT)) * 100, 2) AS Default_Rate_Pct
FROM Employment_Bands
GROUP BY Emp_Length_Band
ORDER BY Default_Rate_Pct DESC;
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P6.png?raw=true)

_Riesgo de incumplimiento según niveles de experiencia en el empleo actual_

Se verifica la hipótesis bancaria clásica: a mayor estabilidad en el empleo actual, suele existir una tendencia hacia una menor probabilidad de caer en mora.

### Pregunta #7: ¿Cuál es la probabilidad de impago en jóvenes con bajos ingresos y manchas crediticias previas?

Crucé múltiples condiciones restrictivas empleando operadores AND en el WHERE para encontrar solicitantes menores de 25 años, con ingresos inferiores a 50,000 y un historial de default en buró (cb_person_default_on_file = 'Y'). Sobre este grupo cerrado, determiné la tasa actual de préstamos fallidos.

```sql
-- Tasa de mora en jóvenes de bajos ingresos con historial de default --

SELECT COUNT(*) AS Target_Count, 
    ROUND(AVG(CAST(loan_status AS FLOAT)) * 100, 2) AS Default_Rate_Pct
FROM credit_risk_data
WHERE person_age < 25 
    AND person_income < 50000 
    AND cb_person_default_on_file = 'Y';
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P7.png?raw=true)

_Análisis de reincidencia en perfiles jóvenes vulnerables_

Este análisis evalúa numéricamente el riesgo de dar "segundas oportunidades". Ayuda a definir si este nicho de mercado es eventualmente rentable o si genera exclusivamente pérdidas financieras.

### Pregunta #8: Identificar los 3 préstamos en mora con las peores tasas de interés dentro de cada propósito de préstamo.

Apliqué la función de ventana analítica ROW_NUMBER() OVER(PARTITION BY... ORDER BY...) dentro de una CTE para crear un ranking de tasas de interés de mayor a menor, particionado y reiniciado por cada propósito del préstamo. Finalmente, en la consulta principal filtré solo a los tres peores (Rank_Idx <= 3).

```sql
-- Top 3 préstamos morosos con mayores tasas, particionado por propósito --

WITH RankedLoans AS (
    SELECT loan_intent, 
        loan_amnt, 
        loan_int_rate,
        person_income,
        ROW_NUMBER() OVER(PARTITION BY loan_intent ORDER BY loan_int_rate DESC) AS Rank_Idx
    FROM credit_risk_data
    WHERE loan_status = 1 
        AND loan_int_rate IS NOT NULL
)
SELECT loan_intent, 
    loan_amnt, 
    loan_int_rate, 
    person_income
FROM RankedLoans
WHERE Rank_Idx <= 3;
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P8.png?raw=true)

_Ranking de los peores créditos segmentados por categoría_

Resulta ser una herramienta invaluable para las auditorías internas, ya que permite revisar a detalle los expedientes individuales que resultaron en impagos severos bajo condiciones iniciales altamente punitivas.

### Pregunta #9: ¿Existen solicitudes atípicas que superen el doble del monto promedio solicitado por clientes de su mismo segmento?

Usé la función de ventana AVG() OVER(PARTITION BY...) para calcular el promedio dinámico de los montos según la calificación de riesgo y el propósito del préstamo. En la consulta principal, en lugar de listar cada caso de forma individual, filtré y agrupé los resultados para contar cuántas anomalías (solicitudes que superan el doble de su promedio calculado) existen por cada segmento, obteniendo así un panorama gerencial claro.

```sql
-- Detección de anomalías en montos solicitados por segmento --

WITH AvgSegmentAmounts AS (
    SELECT loan_intent, 
        loan_grade, 
        loan_amnt, 
        AVG(loan_amnt) OVER(PARTITION BY loan_grade, loan_intent) AS Avg_Segment_Amnt
    FROM credit_risk_dataset
)
SELECT loan_intent, 
    loan_grade, 
    COUNT(*) AS Total_Anomalies, 
    ROUND(AVG(loan_amnt), 2) AS Avg_Anomalous_Amount
FROM AvgSegmentAmounts
WHERE loan_amnt > (2 * Avg_Segment_Amnt)
GROUP BY loan_intent, loan_grade
ORDER BY Total_Anomalies DESC;
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P9.png?raw=true)

_Conteo y monto promedio de préstamos atípicos (outliers) agrupados por segmento_

Al agrupar las solicitudes atípicas, identificamos rápidamente en qué propósitos de préstamo y niveles de riesgo se concentran los intentos de fraude o sobreendeudamiento extremo. Esta lógica actúa como una alerta temprana y gerencial clave, permitiendo al equipo de prevención auditar bloques de alto riesgo en lugar de perderse en un listado interminable de casos aislados.

### Pregunta #10: ¿Cómo es el comportamiento de pago si dividimos a la población equitativamente en cuatro grupos según sus ingresos?

Empleé la potente función de ventana NTILE(4) OVER(ORDER BY person_income) para distribuir a toda la base de clientes exactamente en cuatro cuartiles basados en sus ingresos declarados. Luego, agrupé por cada cuartil generado para extraer el rango salarial (mínimo y máximo) y su respectiva probabilidad de impago.

```sql
-- Análisis de rendimiento por cuartiles de ingresos equitativos --

WITH IncomeQuartiles AS (
    SELECT person_income, 
        loan_status, 
        loan_amnt, 
        NTILE(4) OVER(ORDER BY person_income) AS Income_Quartile
    FROM credit_risk_data
)
SELECT Income_Quartile, 
    MIN(person_income) AS Min_Income_In_Q,
    MAX(person_income) AS Max_Income_In_Q,
    ROUND(AVG(CAST(loan_status AS FLOAT)) * 100, 2) AS Default_Rate_Pct, 
    ROUND(AVG(loan_amnt), 2) AS Avg_Loan
FROM IncomeQuartiles
GROUP BY Income_Quartile
ORDER BY Income_Quartile;
```

![image](https://github.com/alexander9484/-SQL-SERVER-PROJECT-CREDIT-RISK/blob/main/Picture/P10.png?raw=true)

_Tasa de morosidad y ticket promedio agrupados por cuartiles de ingresos_

Ofrece una visión macro y estratégica vital para fijar políticas de precios basados en riesgo (Risk-Based Pricing). Permite visualizar sin sesgos qué estrato poblacional está inyectando capital al banco y cuál sector representa las mayores pérdidas.

### Conclusión
Este análisis estructurado proporcionó respuestas contundentes sobre el portafolio de créditos. Logramos validar que variables como la relación Deuda-Ingreso (DTI), las altas tasas de interés y los historiales de mora previos son fuertes detractores del cumplimiento de pago.

Además de responder a preguntas de negocio inmediatas para ajustar políticas crediticias, las variables analizadas (y sus segmentaciones en cuartiles/bandas) son fuertes candidatas predictoras (Features). Esta extracción de características deja el terreno perfectamente preparado para la ingesta de estos datos en pipelines de preprocesamiento, de cara al entrenamiento de modelos de Machine Learning para la predicción probabilística de impagos.

