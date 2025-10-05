# Презентація 15. Хмарні бази даних та Database-as-a-Service

## План лекції

1. Вступ до хмарних обчислень
2. Моделі хмарних сервісів
3. Database-as-a-Service (DBaaS)
4. Провідні хмарні платформи
5. Міграція до хмари

## **☁️ Основні поняття:**

**Хмарні обчислення** — надання обчислювальних ресурсів через інтернет на основі оплати за використання.

**DBaaS** (Database as a Service) — керована база даних як сервіс у хмарі.

**Serverless** — модель виконання, де провайдер автоматично керує інфраструктурою.

**Multi-cloud** — використання кількох хмарних провайдерів одночасно.

## **1. Вступ до хмарних обчислень**

## Еволюція інфраструктури БД

### 📜 **Історичний розвиток:**

```mermaid
timeline
    title Еволюція інфраструктури баз даних
    1980-2000 : On-Premises
              : Власні сервери
              : Повний контроль
    2000-2010 : Віртуалізація
              : VMware, Hyper-V
              : Кращe використання ресурсів
    2010-2015 : IaaS
              : AWS EC2, Azure VMs
              : Еластичність
    2015-2020 : DBaaS
              : RDS, Cloud SQL
              : Керовані сервіси
    2020-сьогодні : Serverless
                  : Aurora Serverless
                  : Автомасштабування
```

## On-Premises vs Хмара

### 🏢 **Традиційний підхід:**

**On-Premises:**
- Власні сервери в дата-центрі
- Повний контроль над обладнанням
- Капітальні інвестиції (CapEx)
- Власна команда адміністрування

```mermaid
graph TB
    A[🏢 КОМПАНІЯ] --> B[Серверна кімната]
    B --> C[🖥️ Сервери БД]
    B --> D[🔌 Мережеве обладнання]
    B --> E[❄️ Охолодження]
    B --> F[⚡ ДБЖ]

    G[👨‍💼 DBA] --> C
    H[👨‍🔧 Сисадмін] --> D
    I[🔧 Інженер] --> E
```

### ✅ **Переваги:** Повний контроль, дані на місці
### ❌ **Недоліки:** Висока вартість, складність масштабування

## Переваги хмарних БД

### ☁️ **Чому хмара?**

**Економічні переваги:**
- 💰 **OpEx замість CapEx** — оплата за використання
- 📉 **Зниження TCO** — немає витрат на обладнання
- 🎯 **Прогнозовані витрати** — оплата за ресурси

**Технічні переваги:**
- ⚡ **Швидкий запуск** — БД за хвилини
- 📈 **Еластичність** — автомасштабування
- 🌍 **Глобальне покриття** — регіони по всьому світу
- 🔄 **Автоматичні бекапи** — вбудовані механізми
- 🛡️ **Відмовостійкість** — SLA 99.95-99.99%

**Operational переваги:**
- 🤖 **Автоматичні оновлення** — патчі та версії
- 📊 **Вбудований моніторинг** — метрики з коробки
- 🔒 **Безпека** — шифрування, compliance

## Недоліки хмарних БД

### ⚠️ **Виклики та обмеження:**

**Залежність від провайдера (Vendor Lock-in):**
- Специфічні API та сервіси
- Складність міграції
- Прив'язка до екосистеми

**Проблеми контролю:**
- Обмежений доступ до ОС
- Неможливість тонкого налаштування
- Залежність від провайдера

**Вартість:**
- Може бути дорожче для стабільного навантаження
- Непередбачувані витрати при сплесках
- Витрати на трафік

**Латентність:**
- Мережеві затримки
- Географічні обмеження

## **2. Моделі хмарних сервісів**

## IaaS, PaaS, SaaS

### 🏗️ **Три рівні хмарних сервісів:**

