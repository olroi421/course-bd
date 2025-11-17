# Лекція 19. Системи підтримки прийняття рішень та OLAP

## Вступ

Сучасні організації накопичують величезні обсяги даних у процесі своєї операційної діяльності. Однак сама наявність даних не гарантує можливості приймати обґрунтовані управлінські рішення. Для перетворення сирих даних у цінну інформацію та знання необхідні спеціалізовані системи, які забезпечують ефективний аналіз великих обсягів історичних даних, виявлення закономірностей та тенденцій, підтримку багатовимірного аналізу. Такі системи отримали назву систем підтримки прийняття рішень та OLAP систем.

Розуміння відмінностей між операційними та аналітичними системами, знання архітектури сховищ даних та принципів багатовимірного моделювання є критично важливим для розробників сучасних інформаційних систем, оскільки майже кожна корпоративна система рано чи пізно потребує аналітичних можливостей.

## Основні визначення та концепції

**Система підтримки прийняття рішень (Decision Support System, DSS)** — це комп'ютеризована інформаційна система, призначена для підтримки управлінської діяльності в умовах напівструктурованих та неструктурованих завдань. Така система надає менеджерам інструменти для аналізу великих обсягів даних, моделювання ситуацій та прогнозування наслідків управлінських рішень.

**OLAP (Online Analytical Processing)** — технологія обробки даних, яка дозволяє аналітикам, менеджерам та керівникам швидко, інтерактивно та з різних точок зору отримувати доступ до інформації, що була перетворена з сирих даних для відображення реального розуміння підприємства користувачем.

**Сховище даних (Data Warehouse)** — предметно-орієнтована, інтегрована, незмінна та історична колекція даних, призначена для підтримки прийняття управлінських рішень. Термін був введений Біллом Інмоном у 1990 році.

**ETL (Extract, Transform, Load)** — процес вилучення даних з різних джерел, їх трансформації у необхідний формат та завантаження у цільову систему, зазвичай сховище даних.

**Куб даних (Data Cube)** — багатовимірне представлення даних, де кожен вимір відповідає певному атрибуту або категорії аналізу, а комірки куба містять вимірювані показники.

## OLTP versus OLAP: фундаментальні відмінності

### Характеристики робочих навантажень

Для розуміння необхідності створення окремих аналітичних систем важливо усвідомити фундаментальні відмінності між двома типами робочих навантажень баз даних.

**OLTP (Online Transaction Processing)** системи призначені для підтримки повсякденної операційної діяльності організації. Вони обслуговують велику кількість користувачів, які виконують короткі транзакції з невеликою кількістю записів. Типові операції включають додавання нового замовлення, оновлення інформації про клієнта, реєстрацію платежу.

**OLAP (Online Analytical Processing)** системи призначені для аналітичної обробки та підтримки прийняття рішень. Вони обслуговують меншу кількість користувачів, переважно аналітиків та менеджерів, які виконують складні запити над великими обсягами історичних даних для виявлення тенденцій та закономірностей.

**Порівняльна таблиця характеристик:**

| Характеристика | OLTP | OLAP |
|----------------|------|------|
| Призначення | Підтримка операційної діяльності | Підтримка аналітичної діяльності та прийняття рішень |
| Джерело даних | Поточні, оперативні дані | Історичні, консолідовані дані |
| Тип операцій | Вставка, оновлення, видалення | Складні запити, агрегація |
| Розмір транзакцій | Короткі, прості запити | Довгі, складні запити |
| Кількість записів на запит | Від одиниць до сотень | Від тисяч до мільйонів |
| Користувачі | Тисячі співробітників | Десятки аналітиків |
| Час відгуку | Мілісекунди | Секунди або хвилини |
| Нормалізація | Висока нормалізація (3НФ і вище) | Часто денормалізована (зоряна схема) |
| Історичність даних | Поточний стан | Історія змін протягом років |
| Розмір бази даних | Гігабайти | Терабайти та петабайти |
| Оновлення даних | Постійні, в реальному часі | Періодичні, пакетні |

### Приклади типових операцій

**Типова OLTP операція — створення замовлення:**

```sql
BEGIN TRANSACTION;

INSERT INTO orders (customer_id, order_date, status)
VALUES (12345, CURRENT_TIMESTAMP, 'NEW')
RETURNING order_id;

INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES
    (98765, 101, 2, 29.99),
    (98765, 205, 1, 149.99);

UPDATE products
SET stock_quantity = stock_quantity - 2
WHERE product_id = 101;

UPDATE products
SET stock_quantity = stock_quantity - 1
WHERE product_id = 205;

COMMIT;
```

Ця транзакція торкається лише кількох записів, виконується швидко та відразу фіксує зміни в базі даних.

**Типова OLAP операція — аналіз продажів за регіонами:**

```sql
SELECT
    r.region_name,
    p.category,
    DATE_TRUNC('month', o.order_date) as month,
    COUNT(DISTINCT o.order_id) as total_orders,
    SUM(oi.quantity) as total_units,
    SUM(oi.quantity * oi.price) as total_revenue,
    AVG(oi.quantity * oi.price) as avg_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN customers c ON o.customer_id = c.customer_id
JOIN regions r ON c.region_id = r.region_id
WHERE o.order_date >= DATE_TRUNC('year', CURRENT_DATE - INTERVAL '3 years')
GROUP BY r.region_name, p.category, DATE_TRUNC('month', o.order_date)
ORDER BY r.region_name, p.category, month;
```

