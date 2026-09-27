# Лекція 13 Безпека та адміністрування СУБД

## Вступ

Адміністрування бази даних — це не лише налаштування реплікації чи вибір хмарного провайдера, про які йшлося в попередній лекції. Це насамперед щоденна відповідальність за те, щоб система залишалася (1) захищеною від зловмисників і витоків та (2) продуктивною під реальним навантаженням. Ці дві задачі — безпека та моніторинг продуктивності — традиційно викладаються окремо, але на практиці адміністратор бази даних (DBA) або інженер із надійності даних (Database Reliability Engineer, DBRE) виконує їх у зв'язці: одна й та сама інфраструктура логування використовується і для аудиту безпеки, і для аналізу повільних запитів; один і той самий Prometheus/Grafana-стек збирає метрики як про підозрілі спроби входу, так і про використання CPU.

Статистика кіберзлочинів показує, що витік даних може коштувати компаніям мільйони доларів у вигляді штрафів, судових позовів та втрати репутації. Регуляторні вимоги — GDPR у Європі чи HIPAA у США — встановлюють суворі стандарти захисту даних зі значними штрафами за порушення. Водночас повільні запити безпосередньо впливають на бізнес-показники: затримки в роботі застосунків знижують конверсію та задоволеність користувачів. Тому ця лекція розглядає обидві теми послідовно, а в підсумку показує, як вони сходяться в єдиній практиці адміністрування.

## Частина I. Безпека баз даних

### Моделі контролю доступу

Контроль доступу визначає, які користувачі або процеси мають право виконувати певні операції з даними. Існують три основні моделі організації контролю доступу.

#### Дискреційна модель контролю доступу (DAC)

Дискреційна модель надає власникам об'єктів право самостійно визначати права доступу для інших користувачів. Це найпоширеніша модель у комерційних СУБД: кожен об'єкт має власника, який може надавати або відкликати права доступу іншим користувачам, і ці права можуть передаватися далі, якщо це явно дозволено.

```sql
-- Створення ролей користувачів
CREATE ROLE app_admin LOGIN PASSWORD 'secure_admin_password';
CREATE ROLE app_developer LOGIN PASSWORD 'secure_dev_password';
CREATE ROLE app_readonly LOGIN PASSWORD 'secure_readonly_password';

CREATE DATABASE application_db OWNER app_admin;

\c application_db

CREATE SCHEMA app_data AUTHORIZATION app_admin;
CREATE SCHEMA app_logs AUTHORIZATION app_admin;

CREATE TABLE app_data.users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Надання прав розробникам
GRANT CONNECT ON DATABASE application_db TO app_developer;
GRANT USAGE ON SCHEMA app_data TO app_developer;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app_data TO app_developer;

-- Надання прав тільки для читання
GRANT CONNECT ON DATABASE application_db TO app_readonly;
GRANT USAGE ON SCHEMA app_data TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA app_data TO app_readonly;

-- Автоматичне надання прав для нових таблиць
ALTER DEFAULT PRIVILEGES IN SCHEMA app_data
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_developer;

-- Надання прав з можливістю передачі
GRANT SELECT ON app_data.users TO app_developer WITH GRANT OPTION;
```

Перегляд поточних прав доступу:

```sql
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type,
    is_grantable
FROM information_schema.role_table_grants
WHERE table_schema = 'app_data'
ORDER BY table_name, grantee;
```