```mermaid
graph TB
    subgraph "On-Premises"
        A1[Застосунок]
        A2[Дані]
        A3[Runtime]
        A4[Middleware]
        A5[ОС]
        A6[Віртуалізація]
        A7[Сервери]
        A8[Сховище]
        A9[Мережа]
    end

    subgraph "IaaS"
        B1[Застосунок]
        B2[Дані]
        B3[Runtime]
        B4[Middleware]
        B5[ОС]
        B6[Віртуалізація ☁️]
        B7[Сервери ☁️]
        B8[Сховище ☁️]
        B9[Мережа ☁️]
    end

    subgraph "PaaS"
        C1[Застосунок]
        C2[Дані]
        C3[Runtime ☁️]
        C4[Middleware ☁️]
        C5[ОС ☁️]
        C6[Віртуалізація ☁️]
        C7[Сервери ☁️]
        C8[Сховище ☁️]
        C9[Мережа ☁️]
    end

    subgraph "SaaS"
        D1[Застосунок ☁️]
        D2[Дані ☁️]
        D3[Runtime ☁️]
        D4[Middleware ☁️]
        D5[ОС ☁️]
        D6[Віртуалізація ☁️]
        D7[Сервери ☁️]
        D8[Сховище ☁️]
        D9[Мережа ☁️]
    end
```

## IaaS — Infrastructure as a Service

### 🖥️ **Віртуальна інфраструктура:**

**Що надається:**
- Віртуальні машини
- Сховище даних
- Мережева інфраструктура
- Балансувальники навантаження

**Приклади для БД:**
- AWS EC2 + EBS
- Azure Virtual Machines
- Google Compute Engine

**Використання:**
```bash
# Запуск PostgreSQL на AWS EC2
aws ec2 run-instances \
    --image-id ami-12345678 \
    --instance-type t3.large \
    --key-name mykey \
    --security-groups postgres-sg

# Підключення та встановлення
ssh -i mykey.pem ec2-user@instance-ip
sudo yum install postgresql14-server
sudo postgresql-setup initdb
sudo systemctl start postgresql
```

### ✅ **Переваги:** Гнучкість, повний контроль
### ❌ **Недоліки:** Потребує адміністрування

## PaaS — Platform as a Service

### 🎯 **Керована платформа:**

**Що надається:**
- Керована БД
- Автоматичні бекапи
- Масштабування
- Моніторинг та алерти
- Патчі та оновлення

**Приклади:**
- AWS RDS (PostgreSQL, MySQL, MariaDB)
- Azure Database for PostgreSQL
- Google Cloud SQL
- Heroku Postgres

**Використання:**
```bash
# Створення PostgreSQL на AWS RDS
aws rds create-db-instance \
    --db-instance-identifier mydb \
    --db-instance-class db.t3.medium \
    --engine postgres \
    --master-username admin \
    --master-user-password secret123 \
    --allocated-storage 100 \
    --backup-retention-period 7 \
    --multi-az
```

### ✅ **Переваги:** Менше адміністрування, швидкий старт
### ❌ **Недоліки:** Менше контролю, vendor lock-in

## DBaaS — Database as a Service

### 🗄️ **Спеціалізовані сервіси БД:**

**Повністю керовані БД:**
- Автоматичне масштабування
- Вбудована реплікація
- Point-in-time recovery
- Глобальний розподіл
- Автоматичні патчі

**Приклади:**
- Amazon Aurora (MySQL/PostgreSQL сумісний)
- Google Cloud Spanner
- Azure Cosmos DB
- MongoDB Atlas
- PlanetScale (MySQL)

**Особливості Aurora Serverless:**
- Автоматичне вмикання/вимикання
- Масштабування за секунди
- Оплата за використання (ACU - Aurora Capacity Units)

## Порівняння моделей для БД

| Модель | Контроль | Адміністрування | Масштабування | Вартість |
|--------|----------|-----------------|---------------|----------|
| **IaaS** | 🟢 Високий | 🔴 Повне | 🟡 Ручне | 💰 Низька |
| **PaaS** | 🟡 Середній | 🟡 Часткове | 🟢 Керовані | 💰💰 Середня |
| **DBaaS** | 🔴 Обмежений | 🟢 Мінімальне | 🟢 Автоматичне | 💰💰💰 Висока |

**Вибір залежить від:**
- Розміру команди
- Експертизи
- Бюджету
- Вимог до контролю

## **3. Database-as-a-Service (DBaaS)**

## Amazon RDS

### 🔶 **Керовані реляційні БД від AWS:**

**Підтримувані движки:**
- PostgreSQL
- MySQL
- MariaDB
- Oracle
- SQL Server
- Amazon Aurora

