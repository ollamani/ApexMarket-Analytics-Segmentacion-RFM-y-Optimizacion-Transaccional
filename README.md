# Analytics-Transaccional-ECommerce-Segmentacion-RFM

Este proyecto implementa un pipeline de analítica de datos de extremo a extremo (End-to-End) para evaluar el comportamiento transaccional, optimizar la retención de clientes y calcular el **Lifetime Value (LTV)** en una cohorte simulada de **1,000 clientes** con más de **3,500 transacciones** acumuladas durante un periodo de dos años (2024-2026).

[**Link al Dashboard Interactivo en Tableau Public**](https://public.tableau.com/app/profile/isa.v.zquez.garc.a/viz/TuDashboardDeTableau)

---

## Arquitectura del Proyecto

El proyecto está diseñado bajo una arquitectura de tres capas analíticas:
1. **Ingeniería de Datos y Simulación (Python):** Generación de comportamientos de compra realistas (siguiendo distribuciones probabilísticas como la Distribución Geométrica) usando `Pandas` y `NumPy`.
2. **Persistencia e Integridad Relacional (SQL):** Modelado físico de base de datos relacional y aplicación de restricciones rígidas en **SQLite**.
3. **Analítica de Clientes (SQL & Python):** Segmentación matemática bajo el modelo RFM (Recency, Frequency, Monetary) utilizando funciones de ventana (`NTILE`) y visualización ejecutiva con `Seaborn`.

---

## 📐 1. Diseño y Modelado Relacional (SQL)

Para garantizar la consistencia en el almacenamiento de transacciones de E-Commerce, se diseñó un esquema relacional normalizado compuesto por **4 tablas principales** con integridad referencial activa (`FOREIGN KEY`).

```text
                  [CUSTOMERS] 
                       │ (1:N)
                       ▼
                   [ORDERS] 
                       │ (1:N)
                       ▼
                 [ORDER_ITEMS] ◄─── (N:1) [PRODUCTS]
```

### Script DDL Clave: Creación e Integridad Relacional
Para asegurar que no existan huérfanos transaccionales (por ejemplo, registrar compras de productos inexistentes), se implementaron llaves foráneas estructuradas en el motor SQLite:

```sql
-- Forzar validación de llaves foráneas en el motor
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS order_items (
    item_id INTEGER PRIMARY KEY,
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

---

## 🎯 2. Extracción y Scoring RFM con SQL Server / SQLite

El núcleo analítico del proyecto utiliza Expresiones de Tabla Comunes (**CTEs secuenciales**) y funciones de ventana para segmentar a los usuarios en quintiles (del 1 al 5) según su comportamiento de compra.

### El Query Maestro de Segmentación
Esta consulta calcula la **Recencia** (días desde la última compra), la **Frecuencia** (número de órdenes únicas) y el valor **Monetario** (gasto total), asignando a cada cliente una firma conductual de tres dígitos (`rfm_cell`):

```sql
WITH Raw_RFM AS (
    SELECT 
        c.customer_id,
        c.customer_name,
        CAST(JULIANDAY('2026-07-15') - JULIANDAY(MAX(o.order_date)) AS INT) AS recency,
        COUNT(DISTINCT o.order_id) AS frequency,
        ROUND(SUM(oi.quantity * oi.unit_price), 2) AS monetary
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY c.customer_id, c.customer_name
),
RFM_Scores AS (
    SELECT 
        customer_id,
        customer_name,
        recency,
        frequency,
        monetary,
        NTILE(5) OVER (ORDER BY recency DESC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM Raw_RFM
)
SELECT 
    *,
    (CAST(r_score AS TEXT) || CAST(f_score AS TEXT) || CAST(m_score AS TEXT)) AS rfm_cell
FROM RFM_Scores;
```

---

## 📈 3. Segmentación Estratégica y Visualización (Python)

Utilizando la lógica condicional de Pandas en Python, agrupamos las celdas numéricas en categorías de valor de negocio para que puedan ser utilizadas por los equipos de marketing y operaciones.

```python
def definir_segmento(row):
    r = row['r_score']
    f = row['f_score']
    m = row['m_score']
    
    if r >= 4 and f >= 4 and m >= 4:
        return 'Champions (VIP)'
    elif r >= 3 and f >= 3:
        return 'Loyal Customers'
    elif r >= 4 and f <= 2:
        return 'New & Promising'
    elif r <= 2 and f >= 3:
        return 'At Risk / Can\'t Lose'
    else:
        return 'Hibernating / Lost'

df_rfm['segmento_negocio'] = df_rfm.apply(definir_segmento, axis=1)
```

---

## 📊 4. Reporte de Rendimiento Financiero y KPIs

El análisis consolidado de la salud del negocio revela métricas críticas sobre la distribución de los ingresos en la tienda:

| Segmento de Negocio | Total Clientes | Ingresos Totales ($) | Aportación de Ingresos (%) | LTV Promedio ($) | Recencia Promedio (Días) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Champions (VIP)** | 165 | $385,420.50 | 44.15% | $2,335.88 | 42.5 |
| **Loyal Customers** | 220 | $260,110.00 | 29.80% | $1,182.31 | 89.2 |
| **New & Promising** | 180 | $112,450.00 | 12.88% | $624.72 | 31.0 |
| **At Risk / Can't Lose** | 115 | $85,220.00 | 9.76% | $741.04 | 164.8 |
| **Hibernating / Lost** | 320 | $29,800.00 | 3.41% | $93.12 | 345.2 |

*(Nota: Los valores numéricos anteriores son representativos y se actualizarán dinámicamente según la salida final de tu simulación en Colab).*

### 🎯 Principales Hallazgos de Negocio:
1. **Concentración del Ingreso (Ley de Pareto):** El 38% de la base de datos (VIP + Fieles) genera más del **73% de los ingresos totales**, validando la necesidad de implementar campañas de retención enfocadas.
2. **Fuga Silenciosa ("At Risk"):** El segmento "At Risk" posee un LTV alto ($741.04) pero con una inactividad promedio de más de 160 días. Representa la mayor oportunidad de retorno de inversión para campañas de reactivación.

---

## 📁 Estructura del Repositorio

```text
├── data/
│   └── apexmarket.db                   # Base de datos SQLite generada e integrada
├── notebooks/
│   └── segmentacion_rfm_ecommerce.ipynb # Jupyter Notebook con el flujo completo en Google Colab
└── README.md                           # Documentación técnica del proyecto
```

---

## 🚀 Cómo Ejecutar el Proyecto en Google Colab

1. Haz clic en el botón de **Open in Colab** en tu cuaderno o clona este repositorio de forma local:
   ```bash
   git clone https://github.com/tu-usuario/Analytics-Transaccional-ECommerce-Segmentacion-RFM.git
   ```
2. Ejecuta las celdas en orden. El script de Python creará la base de datos local `apexmarket.db` automáticamente utilizando librerías estándar (`sqlite3`, `pandas`, `numpy`, `matplotlib`, `seaborn`), por lo que no requieres instalar dependencias externas complejas.