Цей запит обробляє мільйони записів за кілька років, виконує множинні з'єднання та агрегації, повертає тисячі рядків результату.

### Конфлікт вимог та необхідність розділення

Спроби виконувати складні аналітичні запити безпосередньо на операційній базі даних призводять до низки проблем. По-перше, такі запити створюють значне навантаження на систему, конкуруючи за ресурси з операційними транзакціями та потенційно уповільнюючи критично важливі бізнес-процеси. По-друге, нормалізована структура операційної бази даних оптимізована для транзакцій, а не для аналітичних запитів, що робить останні неефективними. По-третє, операційні системи зазвичай зберігають лише поточний стан даних або обмежену історію, тоді як для аналізу часто потрібні роки історичних даних.

Саме ці фундаментальні відмінності у вимогах призвели до появи концепції окремих систем для аналітичної обробки, побудованих на інших принципах організації даних та оптимізованих для читання великих обсягів інформації замість швидкої обробки транзакцій.

## Архітектура сховищ даних

### Компоненти класичної архітектури

Архітектура сховища даних складається з кількох ключових компонентів, кожен з яких виконує специфічну функцію в процесі перетворення операційних даних у аналітичну інформацію.

```mermaid
graph TB
    A[Операційні системи<br/>OLTP БД] --> B[ETL процеси<br/>Extract, Transform, Load]
    C[Зовнішні джерела<br/>Файли, API] --> B
    D[Legacy системи<br/>Старі додатки] --> B

    B --> E[Staging Area<br/>Проміжна зона]
    E --> F[Сховище даних<br/>Data Warehouse]

    F --> G[Data Marts<br/>Відділові вітрини]

    G --> H[OLAP сервер<br/>Куби даних]
    G --> I[Звіти та дашборди<br/>Reporting Tools]
    G --> J[Data Mining<br/>Аналітика]

    H --> K[Аналітики та<br/>менеджери]
    I --> K
    J --> K
```

**Джерела даних** включають різноманітні системи, з яких витягуються дані. Зазвичай це операційні бази даних різних відділів організації, зовнішні джерела інформації, файли різних форматів, API сторонніх сервісів. Важливою характеристикою є гетерогенність цих джерел, вони можуть використовувати різні СУБД, формати даних, семантику полів.

**Проміжна зона (Staging Area)** — це тимчасове сховище, куди завантажуються дані з джерел перед їх обробкою. Тут дані зберігаються у формі, близькій до оригіналу, що дозволяє відновити процес завантаження у разі збою. Проміжна зона також використовується для валідації даних перед їх інтеграцією у сховище.

**Сховище даних (Data Warehouse)** — це центральне сховище інтегрованих даних, організоване за предметними областями. Тут дані структуровані таким чином, щоб забезпечити ефективний аналіз, зберігається повна історія змін, дані не видаляються, а позначаються як неактуальні.

**Відділові вітрини даних (Data Marts)** — це підмножини сховища даних, сфокусовані на конкретних предметних областях або потребах окремих відділів. Наприклад, може існувати вітрина для відділу продажів, окрема вітрина для маркетингу, фінансова вітрина. Вони містять агреговані та оптимізовані дані для специфічних аналітичних завдань.

**OLAP сервер** будує багатовимірні куби на основі даних зі сховища або вітрин, забезпечуючи швидкий інтерактивний аналіз. Існують різні архітектури OLAP серверів, про які детальніше буде сказано далі.

### Процес ETL: Extract, Transform, Load

ETL процес є критично важливим компонентом будь-якого сховища даних. Він забезпечує вилучення даних з різнорідних джерел, їх очищення та трансформацію у узгоджений формат, завантаження у цільову систему.

**Етап вилучення (Extract)** полягає у читанні даних з різних джерел. Це може бути повне вилучення всіх даних або інкрементальне вилучення лише змінених записів. Основні виклики на цьому етапі включають роботу з різними форматами даних, мінімізацію впливу на продуктивність джерельних систем, забезпечення узгодженості даних при вилученні з кількох джерел.

**Приклад вилучення даних з кількох джерел:**

```python
import pandas as pd
from sqlalchemy import create_engine

# Підключення до операційної бази даних PostgreSQL
oltp_engine = create_engine('postgresql://user:pass@host1/sales_db')

# Вилучення замовлень за останню добу
orders_query = """
    SELECT
        order_id,
        customer_id,
        order_date,
        status,
        total_amount
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '1 day'
"""
orders_df = pd.read_sql(orders_query, oltp_engine)

# Підключення до CRM системи на MySQL
crm_engine = create_engine('mysql://user:pass@host2/crm_db')

# Вилучення інформації про клієнтів
customers_query = """
    SELECT
        customer_id,
        customer_name,
        segment,
        region
    FROM customers
    WHERE customer_id IN ({})
""".format(','.join(map(str, orders_df['customer_id'].unique())))

customers_df = pd.read_sql(customers_query, crm_engine)

# Вилучення даних з CSV файлу
products_df = pd.read_csv('/data/product_catalog.csv')
```

**Етап трансформації (Transform)** є найбільш складним та трудомістким. Тут виконується очищення даних від помилок та дублікатів, стандартизація форматів та значень, збагачення даних додатковою інформацією, агрегація та обчислення похідних показників, встановлення зв'язків між даними з різних джерел. Також на цьому етапі виконується обчислення сурогатних ключів для забезпечення незалежності від джерельних систем.