**Ключові можливості:**
- 🔄 **Automated backups** — до 35 днів
- 🌍 **Multi-AZ deployment** — відмовостійкість
- 📖 **Read replicas** — масштабування читання
- 📊 **Performance Insights** — аналіз продуктивності
- 🔒 **Encryption** — at rest and in transit

## Архітектура RDS Multi-AZ

### 🏗️ **Високодоступна конфігурація:**

```mermaid
graph TB
    A[🌐 Застосунок] --> B[📍 RDS Endpoint]

    B --> C[🎯 PRIMARY INSTANCE<br/>AZ-A]
    C -.->|Синхронна<br/>реплікація| D[📋 STANDBY INSTANCE<br/>AZ-B]

    C --> E[(💾 EBS Storage<br/>AZ-A)]
    D --> F[(💾 EBS Storage<br/>AZ-B)]

    G[⚠️ Failure Detection] -.->|Автоматичний<br/>failover| D
    G --> C
```

**Failover процес:**
1. Виявлення збою (30-120 секунд)
2. Автоматичне перемикання DNS
3. Standby стає Primary
4. Відновлення нового Standby

**RTO:** ~60-120 секунд
**RPO:** ~0 (синхронна реплікація)

## Amazon Aurora

### ⚡ **MySQL/PostgreSQL-сумісний движок від AWS:**

**Інновації Aurora:**
- 🚀 **До 5x швидше** за MySQL
- 🌍 **Глобальні бази даних** — cross-region реплікація
- 🔄 **До 15 read replicas** — низька латентність
- 💾 **Автомасштабування сховища** — до 128 TB
- ⚡ **Швидке відновлення** — crash recovery < 60 сек

**Архітектура сховища:**
```mermaid
graph TB
    A[Aurora Instance] --> B[Віртуальний<br/>Storage Layer]

    B --> C[AZ 1<br/>Копія 1-2]
    B --> D[AZ 2<br/>Копія 3-4]
    B --> E[AZ 3<br/>Копія 5-6]

    C --> F[10GB Segment]
    D --> G[10GB Segment]
    E --> H[10GB Segment]
```

**Особливості:** 6 копій даних у 3 AZ, кворум для запису (4/6)

## Aurora Serverless

### 🌟 **Автомасштабування на попит:**

**Можливості:**
- 💤 **Auto-pause** — вимикається при відсутності активності
- 📈 **Auto-scaling** — від 0.5 ACU до 128 ACU
- 💰 **Pay per second** — оплата за фактичне використання
- 🔌 **Data API** — HTTP endpoint для запитів

**Ідеальні сценарії:**
- Нерегулярне навантаження
- Розробка та тестування
- Нові застосунки з невідомим навантаженням
- Періодичні звіти та аналітика

**Приклад налаштування:**
```yaml
MinCapacity: 2 ACU
MaxCapacity: 16 ACU
AutoPause: true
SecondsUntilAutoPause: 300
```

## Google Cloud SQL

### 🔵 **Керовані БД від Google:**

**Підтримка:**
- PostgreSQL
- MySQL
- SQL Server

**Ключові можливості:**
- 🌍 **High Availability** — 99.95% SLA
- 📖 **Read replicas** — до 10 реплік
- 🔄 **Автоматичне резервне копіювання**
- 🔧 **Автоматичне масштабування сховища**
- 🔒 **Customer-managed encryption keys**

**Особливості:**
- Інтеграція з GCP екосистемою
- BigQuery Federation
- Cloud SQL Insights для аналізу

## Google Cloud Spanner

### 🌐 **Глобально розподілена SQL БД:**

**Унікальні характеристики:**
- 🌍 **Глобальна консистентність** — ACID транзакції
- 📈 **Горизонтальне масштабування** — петабайти даних
- 🎯 **99.999% доступність** — multi-region
- ⏱️ **TrueTime API** — синхронізація часу

**Архітектура:**
```mermaid
graph TB
    A[Глобальний<br/>Cloud Spanner] --> B[Region: US]
    A --> C[Region: EU]
    A --> D[Region: Asia]

    B --> E[Zone 1]
    B --> F[Zone 2]
    B --> G[Zone 3]

    E & F & G --> H[Paxos<br/>Consensus]
```

