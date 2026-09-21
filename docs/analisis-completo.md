# Optimización de tarifa eléctrica con Power BI y DAX

> Análisis completo del caso: contexto, pregunta de investigación, recorrido por el dashboard pestaña a pestaña y decisión final con su desglose de impacto.
---
## Índice de contenidos

- [Parte 1: El caso de Antonio](#parte-1-el-caso-de-antonio)
  - [1.1. Contexto](#11-contexto)
  - [1.2. Pregunta de análisis](#12-pregunta-de-análisis)
  - [1.3. Estructura del dashboard](#13-estructura-del-dashboard)
    - [1.3.1. Consumo](#131-consumo)
    - [1.3.2. Potencia](#132-potencia)
    - [1.3.3. Comparativa](#133-comparativa)
    - [1.3.4. Ahorro](#134-ahorro)
  - [1.4. Decisión final e impacto](#14-decisión-final-e-impacto)
    - [1.4.1. De dónde sale cada euro del ahorro](#141-de-dónde-sale-cada-euro-del-ahorro)
    - [1.4.2. Impacto](#142-impacto)

---
## Parte 1: El caso de Antonio
### 1.1. Contexto

Antonio tiene 66 años y está jubilado. Durante muchos años desarrolló una actividad profesional que requería el uso continuado de maquinaria en una instalación que compartía suministro eléctrico con su vivienda, lo que justificaba un elevado consumo energético y una potencia contratada de **16,44 kW**.

Tras su jubilación, Antonio cesa esa actividad y vende la maquinaria. Sin embargo, aunque sus necesidades energéticas cambian radicalmente, la instalación eléctrica permanece igual, porque vivienda y negocio nunca tuvieron contadores ni contratos separados.

Mes tras mes, Antonio continúa recibiendo facturas que ya no parecen corresponderse con su nueva realidad. Ante esta situación, decide buscar asesoramiento.

> _La identidad de Antonio y cualquier dato que pueda permitir su identificación no se ha expuesto en este análisis, de acuerdo con la normativa aplicable en materia de protección de datos._

---

### 1.2. Pregunta de análisis

Todo el proyecto gira en torno a una única pregunta, la que Antonio planteó al contactar:

> **¿Estoy pagando de más?**

De esta pregunta raíz se derivan las cuestiones técnicas que estructuran el análisis:

1. ¿Cuánta energía demandaba Antonio cuando utilizaba la maquinaria y cuánto demanda ahora? ¿En qué franjas concretas del día se concentra su demanda?
2. ¿Cuál es la potencia máxima que realmente necesita su instalación? ¿Cómo se distribuye esa potencia máxima entre los periodos tarifarios P1-P6?
3. ¿Qué margen existe respecto a los 16,44 kW contratados?
4. ¿Qué tarifa y qué potencia contratada se ajustan mejor a sus necesidades actuales?
5. ¿Cuánto dinero puede suponer realmente este cambio? ¿Cómo evoluciona ese ahorro mes a mes, y no solo en términos anuales?

---

### 1.3. Estructura del dashboard

El dashboard de Power BI se organiza en cuatro pestañas, cada una respondiendo a una parte del análisis.

#### 1.3.1. Consumo

Evolución del consumo entre 2024 y 2025, a nivel mensual y horario.

- El consumo de 2025 fue de 3.935 kWh, un -8,72 % respecto a 2024. Desde abril de 2024, coincidiendo con el cese de actividad, cae por debajo de la mediana y se estabiliza (frente a los 861 kWh de enero).
- **El pico de agosto (638 kWh) es un evento puntual de mayor ocupación y climatización**, no un cambio estructural. El exceso se concentra del 9 al 18 de agosto (23-35 kWh/día frente a una base de 11-22 kWh/día). El crecimiento respecto a agosto de 2024 es mayor en Noche (+271 %), Madrugada (+188 %) y Tarde (+146 %) que en Mañana (+55 %), la franja de la actividad profesional; en términos absolutos, la Tarde aporta el mayor incremento (+151,5 kWh).
- **El desglose por franja horaria y tipo de jornada confirma el cese de actividad.** Laborable-Mañana cae un -33,32 % y Laborable-Noche un -24,69 %; No laborable-Mañana también cae (-23,74 %), evidenciando que el patrón no distinguía entre laborables y festivos. Filtrando a Laborable-Mañana, los meses previos a abril de 2024 quedan claramente por encima de la mediana; todo 2025 se mantiene próximo a ella.
- **El consumo de madrugada sube en 2025** (+79,98 % en laborables, +70,26 % en no laborables), concentrado entre junio y octubre. Al afectar por igual a laborables y no laborables, apunta a un cambio de hábito doméstico —probablemente climatización nocturna— no vinculado al horario de trabajo.
- **La energía reactiva confirma el cese de forma independiente al testimonio de Antonio.** La reactiva inductiva, generada por motores, cae un -56,29 %. La caída no es uniforme: en las cuatro horas de trabajo de la maquinaria (7:00, 8:00, 19:00 y 20:00) desciende entre un 84,6 % y un 90,6 % (de 1.098,5 a 138,5 kVARh), frente al -22,6 % del resto del día.


![Pestaña Consumo](img/dashboard-pagina1-consumo.png)

#### 1.3.2. Potencia

Compara la potencia contratada frente a la demanda real de la instalación.

- Potencia contratada: **16,44 kW**
- Potencia máxima real registrada: **6 kW**
- Utilización de la potencia contratada: **36,50 %**
- Margen disponible: **10,44 kW**
- Potencia a contratar: 10 kW

Estos datos evidencian un sobredimensionamiento claro del contrato respecto a las necesidades actuales.

**La reducción de potencia y el cambio de tarifa no son dos decisiones independientes: la normativa las acopla.** La CNMC (Circular 3/2020) reserva la tarifa de acceso 3.0TD a los suministros de baja tensión con potencia contratada superior a 15 kW en algún periodo, y la 2.0TD a los de 15 kW o menos. Como la propuesta reduce la potencia de Antonio de 16,44 kW a 10 kW, el cambio de peaje de 3.0TD a 2.0TD **no es una opción**, es una consecuencia obligatoria de bajar de 15 kW. Por eso la comparativa de este proyecto combina ambas variables en un único escenario en lugar de tratarlas por separado.

![Pestaña Potencia](img/dashboard-pagina2-potencia.png)

#### 1.3.3. Comparativa

Compara tres escenarios de tarifa y potencia usando el mismo consumo real de 2025. Las cifras representan **cuánto habría pagado Antonio en concepto de energía y potencia durante 2025 si hubiera tenido contratada cada una de estas tarifas**, no una proyección de lo que pagará en el futuro.

| Escenario (etiqueta en el dashboard) | Nomenclatura oficial                                                                        | Potencia | Coste energético estimado (2025) |
| ------------------------------------ | ------------------------------------------------------------------------------------------- | -------- | -------------------------------- |
| Actual                               | 3.0TD (6 periodos, P1-P6) — incluye un descuento del 15 % sobre los kWh consumidos cada mes | 16,44 kW | 1.549,64 €                       |
| 24 horas                             | 2.0TD, modalidad SDH (precio único, sin discriminación horaria)                             | 10 kW    | 963,57 €                         |
| Discriminación horaria               | 2.0TD, modalidad DH (3 periodos: punta, llano, valle)                                       | 10 kW    | 961,71 €                         |

Los precios son fijos durante los 12 meses simulados: la comparativa responde a "qué habría pagado con esta oferta", no a una proyección con precios variables.

Estas cifras cubren únicamente los conceptos de **potencia y energía consumida**, que son los únicos sobre los que Antonio tiene capacidad real de decisión. Quedan excluidos impuestos, peajes regulados y otros cargos ajenos a la tarifa contratada; el detalle y el motivo de cada exclusión se explica en la sección [Conceptos excluidos del análisis](arquitectura-tecnica.md#23-conceptos-excluidos-del-análisis) de la arquitectura técnica.

![Pestaña Comparativa](img/dashboard-pagina3-comparativa.png)

#### 1.3.4. Ahorro

Cuantifica el impacto económico de cada propuesta frente al coste actual, siempre sobre la base de lo que Antonio habría pagado en 2025 en concepto de energía y potencia.

- Ahorro estimado con tarifa 2.0TD SDH (24h): **586,07 €/año** (-37,8 %)
- Ahorro estimado con tarifa 2.0TD DH (3 periodos): **587,93 €/año** (-37,9 %)
- Diferencia entre ambas propuestas: **1,86 €/año**

>**La diferencia de 1,86 €/año entre SDH y DH no es distinguible del error de medición.** Dado que el consumo horario tiene una resolución de 1 kWh (ver limitación de precisión del dato), el error de redondeo esperado sobre la diferencia de coste entre ambas tarifas es de aproximadamente ±1,23 €/año. Bastaría con que 17 kWh de los 3.935 kWh anuales estuvieran mal clasificados entre periodos para invertir el resultado. En la práctica, **SDH y DH son económicamente equivalentes** para el patrón de consumo de Antonio, y la elección entre ambas se resuelve por preferencia del cliente (ver más abajo), no por la diferencia de 1,86 €.


Desglose de ahorro por potencia contratada y energía consumida (visible en gráfico de cascada):

| Componente del ahorro | Tarifa Actual  | Tarifa 24h (SDH) | Ahorro       | % del ahorro total |
| --------------------- | -------------- | ---------------- | ------------ | ------------------ |
| Energía consumida     | 788,23 €       | 621,69 €         | 166,54 €     | 28,4 %             |
| Potencia contratada   | 761,40 €       | 341,88 €         | 419,52 €     | 71,6 %             |
| **Total**             | **1.549,64 €** | **963,57 €**     | **586,07 €** | **100 %**          |


Dentro del ahorro de potencia (419,52 €), **298,26 €** provienen de reducir la potencia contratada de 16,44 a 10 kW manteniendo la estructura de precios de la 3.0TD, y **121,26 €** de cambiar a la estructura de precio de la 2.0TD. El 71,6 % del ahorro total viene de la potencia, no de la tarifa de energía: el cambio de contrato es, sobre todo, una corrección de sobredimensionamiento, no una mejora de tarifa.


![Pestaña Ahorro](img/dashboard-pagina4-ahorro.png)

---

### 1.4. Decisión final e impacto

Aunque la tarifa 2.0TD en modalidad DH (3 periodos) resulta marginalmente más económica (1,86 € al año), la propuesta final se decanta por la **tarifa 2.0TD en modalidad SDH (precio fijo 24 horas)**, junto con una reducción de la potencia contratada de **16,44 kW a 10 kW**.

Esta decisión responde a las preferencias explícitas de Antonio, que prioriza:

- **Libertad de consumo** sin depender de franjas horarias.
- Un **margen de seguridad de potencia** (10 kW frente a los 6 kW de demanda máxima real) para evitar cortes ante situaciones puntuales.

Se presenta igualmente la tabla con todas las opciones de contratación de potencia para evaluar los múltiples escenarios:

| Potencia contratada | Coste anual del término de potencia | Sobrecoste vs. 7 kW (mínimo con margen de 1 kW) |
|---|---|---|
| 6 kW (= máximo real registrado) | 205,13 € | -34,19 € |
| 7 kW | 239,32 € | 0,00 € (referencia) |
| 8 kW | 273,50 € | +34,19 € |
| 9 kW | 307,69 € | +68,38 € |
| **10 kW (propuesta)** | **341,88 €** | **+102,56 €** |

Antonio paga **102,56 €/año adicionales** por contratar 10 kW en lugar de los 7 kW que ya suponen un margen de seguridad del 17 % sobre su demanda máxima real (6 kW). Esta cifra es 55 veces mayor que la diferencia de 1,86 €/año entre las dos tarifas 2.0TD, y es la decisión económica más relevante que Antonio toma en este proyecto después de reducir la potencia desde 16,44 kW. Se mantiene igualmente porque, a diferencia de la elección de tarifa, aquí sí hay una consecuencia real de equivocarse a la baja: un corte de suministro por exceso de potencia. Antonio prioriza ese margen de seguridad sobre el ahorro adicional.

**El desglose del consumo por periodo tarifario (P1/P2/P3) explica con precisión por qué la propuesta de discriminación horaria (DH) resulta solo marginalmente más económica que la de precio único (SDH).** La caída de consumo de Antonio tras el cese de su actividad se concentra casi por completo en los periodos punta y llano: P1 cae un -19,02 % y P2 un -9,19 % respecto a 2024, mientras que P3 se mantiene prácticamente estable (+0,57 %). Como la ventaja económica de la tarifa DH depende de concentrar consumo en P3 (el periodo más barato), y es precisamente el único periodo que no ha bajado, la diferencia entre ambas tarifas se reduce a apenas 1,86 €/año (587,93 € de ahorro con DH frente a 586,07 € con SDH). Esto confirma el criterio de decisión final: elegir SDH por la preferencia de Antonio por la libertad de consumo.

#### 1.4.1. De dónde sale cada euro del ahorro

| Palanca                                                      | Euros/año    | % del ahorro | ¿Es una decisión de tarifa?         |
| ------------------------------------------------------------ | ------------ | ------------ | ----------------------------------- |
| Reducir la potencia contratada de 16,44 kW a 10 kW           | 298,26 €     | 50,9 %       | No: es dimensionamiento             |
| Cambio de peaje de acceso 3.0TD → 2.0TD                      | 121,26 €     | 20,7 %       | No: es automático al bajar de 15 kW |
| Precio de energía de la nueva oferta (0,2003 → 0,1580 €/kWh) | 166,54 €     | 28,4 %       | Sí: elección de oferta comercial    |
| **Total (escenario elegido: 2.0TD SDH)**                     | **586,07 €** | **100 %**    |                                     |
| _Palanca alternativa: modalidad DH en lugar de SDH_          | _+1,86 €_    | _+0,3 %_     | _Sí: elección de modalidad horaria_ |

La última fila no forma parte del total porque es excluyente con él: o se contrata SDH (586,07 €) o se contrata DH (587,93 €). Se muestra aparte para dimensionar la decisión que más tiempo consume en una comparativa comercial y que menos dinero mueve.

La lectura que importa para Antonio: **el 71,6 % del ahorro está en el término de potencia**, no en el de energía. La decisión que más dinero mueve no es "qué tarifa elijo" sino "cuántos kW contrato", y esa decisión estaba disponible desde el día en que vendió la maquinaria. La elección entre SDH y DH, que es la que más tiempo consume en una comparativa comercial, vale 1,86 €/año: menos del 0,4 % del resultado.

>**Matiz normativo:** la 2.0TD aplica a suministros de hasta 15 kW y la 3.0TD por encima (Circular 3/2020 de la CNMC). Al bajar a 10 kW el cambio de peaje no es una opción a valorar, es una consecuencia. Las dos primeras palancas de la tabla son por tanto una sola decisión con dos efectos separables.
#### 1.4.2. Impacto

El resultado de la simulación muestra una diferencia significativa: la propuesta supone una **reducción del 37,8 %** en los costes de energía y potencia analizados sobre el histórico de 2025, equivalente a un **ahorro aproximado de 586,07 € al año**.

No se trata únicamente de reducir una factura. Se trata de adaptar el contrato energético a las **necesidades reales** de Antonio, evitando pagar por una potencia que ya no utiliza y escogiendo una tarifa coherente con sus hábitos y preferencias actuales.

Tras presentar los resultados, Antonio acepta la propuesta. Se recomienda mantener un **seguimiento y monitorización** de sus hábitos energéticos para detectar posibles cambios futuros y continuar buscando oportunidades de mejora.

Este caso, basado en una historia real, muestra cómo los datos disponibles pueden utilizarse para analizar una situación y tomar decisiones adaptadas a las necesidades concretas de una persona.

---