**Приклад трансформації даних:**

```python
# Очищення та стандартизація
customers_df['customer_name'] = customers_df['customer_name'].str.strip().str.title()
customers_df['region'] = customers_df['region'].str.upper()

# Обробка пропущених значень
customers_df['segment'].fillna('UNKNOWN', inplace=True)

# Обчислення похідних атрибутів
orders_df['order_year'] = orders_df['order_date'].dt.year
orders_df['order_month'] = orders_df['order_date'].dt.month
orders_df['order_quarter'] = orders_df['order_date'].dt.quarter

# З'єднання даних з різних джерел
enriched_orders = orders_df.merge(
    customers_df,
    on='customer_id',
    how='left'
)

enriched_orders = enriched_orders.merge(
    products_df,
    on='product_id',
    how='left'
)

# Обчислення агрегатів
daily_summary = enriched_orders.groupby(['order_date', 'region']).agg({
    'order_id': 'count',
    'total_amount': 'sum',
    'customer_id': 'nunique'
}).reset_index()

daily_summary.columns = ['date', 'region', 'order_count', 'revenue', 'unique_customers']

# Генерація сурогатних ключів
enriched_orders['surrogate_order_key'] = range(1, len(enriched_orders) + 1)
```

**Етап завантаження (Load)** полягає у записі трансформованих даних у сховище. Існує два основні підходи до завантаження. Повне завантаження видаляє всі існуючі дані та завантажує нову версію повністю, що просто у реалізації, але вимагає багато часу та ресурсів. Інкрементальне завантаження додає лише нові або змінені записи, що більш ефективно, але складніше у реалізації. Часто використовується гібридний підхід, коли частина таблиць завантажується повністю, а частина інкрементально.

**Приклад завантаження даних у сховище:**

```python
# Підключення до сховища даних
dwh_engine = create_engine('postgresql://user:pass@dwh_host/warehouse_db')

# Завантаження фактів замовлень
enriched_orders.to_sql(
    'fact_orders',
    dwh_engine,
    if_exists='append',
    index=False,
    method='multi',
    chunksize=10000
)

# Завантаження агрегованих даних
daily_summary.to_sql(
    'fact_daily_sales',
    dwh_engine,
    if_exists='append',
    index=False
)

# Оновлення вимірів методом slowly changing dimension type 2
with dwh_engine.connect() as conn:
    # Закриття старих версій записів
    conn.execute("""
        UPDATE dim_customer
        SET valid_to = CURRENT_DATE,
            is_current = FALSE
        WHERE customer_id IN (
            SELECT DISTINCT customer_id FROM staging_customers
        )
        AND is_current = TRUE
    """)

    # Вставка нових версій
    conn.execute("""
        INSERT INTO dim_customer (
            customer_id,
            customer_name,
            segment,
            region,
            valid_from,
            valid_to,
            is_current
        )
        SELECT
            customer_id,
            customer_name,
            segment,
            region,
            CURRENT_DATE,
            '9999-12-31',
            TRUE
        FROM staging_customers
    """)

    conn.commit()
```

### Моделювання багатовимірних даних

Центральною концепцією у сховищах даних є багатовимірне моделювання, яке організовує дані навколо бізнес-процесів та вимірів їх аналізу. Найпоширенішими схемами багатовимірного моделювання є зоряна схема та сніжинка.

**Зоряна схема (Star Schema)** складається з центральної таблиці фактів, оточеної таблицями вимірів. Таблиця фактів містить вимірювані показники та зовнішні ключі до таблиць вимірів. Таблиці вимірів містять описові атрибути та є денормалізованими для швидкого доступу.

**Приклад зоряної схеми для аналізу продажів:**

```sql
-- Таблиця фактів продажів
CREATE TABLE fact_sales (
    sale_id BIGSERIAL PRIMARY KEY,
    date_key INTEGER NOT NULL,
    product_key INTEGER NOT NULL,
    customer_key INTEGER NOT NULL,
    store_key INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    total_amount DECIMAL(12,2) NOT NULL,
    cost_amount DECIMAL(12,2) NOT NULL,
    profit_amount DECIMAL(12,2) NOT NULL,
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (product_key) REFERENCES dim_product(product_key),
    FOREIGN KEY (customer_key) REFERENCES dim_customer(customer_key),
    FOREIGN KEY (store_key) REFERENCES dim_store(store_key)
);

-- Вимір дати
CREATE TABLE dim_date (
    date_key INTEGER PRIMARY KEY,
    full_date DATE NOT NULL,
    day_of_week VARCHAR(10),
    day_of_month INTEGER,
    day_of_year INTEGER,
    week_of_year INTEGER,
    month_number INTEGER,
    month_name VARCHAR(10),
    quarter_number INTEGER,
    quarter_name VARCHAR(2),
    year_number INTEGER,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    holiday_name VARCHAR(50)
);

-- Вимір продукту
CREATE TABLE dim_product (
    product_key INTEGER PRIMARY KEY,
    product_id VARCHAR(50) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    product_description TEXT,
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    supplier_name VARCHAR(200),
    unit_cost DECIMAL(10,2),
    valid_from DATE NOT NULL,
    valid_to DATE NOT NULL,
    is_current BOOLEAN DEFAULT TRUE
);

-- Вимір клієнта
CREATE TABLE dim_customer (
    customer_key INTEGER PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    customer_name VARCHAR(200) NOT NULL,
    customer_type VARCHAR(50),
    segment VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(20),
    city VARCHAR(100),
    region VARCHAR(100),
    country VARCHAR(100),
    postal_code VARCHAR(20),
    registration_date DATE,
    valid_from DATE NOT NULL,
    valid_to DATE NOT NULL,
    is_current BOOLEAN DEFAULT TRUE
);

-- Вимір магазину
CREATE TABLE dim_store (
    store_key INTEGER PRIMARY KEY,
    store_id VARCHAR(50) NOT NULL,
    store_name VARCHAR(200) NOT NULL,
    store_type VARCHAR(50),
    address VARCHAR(300),
    city VARCHAR(100),
    region VARCHAR(100),
    country VARCHAR(100),
    postal_code VARCHAR(20),
    phone VARCHAR(20),
    manager_name VARCHAR(200),
    opening_date DATE,
    square_meters INTEGER,
    is_active BOOLEAN DEFAULT TRUE
);
```