**Використання:**
- Глобальні фінансові системи
- Gaming leaderboards
- Retail inventory

## Azure Database Services

### 🔷 **Пропозиції Microsoft Azure:**

**Azure Database for PostgreSQL:**
- Single Server
- Flexible Server (новіша версія)
- Hyperscale (Citus) — шардинг

**Azure SQL Database:**
- SQL Server у хмарі
- Elastic pools
- Serverless tier

**Azure Cosmos DB:**
- Multi-model (документи, графи, key-value)
- Глобальний розподіл
- 5 рівнів консистентності

**Особливості Azure:**
- Інтеграція з Active Directory
- Azure Monitor
- Defender for Cloud

## MongoDB Atlas

### 🍃 **DBaaS для MongoDB:**

**Можливості:**
- 🌍 **Multi-cloud** — AWS, Azure, GCP
- 📈 **Автомасштабування** — compute та storage
- 🔍 **Atlas Search** — повнотекстовий пошук
- 📊 **Performance Advisor** — рекомендації
- 🔄 **Continuous backup** — point-in-time recovery

**Рівні сервісу:**
- **Shared** — безкоштовний/початковий
- **Dedicated** — виділені кластери
- **Serverless** — оплата за операції

**Приклад підключення:**
```javascript
const { MongoClient } = require('mongodb');

const uri = "mongodb+srv://user:pass@cluster.mongodb.net/";
const client = new MongoClient(uri);

await client.connect();
const database = client.db('mydb');
const collection = database.collection('users');
```

## Порівняння провайдерів

| Провайдер | Сильні сторони | Популярні сервіси |
|-----------|----------------|-------------------|
| **AWS** | Найбільший вибір, зрілість | Aurora, RDS, DynamoDB |
| **Google Cloud** | Інновації, BigData | Cloud Spanner, BigQuery |
| **Azure** | Інтеграція з MS, hybrid | Cosmos DB, SQL Database |
| **MongoDB Atlas** | Expertise в NoSQL | Managed MongoDB |

**Вибір провайдера залежить від:**
- Існуючої екосистеми
- Регіональних вимог
- Спеціалізації команди
- Бюджету

## **4. Провідні хмарні платформи**

## AWS для баз даних

### 🔶 **Екосистема Amazon Web Services:**

**Реляційні БД:**
- **RDS** — керовані БД (PostgreSQL, MySQL, etc.)
- **Aurora** — високопродуктивний движок
- **Redshift** — аналітичне сховище даних

**NoSQL:**
- **DynamoDB** — key-value, документи
- **DocumentDB** — MongoDB-сумісний
- **Keyspaces** — Cassandra-сумісний

**Спеціалізовані:**
- **Neptune** — графова БД
- **Timestream** — часові ряди
- **QLDB** — Quantum Ledger (immutable)

## DynamoDB — приклад

### ⚡ **Serverless NoSQL від AWS:**

**Характеристики:**
- Мілісекундна латентність
- Автоматичне масштабування
- Вбудована реплікація
- Global Tables (multi-region)

**Приклад використання:**
```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Запис
table.put_item(
    Item={
        'user_id': '12345',
        'name': 'John Doe',
        'email': 'john@example.com',
        'created_at': '2024-10-05'
    }
)

# Читання
response = table.get_item(Key={'user_id': '12345'})
user = response['Item']

# Query з індексом
response = table.query(
    IndexName='email-index',
    KeyConditionExpression=Key('email').eq('john@example.com')
)
```

## Google Cloud Platform для БД

### 🔵 **Пропозиції GCP:**

**Керовані SQL:**
- **Cloud SQL** — PostgreSQL, MySQL, SQL Server
- **Cloud Spanner** — глобальна SQL БД
- **AlloyDB** — PostgreSQL-сумісний (новий)

**NoSQL та аналітика:**
- **Firestore** — документна БД
- **Bigtable** — wide-column store
- **BigQuery** — serverless data warehouse

**Особливості GCP:**
- Висока продуктивність мережі
- Інтеграція з ML/AI сервісами
- Сильна аналітика (BigQuery)

## BigQuery

### 📊 **Serverless аналітичне сховище:**

