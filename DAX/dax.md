# DAX

## Columnas calculadas

El modelo incluye cinco columnas calculadas, repartidas en tres tablas, que preparan el terreno para los desgloses y comparativas de los visuales sin necesidad de duplicar medidas por segmento.

**Calendario**
```dax
Calendario =
VAR FechaMin = DATE(2024, 1, 1)
VAR FechaMax = DATE(2025, 12, 31)
RETURN
ADDCOLUMNS(
    CALENDAR(FechaMin, FechaMax),
    "Año", YEAR([Date]),
    "Mes Num", MONTH([Date]),
    "Mes Nombre", FORMAT([Date], "MMMM"),
    "Día Semana Num", WEEKDAY([Date], 2),
    "id_periodo_mes", (YEAR([Date]) - 2024) * 12 + MONTH([Date])
)
```

**Columna `AñoMes`**
```DAX
-- Fecha truncada al mes y año ej: "ene 2024".
-- Permite comparar el mismo mes entre años usando EDATE() (ver medida "% Var Consumo 25 vs 24")
AñoMes = DATE(Calendario[Año], Calendario[Mes Num], 1)
```

**Momento del día**

```DAX
-- Agrupa cada hora del día en una de cuatro franjas: Madrugada (00-06, 6h),
-- Mañana (06-12, 6h), Tarde (12-19, 7h) y Noche (19-24, 5h). Las franjas
-- de Tarde y Noche no son simétricas: se ajustan a la puesta de sol y al
-- patrón de actividad habitual, no a bloques de 6 horas exactos.
-- Alimenta la tabla cruzada de consumo por franja horaria (sección Consumo)
-- y permite filtrar el gráfico "kWh consumidos por año" por segmento.
Momento del Día =
SWITCH(
    TRUE(),
    HOUR(T_consumo[HoraFormato]) >= 0 && HOUR(T_consumo[HoraFormato]) < 6, "Madrugada",
    HOUR(T_consumo[HoraFormato]) >= 6 && HOUR(T_consumo[HoraFormato]) < 12, "Mañana",
    HOUR(T_consumo[HoraFormato]) >= 12 && HOUR(T_consumo[HoraFormato]) < 19, "Tarde",
    "Noche"
)
```

**Tipo de día**
```DAX
Tipo de Día =
VAR FechaActual = T_consumo[Fecha]
VAR EsFinDeSemana = RELATED(Calendario[Día Semana Num]) IN {6, 7}
VAR EsFestivo =
    FechaActual IN {
        DATE(2024,1,1), DATE(2024,1,6), DATE(2024,3,29), DATE(2024,5,1),
        DATE(2024,8,15), DATE(2024,10,12), DATE(2024,11,1), DATE(2024,12,6),
        DATE(2024,12,9), DATE(2024,12,25),
        DATE(2025,1,1), DATE(2025,1,6), DATE(2025,4,18), DATE(2025,5,1),
        DATE(2025,8,15), DATE(2025,11,1), DATE(2025,12,6), DATE(2025,12,8),
        DATE(2025,12,25)
    }
RETURN
IF(EsFinDeSemana || EsFestivo, "No laborable", "Laborable")
```

**Leyenda Potencia**
```DAX
-- Resuelve una asimetría de la normativa tarifaria española: en la tarifa actual (3.0TD, id 1)
-- la potencia contratada varía por periodo (P1-P6), así que se muestra el periodo.
-- En las tarifas 2.0TD (24h y DH, id 2 y 3) la potencia es fija, así que se muestra "Precio Único".
-- La rama por defecto queda preparada para una futura cuarta tarifa no contemplada en T_tarifas.
Leyenda Potencia =
SWITCH(
    T_comparativa[id_tarifa],
    1, RELATED(T_periodo_tarifario[Periodo]),  // Actual: potencia variable -> muestra P1, P2...
    2, "Precio Único",                          // 24 horas: potencia fija
    3, "Precio Único",                          // Discriminación horaria: potencia fija
    RELATED(T_periodo_tarifario[Periodo])       // Por defecto, si hay más tarifas
)
```