Зоряна схема має кілька важливих характеристик. Вона проста для розуміння бізнес-користувачами, оскільки відображає природний спосіб мислення про бізнес-процеси. Запити виконуються швидко завдяки мінімальній кількості з'єднань. Денормалізована структура вимірів полегшує написання запитів та підвищує продуктивність читання. Однак ця схема може містити деяку надмірність у таблицях вимірів та вимагає більше місця для зберігання порівняно з нормалізованими схемами.

**Схема сніжинки (Snowflake Schema)** є варіацією зоряної схеми, де таблиці вимірів нормалізовані шляхом розбиття їх на додаткові таблиці. Це зменшує надмірність даних, але ускладнює структуру та може уповільнити запити через необхідність додаткових з'єднань.

**Приклад нормалізації виміру продукту у схему сніжинки:**

```sql
-- Основна таблиця виміру продукту
CREATE TABLE dim_product (
    product_key INTEGER PRIMARY KEY,
    product_id VARCHAR(50) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    product_description TEXT,
    subcategory_key INTEGER NOT NULL,
    brand_key INTEGER NOT NULL,
    supplier_key INTEGER NOT NULL,
    unit_cost DECIMAL(10,2),
    valid_from DATE NOT NULL,
    valid_to DATE NOT NULL,
    is_current BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (subcategory_key) REFERENCES dim_subcategory(subcategory_key),
    FOREIGN KEY (brand_key) REFERENCES dim_brand(brand_key),
    FOREIGN KEY (supplier_key) REFERENCES dim_supplier(supplier_key)
);

-- Таблиця підкатегорій
CREATE TABLE dim_subcategory (
    subcategory_key INTEGER PRIMARY KEY,
    subcategory_name VARCHAR(100) NOT NULL,
    category_key INTEGER NOT NULL,
    FOREIGN KEY (category_key) REFERENCES dim_category(category_key)
);

-- Таблиця категорій
CREATE TABLE dim_category (
    category_key INTEGER PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL,
    department_key INTEGER NOT NULL,
    FOREIGN KEY (department_key) REFERENCES dim_department(department_key)
);

-- Таблиця відділів
CREATE TABLE dim_department (
    department_key INTEGER PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL
);

-- Таблиця брендів
CREATE TABLE dim_brand (
    brand_key INTEGER PRIMARY KEY,
    brand_name VARCHAR(100) NOT NULL,
    country_of_origin VARCHAR(100)
);

-- Таблиця постачальників
CREATE TABLE dim_supplier (
    supplier_key INTEGER PRIMARY KEY,
    supplier_name VARCHAR(200) NOT NULL,
    contact_name VARCHAR(200),
    phone VARCHAR(20),
    email VARCHAR(100),
    address VARCHAR(300),
    city VARCHAR(100),
    country VARCHAR(100)
);
```

Вибір між зоряною схемою та сніжинкою залежить від конкретних вимог проєкту. Зоряна схема зазвичай надає кращу продуктивність запитів та простіше у супроводі, тому є найпоширенішим вибором для більшості сховищ даних. Схема сніжинки може бути доречною, коли важливо мінімізувати надмірність даних або коли ієрархії вимірів дуже складні та часто змінюються.

## OLAP операції та багатовимірний аналіз

### Концепція куба даних

Багатовимірна модель даних представляє інформацію у вигляді куба, де кожен вимір відповідає певній характеристиці аналізу, а комірки куба містять вимірювані показники або метрики.

**Візуалізація тривимірного куба продажів:**

```mermaid
graph TB
    subgraph Куб_Продажів
        A[Продукт: Ноутбуки<br/>Регіон: Київ<br/>Час: 2024 Q1<br/>Продажі: 1,250,000 грн]
        B[Продукт: Ноутбуки<br/>Регіон: Львів<br/>Час: 2024 Q1<br/>Продажі: 850,000 грн]
        C[Продукт: Телефони<br/>Регіон: Київ<br/>Час: 2024 Q1<br/>Продажі: 2,100,000 грн]
    end

    D[Виміри:<br/>- Продукт<br/>- Регіон<br/>- Час] --> A
    D --> B
    D --> C

    E[Метрики:<br/>- Обсяг продажів<br/>- Кількість одиниць<br/>- Прибуток] --> A
```

