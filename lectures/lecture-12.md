# Лекція 12 Хмарні та розподілені СУБД

## Вступ

Сучасні інформаційні системи дедалі частіше переміщуються в хмарне середовище, і водночас повинні обробляти постійно зростаючі обсяги даних та навантаження від мільйонів користувачів по всьому світу. Ці дві тенденції — перехід у хмару та потреба в масштабуванні — насправді нерозривно пов'язані: більшість переваг, які хмарні провайдери обіцяють своїм клієнтам (еластичність, відмовостійкість, географічний розподіл), технічно реалізуються саме через розподілені архітектури баз даних. Хмара — це не просто «чужий сервер», а керована платформа, побудована на принципах горизонтального масштабування, реплікації та консенсусу, про які й піде мова в цій лекції.

Тому логічно розглядати ці дві теми разом, у межах єдиного розділу. Спочатку ми розберемо, що таке хмарні бази даних та модель Database-as-a-Service (DBaaS) — з якими моделями обслуговування, архітектурними патернами й провайдерами доводиться працювати інженеру програмного забезпечення. Далі ми зануримось у те, що відбувається «під капотом» цих хмарних сервісів: чому й коли системи масштабують вертикально, а коли — горизонтально, як розподіляють дані між серверами (шардинг), як підтримують кілька копій даних (реплікація), і як розподілені вузли домовляються між собою про єдину версію істини (консенсус). Насамкінець ми покажемо, як обидві теми сходяться в одній точці — у класі систем NewSQL / розподілений SQL, які за останні кілька років перетворилися з академічної екзотики на промислові пропозиції всіх провідних хмарних провайдерів.

Розуміння цих принципів є критично важливим для сучасних фахівців з програмної інженерії: більшість нових проєктів стартують одразу в хмарному середовищі, а існуючі системи активно мігрують до хмари саме заради масштабованості, надійності та економічної ефективності, недосяжних у межах одного власного сервера.

---

# Частина I. Хмарні бази даних та Database-as-a-Service

## Моделі хмарних обчислень для баз даних

### Три основні моделі обслуговування

Хмарні обчислення пропонують три фундаментальні моделі надання послуг, кожна з яких визначає рівень контролю та відповідальності між провайдером та клієнтом.

#### Infrastructure as a Service (IaaS)

Модель IaaS надає базову обчислювальну інфраструктуру як послугу. У контексті баз даних це означає, що провайдер надає віртуальні машини, мережу та сховище, а клієнт самостійно встановлює та конфігурує СУБД.

Клієнт отримує повний контроль над операційною системою та СУБД: він самостійно встановлює необхідну версію СУБД, конфігурує її параметри, налаштовує безпеку та відповідає за оновлення. Провайдер забезпечує лише базову інфраструктуру — обчислювальні ресурси, дискове сховище та мережеву зв'язність.

Приклади IaaS рішень: Amazon EC2 з самостійно встановленою базою даних, Google Compute Engine з власною конфігурацією СУБД або Azure Virtual Machines з розгорнутою БД на вибір клієнта.

**Переваги моделі IaaS:** максимальна гнучкість конфігурації дозволяє налаштувати систему під специфічні потреби; можливість використання будь-якої СУБД та її версії надає повну свободу вибору технологій; повний контроль над оптимізацією дає змогу досягти максимальної продуктивності для конкретного застосунку; легкість міграції існуючих систем спрощує перехід до хмари.

**Недоліки моделі IaaS:** висока складність адміністрування вимагає кваліфікованих фахівців; повна відповідальність за безпеку та оновлення лягає на клієнта; необхідність ручного масштабування ускладнює адаптацію до змінного навантаження; відсутність автоматизованих механізмів відновлення потребує додаткових зусиль.

#### Platform as a Service (PaaS)

Модель PaaS надає платформу для розробки та розгортання застосунків, включаючи керовані бази даних. Провайдер бере на себе управління операційною системою, СУБД та базовою інфраструктурою, автоматично керуючи оновленнями, резервним копіюванням, моніторингом продуктивності та масштабуванням. Клієнт зосереджується на схемі бази даних, запитах та логіці застосунку, не турбуючись про адміністрування інфраструктури.

Приклади PaaS рішень: Amazon RDS надає керовані реляційні бази даних з автоматичним резервним копіюванням та патчингом; Google Cloud SQL пропонує повністю керовані MySQL, PostgreSQL та SQL Server із вбудованою високою доступністю; Azure SQL Database забезпечує керовану хмарну базу даних з інтелектуальною оптимізацією продуктивності.

**Переваги моделі PaaS:** зниження операційних витрат за рахунок автоматизації адміністративних задач; автоматичне резервне копіювання та відновлення без додаткових зусиль; вбудовані механізми масштабування; висока доступність через автоматичну реплікацію та відмовостійкість.

**Недоліки моделі PaaS:** обмежена гнучкість конфігурації не дозволяє налаштувати всі параметри СУБД; залежність від провайдера ускладнює міграцію до іншої платформи (так званий *vendor lock-in*); можливі обмеження у версіях СУБД та розширеннях; вища вартість порівняно з IaaS при еквівалентних ресурсах.

#### Software as a Service (SaaS)

Модель SaaS надає повністю готовий застосунок, який працює на основі бази даних, прихованої від користувача. Провайдер керує всіма аспектами: інфраструктурою, СУБД, схемою даних, безпекою та масштабуванням; клієнт має доступ лише до функцій застосунку.

Приклади SaaS систем із базами даних: Salesforce CRM зберігає клієнтські дані в захищених хмарних базах даних; Google Workspace використовує розподілені бази даних для зберігання документів і пошти; Microsoft 365 працює на основі потужних хмарних СУБД для забезпечення співпраці.

### Порівняльна таблиця моделей

| Критерій | IaaS | PaaS | SaaS |
|---|---|---|---|
| Рівень контролю | Повний контроль над СУБД та інфраструктурою | Контроль обмежений схемою та конфігурацією БД | Лише контроль над даними застосунку |
| Відповідальність за адміністрування | Повністю на клієнті | Провайдер керує СУБД | Провайдер керує всім стеком |
| Складність управління | Вимагає високої кваліфікації адміністраторів | Потребує знань схем баз даних | Не вимагає знань про бази даних |
| Гнучкість | Максимальна | Помірна | Фіксована функціональність |
| Вартість | Найдешевша при оптимізації | Середня | Найвища для еквівалентних можливостей |

## Архітектурні патерни хмарних СУБД

### Multi-tenancy: багатоорендність

Multi-tenancy — один із фундаментальних патернів хмарних баз даних, що дозволяє ефективно використовувати ресурси між багатьма клієнтами (орендарями, *tenants*). Багатоорендність означає, що єдина інстанція СУБД обслуговує множину клієнтів, забезпечуючи при цьому ізоляцію їхніх даних та ресурсів. Існує три основні моделі реалізації цього підходу.

**Модель окремих баз даних** — кожен клієнт має власну базу даних на спільному сервері. Забезпечує високу ізоляцію даних та спрощує міграцію окремих клієнтів, але вимагає більше ресурсів та ускладнює оновлення схеми для всіх клієнтів одночасно.

**Модель окремих схем** — використовує єдину базу даних із різними схемами для кожного клієнта. Балансує ізоляцію та ефективність використання ресурсів, дозволяє кастомізувати схему для кожного клієнта, але ускладнює управління великою кількістю схем.

**Модель спільної схеми** — застосовує єдину схему для всіх клієнтів з ідентифікатором орендаря (`tenant_id`) у кожній таблиці. Максимально ефективно використовує ресурси й спрощує оновлення схеми, але потребує ретельного контролю доступу та може ускладнити кастомізацію для окремих клієнтів.

Приклад спільної схеми з ідентифікатором орендаря:

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    customer_name VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_tenant FOREIGN KEY (tenant_id)
        REFERENCES tenants(tenant_id)
);

CREATE INDEX idx_tenant_customers ON customers(tenant_id);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    customer_id INTEGER NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10, 2),
    CONSTRAINT fk_tenant FOREIGN KEY (tenant_id)
        REFERENCES tenants(tenant_id),
    CONSTRAINT fk_customer FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
);

CREATE INDEX idx_tenant_orders ON orders(tenant_id);
```

Ізоляція даних між орендарями забезпечується через Row-Level Security (RLS) — механізм, вбудований у PostgreSQL, який автоматично фільтрує рядки на рівні СУБД, а не на рівні застосунку:

```sql
CREATE POLICY tenant_isolation_customers ON customers
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);

CREATE POLICY tenant_isolation_orders ON orders
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);

ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
```

**Переваги багатоорендності:** економія ресурсів через спільне використання інфраструктури; спрощене обслуговування та оновлення для всіх клієнтів одночасно; можливість динамічного перерозподілу ресурсів між орендарями.

**Виклики багатоорендності:** забезпечення ізоляції даних та продуктивності; складність балансування ресурсів між клієнтами з різними потребами; ризик каскадних збоїв, які можуть вплинути на всіх орендарів (проблема так званого «шумного сусіда», *noisy neighbor*).

### Elasticity: еластичність

Еластичність визначає здатність системи автоматично адаптувати ресурси до поточного навантаження.

#### Вертикальна еластичність

Вертикальна еластичність означає зміну потужності окремого сервера шляхом додавання або зменшення ресурсів процесора, пам'яті чи дискового простору.

```python
import boto3