**Leyenda consumo**
```DAX
-- Espejo de "Leyenda Potencia": aquí es el consumo el que varía por periodo (P1-P3)
-- solo en la tarifa de discriminación horaria (id 3); en las demás el precio de la
-- energía es único, aunque su potencia se trate de forma distinta (ver columna anterior).
Leyenda Consumo =
SWITCH(
    T_comparativa[id_tarifa],
    1, "Precio Único",                          // Actual: consumo a precio único
    2, "Precio Único",                          // 24 horas: consumo a precio único
    3, RELATED(T_periodo_tarifario[Periodo]),  // Discriminación horaria: consumo variable -> P1, P2, P3
    RELATED(T_periodo_tarifario[Periodo])       // Por defecto, si hay más tarifas
)
```

---

## Medidas

Las medidas del modelo se agrupan según la pestaña del dashboard en la que se utilizan.

## Medidas de consumo

```DAX
-- Total de kWh consumidos durante 2024
Consumo 2024 =
CALCULATE(
    SUM(T_consumo[Consumida Kwh]),
    Calendario[Año] = 2024
)
```

```DAX
-- Total de kWh consumidos durante 2025
Consumo 2025 =
CALCULATE(
    SUM(T_consumo[Consumida Kwh]),
    Calendario[Año] = 2025
)
```

```DAX
% Var Consumo 25 vs 24 =
// Captura el punto exacto pulsado en el gráfico (columna AñoMes = DATE(Año, Mes Num, 1)).
// Si no hay ningún clic (vista de año completo), esto devuelve BLANK porque hay
// 12 valores distintos de AñoMes en el contexto sin filtrar.
VAR FechaSeleccionada = SELECTEDVALUE(Calendario[AñoMes])

// Consumo de 2024:
// - Sin clic (FechaSeleccionada en blanco) -> usa la medida simple [Consumo 2024],
//   que da el año 2024 completo (comportamiento estable para el KPI).
// - Con clic -> retrocede 12 meses desde el mes pulsado con EDATE(-12) y filtra
//   Calendario a esa única fecha, dando el consumo de ese mismo mes pero en 2024.
VAR Consumo24 =
    IF(
        ISBLANK(FechaSeleccionada),
        [Consumo 2024],
        CALCULATE(
            SUM(T_consumo[Consumida Kwh]),
            FILTER(ALL(Calendario), Calendario[AñoMes] = EDATE(FechaSeleccionada, -12))
        )
    )

// Consumo de 2025: misma lógica que arriba, pero sin desplazar el mes
// (se queda en la fecha pulsada tal cual, ya que se asume que el clic
// se hace sobre un punto de 2025).
VAR Consumo25 =
    IF(
        ISBLANK(FechaSeleccionada),
        [Consumo 2025],
        CALCULATE(
            SUM(T_consumo[Consumida Kwh]),
            FILTER(ALL(Calendario), Calendario[AñoMes] = FechaSeleccionada)
        )
    )

// Variación porcentual con protección de división por cero (DIVIDE devuelve 0
// en vez de error si Consumo24 es 0 o BLANK).
RETURN
DIVIDE(Consumo25 - Consumo24, Consumo24, 0)
```

```DAX
-- Total de kVARh de energía reactiva inductiva consumida. A diferencia de la energía
-- activa, la reactiva inductiva la generan específicamente los motores eléctricos, por lo
-- que sirve como confirmación física del cese de actividad de Antonio, independiente de
-- su testimonio y de la estacionalidad del consumo doméstico.
Reactiva kVARh = SUM(T_consumo[Reactiva inductiva consumida kVARh])
```

```DAX
-- Variación interanual de la energía reactiva. Mismo patrón que las medidas de consumo
-- de energía activa, con una diferencia deliberada: aquí no se implementa la lógica de
-- SELECTEDVALUE(Calendario[AñoMes]) de "% Var Consumo 25 vs 24", por lo que esta medida
-- solo devuelve un valor a nivel de año completo (BLANK si se filtra a un único mes,
-- porque R24 y R25 no tendrían ambos datos en ese contexto). Es la que se usa para el KPI
-- de -56,29 % citado en el README, no una medida de clic mes a mes.
% Var Reactiva 25 vs 24 =
VAR R24 = CALCULATE([Reactiva kVARh], Calendario[Año] = 2024)
VAR R25 = CALCULATE([Reactiva kVARh], Calendario[Año] = 2025)
RETURN DIVIDE(R25 - R24, R24, BLANK())
```

## Medidas de potencia

```DAX
-- Potencia máxima real registrada por la instalación
Potencia Máxima Real = MAX(T_potencia[kW])
```