Для аналізу продажів можна створити куб з вимірами Продукт, Час та Регіон. Кожна комірка куба буде містити показники, такі як обсяг продажів, кількість проданих одиниць, прибуток. Наприклад, комірка для Ноутбуків, проданих у Києві у першому кварталі 2024 року, міститиме відповідні значення метрик.

### Основні OLAP операції

OLAP системи підтримують набір стандартних операцій, які дозволяють аналітикам інтерактивно досліджувати дані з різних перспектив.

**Drill-Down (деталізація)** — операція переходу від більш узагальнених даних до більш детальних. Наприклад, від річних продажів до квартальних, від квартальних до місячних, від категорій продуктів до конкретних найменувань.

**Приклад послідовності drill-down запитів:**

```sql
-- Рівень 1: Річні продажі по регіонах
SELECT
    r.region_name,
    d.year_number,
    SUM(f.total_amount) as annual_sales
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_store s ON f.store_key = s.store_key
JOIN dim_region r ON s.region_key = r.region_key
WHERE d.year_number = 2024
GROUP BY r.region_name, d.year_number;

-- Рівень 2: Drill-down до квартальних продажів
SELECT
    r.region_name,
    d.year_number,
    d.quarter_number,
    d.quarter_name,
    SUM(f.total_amount) as quarterly_sales
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_store s ON f.store_key = s.store_key
JOIN dim_region r ON s.region_key = r.region_key
WHERE d.year_number = 2024
  AND r.region_name = 'Київська область'
GROUP BY r.region_name, d.year_number, d.quarter_number, d.quarter_name
ORDER BY d.quarter_number;

-- Рівень 3: Drill-down до місячних продажів
SELECT
    r.region_name,
    d.year_number,
    d.month_number,
    d.month_name,
    SUM(f.total_amount) as monthly_sales,
    COUNT(DISTINCT f.customer_key) as unique_customers,
    COUNT(DISTINCT f.sale_id) as transaction_count
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_store s ON f.store_key = s.store_key
JOIN dim_region r ON s.region_key = r.region_key
WHERE d.year_number = 2024
  AND d.quarter_number = 1
  AND r.region_name = 'Київська область'
GROUP BY r.region_name, d.year_number, d.month_number, d.month_name
ORDER BY d.month_number;
```

**Roll-Up (узагальнення)** — операція, протилежна до drill-down, перехід від детальних даних до узагальнених. Наприклад, від денних продажів до тижневих, від окремих магазинів до регіонів, від продуктів до категорій.

**Приклад roll-up запиту:**

```sql
-- Узагальнення від магазинів до регіонів і від місяців до кварталів
SELECT
    r.region_name,
    d.year_number,
    d.quarter_name,
    COUNT(DISTINCT s.store_key) as store_count,
    SUM(f.quantity) as total_units_sold,
    SUM(f.total_amount) as total_revenue,
    AVG(f.total_amount) as avg_transaction_value
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_store s ON f.store_key = s.store_key
JOIN dim_region r ON s.region_key = r.region_key
WHERE d.year_number BETWEEN 2023 AND 2024
GROUP BY r.region_name, d.year_number, d.quarter_name
ORDER BY d.year_number, d.quarter_name, r.region_name;
```

**Slice (зріз)** — операція фіксації одного виміру на певному значенні та аналізу решти вимірів. Наприклад, аналіз продажів лише за січень 2024 року по всіх продуктах та регіонах, вимір часу зафіксовано на січні.

**Приклад slice операції:**

```sql
-- Зріз куба для конкретного місяця
SELECT
    p.category,
    r.region_name,
    SUM(f.quantity) as units_sold,
    SUM(f.total_amount) as revenue,
    SUM(f.profit_amount) as profit,
    ROUND(SUM(f.profit_amount) / SUM(f.total_amount) * 100, 2) as profit_margin
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_store s ON f.store_key = s.store_key
JOIN dim_region r ON s.region_key = r.region_key
WHERE d.year_number = 2024
  AND d.month_number = 1
GROUP BY p.category, r.region_name
ORDER BY revenue DESC;
```

**Dice (підкуб)** — операція вибору підкуба шляхом застосування умов до кількох вимірів одночасно. Наприклад, аналіз продажів ноутбуків та планшетів у Києві та Львові за перше півріччя 2024 року.

**Приклад dice операції:**

```sql
-- Вибір підкуба з обмеженнями по кількох вимірах
SELECT
    p.product_name,
    r.region_name,
    d.month_name,
    SUM(f.quantity) as units_sold,
    SUM(f.total_amount) as revenue
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_store s ON f.store_key = s.store_key
JOIN dim_region r ON s.region_key = r.region_key
WHERE d.year_number = 2024
  AND d.month_number BETWEEN 1 AND 6
  AND p.category IN ('Ноутбуки', 'Планшети')
  AND r.region_name IN ('Київська область', 'Львівська область')
GROUP BY p.product_name, r.region_name, d.month_name
ORDER BY revenue DESC;
```

**Pivot (поворот)** — операція зміни орієнтації куба, коли виміри міняються місцями для отримання іншої перспективи аналізу. Наприклад, від представлення продуктів по рядках та часу по стовпцях до часу по рядках та продуктів по стовпцях.

**Приклад pivot операції з використанням crosstab у PostgreSQL:**

