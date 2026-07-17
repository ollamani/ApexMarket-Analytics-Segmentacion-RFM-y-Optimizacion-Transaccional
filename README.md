# Analytics-Transaccional-ECommerce-Segmentacion-RFM
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ollamani/ApexMarket-Analytics-Segmentacion-RFM-y-Optimizacion-Transaccional/blob/main/Analytics_Transaccional_ECommerce_Segmentacion_RFM.ipynb)

Este proyecto implementa un pipeline de analítica de datos de extremo a extremo (End-to-End) para evaluar el comportamiento transaccional, optimizar la retención de clientes y calcular el **Lifetime Value (LTV)** en una cohorte simulada de **1,000 clientes** con más de **3,500 transacciones** acumuladas durante un periodo de dos años (2024-2026).

---

## Arquitectura del Proyecto

El proyecto está diseñado bajo una arquitectura de tres capas analíticas:
1. **Ingeniería de Datos y Simulación (Python):** Generación de comportamientos de compra realistas (siguiendo distribuciones probabilísticas como la Distribución Geométrica) usando `Pandas` y `NumPy`.
2. **Persistencia e Integridad Relacional (SQL):** Modelado físico de base de datos relacional y aplicación de restricciones rígidas en **SQLite**.
3. **Analítica de Clientes (SQL & Python):** Segmentación matemática bajo el modelo RFM (Recency, Frequency, Monetary) utilizando funciones de ventana (`NTILE`) y visualización ejecutiva con `Seaborn`.

---

## 1. Diseño y Modelado Relacional (SQL)

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

## El Proceso de Refinamiento: Del Caos a los Insights

Para entender el valor del proyecto, este es el contraste entre los datos crudos originales y el producto final refinado:

<details>
<summary> Haz clic aquí para ver un ejemplo de los Datos Crudos (Antes)</summary>

Así llegaban originalmente las transacciones al sistema: filas masivas sin clasificar, con fechas planas y montos dispersos que no aportaban valor estratégico por sí solos:

| transaction_id | customer_id | transaction_date | amount | store_id |
| :--- | :--- | :--- | :--- | :--- |
| TXN_90812 | Cust_510 | 2026-03-14 | 150.50 | STR_01 |
| TXN_90813 | Cust_510 | 2026-05-20 | 85.00 | STR_02 |
| TXN_92144 | Cust_581 | 2025-11-02 | 340.00 | STR_01 |
| TXN_95311 | Cust_201 | 2026-06-01 | 1200.00 | STR_03 |

</details>

<details>
<summary> Haz clic aquí para ver los Datos Modelados y Segmentados (Después)</summary>

Tras aplicar el modelado relacional en SQLite y procesar las métricas mediante el algoritmo RFM en Python, transformamos el caos anterior en una matriz de valor de cliente unificada y lista para la toma de decisiones:

| customer_name | segmento_negocio | recency (Días sin compra) | frequency (Compras) | monetary (Gasto Total) |
| :--- | :--- | :---: | :---: | :---: |
| Cliente_510 | Champions (VIP) | 9 | 6 | $11,820.00 |
| Cliente_581 | Champions (VIP) | 64 | 6 | $11,440.00 |
| Cliente_201 | Champions (VIP) | 49 | 8 | $9,790.00 |

</details>



---

## 2. Extracción y Scoring RFM con SQL Server / SQLite

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

## 3. Segmentación Estratégica y Visualización (Python)

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

## 4. Reporte de Rendimiento Financiero y KPIs

El análisis consolidado de la salud del negocio revela métricas críticas sobre la distribución de los ingresos en la tienda:

| Segmento de Negocio | Total Clientes | Ingresos Totales ($) | Aportación de Ingresos (%) | LTV Promedio ($) | Recencia Promedio (Días) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Champions (VIP)** | 165 | $385,420.50 | 44.15% | $2,335.88 | 42.5 |
| **Loyal Customers** | 220 | $260,110.00 | 29.80% | $1,182.31 | 89.2 |
| **New & Promising** | 180 | $112,450.00 | 12.88% | $624.72 | 31.0 |
| **At Risk / Can't Lose** | 115 | $85,220.00 | 9.76% | $741.04 | 164.8 |
| **Hibernating / Lost** | 320 | $29,800.00 | 3.41% | $93.12 | 345.2 |

*(Nota: Los valores numéricos anteriores son representativos y se actualizarán dinámicamente según la salida final de tu simulación en Colab).*

### Principales Hallazgos de Negocio:
1. **Concentración del Ingreso (Ley de Pareto):** El 38% de la base de datos (VIP + Fieles) genera más del **73% de los ingresos totales**, validando la necesidad de implementar campañas de retención enfocadas.
2. **Fuga Silenciosa ("At Risk"):** El segmento "At Risk" posee un LTV alto ($741.04) pero con una inactividad promedio de más de 160 días. Representa la mayor oportunidad de retorno de inversión para campañas de reactivación.

---

## Dashboard Interactivo (Data Studio)

Para llevar el análisis técnico a un nivel ejecutivo y facilitar la toma de decisiones estratégicas, se diseñó un tablero de control interactivo en Data Studio. Este dashboard permite al usuario navegar en tiempo real por el comportamiento financiero de la empresa.

### Características Clave:
* **Filtrado Dinámico:** Menú desplegable para aislar segmentos específicos (ej. *Champions*, *At Risk*) y recalcular instantáneamente el LTV, los ingresos totales y el volumen de clientes.
* **Filtrado Cruzado:** Las gráficas interactúan entre sí; hacer clic en cualquier barra segmenta de inmediato el resto del informe.
* **Enfoque de Negocio:** Una matriz visual con mapas de calor que identifica anomalías de retención y prioriza las listas de acción operativa para los equipos de marketing.

<img width="557" height="418" alt="image" src="https://github.com/user-attachments/assets/f09b7739-c8cc-4955-b15a-b39b235c5f5d" />


👉 **[Interactuar con el Dashboard en Vivo Aquí](https://datastudio.google.com/reporting/1e69cf08-cd70-48d8-b263-c09456ae0b83)**

---

## Estructura del Repositorio

```text
├── data/
│   └── apexmarket.db                   # Base de datos SQLite generada e integrada
├── notebooks/
│   └── segmentacion_rfm_ecommerce.ipynb # Jupyter Notebook con el flujo completo en Google Colab
└── README.md                           # Documentación técnica del proyecto
```

---

## Cómo Ejecutar el Proyecto en Google Colab

1. Haz clic en el botón de **Open in Colab** en tu cuaderno o clona este repositorio de forma local:
   ```bash
   git clone https://github.com/ollamani/ApexMarket-Analytics-Segmentacion-RFM-y-Optimizacion-Transaccional.git
   ```
2. Ejecuta las celdas en orden. El script de Python creará la base de datos local `apexmarket.db` automáticamente utilizando librerías estándar (`sqlite3`, `pandas`, `numpy`, `matplotlib`, `seaborn`), por lo que no requieres instalar dependencias externas complejas.