```DAX
-- Potencia contratada en el escenario actual (id 1). Se filtra por T_tarifas en lugar de
-- T_comparativa para partir siempre del catálogo de tarifas, y se limpian tanto el filtro
-- de tarifa como el de fecha (REMOVEFILTERS(Calendario)) para que el KPI no se vacíe si el
-- usuario hace clic en un mes o visual que no tenga fila para la tarifa 1 en ese contexto.
-- Se usa MAX en lugar de SELECTEDVALUE para no devolver BLANK en silencio si en algún
-- momento el histórico incluyera más de un valor de potencia contratada.
Potencia Contratada =
CALCULATE(
    MAX(T_comparativa[Potencia]),
    T_tarifas[id_tarifa] = 1,
    REMOVEFILTERS(T_tarifas),
    REMOVEFILTERS(Calendario)
)
```

```DAX
-- Relación entre la potencia máxima real demandada y la potencia contratada
% Utilización Potencia =
DIVIDE(
    [Potencia Máxima Real],
    [Potencia Contratada],
    0
)
```

```DAX
-- Margen entre la potencia contratada actual (16,44 kW) y la demanda máxima real
Margen Potencia Actual =
VAR ContratadaActual = [Potencia Contratada]
VAR MaximaDemandada = [Potencia Máxima Real]
RETURN
IF(
    ISBLANK(ContratadaActual) || ISBLANK(MaximaDemandada),
    BLANK(),
    ContratadaActual - MaximaDemandada
)
```

```DAX
-- Potencia propuesta en los escenarios de tarifa 2.0TD (SDH y DH)
Nueva Potencia Contratada =
CALCULATE(
    MAX(T_comparativa[Potencia]),
    T_comparativa[id_tarifa] IN {2, 3},
    -- ALLEXCEPT limpia todos los filtros de la tabla al hacer clic (periodos, leyendas, etc.)
    -- y SOLO respeta el filtro de la tarifa.
    ALLEXCEPT(T_comparativa, T_comparativa[id_tarifa])
)
```

```DAX
-- Margen entre la nueva potencia propuesta (10 kW) y la demanda máxima real
Margen Nueva Potencia =
VAR NuevaContratada = [Nueva Potencia Contratada]
VAR MaximaDemandada = [Potencia Máxima Real]
RETURN
IF(
    ISBLANK(NuevaContratada) || ISBLANK(MaximaDemandada),
    BLANK(),
    NuevaContratada - MaximaDemandada
)
```

## Medidas de comparativa

Respetan la diferencia estructural entre una tarifa de precio único y una tarifa por periodos, aplicando el cálculo fila a fila solo cuando es necesario. El término `Descuento` de la rama SDH aplica el 15 % de descuento sobre los kWh consumidos que incluye la tarifa actual (id 1).

```DAX
-- Coste de la energía consumida; la tarifa 1 (actual) incluye un 15 % de descuento sobre los kWh
Coste Consumo =
SUMX(
    SUMMARIZE(
        T_comparativa,
        Calendario[id_periodo_mes],
        T_tarifas[id_tarifa]
    ),
    VAR TarifaActual = T_tarifas[id_tarifa]
    RETURN
    IF(
        TarifaActual IN {1, 2},
        CALCULATE(SUM(T_comparativa[Kwh])) *
        CALCULATE(SELECTEDVALUE(T_comparativa[€/Kwh])) *
        COALESCE(CALCULATE(SELECTEDVALUE(T_comparativa[Descuento])), 1),
        CALCULATE(
            SUMX(
                T_comparativa,
                T_comparativa[Kwh] * T_comparativa[€/Kwh] * COALESCE(T_comparativa[Descuento], 1)
            )
        )
    )
)
```

```DAX
-- Coste del término de potencia, con cálculo mensual único para tarifas por periodos y fila a fila para la tarifa actual
Coste Potencia =
SUMX(
    SUMMARIZE(
        T_comparativa,
        Calendario[id_periodo_mes],
        T_tarifas[id_tarifa]
    ),
    VAR TarifaActual = T_tarifas[id_tarifa]
    RETURN
    IF(
        TarifaActual IN {2, 3},
        CALCULATE(SELECTEDVALUE(T_comparativa[Dias])) *
        CALCULATE(SELECTEDVALUE(T_comparativa[Potencia])) *
        CALCULATE(SELECTEDVALUE(T_comparativa[€/kw/dia])),
        CALCULATE(
            SUMX(
                T_comparativa,
                T_comparativa[Dias] * T_comparativa[Potencia] * T_comparativa[€/kw/dia]
            )
        )
    )
)
```