```sql
-- Створення перехресної таблиці з продуктами по рядках та місяцями по стовпцях
SELECT * FROM crosstab(
  'SELECT
     p.category,
     d.month_name,
     SUM(f.total_amount)::bigint as revenue
   FROM fact_sales f
   JOIN dim_date d ON f.date_key = d.date_key
   JOIN dim_product p ON f.product_key = p.product_key
   WHERE d.year_number = 2024
     AND d.quarter_number = 1
   GROUP BY p.category, d.month_name, d.month_number
   ORDER BY 1, d.month_number',
  'SELECT DISTINCT month_name
   FROM dim_date
   WHERE year_number = 2024 AND quarter_number = 1
   ORDER BY month_number'
) AS pivot_table(
  category VARCHAR(100),
  january BIGINT,
  february BIGINT,
  march BIGINT
);
```

### Типи OLAP архітектур

Існує кілька підходів до реалізації OLAP функціональності, кожен з яких має свої переваги та недоліки.

**MOLAP (Multidimensional OLAP)** зберігає дані безпосередньо у багатовимірному форматі, зазвичай у спеціалізованих багатовимірних масивах. Цей підхід забезпечує найвищу швидкість аналізу завдяки попередньо обчисленим агрегатам. Дані компактно зберігаються завдяки спеціалізованим методам стиснення. Підтримуються складні обчислення та аналітичні функції. Однак MOLAP має обмеження щодо розміру даних, які можна ефективно обробляти, вимагає довгого часу на побудову кубів при великих обсягах даних, менш гнучкий у додаванні нових вимірів або показників.

**ROLAP (Relational OLAP)** працює безпосередньо з реляційною базою даних, динамічно генеруючи SQL запити для виконання аналітичних операцій. Основні переваги включають здатність працювати з великими обсягами даних без попереднього агрегування, гнучкість у додаванні нових вимірів та показників, використання існуючої інфраструктури реляційних СУБД. Недоліками є повільніші запити порівняно з MOLAP, особливо для складного аналізу, залежність продуктивності від оптимізації SQL запитів та індексування, вищі вимоги до ресурсів реляційної СУБД.

**HOLAP (Hybrid OLAP)** поєднує переваги обох підходів, зберігаючи агреговані дані у багатовимірному форматі для швидкого доступу, а детальні дані в реляційній базі. Це забезпечує швидкий аналіз для узагальнених даних та можливість деталізації при потребі. Недоліком є складність архітектури та управління двома типами сховищ.

## Колонково-орієнтовані СУБД для аналітики

### Принципи колонкового зберігання

Традиційні реляційні СУБД зберігають дані по рядках, коли всі значення одного запису розміщуються послідовно на диску. Це оптимально для транзакційних систем, де зазвичай потрібен доступ до всіх полів запису одночасно. Однак для аналітичних запитів, які часто працюють лише з кількома стовпцями з мільйонів записів, такий підхід неефективний.

Колонково-орієнтовані СУБД зберігають дані по стовпцях, де всі значення одного атрибута для всіх записів розміщуються послідовно. Це дає кілька важливих переваг для аналітичних навантажень.

**Порівняння рядкового та колонкового зберігання:**

```mermaid
graph TB
    subgraph Рядкове_зберігання
        R1[ID:1|Name:Petrov|Age:25|City:Kyiv|Salary:5000]
        R2[ID:2|Name:Ivanov|Age:30|City:Lviv|Salary:6000]
        R3[ID:3|Name:Sydorov|Age:28|City:Kyiv|Salary:5500]
    end

    subgraph Колонкове_зберігання
        C1[ID: 1|2|3]
        C2[Name: Petrov|Ivanov|Sydorov]
        C3[Age: 25|30|28]
        C4[City: Kyiv|Lviv|Kyiv]
        C5[Salary: 5000|6000|5500]
    end
```

Переваги колонкового зберігання для аналітики полягають у кількох аспектах. По-перше, ефективність вибірки даних, оскільки для запиту, який використовує лише кілька стовпців, зчитуються лише ці стовпці, а не всі дані таблиці. По-друге, краще стиснення даних, адже значення одного стовпця зазвичай мають подібний тип та розподіл, що дозволяє використовувати ефективніші алгоритми стиснення. По-третє, векторизоване виконання, коли сучасні процесори можуть обробляти масиви однотипних даних набагато швидше завдяки SIMD інструкціям. По-четверте, ефективна агрегація, оскільки обчислення сум, середніх, максимумів над стовпцем виконується швидше, коли дані розміщені послідовно.

### Популярні колонкові СУБД

**ClickHouse** — розроблена компанією Yandex відкрита колонкова СУБД, оптимізована для аналітичних запитів у реальному часі. Вона може обробляти мільярди записів за секунди завдяки колонковому зберіганню, векторизованому виконанню запитів, паралелізму на рівні ядер та серверів, ефективному стисненню даних.

**Приклад створення таблиці у ClickHouse:**

```sql
CREATE TABLE sales_analytics (
    date Date,
    product_id UInt32,
    product_name String,
    category String,
    region String,
    quantity UInt32,
    unit_price Decimal(10, 2),
    total_amount Decimal(12, 2)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (category, region, date, product_id);

-- Приклад аналітичного запиту
SELECT
    category,
    region,
    toStartOfMonth(date) as month,
    sum(quantity) as total_units,
    sum(total_amount) as revenue,
    avg(unit_price) as avg_price
FROM sales_analytics
WHERE date >= '2024-01-01'
GROUP BY category, region, month
ORDER BY revenue DESC;
```

