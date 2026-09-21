# Optimización de tarifa eléctrica con Power BI y DAX

> Documentación técnica del modelo: arquitectura de datos, columnas calculadas, medidas DAX, origen de los datos y limitaciones conocidas.

---
## Índice de contenidos

- [Parte 2: Cómo se ha construido el proyecto](#parte-2-cómo-se-ha-construido-el-proyecto)
  - [2.1. Arquitectura](#21-arquitectura)
    - [2.1.1. Columnas calculadas](#211-columnas-calculadas)
  - [2.2. Origen de los datos](#22-origen-de-los-datos)
  - [2.3. Conceptos excluidos del análisis](#23-conceptos-excluidos-del-análisis)
  - [2.4. Por qué mediana y no promedio](#24-por-qué-mediana-y-no-promedio)
  - [2.5. Diccionario de medidas DAX](#25-diccionario-de-medidas-dax)
  - [2.6. Limitaciones y líneas futuras](#26-limitaciones-y-líneas-futuras)
    - [2.6.1. Limitaciones](#261-limitaciones)
    - [2.6.2. Líneas futuras](#262-líneas-futuras)
---

## Parte 2: Cómo se ha construido el proyecto

### 2.1. Arquitectura

El dashboard de Power BI usa **modo de importación (Import)** sobre un archivo Excel que actúa como fuente de datos única, sin base de datos intermedia. El modelo de datos relaciona las siguientes tablas mediante un calendario común (`Calendario`), que actúa como tabla puente entre `T_consumo`, `T_potencia`, `T_facturas` y `T_comparativa`:

- `_Medidas`: tabla auxiliar sin datos propios, usada como buena práctica para centralizar todas las medidas DAX del modelo en un único lugar visible, en lugar de dispersarlas entre las tablas de hechos.
- `Calendario`: tabla de fechas que habilita funciones de time intelligence (comparativas año/mes, ordenación cronológica) y actúa como puente entre las tablas de hechos del modelo. El campo `id_periodo_mes` es una clave secuencial global del `Calendario` que numera todos los meses del histórico de forma continua (2024 = 1-12, 2025 = 13-24), lo que explica por qué `T_Facturas` referencia valores 13-24 pese a cubrir únicamente los 12 meses de 2025.
- `T_consumo`: consumo horario en kWh. Incluye dos columnas calculadas añadidas para el análisis de patrones horarios: `Momento del Día` (Madrugada/Mañana/Tarde/Noche, según franjas horarias de 6 horas) y `Tipo de Día` (Laborable/No laborable, derivada de `Día Semana Num` en `Calendario` donde 1=lunes y 7=domingo combinado con una lista de festivos nacionales de 2024 y 2025 codificada directamente en la medida, sin tabla auxiliar de calendario laboral)
- `T_potencia`: potencia máxima registrada por periodo y mes.
- `T_facturas`: importe real facturado a Antonio, mes a mes.
- `T_comparativa`: precios de energía y potencia por tarifa y periodo, usada para simular los distintos escenarios.
- `T_tarifas`: catálogo de tarifas comparadas (**Actual**, **24 horas**, **Discriminación horaria**), con las etiquetas exactas usadas en el modelo.
- `T_periodo_tarifario`: asocia cada `id_periodo_tarifario` con su etiqueta P1-P6, usada para desglosar consumo y potencia por periodo tarifario en los gráficos correspondientes. **Esta tabla actúa como dimensión compartida entre dos normativas tarifarias distintas y no debe interpretarse como una jerarquía única**: `T_consumo` solo utiliza los valores P1-P3 (estructura de la tarifa 2.0TD), mientras que `T_potencia` y `T_comparativa` (en el escenario "Actual") utilizan P1-P6 (estructura de la tarifa 3.0TD).
- `Palancas`: tabla auxiliar desconectada, sin relación con el resto del modelo. No almacena datos del negocio: existe únicamente para ordenar y etiquetar las categorías del gráfico de cascada de la pestaña Ahorro (ver medida `Valor Palanca`). Es el mismo patrón que `_Medidas`, pero orientado a un visual concreto en lugar de a organizar código DAX.

Las medidas DAX calculan de forma dinámica los costes, márgenes y comparativas según la tarifa seleccionada en el propio dashboard.

#### 2.1.1. Columnas calculadas

El modelo incluye cinco columnas calculadas, repartidas en tres tablas, que preparan el terreno para los desgloses y comparativas de los visuales sin necesidad de duplicar medidas por segmento (`Calendario`, `AñoMes`, `Momento del Día`, `Tipo de Día`, `Leyenda Potencia` y `Leyenda Consumo`).

El código completo y la explicación de cada una de ellas se encuentra detallado en [`DAX/dax.md`](../DAX/dax.md#columnas-calculadas).

### 2.2. Origen de los datos

**`T_consumo`**: datos de consumo horario extraídos directamente de la distribuidora (e-distribución) en formato CSV. Se mantienen los campos `Fecha`, `Hora`, `AE_kWh` (energía activa consumida) y `R1_kVARh` (energía reactiva inductiva consumida). Se excluyen del modelo: `CUPS`, `AS_kWh`, `R2_kVARh`, `R3_kVARh`, `R4_kVARh`, `AE_AUTOCONS_kWh` y `REAL/ESTIMADO`, por no ser relevantes para este análisis o por motivos de privacidad (`CUPS`). La columna `id_periodo_tarifario` no viene en el CSV original de e-distribución. Se generó mediante tratamiento con IA sobre las 17.542 filas horarias, aplicando el calendario oficial de festivos nacionales de 2024 y 2025 y la distinción laborable/fin de semana, para asignar cada fila a su periodo P1-P3 según la normativa de la tarifa 2.0TD. El criterio de festivos aplicado toma los festivos nacionales de fecha fija (1 de enero, 6 de enero, 1 de mayo, 15 de agosto, 12 de octubre, 1 de noviembre, 6 de diciembre, 8 de diciembre y 25 de diciembre) más el Viernes Santo (festivo nacional de fecha móvil, festivo en la mayor parte del territorio salvo excepciones autonómicas). No se traslada al lunes siguiente ningún festivo que caiga en fin de semana, ni se aplican traslados o festivos autonómicos/locales adicionales.

> **Precisión del dato de consumo horario.** El export de e-distribución entrega el consumo horario mayoritariamente con resolución de 1 kWh: 16.846 de las 17.542 filas (96,0 %) son valores enteros. Las 696 restantes sí incluyen decimales (rango 0,094 a 5,929 kWh, 439,97 kWh en total), concentradas en 2025 (498 filas). Para el 96 % del dato, esto implica un error de redondeo de hasta ±0,5 kWh por hora registrada que se acumula en agregados finos como el desglose por periodo tarifario. Los totales mensuales y anuales, al sumar miles de horas, diluyen este error hasta hacerlo irrelevante.

**`T_potencia`**: datos de potencia máxima extraídos de la distribuidora (e-distribución). Para cada uno de los 19 meses del histórico (junio 2024-diciembre 2025), la tabla recoge los 3 mayores registros del maxímetro de ese mes, con marca de fecha/hora en tramos de 15 minutos y el periodo tarifario (P1-P6) en el que se produjo cada pico. No se trata, por tanto, de un máximo garantizado para cada uno de los seis periodos: algunos periodos (p. ej. P5) aparecen en muy pocos meses de la muestra, por lo que el desglose "Potencia Máxima Real por Periodo" del dashboard refleja los picos capturados, no necesariamente el máximo histórico real de cada periodo. Se excluye el campo `CUPS` por motivos de privacidad.

>**Precisión del dato de potencia máxima.** El export de e-distribución entrega la potencia máxima registrada por el maxímetro con resolución de 1 kW (sin decimales). El valor de 6 kW como máximo real registrado puede representar cualquier valor entre 5,5 y 6,5 kW; el margen del 36,50 % de utilización y el margen disponible de 10,44 kW deben leerse con ese margen de error.

**`T_facturas`**: importe total facturado a Antonio mes a mes, extraído de las facturas reales de 2025 de la comercializadora (nombre omitido por confidencialidad) mediante tratamiento con IA. Para preservar la privacidad, las facturas se recortaron previamente al procesamiento de forma que solo se incluyeran las secciones "detalle de factura" y "lecturas", excluyendo así cualquier dato identificativo o sensible antes de que la IA accediera al documento. Esta tabla permite comparar la factura real que paga Antonio frente a los costes de energía y potencia estimados en cada escenario simulado. 

La cardinalidad de esta relación es 1:* con **`T_Facturas` como tabla "una"** (cada `id_periodo_mes` aparece una sola vez, una fila por mes) y **`Calendario` como tabla "muchos"** (el mismo `id_periodo_mes` se repite en cada día del mes). Esto invierte el patrón habitual del resto del modelo, donde `Calendario` actúa como dimensión en el lado "uno" frente a las tablas de hechos diarias (`T_consumo`, `T_potencia`). La causa es que `T_Facturas` tiene grano mensual mientras que `Calendario` tiene grano diario: `T_Facturas` funciona aquí como una tabla de hechos "resumen" a nivel de mes, no como una dimensión. Esta inversión de cardinalidad es la razón real por la que el filtrado cruzado bidireccional es necesario (ver más abajo), no una elección arbitraria de configuración además se mantiene el filtrado cruzado bidireccional porque el visual "Resumen de costes energéticos para cada tarifa" (pestaña Comparativa) necesita propagar el contexto de fecha entre `T_comparativa` y `T_facturas` a través de `Calendario` en ambos sentidos; con dirección única, la serie de facturas reales se aplana al no poder alcanzar el contexto de mes correcto.

`T_comparativa`: **Precio de potencia de las tarifas 2.0TD.** Aunque la tarifa de acceso 2.0TD define dos periodos de potencia (punta: 8h-24h en laborables; valle: resto de horas, fines de semana y festivos), la oferta comercial comparada en este análisis factura la potencia contratada a un precio único independiente del periodo. Esto es una característica de la oferta comercial simulada, no una simplificación del modelo.
### 2.3. Conceptos excluidos del análisis

Los conceptos analizados (**potencia contratada** y **energía consumida**) representan aproximadamente el **72,6 % de la factura total real de 2025** (1.549,64 € de un total facturado de 2.135,73 €). El 27,4 % restante corresponde a impuestos, alquiler del contador y financiación del bono social, conceptos sobre los que Antonio no tiene capacidad de decisión al elegir tarifa. El peaje de transporte y distribución y los cargos regulados **sí están incluidos** en el 72,6 % analizado.

| Concepto                                 | Motivo de exclusión                                                                                                                     |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| IVA                                      | Ha variado entre el 21 % y el 5 % según medidas gubernamentales de crisis en distintos periodos, un factor ajeno a la tarifa contratada |
| Impuesto eléctrico                       | Sujeto a variaciones similares a las del IVA por medidas regulatorias temporales                                                        |
| Alquiler del equipo de medida (contador) | Varía según la naturaleza y titularidad de la instalación, no según la tarifa contratada                                                |
| Financiación del bono social             | Importe residual, inferior a 1 € mensual en la factura, sin impacto significativo en la comparativa entre escenarios                    |

Como consecuencia, el ahorro real que Antonio obtenga en su factura completa puede variar al alza o a la baja respecto a los 586,07 € estimados. Sin embargo, el criterio de decisión (qué tarifa y qué potencia elegir) permanece inalterado, porque se basa en los únicos conceptos sobre los que existe capacidad real de decisión.

### 2.4. Por qué mediana y no promedio

Los gráficos de consumo por mes y por hora usan la **mediana** como línea de referencia, en lugar del promedio. Esta elección no es arbitraria: el consumo de enero de 2024 (861 kWh) y el pico de agosto de 2025 (638 kWh, evento puntual analizado en 1.3.1.) son valores atípicos muy alejados del resto de meses. Si se usara el promedio, ambos outliers desplazarían la referencia hacia arriba y diluirían el contraste real entre el consumo antes y después del cese de actividad de Antonio, que es el argumento central del proyecto. La mediana, al no verse afectada por la magnitud de esos picos, refleja mejor cuál es el mes "típico" de Antonio.

### 2.5. Diccionario de medidas DAX

La consulta en detalle del código y la lógica de las medidas DAX se realiza en [`DAX/dax.md`](../DAX/dax.md).

#### Consumo

| Medida | Descripción |
| :----- | :---------- |
| `Consumo 2024` | Total de kWh consumidos en 2024 |
| `Consumo 2025` | Total de kWh consumidos en 2025 |
| `% Var Consumo 25 vs 24` | Variación interanual del consumo; soporta clic mes a mes mediante `SELECTEDVALUE` |
| `Reactiva kVARh` | Total de kVARh de energía reactiva inductiva consumida |
| `% Var Reactiva 25 vs 24` | Variación interanual de la energía reactiva (solo nivel de año completo) |

#### Potencia

| Medida | Descripción |
| :----- | :---------- |
| `Potencia Máxima Real` | Máximo kW demandado registrado en el maxímetro |
| `Potencia Contratada` | kW contratados en el escenario actual (tarifa 1 — 16,44 kW) |
| `% Utilización Potencia` | Ratio entre la potencia máxima real y la contratada |
| `Margen Potencia Actual` | Diferencia entre la potencia contratada y la demanda máxima real |
| `Nueva Potencia Contratada` | kW propuestos en los escenarios 2.0TD (10 kW) |
| `Margen Nueva Potencia` | Diferencia entre la nueva potencia propuesta y la demanda máxima real |

#### Comparativa

| Medida | Descripción |
| :----- | :---------- |
| `Coste Consumo` | Coste de los kWh consumidos; aplica el 15 % de descuento para la tarifa actual |
| `Coste Potencia` | Coste del término de potencia según tarifa y días del periodo |
| `Costes potencia y consumo` | Suma de `Coste Potencia` y `Coste Consumo`; base de las medidas de ahorro |

#### Ahorro

| Medida | Descripción |
| :----- | :---------- |
| `Ahorro €` | Diferencia en euros entre el coste de la tarifa actual y la seleccionada |
| `Ahorro %` | Diferencia porcentual entre el coste de la tarifa actual y la seleccionada |
| `Facturas reales` | Total facturado realmente a Antonio (fuente: `T_Facturas`) |
| `Coste Potencia Actual a Nueva Potencia` | Recalcula el término de potencia actual aplicando los 10 kW propuestos |
| `Ahorro € por reducción de potencia` | Ahorro atribuible exclusivamente a reducir la potencia contratada |
| `Ahorro € por precio de energía` | Ahorro atribuible al precio del kWh entre la tarifa actual y la 2.0TD SDH |
| `Ahorro € por cambio de peaje` | Ahorro atribuible al cambio de peaje de 3.0TD a 2.0TD |
| `Valor Palanca` | Valor de cada palanca para el gráfico de cascada (`SWITCH` sobre tabla `Palancas`) |

---

### 2.6. Limitaciones y líneas futuras

Este análisis se ha construido con el máximo rigor posible dentro de los datos disponibles, pero es importante ser transparente sobre su alcance y sus puntos de mejora futuros.

#### 2.6.1. Limitaciones

- **Un único año de datos horarios completos.** El análisis horario y de potencia se basa en los datos de 2025. La comparación 2024 vs. 2025 permite contrastar el consumo antes y después del cese de actividad de Antonio, pero no captura variabilidad interanual más allá de esos dos años (por ejemplo, el efecto de inviernos especialmente fríos o veranos especialmente calurosos en años sucesivos).
- **Los días de cambio a horario de verano tienen 23 registros horarios en lugar de 24.** El 31/03/2024 y el 30/03/2025 tienen 23 filas. Los días de cambio a horario de invierno (27/10/2024 y 26/10/2025) **no** incorporan la hora duplicada: el export de e-distribución entrega 24 filas en esos días, con `HoraNum` de 0 a 23. Como consecuencia el histórico completo suma 17.542 filas y no las 17.544 que corresponderían a dos años naturales. El impacto en los totales es de dos horas de madrugada (periodo P3/valle, consumo típico inferior a 1 kWh cada una), pero se documenta para que cualquier análisis futuro que asuma 24 filas por día tenga en cuenta la excepción.
- **El pico de agosto de 2025 (638 kWh) se interpreta como un evento puntual, no como parte del patrón estructural.** Antonio lo atribuyó a una reforma en la vivienda; el perfil horario (crecimiento máximo en Madrugada y Noche, y días altos en sábado y domingo) apunta más bien a mayor ocupación con climatización. No ha sido posible verificar ninguna de las dos causas de forma independiente. Para el dimensionamiento la distinción es irrelevante: en ambos casos es puntual.
- **La relación entre `T_Facturas` y `Calendario` es 1:*, con filtrado cruzado bidireccional.** La bidireccionalidad no es un ajuste manual evitable, sino un requisito funcional: el visual "Resumen de costes energéticos para cada tarifa" necesita propagar el contexto de fecha entre `T_comparativa` y `T_Facturas` a través de `Calendario`, y con dirección única la serie de facturas reales pierde esa propagación y se aplana. El detalle se documenta en la sección de arquitectura.
- **`T_comparativa` solo contiene una fila de referencia por el día 1 de cada mes, pero se relaciona con `Calendario[Date]` a grano diario.** Cualquier filtro de fecha que no incluya explícitamente el día 1 de un mes (por ejemplo, un rango del 10 al 20 de marzo aplicado desde otro visual con filtrado cruzado) deja ese mes sin filas en `T_comparativa` y los visuales de coste/ahorro de ese mes aparecen vacíos. Los visuales actuales del dashboard siempre filtran por mes completo, por lo que el problema no se manifiesta hoy, pero cualquier visual nuevo que filtre por rango de fechas dentro de un mes debe tenerlo en cuenta.
- **El desglose de potencia máxima por periodo tarifario (P1-P6) se basa en una muestra parcial.** `T_potencia` registra los 3 mayores picos del maxímetro por mes, no un máximo por cada periodo, por lo que periodos como P5 solo están representados en una fracción de los meses del histórico. El valor global de potencia máxima real (6 kW) sí es fiable; el desglose por periodo debe interpretarse como orientativo.
- **`T_periodo_tarifario` no es una dimensión conforme en sentido estricto.** Sirve simultáneamente a la estructura de 3 periodos de la 2.0TD y a la de 6 periodos de la 3.0TD, lo que puede generar selecciones sin sentido en los segmentadores que la usan de forma cruzada entre pestañas. Una mejora futura sería separarla en dos tablas de periodo independientes.
- **El ahorro estimado (586,07 €/año) cubre únicamente los conceptos de potencia y energía consumida.** Como se detalla en la sección "Conceptos excluidos del análisis", quedan fuera impuestos, peajes regulados, alquiler del contador y financiación del bono social. El ahorro real en la factura completa de Antonio puede variar al alza o a la baja respecto a esta cifra, aunque el criterio de decisión (qué tarifa y qué potencia elegir) no se ve afectado.
- **La comparación de "Facturas reales" frente a los escenarios simulados** permite validar si el modelo de costes estimados se aproxima a lo que Antonio paga en la práctica, pero esa validación depende de la calidad y completitud de los datos de factura extraídos, que fueron procesados mediante IA a partir de documentos recortados por privacidad.
- **El proyecto está diseñado como un entregable cerrado para el caso concreto de Antonio, no como una plantilla escalable.** Las medidas de coste (`Coste Consumo`, `Coste Potencia`) contemplan explícitamente las tres tarifas comparadas (id_tarifa 1, 2 y 3); incorporar una cuarta tarifa u otro cliente con distinta estructura de facturación requeriría revisar esa lógica condicional, no solo cargar nuevos datos.

#### 2.6.2. Líneas futuras

- **Seguimiento del consumo tras la implementación.** Se recomienda monitorizar el consumo y la potencia máxima real de Antonio durante los primeros meses tras el cambio de tarifa y potencia, para confirmar que los 10 kW contratados siguen siendo suficientes y que no se producen penalizaciones por exceso de potencia.
- **Incorporar datos de más de un año** conforme estén disponibles, para robustecer la comparación de patrones estacionales y detectar si el consumo de 2025 es representativo o si contiene más eventos puntuales similares al de agosto.
- **Extender la comparativa a más compañías comercializadoras**, ya que este análisis compara tarifas dentro de una misma oferta comercial (SDH vs. DH) pero no explora si otras comercializadoras ofrecen condiciones más ventajosas para el mismo peaje de acceso.

---