**Можливості:**
- Петабайтне масштабування
- SQL-подібна мова запитів
- Вбудоване ML (BigQuery ML)
- Інтеграція з Looker, Data Studio

**Приклад аналізу:**
```sql
-- Аналіз мільйонів записів за секунди
SELECT
    country,
    COUNT(*) as users,
    AVG(order_amount) as avg_order,
    SUM(order_amount) as total_revenue
FROM `project.dataset.orders`
WHERE order_date >= '2024-01-01'
GROUP BY country
ORDER BY total_revenue DESC
LIMIT 10;

-- ML безпосередньо в SQL
CREATE MODEL `project.dataset.churn_model`
OPTIONS(model_type='logistic_reg') AS
SELECT
    features.*,
    churned as label
FROM `project.dataset.customer_features` as features;
```

## Microsoft Azure для БД

### 🔷 **Платформа Azure:**

**SQL сервіси:**
- **Azure SQL Database** — PaaS SQL Server
- **Azure Database for PostgreSQL/MySQL**
- **Azure Synapse Analytics** — аналітика

**NoSQL:**
- **Cosmos DB** — multi-model БД
- **Table Storage** — key-value сховище

**Унікальні можливості:**
- Гібридні сценарії (on-prem + cloud)
- Azure Arc для управління всюди
- Глибока інтеграція з .NET та MS стеком

## Azure Cosmos DB

### 🌌 **Глобально розподілена multi-model БД:**

**5 рівнів консистентності:**
```mermaid
graph LR
    A[Strong] --> B[Bounded<br/>Staleness]
    B --> C[Session]
    C --> D[Consistent<br/>Prefix]
    D --> E[Eventual]

    F[Висока<br/>консистентність] -.-> A
    G[Низька<br/>латентність] -.-> E
```

**API підтримка:**
- SQL API (документи)
- MongoDB API
- Cassandra API
- Gremlin API (графи)
- Table API

**Глобальний розподіл:**
- Реплікація в кілька регіонів
- Multi-region writes
- Автоматичний failover

## **5. Міграція до хмари**

## Стратегії міграції

### 🚀 **6R стратегій:**

**1. Rehost (Lift and Shift):**
- Переміщення без змін
- Найшвидша міграція
- Мінімальна оптимізація

**2. Replatform (Lift, Tinker and Shift):**
- Незначні оптимізації
- Використання керованих сервісів
- Баланс швидкості та вигоди

**3. Refactor/Re-architect:**
- Переписування для хмари
- Максимальна оптимізація
- Найдовша міграція

**4. Repurchase:**
- Перехід на SaaS
- Заміна існуючого рішення

**5. Retire:**
- Виведення з експлуатації

**6. Retain:**
- Залишити on-premises

## Процес міграції БД

### 📋 **Покрокова міграція:**

```mermaid
graph TB
    A[1️⃣ Оцінка] --> B[2️⃣ Планування]
    B --> C[3️⃣ Підготовка]
    C --> D[4️⃣ Міграція даних]
    D --> E[5️⃣ Тестування]
    E --> F[6️⃣ Cutover]
    F --> G[7️⃣ Оптимізація]

    A -.->|Вибір стратегії| H[AWS DMS<br/>Azure Migrate<br/>GCP Database<br/>Migration Service]
```

**1. Оцінка:**
- Інвентаризація БД
- Аналіз залежностей
- Оцінка складності

**2. Планування:**
- Вибір цільової платформи
- Визначення downtime вікна
- План відкату

## AWS Database Migration Service

### 🔄 **Інструмент міграції від AWS:**

**Можливості:**
- Гомогенна міграція (Oracle → Oracle)
- Гетерогенна міграція (Oracle → PostgreSQL)
- Continuous replication
- Мінімальний downtime

**Підтримувані джерела:**
- Oracle, SQL Server, MySQL, PostgreSQL
- MongoDB, SAP, DB2
- S3, Amazon RDS

**Приклад процесу:**
```mermaid
sequenceDiagram
    participant S as Source DB<br/>(On-Prem)
    participant DMS as AWS DMS
    participant T as Target DB<br/>(RDS)

    S->>DMS: Full Load
    DMS->>T: Initial data copy
    S->>DMS: CDC (Change Data Capture)
    DMS->>T: Continuous replication
    Note over S,T: Applications cutover
    T->>T: Production traffic
```