**Amazon Redshift** — хмарне сховище даних на базі PostgreSQL з колонковим зберіганням, призначене для аналітики петабайтних обсягів даних. Redshift автоматично розподіляє дані та запити по кластеру вузлів, підтримує масове паралельне виконання запитів, інтегрується з екосистемою AWS.

**Google BigQuery** — повністю керований serverless сервіс для аналітики великих даних, який автоматично масштабується залежно від навантаження, підтримує SQL запити до петабайтів даних за секунди, має вбудовану інтеграцію з машинним навчанням, працює за моделлю оплати за використання.

**Apache Druid** — орієнтована на роботу у реальному часі колонкова СУБД, призначена для швидкого запиту та аналізу подій, що надходять безперервним потоком. Druid оптимізована для часових рядів даних, підтримує sub-second запити до великих обсягів історичних даних, забезпечує горизонтальне масштабування та високу доступність.

Вибір колонкової СУБД залежить від конкретних вимог проєкту, таких як обсяги даних, частота оновлення, складність запитів, бюджет та інфраструктура.

## Інструменти бізнес-інтелекту та візуалізації

### Екосистема BI інструментів

Сховища даних та OLAP системи є фундаментом для побудови рішень бізнес-інтелекту, але кінцеві користувачі взаємодіють з даними через спеціалізовані інструменти візуалізації та аналітики.

**Tableau** — один з найпопулярніших інструментів візуалізації даних, що надає інтуїтивно зрозумілий інтерфейс для створення інтерактивних дашбордів. Tableau підтримує підключення до десятків джерел даних, включаючи реляційні та NoSQL бази, хмарні сервіси, файли різних форматів. Користувачі можуть створювати складні візуалізації просто перетягуючи поля на полотно, без написання коду.

**Power BI** від Microsoft — комплексна платформа бізнес-аналітики, що тісно інтегрується з екосистемою Microsoft. Power BI включає можливості підготовки даних, моделювання, візуалізації та публікації звітів. Десктопна версія безкоштовна, хмарний сервіс працює за моделлю підписки. Особливо ефективний для організацій, які вже використовують продукти Microsoft.

**Apache Superset** — відкритий інструмент візуалізації даних та дослідження, розроблений Airbnb. Superset підтримує широкий спектр типів візуалізацій, від простих графіків до складних геопросторових карт. Має вбудовану підтримку для SQL запитів та підключення до більшості популярних баз даних.

**Metabase** — відкритий інструмент для створення запитань та дашбордів, який акцентує на простоті використання. Metabase дозволяє бізнес-користувачам створювати запити без знання SQL через графічний конструктор, автоматично генерує SQL код, підтримує планування автоматичного оновлення звітів та розсилку результатів.

### Приклад інтеграції аналітичної системи

Типова архітектура сучасної аналітичної системи інтегрує кілька компонентів у єдиний потік обробки даних.

```mermaid
graph LR
    A[Операційні БД<br/>PostgreSQL, MySQL] --> B[ETL/ELT<br/>Airflow, dbt]
    C[API та зовнішні<br/>джерела] --> B
    D[Файли та логи<br/>CSV, JSON] --> B

    B --> E[Сховище даних<br/>Snowflake, BigQuery]

    E --> F[BI інструменти<br/>Tableau, Power BI]
    E --> G[Ad-hoc аналіз<br/>SQL клієнти]
    E --> H[ML платформи<br/>Python, R]

    F --> I[Керівники та<br/>менеджери]
    G --> J[Аналітики]
    H --> K[Data Scientists]
```

Розглянемо конкретний приклад інтеграції з використанням популярних відкритих інструментів.

**Налаштування ETL процесу з Apache Airflow:**

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime, timedelta
import pandas as pd

default_args = {
    'owner': 'analytics_team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'daily_sales_etl',
    default_args=default_args,
    description='Щоденне завантаження даних продажів',
    schedule_interval='0 2 * * *',
    catchup=False
)

def extract_sales_data(**context):
    # Вилучення даних з операційної БД
    from sqlalchemy import create_engine

    source_engine = create_engine('postgresql://user:pass@oltp_host/sales_db')

    query = """
        SELECT
            o.order_id,
            o.order_date,
            o.customer_id,
            c.customer_name,
            c.segment,
            oi.product_id,
            p.product_name,
            p.category,
            oi.quantity,
            oi.unit_price,
            oi.discount
        FROM orders o
        JOIN customers c ON o.customer_id = c.customer_id
        JOIN order_items oi ON o.order_id = oi.order_id
        JOIN products p ON oi.product_id = p.product_id
        WHERE o.order_date >= CURRENT_DATE - INTERVAL '1 day'
    """

    df = pd.read_sql(query, source_engine)
    df.to_csv('/tmp/daily_sales.csv', index=False)

    return len(df)