```DAX
-- Suma del coste de potencia y el coste de consumo, base de las medidas de ahorro
Costes potencia y consumo =
[Coste Potencia] + [Coste Consumo]
```

## Medidas de ahorro

```DAX
-- Ahorro en euros de la tarifa seleccionada frente a la tarifa actual (id 1) como referencia fija
Ahorro € =
VAR CosteRef =
    CALCULATE(
        [Costes potencia y consumo],
        -- Limpiamos cualquier nombre o ID de tarifa que esté en el gráfico/leyenda
        REMOVEFILTERS(T_tarifas),
        -- Forzamos a mirar siempre a la Tarifa 1
        T_tarifas[id_tarifa] = 1
    )
VAR CosteActual = [Costes potencia y consumo]
RETURN
    IF(
        ISBLANK(CosteActual),
        BLANK(),
        CosteRef - CosteActual
    )
```

```DAX
-- Ahorro porcentual de la tarifa seleccionada frente a la tarifa actual (id 1) como referencia fija
Ahorro % =
VAR CosteRef =
    CALCULATE(
        [Costes potencia y consumo],
        REMOVEFILTERS(T_tarifas),
        T_tarifas[id_tarifa] = 1
    )
VAR CosteActual = [Costes potencia y consumo]
RETURN
    IF(
        ISBLANK(CosteActual),
        BLANK(),
        DIVIDE(CosteRef - CosteActual, CosteRef, 0)
    )
```

```DAX
-- Coste total real facturado a Antonio, usado como referencia frente a los escenarios simulados
Facturas reales = SUM(T_Facturas[Total_Factura_Eur])
```

```DAX
Coste Potencia Actual a Nueva Potencia =
VAR NuevaPot = [Nueva Potencia Contratada]
RETURN
CALCULATE(
    SUMX(
        T_comparativa,
        T_comparativa[Dias] * NuevaPot * T_comparativa[€/kw/dia]
    ),
    REMOVEFILTERS(T_tarifas),
    T_tarifas[id_tarifa] = 1
)
```

```DAX
Ahorro € por reducción de potencia =
VAR PotenciaActual =
    CALCULATE(
        [Coste Potencia],
        REMOVEFILTERS(T_tarifas),
        T_tarifas[id_tarifa] = 1
    )
RETURN
PotenciaActual - [Coste Potencia Actual a Nueva Potencia]
```

```DAX
Ahorro € por precio de energía =
VAR EnergiaActual =
    CALCULATE(
        [Coste Consumo],
        REMOVEFILTERS(T_tarifas),
        T_tarifas[id_tarifa] = 1
    )
VAR EnergiaTarifaNueva =
    CALCULATE(
        [Coste Consumo],
        REMOVEFILTERS(T_tarifas),
        T_tarifas[id_tarifa] = 2          -- misma tarifa de referencia que uses arriba
    )
RETURN
EnergiaActual - EnergiaTarifaNueva
```

```DAX
Ahorro € por cambio de peaje =
VAR CosteReconfiguradoA10kW = [Coste Potencia Actual a Nueva Potencia]
VAR CostePotenciaTarifaNueva =
    CALCULATE(
        [Coste Potencia],
        REMOVEFILTERS(T_tarifas),
        T_tarifas[id_tarifa] = 2          -- fija aquí la tarifa que quieras usar de referencia (2 = 24 horas, 3 = DH)
    )
RETURN
CosteReconfiguradoA10kW - CostePotenciaTarifaNueva
```

```DAX
Valor Palanca =
VAR PalancaActual = SELECTEDVALUE(Palancas[Palanca])
RETURN
SWITCH(
    PalancaActual,
    "Coste tarifa Actual",          CALCULATE([Costes potencia y consumo], REMOVEFILTERS(T_tarifas), T_tarifas[id_tarifa] = 1),
    "Reducción de potencia",        -[Ahorro € por reducción de potencia],
    "Cambio de peaje 3.0TD→2.0TD",  -[Ahorro € por cambio de peaje],
    "Precio de energía",            -[Ahorro € por precio de energía],
    "Modalidad DH vs SDH",          -(CALCULATE([Costes potencia y consumo], REMOVEFILTERS(T_tarifas), T_tarifas[id_tarifa] = 2) - CALCULATE([Costes potencia y consumo], REMOVEFILTERS(T_tarifas), T_tarifas[id_tarifa] = 3)),
    BLANK()
)
```