rds = boto3.client('rds')

def scale_database_vertically(db_instance_id, new_instance_class):
    response = rds.modify_db_instance(
        DBInstanceIdentifier=db_instance_id,
        DBInstanceClass=new_instance_class,
        ApplyImmediately=True
    )

    print(f"Масштабування БД до {new_instance_class}")
    return response

scale_database_vertically('my-database', 'db.t3.large')
```

**Переваги:** простота реалізації без зміни архітектури застосунку; відсутність необхідності перерозподілу даних; збереження всіх ACID гарантій.

**Недоліки:** обмеження максимальним розміром інстанції; необхідність простою при зміні конфігурації (для більшості керованих СУБД); вища вартість порівняно з горизонтальним масштабуванням.

#### Горизонтальна еластичність

Горизонтальна еластичність передбачає додавання або видалення серверів для розподілу навантаження — найчастіше через додавання read-реплік.

```python
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')
rds = boto3.client('rds')

def check_and_scale_replicas(db_instance_id):
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(minutes=15)

    cpu_stats = cloudwatch.get_metric_statistics(
        Namespace='AWS/RDS',
        MetricName='CPUUtilization',
        Dimensions=[{'Name': 'DBInstanceIdentifier', 'Value': db_instance_id}],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average']
    )

    avg_cpu = sum(point['Average'] for point in cpu_stats['Datapoints']) / len(cpu_stats['Datapoints'])

    if avg_cpu > 75:
        print("Високе навантаження CPU, додаємо read-репліку")
        add_read_replica(db_instance_id)
    elif avg_cpu < 30:
        print("Низьке навантаження CPU, розглядаємо видалення репліки")

def add_read_replica(source_db_id):
    replica_id = f"{source_db_id}-replica-{datetime.now().strftime('%Y%m%d%H%M%S')}"

    rds.create_db_instance_read_replica(
        DBInstanceIdentifier=replica_id,
        SourceDBInstanceIdentifier=source_db_id,
        DBInstanceClass='db.t3.medium'
    )
```

Політики auto-scaling базуються на метриках продуктивності — використанні процесора, кількості з'єднань, пропускній здатності операцій вводу-виводу. Провайдери дозволяють встановлювати порогові значення для автоматичного додавання або видалення ресурсів.

> Забігаючи наперед: горизонтальна еластичність на рівні одного керованого сервіса (read-репліки) — це лише часткова форма горизонтального масштабування. У Частині II ми розглянемо повноцінні розподілені архітектури, де горизонтально масштабується не лише читання, а й запис даних (шардинг, multi-master реплікація).

### Pay-per-use: оплата за використання

Модель оплати за фактичне використання — одна з ключових переваг хмарних баз даних.

**Оплата за обчислювальні ресурси** враховує час роботи інстанцій бази даних та їхню потужність. **Оплата за зберігання даних** розраховується на основі фактичного обсягу даних та типу сховища (SSD дорожчі, але продуктивніші). **Оплата за операції вводу-виводу** враховує кількість запитів читання й запису. **Оплата за передачу даних** включає вартість трафіку між регіонами або до інтернету (передача всередині одного регіону часто безкоштовна).

Приклад оцінки витрат для PostgreSQL на AWS RDS (орієнтовні тарифи, які варто перевіряти в актуальному калькуляторі AWS — ціни регулярно змінюються):

```
Базова конфігурація:
- Тип інстанції: db.t3.medium (2 vCPU, 4 GB RAM)
- Вартість: 0.068 USD за годину
- Місячна вартість інстанції: 0.068 * 730 = 49.64 USD

Зберігання:
- Обсяг: 100 GB General Purpose SSD (gp3)
- Вартість: 0.115 USD за GB на місяць
- Місячна вартість зберігання: 0.115 * 100 = 11.50 USD

Резервні копії:
- Додаткові 50 GB понад розмір БД
- Вартість: 0.095 USD за GB на місяць
- Місячна вартість резервних копій: 0.095 * 50 = 4.75 USD

Загальна місячна вартість: 49.64 + 11.50 + 4.75 = 65.89 USD
```

Дедалі популярнішими стають **serverless** моделі оплати, де кошти стягуються лише за фактичне використання обчислювальних ресурсів у моменти активності:

```
Aurora Serverless v2 — одиниця виміру: Aurora Capacity Units (ACU)
Мінімальна конфігурація: 0.5 ACU (v2 дозволяє масштабуватися майже до нуля)
Максимальна конфігурація: до 256 ACU

Вартість:
- ≈0.06–0.12 USD за ACU-годину (залежно від регіону)
- При середньому використанні 4 ACU протягом місяця:
  4 ACU * 730 годин * 0.06 USD ≈ 175 USD

Переваги:
- Автоматичне масштабування практично від нуля до максимуму за секунди
- Оплата лише за фактичне використання
- Ідеально для непередбачуваних або нерівномірних навантажень (SaaS-стартапи, dev/test середовища)
```

## Порівняльний аналіз провайдерів

### Amazon Web Services (AWS)

AWS пропонує найширший спектр сервісів баз даних серед хмарних провайдерів.

**Amazon RDS** (Relational Database Service) підтримує шість основних СУБД: MySQL, PostgreSQL, MariaDB, Oracle, Microsoft SQL Server та Amazon Aurora. Автоматичне резервне копіювання виконується щоденно з можливістю відновлення на будь-який момент часу протягом періоду збереження (Point-in-Time Recovery). Multi-AZ розгортання забезпечує високу доступність через синхронну реплікацію до резервного екземпляра в іншій зоні доступності. Read replicas дозволяють створювати репліки для читання в тому ж або інших регіонах.

```hcl
resource "aws_db_instance" "production" {
  identifier           = "production-database"
  engine               = "postgres"
  engine_version       = "17.4"
  instance_class       = "db.t3.large"
  allocated_storage    = 100
  storage_type         = "gp3"

  db_name  = "appdb"
  username = "dbadmin"
  password = var.db_password

  multi_az               = true
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  skip_final_snapshot = false
  final_snapshot_identifier = "production-final-snapshot"

  tags = {
    Environment = "Production"
    Application = "MainApp"
  }
}

resource "aws_db_instance" "read_replica" {
  identifier             = "production-database-replica"
  replicate_source_db    = aws_db_instance.production.id
  instance_class         = "db.t3.medium"

  publicly_accessible = false

  tags = {
    Environment = "Production"
    Type        = "ReadReplica"
  }
}
```

**Amazon Aurora** — хмарно-нативна СУБД, сумісна з MySQL та PostgreSQL, яка забезпечує суттєво кращу продуктивність порівняно зі звичайним MySQL завдяки архітектурі з відокремленим обчислювальним рівнем і рівнем зберігання. Розподілене сховище автоматично реплікує дані шість разів у трьох зонах доступності. Автоматичне відновлення після збоїв відбувається без втрати даних завдяки безперервній архівації до S3. Горизонтальне масштабування читання підтримує до 15 read replicas з мілісекундною затримкою реплікації. Aurora Serverless надає автоматичне масштабування обчислювальних ресурсів на основі навантаження з можливістю практично повної паузи при відсутності активності.

> **Актуалізація 2025 року:** у травні 2025 AWS оголосила загальну доступність (GA) **Amazon Aurora DSQL** — принципово нового serverless-сервісу, PostgreSQL-сумісного, з активною-активною мультирегіональною архітектурою «з коробки» (без потреби вручну налаштовувати реплікацію між регіонами). Це фактично злиття ідей DBaaS та розподіленого SQL, про яке детальніше — у Частині III.

### Google Cloud Platform (GCP)

Google Cloud пропонує керовані бази даних через Cloud SQL та власні рішення на кшталт Cloud Spanner.

**Cloud SQL** підтримує MySQL, PostgreSQL та SQL Server із повністю керованою інфраструктурою: автоматичні патчі та оновлення без участі користувача, точкове відновлення в часі (PITR), високодоступні конфігурації з автоматичним перемиканням при збоях.

```yaml
apiVersion: sql.cnrm.cloud.google.com/v1beta1
kind: SQLInstance
metadata:
  name: production-postgres
spec:
  databaseVersion: POSTGRES_17
  region: europe-west1
  settings:
    tier: db-custom-4-16384
    availabilityType: REGIONAL
    backupConfiguration:
      enabled: true
      startTime: "03:00"
      pointInTimeRecoveryEnabled: true
      transactionLogRetentionDays: 7
    ipConfiguration:
      ipv4Enabled: false
      privateNetwork: projects/my-project/global/networks/my-vpc
    maintenanceWindow:
      day: 7
      hour: 4
    databaseFlags:
      - name: max_connections
        value: "200"
      - name: shared_buffers
        value: "4194304"
