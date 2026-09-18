# Optimización de tarifa eléctrica con Power BI y DAX

> Antonio tiene 66 años y está jubilado. Durante muchos años desarrolló una actividad profesional que requería el uso continuado de maquinaria en una instalación que compartía suministro eléctrico con su vivienda, lo que justificaba un elevado consumo energético y una potencia contratada de **16,44 kW**. Tras su jubilación, Antonio cesa esa actividad y vende la maquinaria. Sin embargo, aunque sus necesidades energéticas cambian radicalmente, la instalación eléctrica permanece igual, porque vivienda y negocio nunca tuvieron contadores ni contratos separados.

**Autor**: Jose Maderas
**Stack:** Excel · Power BI · DAX

![Demo del dashboard](docs/img/dashboard.gif)

---

## Resumen ejecutivo

Antonio mantenía 16,44 kW de potencia contratada dos años después de cesar la actividad profesional que los justificaba. El análisis de 17.542 registros horarios de consumo y de 57 registros del maxímetro demuestra que su demanda máxima real es de 6 kW, es decir, un 36,50 % de lo contratado. Simulando tres escenarios de contrato sobre su consumo real de 2025, la reducción a 10 kW con peaje 2.0TD supone 586,07 € anuales menos en los términos de energía y potencia (-37,8 %). El 71,6 % de ese ahorro procede del término de potencia, no de la tarifa de energía: el problema no era qué tarifa tenía contratada, sino cuántos kilovatios.

## Hallazgos clave

|Indicador|Valor|
|---|---|
|Potencia contratada|16,44 kW|
|Potencia máxima real registrada|6 kW|
|Utilización de la potencia contratada|36,50 %|
|Consumo 2025 vs 2024|-8,72 %|
|Energía reactiva 2025 vs 2024|-56,29 %|

La caída del 56,29 % en energía reactiva confirma físicamente el cese de actividad, con una variable que no depende del testimonio del cliente: en las cuatro horas en que operaba la maquinaria (7:00, 8:00, 19:00 y 20:00) la reactiva cae entre un 84,6 % y un 90,6 %, frente a un 22,6 % en el resto del día.

## Propuesta

Peaje 2.0TD en modalidad de precio único (24 h) con 10 kW contratados. El cambio de peaje no es opcional: por debajo de 15 kW la 2.0TD es obligatoria (Circular 3/2020, CNMC).

## Impacto

586,07 € anuales menos en los términos de energía y potencia, un -37,8 % sobre esos dos conceptos. El 71,6 % del ahorro viene del término de potencia, no de la tarifa de energía: es una corrección de sobredimensionamiento, no un cambio de oferta comercial.

> **Alcance de la cifra.** Los 586,07 € cubren únicamente energía consumida y potencia contratada, los dos únicos conceptos sobre los que Antonio decide al elegir tarifa. Quedan fuera impuestos, alquiler del contador y financiación del bono social. El ahorro en la factura completa puede diferir; el criterio de decisión, no.

**Lo que este proyecto no es.** Una proyección de facturas futuras. Los precios se mantienen fijos en los 12 meses simulados, así que la comparativa responde a "qué habría pagado con cada oferta", no a "qué pagará".

---

## Documentación ampliada

- **[Análisis completo](docs/analisis-completo.md)** — el caso de Antonio, la estructura del dashboard pestaña a pestaña y la decisión final con su desglose de impacto.
- **[Arquitectura técnica](docs/arquitectura-tecnica.md)** — modelo de datos, columnas calculadas, medidas DAX, origen de los datos y limitaciones.
- **[Informe ejecutivo](docs/informe-ejecutivo.md)** — el mismo análisis en formato de informe interno de empresa (planteamiento, hallazgos, recomendaciones, próximos pasos, fuentes de datos).

---

## Requisitos y cómo reproducir

### Requisitos

- **Power BI Desktop**
- **Configuración regional**: el modelo usa formato español (decimales con coma, fechas dd/mm/aaaa). Si tu Power BI Desktop está configurado en otra región, algunas medidas pueden requerir ajuste de separador decimal
- **Excel**: el archivo `Datos de costes energéticos.xlsx` debe mantenerse en la ruta relativa `data/` respecto al `.pbix`, ya que el modelo usa modo de importación (Import) sobre este archivo como fuente única

### Cómo reproducir

1. Clona o descarga este repositorio completo, manteniendo la estructura de carpetas (`data/` y `dashboard/` al mismo nivel)
2. Abre `dashboard/dashboard.pbix` con Power BI Desktop
3. Si Power BI solicita reautenticar el origen de datos, apunta a la ruta local de `data/data.xlsx` dentro de tu copia del repositorio
4. Actualiza el modelo (`Inicio > Actualizar`) para confirmar que las relaciones cargan sin errores
5. Las cuatro pestañas (Consumo, Potencia, Comparativa, Ahorro) son interactivas: los filtros de Periodo, Tipo de Día y Tarifa afectan a los visuales relacionados dentro de cada pestaña

### Contenido del repositorio

```
├── data/
│   ├── data.xlsx
│   └── csv/
│	    └── comparativa.csv
│	    └── consumo.csv
│	    └── facturas.csv
│	    └── periodo_tarifario.csv
│	    └── potencia.csv
│	    └── tarifas.csv
├── dashboard/
│   └── dashboard.pbix
├── docs/
│   ├── analisis-completo.md
│   ├── arquitectura-tecnica.md
│   ├── informe-ejecutivo.md
│   └── img/
│	    ├── dashboard-pagina1-consumo.png
│	    ├── dashboard-pagina2-potencia.png
│	    ├── dashboard-pagina3-comparativa.png
│	    ├── dashboard-pagina4-ahorro.png
│	    └── dashboard.gif
├── .gitignore
├── LICENSE
└── README.md
```

## Protección de datos

Los datos de consumo, potencia y facturación utilizados en este proyecto corresponden a una persona real, identificada en la documentación únicamente como "Antonio". Su identidad y cualquier dato que permita identificarle (CUPS, dirección, titular del contrato, comercializadora) han sido omitidos o excluidos del modelo, de acuerdo con la normativa aplicable en materia de protección de datos. Las facturas reales se recortaron antes de su procesamiento, de forma que solo se incluyeran las secciones "detalle de
factura" y "lecturas". Antonio ha dado su consentimiento para el uso de estos datos, anonimizados, con fines demostrativos en este portfolio.

## Licencia

© Jose Maderas. Publicado bajo licencia MIT: se autoriza el uso, copia, modificación y distribución de este proyecto, incluso con fines comerciales, siempre que se mantenga el aviso de copyright original. El software se proporciona "tal cual", sin garantía de ningún tipo. Ver [LICENSE](LICENSE) para el texto completo.