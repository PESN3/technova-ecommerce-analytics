# TechNova Ecommerce Analytics

Analisis de marketing digital y comportamiento de compra para una tienda online de tecnologia en Latinoamerica.

**Dashboard:** [Ver en Tableau Public](https://public.tableau.com/app/profile/pedro.enmanuel.sanchez.noriega/viz/E-commerceAnalyticsAdquisicinMercadoyLTV/EstrategiadeAdquisicinyEmbudo)

---

## Contexto

TechNova es una tienda online de productos tecnologicos (laptops, celulares, tablets, audifonos, accesorios) que opera en Latinoamerica. La empresa invierte en marketing digital pero no sabe que canal rentabiliza mas, que productos generan mas ingresos, ni por que se abandonan los carritos.

Este proyecto analiza 6 meses de datos (enero a junio 2026) para responder 5 preguntas de negocio clave.

---

## Datos

| Tabla | Registros | Descripcion |
|---|---|---|
| productos | 45 | Catalogo con precio, costo, margen y stock |
| clientes | 2,500 | Clientes de 5 paises LATAM con canal de adquisicion |
| campanas_ads | 636 | Inversion diaria en ads (dic 2025 - jun 2026) |
| carritos | 8,000 | Carritos creados (comprados y abandonados) |
| ventas | 4,488 | Compras concretadas |
| abandonos | 3,512 | Carritos abandonados |

**Herramientas:** SQL (Google BigQuery) + Tableau Public

---

## 1. Que canal de marketing rentabiliza mas?

**KPI:** ROAS (Return on Ad Spend) = Ventas / Inversion

| Canal | Inversion Total | Ventas Total | ROAS |
|---|---|---|---|
| Instagram Ads | $12,642 | $51,776 | 15.67x |
| Google Ads | $33,093 | $80,091 | 14.85x |
| Facebook Ads | $18,861 | $57,201 | 13.23x |

**Insight:** Instagram Ads tiene el mejor ROAS (15.67x), pero Google Ads genera mas ventas totales ($80K). La estrategia deberia mantener inversion en Instagram para rentabilidad y escalar Google para volumen.

```sql
SELECT 
  c.canal,
  ROUND(SUM(c.inversion_usd), 2) AS inversion_total,
  ROUND(SUM(v.monto_usd), 2) AS ventas_total,
  ROUND(SUM(v.monto_usd) / SUM(c.inversion_usd), 2) AS roas
FROM `practicas-494415.technova_analytics.campanas` c
LEFT JOIN `practicas-494415.technova_analytics.ventas` v
  ON c.canal = v.canal_venta
GROUP BY c.canal
ORDER BY roas DESC;
```

---

## 2. Que categoria de producto genera mas ingresos y margen?

| Categoria | Unidades Vendidas | Ingresos Totales | Margen Total | Ticket Promedio |
|---|---|---|---|---|
| Celulares | 841 | $456,722 | $139,036 | $543 |
| Laptops | 683 | $433,756 | $154,140 | $635 |
| Tablets | 725 | $246,461 | $99,884 | $340 |
| Audifonos | 1,115 | $122,863 | $68,777 | $110 |
| Accesorios | 1,124 | $51,108 | $31,193 | $45 |

**Insight:** Celulares generan mas ingresos totales, pero Laptops generan mas margen absoluto ($154K vs $139K) a pesar de vender menos unidades. Accesorios venden mucho volumen pero aportan poco margen.

```sql
SELECT 
  p.categoria,
  COUNT(v.carrito_id) AS unidades_vendidas,
  ROUND(SUM(v.monto_usd), 2) AS ingresos_totales,
  ROUND(SUM(p.margen_usd), 2) AS margen_total,
  ROUND(AVG(v.monto_usd), 2) AS ticket_promedio
FROM `practicas-494415.technova_analytics.ventas` v
JOIN `practicas-494415.technova_analytics.productos` p
  ON v.producto_id = p.producto_id
GROUP BY p.categoria
ORDER BY ingresos_totales DESC;
```

---

## 3. Por que se abandonan los carritos?

**KPI:** Tasa de Abandono = Abandonados / Total de Carritos

| Categoria | Total Carritos | Abandonados | Tasa Abandono |
|---|---|---|---|
| Laptops | 1,624 | 941 | 57.94% |
| Celulares | 1,872 | 1,031 | 55.07% |
| Tablets | 1,244 | 519 | 41.72% |
| Audifonos | 1,652 | 537 | 32.51% |
| Accesorios | 1,608 | 484 | 30.10% |

**Insight:** Productos caros (Laptops, Celulares) tienen mas del 55% de abandono. La decision de compra es mas larga cuando el monto es alto. Se recomienda implementar remarketing y recordatorios de carrito abandonado especificamente para estos productos.

```sql
SELECT 
  p.categoria,
  COUNT(*) AS total_carritos,
  SUM(CASE WHEN c.estado = 'Abandonado' THEN 1 ELSE 0 END) AS abandonados,
  ROUND(
    SUM(CASE WHEN c.estado = 'Abandonado' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 
    2
  ) AS tasa_abandono_pct
FROM `practicas-494415.technova_analytics.carritos` c
JOIN `practicas-494415.technova_analytics.productos` p
  ON c.producto_id = p.producto_id
GROUP BY p.categoria
ORDER BY tasa_abandono_pct DESC;
```

---

## 4. Que pais compra mas y con que ticket promedio?

| Pais | Total Ventas | Ingresos Totales | Ticket Promedio |
|---|---|---|---|
| Mexico | 1,304 | $373,959 | $286.78 |
| Colombia | 1,171 | $341,500 | $291.63 |
| Argentina | 947 | $283,130 | $298.98 |
| Chile | 693 | $205,301 | $296.25 |
| Peru | 373 | $107,020 | $286.92 |

**Insight:** Mexico genera mas ingresos por volumen (1,304 ventas). Argentina tiene el ticket promedio mas alto ($298.98) pero menos ventas. Estrategia: aumentar marketing en Argentina para escalar volumen sin bajar precios.

```sql
SELECT 
  cl.pais,
  COUNT(v.carrito_id) AS total_ventas,
  ROUND(SUM(v.monto_usd), 2) AS ingresos_totales,
  ROUND(AVG(v.monto_usd), 2) AS ticket_promedio
FROM `practicas-494415.technova_analytics.ventas` v
JOIN `practicas-494415.technova_analytics.clientes` cl
  ON v.cliente_id = cl.cliente_id
GROUP BY cl.pais
ORDER BY ingresos_totales DESC;
```

---

## 5. Cuanto vale cada cliente segun como llego?

**KPI:** LTV (Lifetime Value) = Ingresos Totales / Clientes Unicos

| Canal Adquisicion | Total Clientes | Ingresos Totales | LTV Promedio |
|---|---|---|---|
| Email Marketing | 243 | $135,470 | $557.49 |
| Referido | 123 | $67,722 | $550.59 |
| Organico | 377 | $205,446 | $544.95 |
| Facebook Ads | 627 | $337,079 | $537.61 |
| Instagram Ads | 498 | $268,988 | $540.14 |
| Google Ads | 632 | $296,206 | $468.68 |

**Insight:** Email Marketing y Referido tienen el LTV mas alto (~$550). Los clientes que llegan por canales organicos o email valen mas que los de ads pagadas. Recomendacion: invertir mas en email marketing y programa de referidos.

```sql
SELECT 
  cl.canal_adquisicion,
  COUNT(DISTINCT cl.cliente_id) AS total_clientes,
  ROUND(SUM(v.monto_usd), 2) AS ingresos_totales,
  ROUND(SUM(v.monto_usd) / COUNT(DISTINCT cl.cliente_id), 2) AS ltv_promedio
FROM `practicas-494415.technova_analytics.clientes` cl
LEFT JOIN `practicas-494415.technova_analytics.ventas` v
  ON cl.cliente_id = v.cliente_id
GROUP BY cl.canal_adquisicion
ORDER BY ltv_promedio DESC;
```

---

## Stack Tecnologico

- Google BigQuery (almacenamiento y analisis SQL)
- SQL ANSI (extraccion de insights, calculo de KPIs)
- Tableau Public (visualizacion interactiva)
- Python Pandas (generacion de dataset sintetico)

---

## Autor

Pedro Enmanuel Sanchez Noriega
- Analista de Datos | SQL, BigQuery, Tableau, Python
- Formacion en Psicologia Organizacional y Gestion Remota
- Google Data Analytics (en curso)

LinkedIn: [tu-linkedin]
GitHub: [tu-github]

---

*Proyecto desarrollado como parte de mi portafolio profesional. Los datos son sinteticos pero reflejan patrones reales de comportamiento de ecommerce en Latinoamerica.*