```

**Cloud Spanner** — глобально розподілена реляційна база даних із горизонтальним масштабуванням та строгою консистентністю. Глобальна консистентність забезпечує ACID транзакції в масштабах кількох континентів (завдяки технології TrueTime — синхронізованих атомарних годинників), автоматичний шардинг розподіляє дані без участі розробника, синхронна реплікація гарантує нульову втрату даних при регіональних збоях. GCP також розвиває **AlloyDB** — PostgreSQL-сумісну СУБД з колонковим прискоренням аналітичних запитів, орієнтовану на конкуренцію з Aurora.

### Microsoft Azure

Azure пропонує широкий спектр сервісів баз даних для різних сценаріїв використання.

**Azure SQL Database** — повністю керована реляційна база даних на основі Microsoft SQL Server. Моделі розгортання: Single Database (ізольована база даних з гарантованими ресурсами), Elastic Pool (групування баз даних для спільного використання ресурсів), Managed Instance (майже повна сумісність з on-premises SQL Server). Рівні обслуговування: DTU-based (комплексна міра продуктивності), vCore-based (незалежне налаштування процесора, пам'яті та сховища), Serverless (автоматичне масштабування та пауза при неактивності).

```bicep
resource sqlServer 'Microsoft.Sql/servers@2023-08-01-preview' = {
  name: 'production-sql-server'
  location: 'westeurope'
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlAdminPassword
    version: '12.0'
    minimalTlsVersion: '1.2'
  }
}

resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  parent: sqlServer
  name: 'production-db'
  location: 'westeurope'
  sku: {
    name: 'GP_Gen5'
    tier: 'GeneralPurpose'
    capacity: 4
  }
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
    maxSizeBytes: 107374182400
    zoneRedundant: true
    readScale: 'Enabled'
    autoPauseDelay: 60
  }
}
```

Крім Azure SQL Database, Azure пропонує **Cosmos DB** (мультимодельна, глобально розподілена NoSQL-СУБД з вибором рівня консистентності) та **Azure Database for PostgreSQL — Flexible Server**.

> **Актуалізація 2025 року:** наприкінці 2025 Microsoft анонсувала **Azure HorizonDB** — новий розподілений SQL-сервіс, який позиціонується як відповідь Azure на Aurora DSQL та Cloud Spanner. Це підтверджує загальну тенденцію: усі три гіперскейлери одночасно рухаються в бік «безшовного» розподіленого SQL, доступного через ту саму DBaaS-модель, що й звичайна керована база даних.

Серед незалежних (multi-cloud) гравців варто згадати **MongoDB Atlas** (керована document-oriented БД, доступна на всіх трьох хмарах), **Neon** (serverless PostgreSQL з миттєвим розгалуженням бази даних — *database branching*, зручний для CI/CD) та **PlanetScale**, яка виросла з керованого MySQL/Vitess у мультидвигунну платформу — з 2024–2025 років вона пропонує також **PlanetScale Postgres**, побудований на власному сховищі («Metal»), не лише на класичному MySQL.

### Порівняльна таблиця провайдерів

| Критерій | AWS | GCP | Azure |
|---|---|---|---|
| Підтримувані СУБД | Найширший вибір + власна Aurora, Aurora DSQL | Стандартні рішення + унікальний Spanner, AlloyDB | Екосистема Microsoft, SQL Server, Cosmos DB, HorizonDB |
| Географічне покриття | Найбільша кількість регіонів | Глобальна присутність, акцент на мережеву продуктивність | Сильні позиції в Європі та Азії |
| Ціноутворення | Найбільше варіантів резервування та заощадження | Найпростіша структура цін | Переваги для клієнтів із ліцензіями Microsoft |
| Моніторинг | AWS CloudWatch — глибока інтеграція | Google Cloud Monitoring — зручна візуалізація | Azure Monitor — найкраща інтеграція з екосистемою Microsoft |

## Міграційні стратегії

### Модель «6R» як загальна рамка

Перш ніж переходити до конкретних технік, варто зауважити: індустрія узагальнила підходи до хмарної міграції в модель **«6R»** — Rehost (lift-and-shift), Replatform, Refactor/Re-architect, Repurchase (заміна на SaaS-аналог), Retire (виведення з експлуатації застарілих систем) та Retain (залишення як є). Дві найпоширеніші для баз даних стратегії — lift-and-shift та re-architecting — розглянемо детальніше.

### Lift-and-shift міграція (Rehost)

Lift-and-shift, або rehosting, — найпростіша стратегія міграції: переміщення існуючої бази даних до хмари з мінімальними змінами.

Підготовчий етап включає аналіз поточної інфраструктури, оцінку обсягів даних та залежностей, планування вікна міграції з мінімальним впливом на бізнес. Етап міграції передбачає створення резервної копії production бази даних, розгортання еквівалентної інфраструктури в хмарі, відновлення резервної копії на хмарному сервері, налаштування мережевого з'єднання та оновлення конфігурації застосунків.

```bash
#!/bin/bash

SOURCE_DB="production-db.company.local"
SOURCE_USER="postgres"
TARGET_DB="production-db.abc123.eu-west-1.rds.amazonaws.com"
TARGET_USER="postgres"
DUMP_FILE="production-backup.sql"

echo "Створення резервної копії джерельної БД..."
pg_dump -h $SOURCE_DB -U $SOURCE_USER -Fc -f $DUMP_FILE production

echo "Завантаження резервної копії до S3..."
aws s3 cp $DUMP_FILE s3://migration-bucket/backups/

echo "Створення RDS інстанції..."
aws rds create-db-instance \
    --db-instance-identifier production-migrated \
    --db-instance-class db.m5.xlarge \
    --engine postgres \
    --engine-version 17.4 \
    --master-username postgres \
    --master-user-password $DB_PASSWORD \
    --allocated-storage 500 \
    --multi-az

echo "Очікування готовності RDS інстанції..."
aws rds wait db-instance-available \
    --db-instance-identifier production-migrated

echo "Відновлення даних на RDS..."
pg_restore -h $TARGET_DB -U $TARGET_USER -d production $DUMP_FILE

echo "Міграція завершена!"
```

Для великих production-баз, де мінімізація простою критична, замість ручного `pg_dump`/`pg_restore` варто застосовувати спеціалізовані сервіси безперервної реплікації, наприклад **AWS Database Migration Service (DMS)** разом зі **Schema Conversion Tool** — вони дозволяють мігрувати дані практично без простою (near-zero-downtime) через безперервний захват змін (CDC) із джерела в цільову базу, з подальшим коротким «cutover»-вікном.

**Переваги lift-and-shift:** швидкість міграції — не потрібно переписувати код застосунку; мінімальний ризик через збереження існуючої архітектури; можливість поетапної оптимізації (спочатку перемістити, потім вдосконалювати).

**Недоліки lift-and-shift:** неповне використання хмарних можливостей; збереження технічного боргу; можлива неоптимальна вартість через застарілі підходи.

### Re-architecting міграція (Refactor)

Re-architecting, або refactoring, передбачає переробку архітектури застосунку для максимального використання хмарних можливостей.

Аналітична фаза включає оцінку поточної архітектури, виявлення вузьких місць та неефективностей, визначення можливостей для використання хмарних сервісів. Фаза проєктування передбачає розробку цільової архітектури з урахуванням хмарних патернів, вибір оптимальних хмарних сервісів, планування стратегії розгортання та переходу.

```mermaid
graph TB
    subgraph "До міграції"
        A[Монолітний застосунок] --> B[(Єдина БД)]
    end

    subgraph "Після міграції"
        C[User Service] --> D[(User DB<br/>RDS PostgreSQL)]
        E[Order Service] --> F[(Order DB<br/>Aurora MySQL)]
        G[Inventory Service] --> H[(Inventory DB<br/>DynamoDB)]
        I[Analytics Service] --> J[(Analytics DB<br/>Redshift)]

        C -.REST API.-> E
        E -.Events.-> G
        G -.Stream.-> I
    end
```

Стратегія поетапної міграції: перша фаза виділяє окремі сервіси з монолітної бази даних зі збереженням зворотної сумісності; друга фаза мігрує критичні сервіси до хмарних керованих баз даних; третя фаза оптимізує використання хмарних можливостей, впроваджує auto-scaling та serverless компоненти.

```python
from flask import Flask, jsonify
import psycopg2
from circuit_breaker import CircuitBreaker

app = Flask(__name__)

# Підключення до окремих БД для різних сервісів
USER_DB_CONFIG = {
    'host': 'user-db.abc123.rds.amazonaws.com',
    'database': 'users',
    'user': 'app_user',
    'password': 'secure_password'
}

ORDER_DB_CONFIG = {
    'host': 'order-db.xyz456.rds.amazonaws.com',
    'database': 'orders',
    'user': 'app_user',
    'password': 'secure_password'
}

@app.route('/api/users/<int:user_id>')
@CircuitBreaker(failure_threshold=5, recovery_timeout=30)
def get_user(user_id):
    with psycopg2.connect(**USER_DB_CONFIG) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            user = cur.fetchone()
            return jsonify(user)

@app.route('/api/orders/<int:order_id>')
@CircuitBreaker(failure_threshold=5, recovery_timeout=30)
def get_order(order_id):
    with psycopg2.connect(**ORDER_DB_CONFIG) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM orders WHERE id = %s", (order_id,))
            order = cur.fetchone()
            return jsonify(order)