**Переваги DAC:** гнучкість управління (власники самостійно контролюють свої об'єкти), простота розуміння, природна відповідність бізнес-процесам.

**Недоліки DAC:** складність централізованого контролю через розподілене управління правами; ризик несанкціонованого поширення прав при необережному використанні `WITH GRANT OPTION`; складність аудиту.

#### Мандатна модель контролю доступу (MAC)

Мандатна модель використовує централізовано визначені правила для контролю доступу на основі міток конфіденційності: рівнів класифікації даних (публічні, внутрішні, конфіденційні, таємні) та рівнів допуску користувачів. Правила встановлюються централізовано й не можуть бути змінені власниками об'єктів.

```sql
CREATE TABLE documents (
    document_id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    classification_level INTEGER NOT NULL CHECK (classification_level BETWEEN 1 AND 4),
    created_by VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- 1 - Public, 2 - Internal, 3 - Confidential, 4 - Secret

CREATE TABLE user_clearance (
    username VARCHAR(50) PRIMARY KEY,
    clearance_level INTEGER NOT NULL CHECK (clearance_level BETWEEN 1 AND 4),
    department VARCHAR(100)
);

ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Політика read-down: користувач читає дані свого рівня та нижче
CREATE POLICY mac_read_down ON documents
    FOR SELECT
    USING (
        classification_level <= (
            SELECT clearance_level FROM user_clearance WHERE username = current_user
        )
    );

-- Політика write-up: користувач створює дані лише свого рівня
CREATE POLICY mac_write_level ON documents
    FOR INSERT
    WITH CHECK (
        classification_level = (
            SELECT clearance_level FROM user_clearance WHERE username = current_user
        )
    );
```

**Переваги MAC:** централізований контроль політики безпеки, автоматичне застосування правил, відповідність військовим/урядовим стандартам.

**Недоліки MAC:** складність впровадження через необхідність класифікації всіх даних, менша гнучкість, потенційно надмірні обмеження.

#### Рольова модель контролю доступу (RBAC)

Рольова модель організовує права доступу навколо ролей, які відображають посади чи функції в організації. Це найпоширеніший компроміс між гнучкістю DAC і суворістю MAC.

```sql
CREATE ROLE sales_role;
CREATE ROLE finance_role;
CREATE ROLE hr_role;
CREATE ROLE management_role;

CREATE SCHEMA sales;
CREATE SCHEMA finance;
CREATE SCHEMA hr;

GRANT USAGE ON SCHEMA sales TO sales_role;
GRANT SELECT, INSERT, UPDATE ON sales.customers, sales.orders TO sales_role;

GRANT USAGE ON SCHEMA sales, finance TO finance_role;
GRANT SELECT ON sales.customers, sales.orders TO finance_role;
GRANT SELECT, INSERT, UPDATE ON finance.invoices TO finance_role;

-- Ієрархія ролей з наслідуванням
CREATE ROLE employee_base;
CREATE ROLE supervisor INHERIT;
CREATE ROLE department_head INHERIT;
CREATE ROLE executive INHERIT;

GRANT employee_base TO supervisor;
GRANT supervisor TO department_head;
GRANT department_head TO executive;

-- Створення користувача з кількома ролями
CREATE ROLE alice_manager LOGIN PASSWORD 'secure_password';
GRANT management_role, sales_role, finance_role TO alice_manager;
```

Динамічне перемикання ролей у сесії дозволяє одному користувачу з кількома ролями явно обмежити свої поточні права:

```sql
SET ROLE sales_role;
SELECT current_role; -- sales_role

-- Спроба доступу до HR даних буде заблокована
SELECT * FROM hr.employees; -- ERROR: permission denied

RESET ROLE;
```

**Переваги RBAC:** простота адміністрування через управління ролями, а не окремими користувачами; відповідність організаційній структурі; легкість аудиту.

**Недоліки RBAC:** можлива надмірність прав, коли ролі занадто широкі; складнощі при нестандартних вимогах; потреба періодичного перегляду ролей.

### Загрози безпеці баз даних

#### SQL ін'єкції

SQL ін'єкція — одна з найнебезпечніших вразливостей веб-застосунків, що дозволяє виконувати довільний SQL код через недостатню валідацію користувацького вводу.

```python
# НЕБЕЗПЕЧНИЙ КОД — конкатенація рядків
def get_user_vulnerable(username, password):
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
    cursor.execute(query)
    return cursor.fetchone()

# Атака: username = "admin' --"
# Результуючий запит:
# SELECT * FROM users WHERE username = 'admin' --' AND password = '...'
# Частина після -- коментується, пароль не перевіряється!
```

Типи SQL ін'єкцій: **Union-based** (витягування даних з інших таблиць через `UNION SELECT`), **Boolean-based blind** (побітове витягування пароля через true/false відповіді), **Time-based blind** (використання `SLEEP()` для витягування інформації через затримки), **Second-order** (шкідливий код зберігається в БД і виконується пізніше в іншому запиті).

**Захист — параметризовані запити:**

```python
# БЕЗПЕЧНИЙ КОД — параметризований запит
def get_user_secure(username, password):
    query = "SELECT * FROM users WHERE username = %s AND password = %s"
    cursor.execute(query, (username, password))
    return cursor.fetchone()
```

ORM (SQLAlchemy й аналогічні) автоматично параметризує запити, надаючи додатковий рівень захисту. Додатково варто застосовувати валідацію та санітизацію вводу (обмеження довжини, whitelist-перевірка формату), а на рівні бази — принцип найменших привілеїв:

```sql
CREATE ROLE app_user LOGIN PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE app_db TO app_user;
GRANT SELECT, INSERT, UPDATE ON users, orders TO app_user;
REVOKE CREATE ON SCHEMA public FROM app_user;

-- Використання представлень замість прямого доступу до таблиць
CREATE VIEW user_safe_view AS
SELECT user_id, username, email, created_at FROM users;

GRANT SELECT ON user_safe_view TO app_user;
REVOKE SELECT ON users FROM app_user;
```

#### Несанкціонований доступ

Несанкціонований доступ може відбуватися через слабкі паролі, викрадені облікові дані, неправильно налаштовані права або експлуатацію вразливостей. Двофакторна автентифікація (2FA/MFA) — стандартний спосіб додаткового захисту:

```python
import pyotp

class TwoFactorAuth:
    def __init__(self, secret_key=None):
        self.secret_key = secret_key or pyotp.random_base32()
        self.totp = pyotp.TOTP(self.secret_key)

    def get_provisioning_uri(self, username, issuer='MyApp'):
        return self.totp.provisioning_uri(name=username, issuer_name=issuer)

    def verify_token(self, token):
        return self.totp.verify(token, valid_window=1)
```

Обмеження спроб входу (rate limiting) і блокування облікового запису після кількох невдалих спроб — стандартна практика проти brute-force атак, зазвичай реалізована через Redis-лічильники з TTL:

```python
class LoginRateLimiter:
    def __init__(self, redis_client, max_attempts=5, lockout_minutes=15):
        self.redis = redis_client
        self.max_attempts = max_attempts
        self.lockout_duration = timedelta(minutes=lockout_minutes)

    def record_failed_attempt(self, identifier):
        attempt_key = f"login:{identifier}:attempts"
        attempts = self.redis.incr(attempt_key)
        self.redis.expire(attempt_key, 300)

        if attempts >= self.max_attempts:
            self.redis.setex(f"lockout:{identifier}",
                              int(self.lockout_duration.total_seconds()), '1')
            return True
        return False
```

Виявлення аномальної активності — множинних невдалих спроб з різних IP, входу з нетипової локації чи «неможливих подорожей» (вхід з двох віддалених локацій за короткий проміжок часу) — реалізується через аналітичні запити над журналом автентифікації:

```sql
CREATE TABLE authentication_logs (
    log_id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    ip_address INET,
    success BOOLEAN,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    geolocation JSONB
);

-- Виявлення "неможливих подорожей"
SELECT a1.username, a1.geolocation, a2.geolocation,
       EXTRACT(EPOCH FROM (a2.timestamp - a1.timestamp))/60 AS time_diff_minutes
FROM authentication_logs a1
JOIN authentication_logs a2 ON a1.username = a2.username
WHERE a1.success AND a2.success
    AND a1.timestamp < a2.timestamp
    AND a2.timestamp - a1.timestamp < INTERVAL '2 hours'
    AND a1.geolocation->>'country' != a2.geolocation->>'country';
```

> **Актуалізація:** PostgreSQL 18 (2025) додав вбудовану підтримку **OAuth 2.0 автентифікації** (метод `oauth` у `pg_hba.conf`), що дозволяє інтегрувати вхід у базу даних із зовнішніми identity-провайдерами (Azure AD, Okta, Google) без власної реалізації MFA на рівні застосунку — це стає рекомендованою практикою замість домашніх рішень на кшталт наведеного вище прикладу з `pyotp`.

#### Витоки даних

Витоки можуть відбуватися через зовнішні атаки, внутрішні загрози, неправильні налаштування безпеки або помилки в коді. Ключова стратегія запобігання — Data Loss Prevention, зокрема маскування чутливих даних для непривілейованих ролей:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE OR REPLACE FUNCTION mask_credit_card(card_number TEXT)
RETURNS TEXT AS $$
BEGIN
    IF card_number IS NULL OR length(card_number) < 4 THEN
        RETURN '****';
    END IF;
    RETURN '****-****-****-' || right(card_number, 4);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

CREATE VIEW customers_masked AS
SELECT customer_id, first_name, last_name,
       mask_credit_card(credit_card_number) as credit_card
FROM customers;

GRANT SELECT ON customers_masked TO analyst_role;
REVOKE ALL ON customers FROM analyst_role;
```

Аналіз підозрілого доступу (масове читання, доступ у незвичний час, доступ до нетипових таблиць) виявляє потенційну ексфільтрацію даних до того, як вона завдасть шкоди:

```sql
-- Масове читання даних одним користувачем за годину
SELECT username, COUNT(*) as record_count
FROM sensitive_data_access_log
WHERE timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
    AND operation = 'SELECT'
GROUP BY username
HAVING COUNT(*) > 1000;
```

### Криптографічні методи захисту

#### Шифрування даних у спокої (Encryption at Rest)

Шифрування на рівні стовпців захищає найчутливіші дані, зберігаючи можливість пошуку по незашифрованих полях:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE secure_customers (
    customer_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    ssn_encrypted BYTEA,
    credit_card_encrypted BYTEA,
    country VARCHAR(2)
);

CREATE OR REPLACE FUNCTION encrypt_data(plain_text TEXT, encryption_key TEXT)
RETURNS BYTEA AS $$
BEGIN
    RETURN pgp_sym_encrypt(plain_text, encryption_key);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE OR REPLACE FUNCTION decrypt_data(encrypted_data BYTEA, encryption_key TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN pgp_sym_decrypt(encrypted_data, encryption_key);
EXCEPTION WHEN OTHERS THEN
    RETURN NULL; -- + логування спроби несанкціонованого розшифрування
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

Управління ключами шифрування в продакшн-середовищі варто доручати спеціалізованим сервісам — **AWS KMS, Azure Key Vault, HashiCorp Vault** — з регулярною ротацією ключів, а не зберігати їх у файлах `.env`, як це часто робиться в навчальних прикладах.

Transparent Data Encryption (TDE) шифрує всю базу даних на рівні файлової системи без змін у коді застосунку — наприклад, через `LUKS`-шифрований розділ диска або вбудовану підтримку шифрування в MySQL (`ENCRYPTION='Y'`).

#### Шифрування даних при передачі (Encryption in Transit)

```conf
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_min_protocol_version = 'TLSv1.2'
```

```conf
# pg_hba.conf — вимагати SSL для віддалених з'єднань
hostssl all  all  0.0.0.0/0  scram-sha-256
hostnossl all all 0.0.0.0/0  reject
```

```python
conn_params = {
    'host': 'database.example.com',
    'sslmode': 'verify-full',  # вимагає перевірку сертифіката
    'sslrootcert': '/path/to/root.crt'
}
conn = psycopg2.connect(**conn_params)
```

> **Актуалізація:** для методу автентифікації в `pg_hba.conf` варто використовувати сучасний `scram-sha-256` замість застарілого `md5` (застарілий метод все ще трапляється в старих навчальних матеріалах, але вважається менш безпечним і поступово виводиться з використання).

### Аудит та резервне копіювання

#### Комплексний аудит операцій

```sql
CREATE TABLE comprehensive_audit_log (
    audit_id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    database_user VARCHAR(50),
    operation_type VARCHAR(20),
    object_name VARCHAR(100),
    old_values JSONB,
    new_values JSONB,
    client_ip INET,
    query_text TEXT,
    success BOOLEAN
) PARTITION BY RANGE (timestamp);

CREATE OR REPLACE FUNCTION comprehensive_audit()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO comprehensive_audit_log (
        database_user, operation_type, object_name,
        old_values, new_values, client_ip, success
    ) VALUES (
        current_user, TG_OP, TG_TABLE_NAME,
        CASE WHEN TG_OP IN ('UPDATE','DELETE') THEN row_to_json(OLD)::JSONB END,
        CASE WHEN TG_OP IN ('INSERT','UPDATE') THEN row_to_json(NEW)::JSONB END,
        inet_client_addr(), true
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION comprehensive_audit();
```

> Зауважте партиціонування таблиці аудиту за часом (`PARTITION BY RANGE (timestamp)`) — журнали аудиту швидко зростають, і без партиціонування запити до них деградують так само, як описано в Частині II про продуктивність.

#### Резервне копіювання та відновлення

Комплексна стратегія поєднує кілька типів бекапів: повний (`pg_dump`, щоденно), інкрементальний (`pg_basebackup`, кожні кілька годин) та безперервне архівування WAL (Write-Ahead Log) для Point-in-Time Recovery.

```bash
# Повне резервне копіювання
pg_dump -h $DB_HOST -U $DB_USER -Fc -f "$BACKUP_FILE" $DB_NAME
gzip "$BACKUP_FILE"
aws s3 cp "$BACKUP_FILE.gz" "$S3_BUCKET/full/"

# Point-in-Time Recovery — відновлення на конкретний момент часу
cat > /var/lib/postgresql/17/main/recovery.signal << EOF
restore_command = 'cp /var/lib/postgresql/17/wal_archive/%f %p'
recovery_target_time = '2026-01-15 14:30:00'
recovery_target_action = 'promote'
EOF
```

Автоматизація через `cron` та обов'язкове **тестове відновлення** (спроба розгорнути бекап у тестову базу й перевірити цілісність) — критично важлива, «непротестований бекап — це не бекап»:

```bash
0 2 * * * /usr/local/bin/backup_postgresql.sh full
0 3 * * 0 /usr/local/bin/verify_backups.sh
```

> **Актуалізація:** для production-середовищ у 2025–2026 роках стандартом де-факто стає **pgBackRest** замість «саморобних» bash-скриптів навколо `pg_dump`/`pg_basebackup` — він з коробки підтримує паралельне резервне копіювання, інкрементальні/диференційні бекапи, стиснення, шифрування та перевірку цілісності, суттєво знижуючи ризик людської помилки в критично важливій процедурі відновлення.

Дотримуйтеся правила **3-2-1**: щонайменше 3 копії даних, на 2 різних типах носіїв, 1 копія поза межами основного дата-центру (наприклад, в іншому хмарному регіоні).

## Частина II. Моніторинг продуктивності та налагодження

### Метрики продуктивності СУБД

#### Пропускна здатність (Throughput)

Пропускна здатність вимірює кількість операцій, які система обробляє за одиницю часу (TPS — транзакції за секунду, QPS — запити за секунду).

```sql
CREATE TABLE performance_metrics (
    metric_id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metric_name VARCHAR(100),
    metric_value NUMERIC,
    database_name VARCHAR(100)
);

CREATE OR REPLACE FUNCTION collect_throughput_metrics()
RETURNS void AS $$
DECLARE
    current_stats RECORD;
BEGIN
    SELECT xact_commit + xact_rollback as total_transactions
    INTO current_stats
    FROM pg_stat_database WHERE datname = current_database();

    INSERT INTO performance_metrics (metric_name, metric_value, database_name)
    VALUES ('total_transactions', current_stats.total_transactions, current_database());
END;
$$ LANGUAGE plpgsql;

-- Автоматичний збір кожні 60 секунд через pg_cron
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('collect-metrics', '60 seconds', 'SELECT collect_throughput_metrics()');
```

Виявлення аномалій у пропускній здатності через статистичний аналіз (значення поза межами середнє ± 3·стандартне_відхилення):

```sql
WITH stats AS (
    SELECT AVG(metric_value) as mean, STDDEV(metric_value) as stddev
    FROM performance_metrics
    WHERE metric_name = 'transactions_per_second'
        AND timestamp > CURRENT_TIMESTAMP - INTERVAL '7 days'
)
SELECT m.timestamp, m.metric_value,
    CASE
        WHEN m.metric_value > s.mean + 3 * s.stddev THEN 'Unusually High'
        WHEN m.metric_value < s.mean - 3 * s.stddev THEN 'Unusually Low'
        ELSE 'Normal'
    END as status
FROM performance_metrics m CROSS JOIN stats s
WHERE m.metric_name = 'transactions_per_second';
```

#### Час відгуку (Response Time) та персентилі

Час відгуку — критична метрика для користувацького досвіду. Аналіз середнього значення недостатній: важливі саме **персентилі** (P50, P95, P99), оскільки середнє маскує «хвіст» повільних запитів, з якими насправді стикається помітна частка користувачів.

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Найповільніші запити
SELECT
    query, calls, mean_exec_time, max_exec_time,
    100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS cache_hit_ratio
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Запити, що споживають найбільше загального часу (найважливіше для оптимізації!)
SELECT
    query, calls, total_exec_time,
    ROUND(100.0 * total_exec_time / SUM(total_exec_time) OVER (), 2) AS percent_total_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

> Ключова методична порада: сортування за `mean_exec_time` показує «найповільніші» одиничні запити, а сортування за `total_exec_time` показує запити, які найбільше навантажують систему сукупно — часто це «швидкі, але надзвичайно часті» запити, а не поодинокі повільні.

Виявлення деградації продуктивності конкретного запиту порівняно з історичною нормою:

```sql
WITH current_metrics AS (
    SELECT query_hash, mean_time as current_mean
    FROM query_response_time_history
    WHERE timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
),
historical_metrics AS (
    SELECT query_hash, AVG(mean_time) as historical_mean, STDDEV(mean_time) as historical_stddev
    FROM query_response_time_history
    WHERE timestamp BETWEEN CURRENT_TIMESTAMP - INTERVAL '7 days' AND CURRENT_TIMESTAMP - INTERVAL '1 day'
    GROUP BY query_hash
)
SELECT h.query_hash, c.current_mean, h.historical_mean,
    ROUND((c.current_mean - h.historical_mean) / h.historical_mean * 100, 2) as percent_change
FROM current_metrics c
JOIN historical_metrics h USING (query_hash)
WHERE c.current_mean > h.historical_mean + 2 * h.historical_stddev;
```

#### Утилізація ресурсів: CPU, пам'ять, диск

Моніторинг ресурсів на рівні застосунку доповнює SQL-статистику даними операційної системи (бібліотека `psutil` у Python):

```python
import psutil

class ResourceMonitor:
    def __init__(self):
        self.process = psutil.Process()

    def collect_cpu(self):
        return {
            'system_cpu': psutil.cpu_percent(interval=1),
            'process_cpu': self.process.cpu_percent(interval=1)
        }

    def collect_memory(self):
        mem = psutil.virtual_memory()
        return {'used_percent': mem.percent, 'available_gb': mem.available / 1024**3}

    def collect_disk_io(self):
        io = psutil.disk_io_counters()
        return {'read_bytes': io.read_bytes, 'write_bytes': io.write_bytes}
```

На рівні самої СУБД ключові запити для аналізу пам'яті та диска:

```sql
-- Ефективність кешування сторінок
SELECT datname, blks_hit, blks_read,
    ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio
FROM pg_stat_database WHERE datname = current_database();

-- Таблиці з найбільшою кількістю Sequential Scan (потенційна відсутність індексу)
SELECT schemaname, tablename, seq_scan, idx_scan, n_live_tup as estimated_rows
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_scan DESC
LIMIT 20;
```

### Профілювання запитів та виявлення вузьких місць

#### EXPLAIN та EXPLAIN ANALYZE

`EXPLAIN` показує план виконання запиту (оцінки оптимізатора), а `EXPLAIN ANALYZE` фактично виконує запит і додає реальну статистику.

```sql
EXPLAIN ANALYZE
SELECT * FROM customers WHERE city = 'Київ';

/*
Seq Scan on customers (cost=0.00..2084.00 rows=10000 width=...)
  (actual time=0.015..15.234 rows=10000 loops=1)
  Filter: ((city)::text = 'Київ'::text)
  Rows Removed by Filter: 90000
Planning Time: 0.123 ms
Execution Time: 16.456 ms
*/
```

Розширений аналіз з деталями про буфери:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT c.first_name, c.last_name, COUNT(o.order_id) as order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.is_active = true
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING COUNT(o.order_id) > 5
ORDER BY order_count DESC
LIMIT 100;
```

Типові проблеми, які виявляє аналіз плану виконання, і відповідні рекомендації:

| Проблема в плані | Симптом | Рекомендація |
|---|---|---|
| **Sequential Scan** великої таблиці | `actual_rows` > 10 000 без індексу | Додати індекс на стовпці фільтрації |
| **Неточна оцінка рядків** | `actual_rows` суттєво відрізняється від `plan_rows` | Оновити статистику: `ANALYZE table_name;` |
| **Високе дискове читання** | велике значення `Shared Read Blocks` | Збільшити `shared_buffers` / `work_mem` |
| **Nested Loop** із тисячами ітерацій | `actual_loops` > 1000 | Форсувати hash- або merge-join, перевірити індекси на JOIN-стовпцях |

> **Актуалізація:** PostgreSQL 18 (2025) додав **skip scan** для багатостовпцевих B-tree індексів — тепер індекс `(a, b)` може ефективно використовуватися й для запитів, що фільтрують лише за `b` без `a`, чого раніше оптимізатор не робив. Це зменшує потребу створювати додаткові окремі індекси лише заради порядку стовпців.

#### Виявлення N+1 проблеми

N+1 проблема виникає, коли один запит отримує список об'єктів, а потім виконується по одному додатковому запиту для кожного об'єкта — класична пастка ORM за замовчуванням.

```python
# НЕПРАВИЛЬНО — N+1: 1 запит на клієнтів + N запитів на замовлення
for customer in customers:
    cur.execute("SELECT * FROM orders WHERE customer_id = %s", (customer.id,))

# ПРАВИЛЬНО — один запит з JOIN
cur.execute("""
    SELECT c.customer_id, c.first_name, o.order_id, o.total_amount
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    WHERE c.customer_id IN (SELECT customer_id FROM customers LIMIT %s)
""", (limit,))

# АЛЬТЕРНАТИВА — eager loading: 2 запити замість N+1
cur.execute("SELECT customer_id, first_name FROM customers LIMIT %s", (limit,))
customer_ids = [c[0] for c in cur.fetchall()]
cur.execute("SELECT customer_id, order_id FROM orders WHERE customer_id = ANY(%s)", (customer_ids,))
```

На практиці різниця в кількості запитів (N+1 замість 2) при 100 клієнтах означає 101 мережевий round-trip замість 2 — типово в 10–50 разів повільніше залежно від мережевої латентності.

### Інструменти моніторингу

#### Вбудовані представлення PostgreSQL

```sql
CREATE OR REPLACE VIEW system_monitoring AS
SELECT
    (SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active') as active_connections,
    (SELECT COUNT(*) FROM pg_locks WHERE granted = false) as blocked_queries,
    pg_size_pretty(pg_database_size(current_database())) as database_size,
    (SELECT ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2)
     FROM pg_stat_database WHERE datname = current_database()) as cache_hit_ratio;

-- Довгі запити (виконуються понад хвилину)
CREATE OR REPLACE VIEW long_running_queries AS
SELECT pid, now() - query_start as duration, usename, state, LEFT(query, 100) as query_preview
FROM pg_stat_activity
WHERE state != 'idle' AND now() - query_start > interval '1 minute'
ORDER BY duration DESC;

-- Блокування — хто кого блокує
CREATE OR REPLACE VIEW blocking_queries AS
SELECT
    blocked_locks.pid AS blocked_pid, blocking_locks.pid AS blocking_pid,
    blocked_activity.query AS blocked_statement, blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

#### Prometheus та Grafana

Для production-моніторингу вбудованих представлень недостатньо — потрібна система збору й візуалізації метрик у часі з алертингом. Найпоширеніший стек — **Prometheus** (збір і зберігання часових рядів метрик) + **Grafana** (дашборди) через `postgres_exporter`:

```python
from prometheus_client import start_http_server, Gauge

class PostgreSQLExporter:
    def __init__(self, connection_string, port=9187):
        self.conn_string = connection_string
        self.port = port
        self.active_connections = Gauge('postgresql_active_connections', 'Active connections')
        self.cache_hit_ratio = Gauge('postgresql_cache_hit_ratio', 'Cache hit ratio', ['database'])

    def collect_metrics(self):
        conn = psycopg2.connect(self.conn_string)
        cur = conn.cursor()
        cur.execute("SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active'")
        self.active_connections.set(cur.fetchone()[0])
        conn.close()

    def start(self, interval=15):
        start_http_server(self.port)
        while True:
            self.collect_metrics()
            time.sleep(interval)
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'postgresql'
    static_configs:
      - targets: ['localhost:9187']
        labels:
          environment: 'production'
```

Серед готових інструментів варто також згадати **pgAdmin** (графічний адміністративний інтерфейс із вбудованим Query Tool та Dashboard) і **pgBadger** (аналізатор лог-файлів PostgreSQL, що генерує детальні HTML-звіти про повільні запити постфактум).

### Планове обслуговування

#### VACUUM та ANALYZE

`VACUUM` видаляє мертві рядки (наслідок MVCC-моделі PostgreSQL — кожен `UPDATE`/`DELETE` залишає стару версію рядка) і запобігає роздуванню таблиць (*bloat*). `ANALYZE` оновлює статистику для оптимізатора запитів.

```sql
ALTER SYSTEM SET autovacuum = on;
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.1;
ALTER SYSTEM SET autovacuum_analyze_scale_factor = 0.05;
SELECT pg_reload_conf();

-- Моніторинг роздування таблиць
SELECT
    schemaname, tablename,
    n_dead_tup, n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_tuple_percent,
    last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

VACUUM (VERBOSE, ANALYZE) customers;
```

`VACUUM FULL` повністю очищує таблицю, але вимагає ексклюзивного блокування — застосовується лише в спеціально виділені вікна обслуговування, а не в межах регулярної автоматизації.

#### Перебудова індексів та архівування

```sql
-- Виявлення невикористовуваних індексів
SELECT schemaname, tablename, indexname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Перебудова без блокування читання/запису
REINDEX INDEX CONCURRENTLY idx_customers_city;

-- Архівування застарілих даних
CREATE OR REPLACE FUNCTION archive_old_orders()
RETURNS void AS $$
BEGIN
    WITH moved_rows AS (
        DELETE FROM orders WHERE order_date < CURRENT_DATE - INTERVAL '2 years'
        RETURNING *
    )
    INSERT INTO orders_archive SELECT * FROM moved_rows;

    EXECUTE 'VACUUM ANALYZE orders';
END;
$$ LANGUAGE plpgsql;

SELECT cron.schedule('archive-old-orders', '0 3 1 * *', 'SELECT archive_old_orders()');
```

> **Актуалізація:** PostgreSQL 18 запровадив нову асинхронну I/O-підсистему (`io_method = io_uring`), яка суттєво прискорює саме операції масового читання й `VACUUM` на дисках NVMe — вартий згадки орієнтир для тих, хто налаштовує розклад обслуговування на нових серверах.

## Частина III. Де сходяться безпека та продуктивність

Обидві теми цієї лекції, попри різні цілі, спираються на спільну інфраструктуру:

- **Журнали як спільний ресурс.** Таблиця аудиту безпеки (`comprehensive_audit_log`) і таблиця історії метрик продуктивності (`query_response_time_history`) мають однакову структурну проблему — швидке зростання обсягу — і однакове рішення: партиціонування за часом, автоматичне архівування застарілих записів, індексація за `timestamp`.
- **Один стек моніторингу для двох задач.** Prometheus/Grafana, розгорнутий для відстеження CPU й cache hit ratio, з тим самим успіхом збирає й security-метрики (кількість невдалих спроб входу, кількість заблокованих запитів через RLS) — немає потреби в окремій системі алертингу для безпеки.
- **Аномалії виявляються однаково.** Статистичний підхід «значення поза межами середнє ± 3·стандартне_відхилення», застосований вище і до пропускної здатності, і до підозрілих спроб входу, — це той самий метод виявлення аномалій (anomaly detection), застосований до різних метрик.
- **Продуктивність — теж питання безпеки.** Погано оптимізований запит, що виконує `Seq Scan` по мільйону рядків, — це не лише проблема швидкодії, а й потенційний вектор DoS-атаки (зловмисник, що знає про відсутність індексу, може навмисно генерувати такі запити для вичерпання ресурсів сервера).

Ця конвергенція відображає ширшу індустріальну тенденцію: роль класичного DBA, що займався виключно адмініструванням однієї бази даних, поступово трансформується в роль **Database Reliability Engineer (DBRE)** — фахівця, який відповідає за надійність, безпеку та продуктивність даних як за єдину, взаємопов'язану систему, а не три окремі напрями роботи.

## Висновки

Безпека баз даних — багатошарова система захисту: контроль доступу (DAC, MAC, RBAC — кожна модель зі своїм балансом гнучкості й централізації), захист від конкретних загроз (параметризовані запити проти SQL-ін'єкцій, MFA/OAuth проти несанкціонованого доступу, маскування й аудит проти витоку даних), криптографія (шифрування в спокої та при передачі, кероване управління ключами) і надійне резервне копіювання за правилом 3-2-1 з обов'язковим тестуванням відновлення.

Моніторинг продуктивності спирається на систематичний збір метрик (пропускна здатність, час відгуку з увагою до персентилів, а не лише середнього, утилізація ресурсів), профілювання запитів через `EXPLAIN ANALYZE` для виявлення вузьких місць (типово — відсутність індексу, неточна статистика, проблема N+1), спеціалізовані інструменти моніторингу (Prometheus/Grafana для метрик у часі, pgBadger для постфактум-аналізу логів) та планове обслуговування (`VACUUM`/`ANALYZE`, перебудова індексів, архівування).

Найважливіший практичний висновок: безпека й продуктивність — не окремі дисципліни з окремими інструментами, а дві грані однієї функції адміністрування даних, дедалі частіше об'єднані в ролі Database Reliability Engineer, яка спирається на спільну інфраструктуру журналювання, моніторингу та автоматизованого реагування на аномалії.