## Schema Conversion Tool

### 🔧 **AWS SCT для конвертації схем:**

**Функції:**
- Автоматична конвертація DDL
- Аналіз складності міграції
- Звіти про несумісність
- Рекомендації з оптимізації

**Підтримувані конвертації:**
- Oracle → PostgreSQL/Aurora
- SQL Server → PostgreSQL/Aurora
- MySQL → PostgreSQL

**Workflow:**
```bash
# 1. Аналіз вихідної БД
aws sct analyze-source-database

# 2. Конвертація схеми
aws sct convert-schema

# 3. Застосування до цільової БД
aws sct apply-schema

# 4. Генерація звіту
aws sct generate-assessment-report
```

## Тестування після міграції

### ✅ **Перевірка коректності:**

**Функціональне тестування:**
- Порівняння кількості записів
- Валідація даних
- Тестування застосунку

**Навантажувальне тестування:**
```python
import time
from locust import HttpUser, task, between

class DatabaseLoadTest(HttpUser):
    wait_time = between(1, 3)

    @task
    def read_query(self):
        self.client.get("/api/users/12345")

    @task(3)
    def complex_query(self):
        self.client.post("/api/analytics/sales",
            json={"period": "monthly"})

    @task
    def write_query(self):
        self.client.post("/api/orders",
            json={"amount": 99.99})

# Запуск
# locust -f load_test.py --host=https://new-db.example.com
```

**Порівняння продуктивності:**
- Response time (P50, P95, P99)
- Throughput (TPS, QPS)
- Resource utilization

## Cutover стратегії

### 🔀 **Перемикання на хмарну БД:**

**Big Bang Cutover:**
```mermaid
graph LR
    A[On-Prem DB<br/>100% traffic] -->|Downtime<br/>window| B[Cloud DB<br/>100% traffic]
```
- Швидко, але ризиковано
- Потрібен downtime

**Phased Cutover:**
```mermaid
graph TB
    A[On-Prem DB] -->|90%| C[Apps]
    B[Cloud DB] -->|10%| C

    D[On-Prem DB] -->|50%| E[Apps]
    F[Cloud DB] -->|50%| E

    G[On-Prem DB] -->|0%| H[Apps]
    I[Cloud DB] -->|100%| H
```
- Поступово, безпечніше
- Без downtime

**Blue-Green Deployment:**
- Дві ідентичні середовища
- Миттєве перемикання
- Легкий rollback

## Оптимізація після міграції

### ⚡ **Налаштування для хмари:**

**Використання хмарних можливостей:**
```sql
-- Read replicas для масштабування
CREATE READ REPLICA my-db-replica-1
    FROM my-db-instance
    IN REGION eu-central-1;

-- Автомасштабування сховища
ALTER DATABASE SET storage_autoscaling = ON
    WITH max_storage = 1000GB;

-- Performance Insights
ENABLE PERFORMANCE_INSIGHTS
    WITH retention_period = 7;
```

**Моніторинг та оптимізація:**
- CloudWatch metrics
- Slow query logs
- Performance schema
- Cost optimization (right-sizing)

## Виклики міграції

### ⚠️ **Типові проблеми:**

**Технічні:**
- Несумісність версій
- Різниці в SQL діалектах
- Зміна лімітів та обмежень
- Мережева латентність

**Організаційні:**
- Опір змінам
- Навчання команди
- Управління ризиками
- Координація команд

**Фінансові:**
- Непередбачувані витрати
- Подвійні витрати під час міграції
- ROI може бути не одразу

**Рішення:** Ретельне планування, pilot проєкти, навчання

## Висновки

### 🎯 **Ключові моменти:**

**Переваги хмарних БД:**
- Швидкий старт та еластичність
- Зниження operational burden
- Глобальна доступність
- Вбудовані можливості HA/DR

**Вибір моделі:**
- IaaS — повний контроль
- PaaS/DBaaS — баланс
- Serverless — мінімум управління

**Провайдери:**
- AWS — найширший вибір
- GCP — інновації та аналітика
- Azure — Microsoft екосистема