```

**Переваги re-architecting:** максимальне використання хмарних можливостей для оптимальної продуктивності й вартості; підвищення масштабованості; покращення відмовостійкості через керовані сервіси.

**Недоліки re-architecting:** тривалість проєкту значно перевищує lift-and-shift; висока складність вимагає глибоких знань хмарних технологій; ризик переривання бізнес-процесів при невдалій міграції.

Практичний вибір стратегії cutover (переходу з джерельної на цільову базу) також важливий: **Big Bang** (одномоментне перемикання, простіше, але ризикованіше), **Phased** (поетапне перенесення частин функціональності) та **Blue-Green** (паралельна робота обох середовищ з поступовим переключенням трафіку) — остання стратегія особливо популярна для мінімізації ризику в критичних production-системах.

## Управління витратами та оптимізація ресурсів

### Моніторинг та аналіз витрат

Ефективне управління витратами починається з детального моніторингу використання ресурсів. AWS Cost Explorer надає візуалізацію витрат із фільтрацією за сервісами, регіонами та тегами. Google Cloud Billing Reports пропонує детальний аналіз витрат із рекомендаціями щодо оптимізації. Azure Cost Management забезпечує бюджетування та прогнозування витрат.

```python
import boto3

budgets = boto3.client('budgets')

def create_database_budget():
    response = budgets.create_budget(
        AccountId='123456789012',
        Budget={
            'BudgetName': 'Monthly-Database-Budget',
            'BudgetLimit': {
                'Amount': '1000',
                'Unit': 'USD'
            },
            'TimeUnit': 'MONTHLY',
            'BudgetType': 'COST',
            'CostFilters': {
                'Service': ['Amazon Relational Database Service']
            }
        },
        NotificationsWithSubscribers=[
            {
                'Notification': {
                    'NotificationType': 'ACTUAL',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 80,
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': 'admin@company.com'
                    }
                ]
            }
        ]
    )
    return response
```

### Стратегії оптимізації витрат

**Правильний вибір типу інстанції.** General Purpose інстанції збалансовані за процесором та пам'яттю для типових застосунків. Memory Optimized інстанції мають підвищений обсяг RAM для баз даних у пам'яті. Burstable інстанції підходять для непередбачуваних навантажень із можливістю короткочасних сплесків.

**Резервування ресурсів.** Reserved Instances дозволяють заощадити до 60% при зобов'язанні на один або три роки використання. Savings Plans надають гнучкість у виборі конфігурацій зі знижкою до 72%.

**Автоматизація керування ресурсами** — зокрема автоматичне зупинення непотрібних середовищ розробки й тестування:

```python
import boto3
from datetime import datetime

rds = boto3.client('rds')

def stop_dev_databases():
    current_hour = datetime.now().hour
    current_day = datetime.now().weekday()

    if current_day < 5 and current_hour >= 19:
        response = rds.describe_db_instances()

        for db in response['DBInstances']:
            tags = rds.list_tags_for_resource(
                ResourceName=db['DBInstanceArn']
            )

            for tag in tags['TagList']:
                if tag['Key'] == 'Environment' and tag['Value'] == 'Development':
                    print(f"Зупинка {db['DBInstanceIdentifier']}")
                    rds.stop_db_instance(
                        DBInstanceIdentifier=db['DBInstanceIdentifier']
                    )

def start_dev_databases():
    current_hour = datetime.now().hour
    current_day = datetime.now().weekday()

    if current_day < 5 and current_hour == 8:
        response = rds.describe_db_instances()

        for db in response['DBInstances']:
            if db['DBInstanceStatus'] == 'stopped':
                tags = rds.list_tags_for_resource(
                    ResourceName=db['DBInstanceArn']
                )

                for tag in tags['TagList']:
                    if tag['Key'] == 'Environment' and tag['Value'] == 'Development':
                        print(f"Запуск {db['DBInstanceIdentifier']}")
                        rds.start_db_instance(
                            DBInstanceIdentifier=db['DBInstanceIdentifier']
                        )
```

**Оптимізація зберігання.** General Purpose SSD (gp3) балансує продуктивність і ціну для більшості застосунків. Provisioned IOPS SSD необхідний для критичних застосунків із високим навантаженням вводу-виводу. Автоматичне стиснення даних і архівування застарілих записів зменшує обсяг потрібного сховища.

**Spot instances** для непродуктивних навантажень можуть коштувати до 90% дешевше звичайних, але можуть бути перервані з коротким попередженням — підходять для аналітичних завдань, тестування, batch-обробки.

---

# Частина II. Масштабування та розподілені архітектури

Хмарні провайдери дають зручний інтерфейс для запуску керованої бази даних, але фізичні обмеження нікуди не зникають: рано чи пізно будь-яка система впирається в межу продуктивності одного сервера. Що робити далі — і про це друга частина лекції.

## Вертикальне та горизонтальне масштабування

### Вертикальне масштабування (Scale Up)

Вертикальне масштабування передбачає збільшення потужності окремого сервера шляхом додавання процесорів, оперативної пам'яті, дискового простору або заміни компонентів на продуктивніші.

Принцип роботи полягає в тому, що замість додавання нових серверів система покращується через модернізацію наявного обладнання. Типові сценарії застосування: реляційні бази даних, які важко розподілити горизонтально; застосунки з високими вимогами до консистентності даних; системи зі складними транзакціями, що охоплюють багато таблиць; legacy застосунки, не призначені для розподіленого виконання.

```sql
-- Перевірка поточних налаштувань
SHOW shared_buffers;
SHOW effective_cache_size;
SHOW work_mem;
SHOW maintenance_work_mem;

-- Оптимізація для сервера з 64 GB RAM
ALTER SYSTEM SET shared_buffers = '16GB';
ALTER SYSTEM SET effective_cache_size = '48GB';
ALTER SYSTEM SET work_mem = '256MB';
ALTER SYSTEM SET maintenance_work_mem = '2GB';
ALTER SYSTEM SET max_connections = 200;

-- Налаштування для SSD дисків
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;

-- Паралельне виконання запитів
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers = 8;

-- Застосування змін
SELECT pg_reload_conf();
```

Моніторинг ефективності після масштабування:

```sql
-- Аналіз використання кешу
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as cache_hit_ratio
FROM pg_statio_user_tables;

-- Перевірка активних з'єднань
SELECT
    count(*) as total_connections,
    count(*) FILTER (WHERE state = 'active') as active_connections,
    count(*) FILTER (WHERE state = 'idle') as idle_connections
FROM pg_stat_activity;

-- Аналіз найповільніших запитів
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

> **Актуалізація:** починаючи з PostgreSQL 18 (вересень 2025) з'явилася нова підсистема асинхронного вводу-виводу на основі `io_uring` (параметр `io_method`), яка суттєво підвищує ефективність використання дискової підсистеми саме на потужних багатоядерних серверах — тобто безпосередньо розширює межі корисності вертикального масштабування ще на один щабель.

**Переваги вертикального масштабування:** простота реалізації без переписування коду застосунку; збереження ACID гарантій у межах однієї машини; відсутність мережевої латентності; простіше адміністрування одного сервера замість кластера.

**Недоліки вертикального масштабування:** обмеження масштабованості — фізична межа потужності одного сервера; єдина точка відмови; висока вартість, яка зростає нелінійно зі збільшенням потужності; необхідність простою при оновленні для більшості конфігурацій.

```
Сучасний high-end сервер (2025–2026):
- Процесор: AMD EPYC 9005 «Turin» (до 192 ядер, 384 потоки) або Intel Xeon 6
- Оперативна пам'ять: до 6 TB DDR5
- Дискова система: до 24+ NVMe SSD дисків (Gen5)
- Мережа: 100–400 Gbps

Обмеження:
- Максимальна пропускна здатність процесора
- Пропускна здатність шини пам'яті
- Швидкість дискової підсистеми
- Можливості охолодження

Коли досягнуті ці межі, єдиний варіант — горизонтальне масштабування
```

### Горизонтальне масштабування (Scale Out)

Горизонтальне масштабування передбачає додавання нових серверів до існуючого кластера для розподілу навантаження та даних між множиною машин. Принцип роботи базується на розподілі даних і обчислень між багатьма відносно недорогими серверами, що працюють координовано.

Типові сценарії застосування: веб-застосунки з великою кількістю користувачів; NoSQL бази даних із масивними обсягами даних; системи аналітики великих даних; мікросервісні архітектури з незалежними компонентами.

```bash
#!/bin/bash

# Конфігурація master-slave реплікації

# На master сервері
cat > /etc/postgresql/17/main/postgresql.conf << EOF
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB
hot_standby = on
EOF

# Створення користувача реплікації
sudo -u postgres psql << EOF
CREATE ROLE replication_user WITH REPLICATION LOGIN PASSWORD 'secure_password';
EOF

# На slave сервері - створення реплікації
sudo -u postgres pg_basebackup -h master.example.com -D /var/lib/postgresql/17/main -U replication_user -P -v -R -X stream -C -S replica_1

# Налаштування PgBouncer для розподілу навантаження
cat > /etc/pgbouncer/pgbouncer.ini << EOF
[databases]
production = host=master.example.com port=5432 dbname=production
production_ro = host=slave1.example.com,slave2.example.com port=5432 dbname=production

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
EOF
```