def transform_sales_data(**context):
    # Трансформація даних
    df = pd.read_csv('/tmp/daily_sales.csv')

    # Обчислення похідних полів
    df['total_amount'] = df['quantity'] * df['unit_price']
    df['discount_amount'] = df['total_amount'] * df['discount'] / 100
    df['net_amount'] = df['total_amount'] - df['discount_amount']

    # Додавання часових вимірів
    df['order_date'] = pd.to_datetime(df['order_date'])
    df['year'] = df['order_date'].dt.year
    df['quarter'] = df['order_date'].dt.quarter
    df['month'] = df['order_date'].dt.month
    df['day_of_week'] = df['order_date'].dt.dayofweek

    df.to_csv('/tmp/transformed_sales.csv', index=False)

def load_to_warehouse(**context):
    # Завантаження у сховище даних
    from sqlalchemy import create_engine

    dwh_engine = create_engine('postgresql://user:pass@dwh_host/warehouse')

    df = pd.read_csv('/tmp/transformed_sales.csv')
    df.to_sql(
        'fact_sales_daily',
        dwh_engine,
        if_exists='append',
        index=False,
        method='multi'
    )

extract_task = PythonOperator(
    task_id='extract_sales',
    python_callable=extract_sales_data,
    dag=dag
)

transform_task = PythonOperator(
    task_id='transform_sales',
    python_callable=transform_sales_data,
    dag=dag
)

load_task = PythonOperator(
    task_id='load_to_warehouse',
    python_callable=load_to_warehouse,
    dag=dag
)

# Послідовність виконання
extract_task >> transform_task >> load_task
```

### Створення аналітичних звітів

Після завантаження даних у сховище аналітики можуть створювати різноманітні звіти та дашборди для підтримки бізнес-рішень.

**Приклад SQL запиту для дашборду продажів:**

```sql
-- Ключові показники ефективності за останній місяць
WITH current_month AS (
    SELECT
        COUNT(DISTINCT order_id) as orders_count,
        COUNT(DISTINCT customer_key) as customers_count,
        SUM(total_amount) as revenue,
        SUM(profit_amount) as profit,
        AVG(total_amount) as avg_order_value
    FROM fact_sales
    WHERE date_key >= (
        SELECT date_key FROM dim_date
        WHERE full_date = DATE_TRUNC('month', CURRENT_DATE)
    )
),
previous_month AS (
    SELECT
        COUNT(DISTINCT order_id) as orders_count,
        COUNT(DISTINCT customer_key) as customers_count,
        SUM(total_amount) as revenue,
        SUM(profit_amount) as profit,
        AVG(total_amount) as avg_order_value
    FROM fact_sales
    WHERE date_key >= (
        SELECT date_key FROM dim_date
        WHERE full_date = DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
    )
    AND date_key < (
        SELECT date_key FROM dim_date
        WHERE full_date = DATE_TRUNC('month', CURRENT_DATE)
    )
)
SELECT
    'Кількість замовлень' as metric,
    cm.orders_count as current_value,
    pm.orders_count as previous_value,
    ROUND((cm.orders_count - pm.orders_count) * 100.0 / pm.orders_count, 2) as growth_percent
FROM current_month cm, previous_month pm
UNION ALL
SELECT
    'Виручка',
    cm.revenue,
    pm.revenue,
    ROUND((cm.revenue - pm.revenue) * 100.0 / pm.revenue, 2)
FROM current_month cm, previous_month pm
UNION ALL
SELECT
    'Прибуток',
    cm.profit,
    pm.profit,
    ROUND((cm.profit - pm.profit) * 100.0 / pm.profit, 2)
FROM current_month cm, previous_month pm
UNION ALL
SELECT
    'Середній чек',
    cm.avg_order_value,
    pm.avg_order_value,
    ROUND((cm.avg_order_value - pm.avg_order_value) * 100.0 / pm.avg_order_value, 2)
FROM current_month cm, previous_month pm;
```

## Висновки

Системи підтримки прийняття рішень та OLAP технології є невід'ємною частиною сучасної інформаційної інфраструктури організацій. Розуміння фундаментальних відмінностей між операційними та аналітичними системами дозволяє правильно проєктувати архітектуру інформаційних систем, виділяючи окремі компоненти для різних типів навантажень.

Сховища даних забезпечують інтеграцію інформації з різноманітних джерел, створюючи єдине джерело істини для аналітичної роботи. ETL процеси є критично важливими для забезпечення якості та узгодженості даних у сховищі. Багатовимірне моделювання з використанням зоряних схем або схем сніжинок надає інтуїтивно зрозумілий спосіб організації даних навколо бізнес-процесів.

OLAP операції надають аналітикам потужні інструменти для інтерактивного дослідження даних з різних перспектив, виявлення закономірностей та тенденцій. Вибір між різними типами OLAP архітектур залежить від конкретних вимог щодо обсягів даних, швидкості відгуку та гнучкості системи.

Колонково-орієнтовані СУБД революціонізували можливості обробки великих обсягів аналітичних даних, забезпечуючи на порядки вищу продуктивність порівняно з традиційними рядковими системами для типових аналітичних навантажень. Сучасні інструменти бізнес-інтелекту роблять можливості аналізу даних доступними для широкого кола бізнес-користувачів, не вимагаючи глибоких технічних знань.

Успішна реалізація аналітичної системи вимагає не лише правильного вибору технологій, але й розуміння бізнес-процесів, чітко визначених метрик та показників, культури прийняття рішень на основі даних в організації. Інвестиції у побудову якісної аналітичної інфраструктури окупаються через покращення якості управлінських рішень, виявлення нових можливостей для бізнесу та підвищення ефективності операційної діяльності.