Розподіл читання й запису на рівні застосунку через connection pooler:

```python
import psycopg2
from psycopg2 import pool
import random

# Пул з'єднань для запису (master)
write_pool = pool.SimpleConnectionPool(
    1, 20,
    host='master.example.com',
    database='production',
    user='app_user',
    password='secure_password'
)

# Пул з'єднань для читання (slaves)
read_pool = pool.SimpleConnectionPool(
    5, 50,
    host='localhost',
    port=6432,
    database='production_ro',
    user='app_user',
    password='secure_password'
)

class DatabaseManager:
    def execute_write(self, query, params=None):
        conn = write_pool.getconn()
        try:
            with conn.cursor() as cur:
                cur.execute(query, params)
                conn.commit()
                return cur.fetchall()
        finally:
            write_pool.putconn(conn)

    def execute_read(self, query, params=None):
        conn = read_pool.getconn()
        try:
            with conn.cursor() as cur:
                cur.execute(query, params)
                return cur.fetchall()
        finally:
            read_pool.putconn(conn)

db = DatabaseManager()

# Запис йде на master
db.execute_write(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    ("John Doe", "john@example.com")
)

# Читання розподіляється між slaves
users = db.execute_read("SELECT * FROM users WHERE active = true")
```

**Переваги горизонтального масштабування:** практично необмежена масштабованість — можна додавати сервери в міру зростання навантаження; відмовостійкість через дублювання даних на кількох серверах; економічна ефективність завдяки використанню звичайного обладнання; географічний розподіл ближче до користувачів у різних регіонах.

**Недоліки горизонтального масштабування:** складність архітектури, що вимагає врахування розподіленості; складніше забезпечення консистентності даних; мережева латентність при координації кількох серверів; складніше адміністрування кластера.

### CAP-теорема та її практичні наслідки

CAP-теорема стверджує, що розподілена система не може одночасно гарантувати всі три властивості: консистентність, доступність та стійкість до розділення мережі.

**Консистентність (Consistency)** означає, що всі вузли бачать однакові дані в один і той самий момент часу — будь-яке читання отримує найновіші записані дані. **Доступність (Availability)** гарантує, що кожен запит отримує відповідь (без гарантії, що вона містить найновіші дані) — система продовжує працювати навіть при відмові частини вузлів. **Стійкість до розділення (Partition tolerance)** означає, що система продовжує працювати навіть при втраті або затримці повідомлень між вузлами.

```
CP системи (Consistency + Partition tolerance):
- Приклади: MongoDB (з majority write concern), HBase, Redis Cluster
- Вибір: Консистентність важливіша за доступність
- Поведінка: При мережевому розділенні частина вузлів стає недоступною
- Сценарії: Фінансові транзакції, інвентаризація товарів

AP системи (Availability + Partition tolerance):
- Приклади: Cassandra, DynamoDB, Couchbase
- Вибір: Доступність важливіша за консистентність
- Поведінка: При мережевому розділенні всі вузли доступні, але дані можуть відрізнятися
- Сценарії: Соціальні мережі, рекомендаційні системи

CA системи (Consistency + Availability):
- Приклади: Традиційні RDBMS в одному дата-центрі
- Обмеження: Не толерантні до мережевого розділення
- Реальність: У розподілених системах мережеві проблеми неминучі
```

Практичний висновок: на практиці «чистих» CA-систем у розподіленому середовищі не буває — питання лише в тому, чи система свідомо жертвує консистентністю заради доступності (AP), чи навпаки (CP). Ми ще повернемось до цього питання, коли розглядатимемо NewSQL-системи, які намагаються звести цей компроміс до мінімуму.

## Стратегії шардингу

Шардинг — метод горизонтального розподілу даних між кількома базами даних або серверами, де кожен шард містить підмножину загальних даних.

### Діапазонний шардинг (Range-based Sharding)

Діапазонний шардинг розподіляє дані на основі діапазонів значень ключа шардингу.

```sql
-- Шард 1: користувачі зареєстровані у 2023 році
CREATE TABLE users_2023 (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    registration_date DATE NOT NULL CHECK (
        registration_date >= '2023-01-01' AND
        registration_date < '2024-01-01'
    ),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Шард 2: користувачі зареєстровані у 2024 році
CREATE TABLE users_2024 (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    registration_date DATE NOT NULL CHECK (
        registration_date >= '2024-01-01' AND
        registration_date < '2025-01-01'
    ),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Шард 3: користувачі зареєстровані у 2025 році
CREATE TABLE users_2025 (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    registration_date DATE NOT NULL CHECK (
        registration_date >= '2025-01-01' AND
        registration_date < '2026-01-01'
    ),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```python
from datetime import datetime
import psycopg2

class ShardRouter:
    def __init__(self):
        self.shards = {
            2023: {
                'host': 'shard1.example.com',
                'database': 'users_db',
                'table': 'users_2023'
            },
            2024: {
                'host': 'shard2.example.com',
                'database': 'users_db',
                'table': 'users_2024'
            },
            2025: {
                'host': 'shard3.example.com',
                'database': 'users_db',
                'table': 'users_2025'
            }
        }

    def get_shard(self, registration_date):
        year = registration_date.year
        if year not in self.shards:
            raise ValueError(f"Шард для року {year} не знайдено")
        return self.shards[year]

    def insert_user(self, username, email, registration_date):
        shard = self.get_shard(registration_date)

        conn = psycopg2.connect(
            host=shard['host'],
            database=shard['database'],
            user='app_user',
            password='secure_password'
        )

        try:
            with conn.cursor() as cur:
                query = f"""
                    INSERT INTO {shard['table']}
                    (username, email, registration_date)
                    VALUES (%s, %s, %s)
                    RETURNING user_id
                """
                cur.execute(query, (username, email, registration_date))
                conn.commit()
                return cur.fetchone()[0]
        finally:
            conn.close()

router = ShardRouter()
user_id = router.insert_user(
    'john_doe',
    'john@example.com',
    datetime(2025, 3, 15)
)
```

**Переваги:** простота концепції; ефективні діапазонні запити (всі дані з певного діапазону — на одному шарді); легкість додавання нових шардів для нових діапазонів.

**Недоліки:** нерівномірний розподіл даних може створити «гарячі точки» (hotspots) — наприклад, шард поточного року завжди навантаженіший за минулі; складність перебалансування при зміні діапазонів.

### Хешований шардинг (Hash-based Sharding)

Хешований шардинг використовує хеш-функцію для визначення, на якому шарді зберігати кожен запис.

```python
import hashlib
from bisect import bisect_right

class ConsistentHashRing:
    def __init__(self, nodes, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []

        for node in nodes:
            self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(key.encode('utf-8')).hexdigest(), 16)

    def add_node(self, node):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = node
            self.sorted_keys.append(hash_value)

        self.sorted_keys.sort()

    def remove_node(self, node):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            del self.ring[hash_value]
            self.sorted_keys.remove(hash_value)

    def get_node(self, key):
        if not self.ring:
            return None

        hash_value = self._hash(str(key))
        index = bisect_right(self.sorted_keys, hash_value)

        if index == len(self.sorted_keys):
            index = 0

        return self.ring[self.sorted_keys[index]]

class ShardedDatabase:
    def __init__(self, shard_configs):
        nodes = [config['name'] for config in shard_configs]
        self.hash_ring = ConsistentHashRing(nodes)

        self.connections = {}
        for config in shard_configs:
            self.connections[config['name']] = {
                'host': config['host'],
                'database': config['database']
            }

    def get_connection(self, key):
        node = self.hash_ring.get_node(key)
        config = self.connections[node]

        return psycopg2.connect(
            host=config['host'],
            database=config['database'],
            user='app_user',
            password='secure_password'
        )

    def insert_user(self, user_id, username, email):
        conn = self.get_connection(user_id)

        try:
            with conn.cursor() as cur:
                cur.execute(
                    """
                    INSERT INTO users (user_id, username, email)
                    VALUES (%s, %s, %s)
                    """,
                    (user_id, username, email)
                )
                conn.commit()
        finally:
            conn.close()

    def get_user(self, user_id):
        conn = self.get_connection(user_id)

        try:
            with conn.cursor() as cur:
                cur.execute(
                    "SELECT user_id, username, email FROM users WHERE user_id = %s",
                    (user_id,)
                )
                return cur.fetchone()
        finally:
            conn.close()

shards = [
    {'name': 'shard1', 'host': 'shard1.example.com', 'database': 'users'},
    {'name': 'shard2', 'host': 'shard2.example.com', 'database': 'users'},
    {'name': 'shard3', 'host': 'shard3.example.com', 'database': 'users'},
    {'name': 'shard4', 'host': 'shard4.example.com', 'database': 'users'}
]

db = ShardedDatabase(shards)
db.insert_user(12345, 'alice', 'alice@example.com')
user = db.get_user(12345)
```

Ключова ідея — **consistent hashing** («узгоджене хешування»): замість прямого хешування `key mod N` (де додавання чи видалення шарду вимагає перерозподілу майже всіх ключів), consistent hashing розміщує вузли й ключі на єдиному хеш-кільці, тому додавання чи видалення вузла зачіпає лише невелику частку ключів.

**Переваги:** рівномірний розподіл даних; передбачуваність розміщення даних (шард обчислюється за ключем без додаткових запитів); добра масштабованість для операцій за ключем.

**Недоліки:** складність діапазонних запитів (вимагає звернення до всіх шардів); перехешування при додаванні шардів (пом'якшується consistent hashing, але не усувається повністю).

### Директорний шардинг (Directory-based Sharding)

Директорний шардинг використовує окрему таблицю-довідник, яка містить інформацію про розміщення даних на шардах.

```sql
-- Lookup таблиця на окремому сервері
CREATE TABLE shard_directory (
    tenant_id INTEGER PRIMARY KEY,
    shard_name VARCHAR(50) NOT NULL,
    shard_host VARCHAR(255) NOT NULL,
    shard_database VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_shard_name ON shard_directory(shard_name);

-- Наповнення довідника
INSERT INTO shard_directory (tenant_id, shard_name, shard_host, shard_database) VALUES
(1001, 'shard_premium', 'premium-shard.example.com', 'tenant_db'),
(1002, 'shard_standard', 'standard-shard1.example.com', 'tenant_db'),
(1003, 'shard_standard', 'standard-shard1.example.com', 'tenant_db'),
(1004, 'shard_premium', 'premium-shard.example.com', 'tenant_db'),
(1005, 'shard_standard', 'standard-shard2.example.com', 'tenant_db');
```

```python
import psycopg2
from functools import lru_cache

class DirectoryBasedSharding:
    def __init__(self, directory_config):
        self.directory_conn = psycopg2.connect(
            host=directory_config['host'],
            database=directory_config['database'],
            user=directory_config['user'],
            password=directory_config['password']
        )
        self.shard_connections = {}

    @lru_cache(maxsize=1000)
    def get_shard_info(self, tenant_id):
        with self.directory_conn.cursor() as cur:
            cur.execute(
                """
                SELECT shard_name, shard_host, shard_database
                FROM shard_directory
                WHERE tenant_id = %s
                """,
                (tenant_id,)
            )
            result = cur.fetchone()

            if not result:
                raise ValueError(f"Шард для tenant_id {tenant_id} не знайдено")

            return {
                'name': result[0],
                'host': result[1],
                'database': result[2]
            }

    def get_shard_connection(self, tenant_id):
        shard_info = self.get_shard_info(tenant_id)
        shard_key = shard_info['name']

        if shard_key not in self.shard_connections:
            self.shard_connections[shard_key] = psycopg2.connect(
                host=shard_info['host'],
                database=shard_info['database'],
                user='app_user',
                password='secure_password'
            )

        return self.shard_connections[shard_key]

    def execute_query(self, tenant_id, query, params=None):
        conn = self.get_shard_connection(tenant_id)

        with conn.cursor() as cur:
            cur.execute(query, params)
            conn.commit()
            return cur.fetchall()

    def migrate_tenant(self, tenant_id, new_shard_name, new_shard_host, new_shard_database):
        old_shard = self.get_shard_info(tenant_id)

        old_conn = self.get_shard_connection(tenant_id)
        new_conn = psycopg2.connect(
            host=new_shard_host,
            database=new_shard_database,
            user='app_user',
            password='secure_password'
        )

        try:
            # Тут має бути логіка копіювання даних
            # Для простоти опущено

            with self.directory_conn.cursor() as cur:
                cur.execute(
                    """
                    UPDATE shard_directory
                    SET shard_name = %s,
                        shard_host = %s,
                        shard_database = %s,
                        updated_at = CURRENT_TIMESTAMP
                    WHERE tenant_id = %s
                    """,
                    (new_shard_name, new_shard_host, new_shard_database, tenant_id)
                )
                self.directory_conn.commit()

            self.get_shard_info.cache_clear()

        finally:
            new_conn.close()

directory_config = {
    'host': 'directory.example.com',
    'database': 'shard_directory',
    'user': 'directory_user',
    'password': 'directory_password'
}

sharding = DirectoryBasedSharding(directory_config)

results = sharding.execute_query(
    tenant_id=1001,
    query="SELECT * FROM orders WHERE order_date > %s",
    params=('2025-01-01',)
)

sharding.migrate_tenant(
    tenant_id=1002,
    new_shard_name='shard_premium',
    new_shard_host='premium-shard.example.com',
    new_shard_database='tenant_db'
)
```

**Переваги:** максимальна гнучкість у розподілі даних; легкість міграції даних між шардами (лише оновлення lookup-таблиці); можливість балансування навантаження шляхом переміщення активних клієнтів на окремі потужні сервери.

**Недоліки:** додаткова точка відмови у вигляді lookup-сервера; затримка на lookup-запит перед кожним запитом до даних; складність підтримки консистентності між довідником і фактичним розміщенням даних.

## Реплікація даних

Реплікація передбачає створення та підтримку копій даних на кількох серверах для забезпечення відмовостійкості й розподілу навантаження читання.

### Синхронна реплікація

Синхронна реплікація гарантує, що дані записані на всі репліки перед підтвердженням транзакції клієнту: клієнт відправляє запит на запис → master записує дані локально → master чекає підтвердження від усіх синхронних реплік → лише після цього транзакція вважається завершеною.

```sql
-- На master сервері
ALTER SYSTEM SET synchronous_commit = 'on';
ALTER SYSTEM SET synchronous_standby_names = 'replica1,replica2';

SELECT pg_reload_conf();

-- Перевірка статусу реплікації
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    replay_lag
FROM pg_stat_replication;
```

```bash
# postgresql.conf на master
synchronous_commit = remote_apply
synchronous_standby_names = 'FIRST 2 (replica1, replica2, replica3)'

# Master чекає на підтвердження від перших двох доступних реплік
# Транзакція завершується тільки після застосування змін на репліках
```

**Переваги:** нульова втрата даних; консистентність читання; автоматичне відновлення після збоїв без втрати транзакцій.

**Недоліки:** підвищена латентність запису; зниження доступності при недоступності реплік; обмеження продуктивності через послідовне підтвердження транзакцій.

### Асинхронна реплікація

Асинхронна реплікація не чекає на підтвердження від реплік перед завершенням транзакції: клієнт відправляє запит → master записує дані локально → master відразу підтверджує транзакцію → зміни асинхронно передаються на репліки.

```sql
-- На master сервері
ALTER SYSTEM SET synchronous_commit = 'local';

SELECT pg_reload_conf();

-- Моніторинг затримки реплікації
SELECT
    application_name,
    client_addr,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replication_lag_bytes,
    EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS replication_lag_seconds
FROM pg_stat_replication;
```

```python
import psycopg2
import smtplib
from email.mime.text import MIMEText

def check_replication_lag():
    conn = psycopg2.connect(
        host='master.example.com',
        database='postgres',
        user='monitor_user',
        password='monitor_password'
    )

    with conn.cursor() as cur:
        cur.execute("""
            SELECT
                application_name,
                EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds
            FROM pg_stat_replication
        """)

        results = cur.fetchall()

        for replica_name, lag in results:
            if lag > 60:
                send_alert(
                    f"Висока затримка реплікації на {replica_name}: {lag:.2f} секунд"
                )

    conn.close()

def send_alert(message):
    msg = MIMEText(message)
    msg['Subject'] = 'Попередження про затримку реплікації'
    msg['From'] = 'monitor@example.com'
    msg['To'] = 'admin@example.com'

    smtp = smtplib.SMTP('localhost')
    smtp.send_message(msg)
    smtp.quit()
```

**Переваги:** низька латентність запису; висока доступність для запису навіть при недоступності реплік; краща продуктивність через відсутність очікування підтвердження.

**Недоліки:** можлива втрата даних при збої master між записом і реплікацією; незначна затримка читання на репліках; складність автоматичного failover через потенційну розбіжність даних.

### Master-slave архітектура

Master-slave — класична архітектура реплікації, де один сервер приймає запити на запис, а інші тільки реплікують дані. Master обробляє всі операції запису та частину читання, slave-сервери обробляють операції читання та зберігають копію даних.

```yaml
# Patroni конфігурація для PostgreSQL HA
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1.example.com:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 2GB
        effective_cache_size: 6GB
        wal_level: replica
        max_wal_senders: 5
        max_replication_slots: 5

  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1.example.com:5432
  data_dir: /var/lib/postgresql/17/main
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: repl_password
    superuser:
      username: postgres
      password: postgres_password

  parameters:
    unix_socket_directories: '/var/run/postgresql'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
```

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import random

class DatabaseRouter:
    def __init__(self, master_url, slave_urls):
        self.master_engine = create_engine(
            master_url,
            pool_size=20,
            max_overflow=40
        )

        self.slave_engines = [
            create_engine(
                url,
                pool_size=30,
                max_overflow=60
            ) for url in slave_urls
        ]

        self.MasterSession = sessionmaker(bind=self.master_engine)

    def get_write_session(self):
        return self.MasterSession()

    def get_read_session(self):
        engine = random.choice(self.slave_engines)
        Session = sessionmaker(bind=engine)
        return Session()

master = "postgresql://user:pass@master.example.com:5432/db"
slaves = [
    "postgresql://user:pass@slave1.example.com:5432/db",
    "postgresql://user:pass@slave2.example.com:5432/db",
    "postgresql://user:pass@slave3.example.com:5432/db"
]

db_router = DatabaseRouter(master, slaves)

# Запис на master
with db_router.get_write_session() as session:
    new_user = User(name="John Doe", email="john@example.com")
    session.add(new_user)
    session.commit()

# Читання з випадкового slave
with db_router.get_read_session() as session:
    users = session.query(User).filter(User.active == True).all()
```

**Переваги:** простота архітектури; розподіл навантаження читання між кількома slave; захист даних через наявність кількох копій.

**Недоліки:** єдина точка відмови для запису; обмеження масштабування запису (не подолується додаванням slave); складність автоматичного «promotion» slave до master при збої.

### Master-master архітектура

Master-master, або multi-master, архітектура дозволяє кільком серверам приймати операції запису одночасно: кожен master реплікує свої зміни на інший master і приймає зміни від нього.

```sql
-- На обох серверах
CREATE EXTENSION bdr;

-- На першому сервері
SELECT bdr.create_node(
    node_name := 'node1',
    local_dsn := 'host=node1.example.com port=5432 dbname=production'
);

-- На другому сервері
SELECT bdr.create_node(
    node_name := 'node2',
    local_dsn := 'host=node2.example.com port=5432 dbname=production'
);

-- Встановлення зв'язку між вузлами
SELECT bdr.create_node_group(
    node_group_name := 'production_group'
);

SELECT bdr.join_node_group(
    join_target_dsn := 'host=node1.example.com port=5432 dbname=production',
    node_group_name := 'production_group'
);
```

```sql
-- Налаштування стратегії розв'язання конфліктів
ALTER TABLE users SET (
    bdr.conflict_detection = 'row_origin',
    bdr.conflict_resolution = 'last_update_wins'
);

-- Кастомна функція розв'язання конфліктів
CREATE OR REPLACE FUNCTION custom_conflict_resolver()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.priority > OLD.priority THEN
        RETURN NEW;
    ELSE
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

**Переваги:** відсутність єдиної точки відмови; розподіл навантаження запису між кількома серверами; географічний розподіл із локальним записом у кожному регіоні.

**Недоліки:** складність розв'язання конфліктів при одночасних змінах; потенційні проблеми консистентності через затримку реплікації; висока складність налаштування й підтримки.

## Консенсус у розподілених системах

Консенсус — процес досягнення згоди між вузлами розподіленої системи про стан даних або послідовність операцій. Саме алгоритми консенсусу лежать в основі того, як master-master та шардовані розподілені системи вирішують, «чия версія даних правильна» без єдиного централізованого арбітра.

### Paxos алгоритм

Paxos — один із перших формалізованих алгоритмів консенсусу, розроблений Леслі Лампортом у 1989 році.

Основні ролі: **Proposer** пропонує значення для досягнення консенсусу; **Acceptor** приймає або відхиляє пропозиції; **Learner** дізнається про досягнутий консенсус.

Фази: фаза підготовки (proposer відправляє prepare-запит із номером пропозиції, acceptor обіцяє не приймати пропозиції з меншим номером), фаза прийняття (accept-запит зі значенням, acceptor приймає пропозицію, якщо не обіцяв підтримувати вищу), фаза фіксації (більшість acceptors прийняла пропозицію, learners дізнаються про консенсус).

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class MessageType(Enum):
    PREPARE = 1
    PROMISE = 2
    ACCEPT = 3
    ACCEPTED = 4

@dataclass
class Message:
    msg_type: MessageType
    proposal_number: int
    value: Optional[any] = None
    accepted_number: Optional[int] = None
    accepted_value: Optional[any] = None

class Acceptor:
    def __init__(self, node_id):
        self.node_id = node_id
        self.promised_number = 0
        self.accepted_number = 0
        self.accepted_value = None

    def handle_prepare(self, proposal_number):
        if proposal_number > self.promised_number:
            self.promised_number = proposal_number
            return Message(
                MessageType.PROMISE,
                proposal_number,
                accepted_number=self.accepted_number,
                accepted_value=self.accepted_value
            )
        return None

    def handle_accept(self, proposal_number, value):
        if proposal_number >= self.promised_number:
            self.promised_number = proposal_number
            self.accepted_number = proposal_number
            self.accepted_value = value
            return Message(
                MessageType.ACCEPTED,
                proposal_number,
                value=value
            )
        return None

class Proposer:
    def __init__(self, node_id, acceptors):
        self.node_id = node_id
        self.acceptors = acceptors
        self.proposal_number = 0

    def propose(self, value):
        self.proposal_number += 1

        # Фаза 1: Prepare
        promises = []
        for acceptor in self.acceptors:
            response = acceptor.handle_prepare(self.proposal_number)
            if response:
                promises.append(response)

        if len(promises) <= len(self.acceptors) // 2:
            return None, "Не досягнуто кворуму на prepare"

        proposed_value = value
        max_accepted = max(
            (p.accepted_number for p in promises if p.accepted_number > 0),
            default=0
        )
        if max_accepted > 0:
            for p in promises:
                if p.accepted_number == max_accepted:
                    proposed_value = p.accepted_value
                    break

        # Фаза 2: Accept
        accepted = []
        for acceptor in self.acceptors:
            response = acceptor.handle_accept(self.proposal_number, proposed_value)
            if response:
                accepted.append(response)

        if len(accepted) > len(self.acceptors) // 2:
            return proposed_value, "Консенсус досягнуто"

        return None, "Не досягнуто кворуму на accept"

acceptors = [Acceptor(i) for i in range(5)]
proposer = Proposer(1, acceptors)

value, status = proposer.propose("initial_value")
print(f"Результат: {value}, Статус: {status}")
```

### Raft алгоритм

Raft розроблений як більш зрозуміла альтернатива Paxos з акцентом на практичність та освітню цінність.

Ролі вузлів: **Leader** приймає запити від клієнтів та реплікує зміни на followers; **Follower** пасивно приймає оновлення від leader; **Candidate** намагається стати leader під час виборів.

Фази: вибори leader (follower не отримує heartbeat протягом таймауту → стає candidate → запитує голоси → вузол із більшістю голосів стає leader), реплікація логу (leader отримує запит від клієнта → додає запис до свого логу → реплікує на followers → підтверджує клієнту після реплікації на більшість).

```python
import time
import random
from enum import Enum
from threading import Thread, Lock

class NodeState(Enum):
    FOLLOWER = 1
    CANDIDATE = 2
    LEADER = 3

class RaftNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.state = NodeState.FOLLOWER

        self.current_term = 0
        self.voted_for = None
        self.log = []

        self.commit_index = 0
        self.last_applied = 0

        self.next_index = {}
        self.match_index = {}

        self.election_timeout = random.uniform(1.5, 3.0)
        self.last_heartbeat = time.time()

        self.lock = Lock()

    def start_election(self):
        with self.lock:
            self.state = NodeState.CANDIDATE
            self.current_term += 1
            self.voted_for = self.node_id
            votes = 1

            print(f"Node {self.node_id} починає вибори для терміну {self.current_term}")

            for peer in self.peers:
                if peer.request_vote(
                    self.current_term,
                    self.node_id,
                    len(self.log),
                    self.log[-1]['term'] if self.log else 0
                ):
                    votes += 1

            if votes > (len(self.peers) + 1) // 2:
                self.become_leader()

    def become_leader(self):
        print(f"Node {self.node_id} став leader для терміну {self.current_term}")
        self.state = NodeState.LEADER

        for peer in self.peers:
            self.next_index[peer.node_id] = len(self.log)
            self.match_index[peer.node_id] = 0

        self.send_heartbeat()

    def request_vote(self, term, candidate_id, last_log_index, last_log_term):
        with self.lock:
            if term < self.current_term:
                return False

            if term > self.current_term:
                self.current_term = term
                self.state = NodeState.FOLLOWER
                self.voted_for = None

            if self.voted_for is None or self.voted_for == candidate_id:
                my_last_log_term = self.log[-1]['term'] if self.log else 0
                my_last_log_index = len(self.log)

                if (last_log_term > my_last_log_term or
                    (last_log_term == my_last_log_term and
                     last_log_index >= my_last_log_index)):
                    self.voted_for = candidate_id
                    self.last_heartbeat = time.time()
                    return True

            return False

    def append_entries(self, term, leader_id, prev_log_index, prev_log_term, entries, leader_commit):
        with self.lock:
            if term < self.current_term:
                return False

            if term > self.current_term:
                self.current_term = term
                self.state = NodeState.FOLLOWER

            self.last_heartbeat = time.time()

            if prev_log_index > 0:
                if len(self.log) < prev_log_index or \
                   self.log[prev_log_index - 1]['term'] != prev_log_term:
                    return False

            for i, entry in enumerate(entries):
                index = prev_log_index + i
                if len(self.log) > index:
                    if self.log[index]['term'] != entry['term']:
                        self.log = self.log[:index]
                        self.log.append(entry)
                else:
                    self.log.append(entry)

            if leader_commit > self.commit_index:
                self.commit_index = min(leader_commit, len(self.log))

            return True

    def send_heartbeat(self):
        if self.state != NodeState.LEADER:
            return

        for peer in self.peers:
            prev_log_index = self.next_index[peer.node_id] - 1
            prev_log_term = self.log[prev_log_index]['term'] if prev_log_index >= 0 else 0

            entries = self.log[self.next_index[peer.node_id]:]

            success = peer.append_entries(
                self.current_term,
                self.node_id,
                prev_log_index,
                prev_log_term,
                entries,
                self.commit_index
            )

            if success:
                self.next_index[peer.node_id] = len(self.log)
                self.match_index[peer.node_id] = len(self.log) - 1
            else:
                self.next_index[peer.node_id] -= 1

    def run(self):
        while True:
            if self.state == NodeState.LEADER:
                self.send_heartbeat()
                time.sleep(0.5)
            else:
                if time.time() - self.last_heartbeat > self.election_timeout:
                    self.start_election()
                time.sleep(0.1)

# Створення кластера з 5 вузлів
nodes = [RaftNode(i, []) for i in range(5)]

for node in nodes:
    node.peers = [n for n in nodes if n.node_id != node.node_id]

for node in nodes:
    thread = Thread(target=node.run, daemon=True)
    thread.start()
```

### Порівняння Paxos та Raft

| Критерій | Paxos | Raft |
|---|---|---|
| Складність розуміння | Складніший для розуміння та імплементації | Спроєктований для зрозумілості |
| Практичність імплементації | Багато варіацій та оптимізацій | Чітка специфікація, менша варіативність |
| Продуктивність | Подібна теоретична продуктивність | Подібна теоретична продуктивність |
| Використання в індустрії | Google Chubby; ZooKeeper (алгоритм Zab, споріднений) | etcd, Consul, CockroachDB, TiDB |

---

# Частина III. Конвергенція: NewSQL та розподілений SQL у хмарі

NewSQL — клас СУБД, що намагається поєднати ACID-гарантії традиційних SQL-баз даних із горизонтальною масштабованістю NoSQL-систем. Основні цілі NewSQL: збереження SQL-інтерфейсу та ACID-гарантій, підтримка горизонтального масштабування та високої доступності, оптимізація для сучасного обладнання з багатьма ядрами й великими обсягами RAM.

Саме тут сходяться обидві частини цієї лекції: NewSQL-системи — це, по суті, розподілені архітектури (шардинг + реплікація + консенсус за алгоритмом Raft чи Paxos-подібним протоколом), запаковані провайдерами у зручну DBaaS-модель із оплатою за використання.

**Google Cloud Spanner** забезпечує глобальну консистентність із горизонтальним масштабуванням, використовуючи атомарні годинники (TrueTime) для синхронізації часу між дата-центрами по всьому світу — це дозволяє уникнути класичного компромісу CAP-теореми ціною спеціалізованого апаратного забезпечення.

**CockroachDB** сумісний із PostgreSQL wire protocol, використовує Raft для консенсусу, підтримує geo-distributed deployment «з коробки».

```sql
-- Створення бази даних з реплікацією
CREATE DATABASE e_commerce;

-- Встановлення політики реплікації
ALTER DATABASE e_commerce CONFIGURE ZONE USING num_replicas = 3;

-- Створення таблиці з партиціонуванням
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING UNIQUE NOT NULL,
    name STRING NOT NULL,
    country STRING NOT NULL,
    created_at TIMESTAMP DEFAULT current_timestamp(),
    INDEX idx_country (country)
) PARTITION BY LIST (country) (
    PARTITION europe VALUES IN ('UK', 'DE', 'FR', 'UA'),
    PARTITION americas VALUES IN ('US', 'CA', 'BR'),
    PARTITION asia VALUES IN ('JP', 'CN', 'IN')
);

-- Налаштування розміщення партицій
ALTER PARTITION europe OF INDEX users@primary
    CONFIGURE ZONE USING constraints = '[+region=eu-west]';

ALTER PARTITION americas OF INDEX users@primary
    CONFIGURE ZONE USING constraints = '[+region=us-east]';

ALTER PARTITION asia OF INDEX users@primary
    CONFIGURE ZONE USING constraints = '[+region=asia-pacific]';

-- Створення таблиці замовлень з колокацією
CREATE TABLE orders (
    order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    order_date TIMESTAMP DEFAULT current_timestamp(),
    total_amount DECIMAL(10, 2) NOT NULL,
    country STRING NOT NULL,
    INDEX idx_user (user_id),
    INDEX idx_date (order_date)
) PARTITION BY LIST (country) (
    PARTITION europe VALUES IN ('UK', 'DE', 'FR', 'UA'),
    PARTITION americas VALUES IN ('US', 'CA', 'BR'),
    PARTITION asia VALUES IN ('JP', 'CN', 'IN')
);

-- Транзакція через кілька партицій
BEGIN;

INSERT INTO users (email, name, country)
VALUES ('user@example.com', 'John Doe', 'US');

INSERT INTO orders (user_id, total_amount, country)
VALUES (
    (SELECT user_id FROM users WHERE email = 'user@example.com'),
    99.99,
    'US'
);

COMMIT;
```

```sql
-- Перевірка розподілу реплік
SHOW RANGES FROM TABLE users;

-- Аналіз продуктивності розподілених запитів
EXPLAIN ANALYZE SELECT u.name, COUNT(o.order_id)
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.country = 'US'
GROUP BY u.name;

-- Перегляд статистики реплікації
SELECT
    range_id,
    start_key,
    end_key,
    replicas,
    lease_holder
FROM crdb_internal.ranges
WHERE table_name = 'users';
```

### Найновіший етап конвергенції: розподілений SQL як DBaaS

За останній рік NewSQL із нішевої академічної теми перетворилась на мейнстрімну пропозицію всіх трьох гіперскейлерів, повністю доступну як звичайна керована послуга (DBaaS):

- **Amazon Aurora DSQL** (GA, травень 2025) — serverless, PostgreSQL-сумісна, активна-активна мультирегіональна база даних. На відміну від класичної Aurora (один регіон запису + репліки для читання), DSQL дозволяє приймати запис одночасно в кількох регіонах, автоматично вирішуючи конфлікти через оптимістичний контроль конкурентності на рівні транзакцій.
- **Azure HorizonDB** (анонс, кінець 2025) — відповідь Microsoft на Spanner та Aurora DSQL, орієнтована на глобально розподілені транзакційні навантаження.
- **YugabyteDB** — ще одна PostgreSQL-сумісна розподілена СУБД з відкритим кодом, яка використовує модифікований Raft-консенсус (подібно до CockroachDB).

Це означає, що межа між «просто хмарною керованою базою даних» (DBaaS із Частини I) і «розподіленою архітектурою зі складним консенсусом» (Частина II) для розробника практично зникає: сучасний інженер підключається до розподіленого SQL так само просто, як до звичайного PostgreSQL, а вся складність шардингу, реплікації та консенсусу захована всередині керованого сервісу.

## Висновки

Хмарні бази даних та модель Database-as-a-Service фундаментально змінили підходи до управління даними в сучасних інформаційних системах. Три моделі хмарних обчислень (IaaS, PaaS, SaaS) надають різні рівні контролю та відповідальності, а архітектурні патерни хмарних СУБД — багатоорендність, еластичність, оплата за використання — забезпечують економічну ефективність і масштабованість, недосяжні в традиційних підходах.

Але ці зручності стають можливими лише завдяки розподіленим архітектурам, що працюють «під капотом» будь-якого хмарного сервісу. Вибір між вертикальним і горизонтальним масштабуванням залежить від специфічних вимог проєкту, бюджету та очікуваних темпів зростання. Стратегії шардингу (діапазонний, хешований, директорний) дозволяють розподілити дані між множиною серверів, кожна — зі своїми перевагами й недоліками. Реплікація забезпечує відмовостійкість і розподіл навантаження читання, а вибір між синхронною й асинхронною реплікацією визначає баланс між консистентністю та продуктивністю. Алгоритми консенсусу — Paxos та Raft — забезпечують узгодженість стану в розподілених системах без єдиної точки контролю.

Найпоказовіший тренд останніх двох років — це остаточна конвергенція обох тем: NewSQL і розподілений SQL (Cloud Spanner, CockroachDB, Aurora DSQL, Azure HorizonDB) поєднують ACID-гарантії традиційних реляційних СУБД з горизонтальною масштабованістю та географічним розподілом — і подаються провайдерами саме як звичайна DBaaS-послуга. Для сучасного фахівця з програмної інженерії це означає, що розуміння хмарних моделей обслуговування та розуміння розподілених архітектур — це вже не дві окремі компетенції, а одна: щоб ефективно обирати й експлуатувати хмарну базу даних, потрібно розуміти, які компроміси (CAP, консистентність реплікації, стратегія шардингу) ховаються за зручним інтерфейсом керованого сервісу.
