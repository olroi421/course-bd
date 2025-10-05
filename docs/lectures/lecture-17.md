# Лекція 17. Безпека та адміністрування баз даних

## Вступ

Безпека баз даних є критично важливим аспектом будь-якої інформаційної системи, оскільки бази даних зберігають найцінніший актив сучасних організацій — інформацію. Від фінансових записів до персональних даних користувачів, від медичних карток до комерційних таємниць — усе це потребує надійного захисту від несанкціонованого доступу, витоку та втрати.

Статистика кіберзлочинів показує, що витік даних може коштувати компаніям мільйони доларів у вигляді штрафів, судових позовів та втрати репутації. Регуляторні вимоги, такі як GDPR в Європі чи HIPAA у США, встановлюють суворі стандарти захисту даних з значними штрафами за порушення.

Адміністрування баз даних охоплює широкий спектр задач від забезпечення безпеки до оптимізації продуктивності, від резервного копіювання до моніторингу стану системи. Ефективне адміністрування є запорукою стабільної, безпечної та продуктивної роботи бази даних.

## Моделі контролю доступу

Контроль доступу визначає, які користувачі або процеси мають право виконувати певні операції з даними. Існують три основні моделі організації контролю доступу, кожна з яких має свої особливості та сфери застосування.

### Дискреційна модель контролю доступу (DAC)

Дискреційна модель надає власникам об'єктів право самостійно визначати права доступу для інших користувачів. Це найпоширеніша модель у комерційних СУБД.

#### Принципи DAC

Основний принцип полягає в тому, що кожен об'єкт має власника, який може надавати або відкликати права доступу іншим користувачам. Права можуть передаватися далі, якщо це явно дозволено.

Реалізація DAC в PostgreSQL:

```sql
-- Створення ролей користувачів
CREATE ROLE app_admin LOGIN PASSWORD 'secure_admin_password';
CREATE ROLE app_developer LOGIN PASSWORD 'secure_dev_password';
CREATE ROLE app_readonly LOGIN PASSWORD 'secure_readonly_password';

-- Створення бази даних
CREATE DATABASE application_db OWNER app_admin;

\c application_db

-- Створення схем
CREATE SCHEMA app_data AUTHORIZATION app_admin;
CREATE SCHEMA app_logs AUTHORIZATION app_admin;

-- Створення таблиць
CREATE TABLE app_data.users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE app_data.orders (
    order_id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES app_data.users(user_id),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending'
);

CREATE TABLE app_logs.access_log (
    log_id SERIAL PRIMARY KEY,
    user_id INTEGER,
    action VARCHAR(50),
    table_name VARCHAR(100),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET
);

-- Надання прав розробникам
GRANT CONNECT ON DATABASE application_db TO app_developer;
GRANT USAGE ON SCHEMA app_data TO app_developer;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app_data TO app_developer;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA app_data TO app_developer;

-- Надання прав тільки для читання
GRANT CONNECT ON DATABASE application_db TO app_readonly;
GRANT USAGE ON SCHEMA app_data TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA app_data TO app_readonly;

-- Автоматичне надання прав для нових таблиць
ALTER DEFAULT PRIVILEGES IN SCHEMA app_data
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_developer;

ALTER DEFAULT PRIVILEGES IN SCHEMA app_data
    GRANT SELECT ON TABLES TO app_readonly;

-- Надання прав з можливістю передачі
GRANT SELECT ON app_data.users TO app_developer WITH GRANT OPTION;
```

Приклад використання прав з передачею:

```sql
-- Користувач app_developer може передати право SELECT іншому користувачу
SET ROLE app_developer;

CREATE ROLE analyst LOGIN PASSWORD 'analyst_password';

-- Передача права, отриманого з WITH GRANT OPTION
GRANT SELECT ON app_data.users TO analyst;

-- Але не може передати права, яких не має з GRANT OPTION
-- Наступний запит викличе помилку
GRANT INSERT ON app_data.users TO analyst; -- ERROR: permission denied
```

Перегляд поточних прав доступу:

```sql
-- Перегляд прав на таблицю
\dp app_data.users

-- Детальна інформація про права
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type,
    is_grantable
FROM information_schema.role_table_grants
WHERE table_schema = 'app_data'
ORDER BY table_name, grantee;

-- Перегляд членства в ролях
SELECT
    r.rolname AS role_name,
    m.rolname AS member_name
FROM pg_roles r
JOIN pg_auth_members am ON r.oid = am.roleid
JOIN pg_roles m ON am.member = m.oid
ORDER BY r.rolname;
```

#### Переваги та недоліки DAC

Переваги дискреційної моделі включають гнучкість управління правами доступу, оскільки власники самостійно контролюють свої об'єкти. Простота розуміння та впровадження робить DAC інтуїтивно зрозумілою для користувачів. Природна відповідність бізнес-процесам дозволяє відображати організаційну структуру в правах доступу.

Недоліки полягають у складності централізованого контролю через розподілене управління правами. Ризик несанкціонованого поширення прав виникає при необережному використанні WITH GRANT OPTION. Складність аудиту та відстеження прав ускладнює виявлення проблем безпеки.

### Мандатна модель контролю доступу (MAC)

Мандатна модель використовує централізовано визначені правила для контролю доступу на основі міток конфіденційності. Суб'єкти та об'єкти мають рівні безпеки, доступ надається на основі порівняння цих рівнів.

#### Принципи MAC

Основні концепції включають рівні класифікації даних, такі як публічні, внутрішні, конфіденційні, таємні. Рівні допуску користувачів визначають максимальний рівень даних, до яких вони мають доступ. Правила контролю доступу встановлюються централізовано та не можуть бути змінені власниками об'єктів.

Реалізація MAC з використанням PostgreSQL та row-level security:

```sql
-- Створення таблиці з рівнями класифікації
CREATE TABLE documents (
    document_id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    classification_level INTEGER NOT NULL CHECK (classification_level BETWEEN 1 AND 4),
    created_by VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Рівні класифікації:
-- 1 - Public (Публічні)
-- 2 - Internal (Внутрішні)
-- 3 - Confidential (Конфіденційні)
-- 4 - Secret (Таємні)

-- Створення таблиці користувачів з рівнями допуску
CREATE TABLE user_clearance (
    username VARCHAR(50) PRIMARY KEY,
    clearance_level INTEGER NOT NULL CHECK (clearance_level BETWEEN 1 AND 4),
    department VARCHAR(100)
);

-- Наповнення даними
INSERT INTO user_clearance (username, clearance_level, department) VALUES
('public_user', 1, 'External'),
('employee', 2, 'Sales'),
('manager', 3, 'Management'),
('security_officer', 4, 'Security');

INSERT INTO documents (title, content, classification_level, created_by) VALUES
('Company Newsletter', 'Public information', 1, 'marketing'),
('Sales Report Q1', 'Internal sales data', 2, 'sales_dept'),
('Strategic Plan', 'Confidential business strategy', 3, 'ceo'),
('Security Protocols', 'Secret security procedures', 4, 'security');

-- Створення політики безпеки на рівні рядків
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Політика read-down: користувач може читати дані свого рівня та нижче
CREATE POLICY mac_read_down ON documents
    FOR SELECT
    USING (
        classification_level <= (
            SELECT clearance_level
            FROM user_clearance
            WHERE username = current_user
        )
    );

-- Політика write-up: користувач може створювати дані свого рівня
CREATE POLICY mac_write_level ON documents
    FOR INSERT
    WITH CHECK (
        classification_level = (
            SELECT clearance_level
            FROM user_clearance
            WHERE username = current_user
        )
    );

-- Політика оновлення: можна змінювати тільки дані свого рівня
CREATE POLICY mac_update_level ON documents
    FOR UPDATE
    USING (
        classification_level = (
            SELECT clearance_level
            FROM user_clearance
            WHERE username = current_user
        )
    );
```

Приклад використання MAC політик:

```sql
-- Користувач з рівнем допуску 2 (Internal)
SET ROLE employee;

-- Може бачити документи рівня 1 та 2
SELECT document_id, title, classification_level
FROM documents;
-- Результат: документи з рівнем 1 та 2

-- Не може бачити документи рівня 3 та 4
SELECT * FROM documents WHERE classification_level > 2;
-- Результат: пусті рядки (політика блокує доступ)

-- Може створити документ рівня 2
INSERT INTO documents (title, content, classification_level, created_by)
VALUES ('Department Report', 'Internal report', 2, 'employee');
-- Успішно

-- Не може створити документ рівня 3
INSERT INTO documents (title, content, classification_level, created_by)
VALUES ('Strategy', 'Confidential plan', 3, 'employee');
-- ERROR: new row violates row-level security policy
```

Моніторинг та аудит MAC:

```sql
-- Створення таблиці аудиту
CREATE TABLE security_audit (
    audit_id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    action VARCHAR(20),
    table_name VARCHAR(100),
    classification_level INTEGER,
    access_granted BOOLEAN,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    details TEXT
);

-- Функція логування спроб доступу
CREATE OR REPLACE FUNCTION log_access_attempt()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO security_audit (
        username,
        action,
        table_name,
        classification_level,
        access_granted,
        details
    ) VALUES (
        current_user,
        TG_OP,
        TG_TABLE_NAME,
        NEW.classification_level,
        true,
        'Document: ' || NEW.title
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Тригер на операції з документами
CREATE TRIGGER audit_document_access
AFTER INSERT OR UPDATE OR DELETE ON documents
FOR EACH ROW EXECUTE FUNCTION log_access_attempt();

-- Перегляд логів безпеки
SELECT
    username,
    action,
    classification_level,
    COUNT(*) as access_count,
    MAX(timestamp) as last_access
FROM security_audit
GROUP BY username, action, classification_level
ORDER BY username, classification_level;
```

#### Переваги та недоліки MAC

Переваги мандатної моделі включають централізований контроль політики безпеки, що забезпечує консистентність. Автоматичне застосування правил зменшує ризик людської помилки. Відповідність військовим та урядовим стандартам безпеки робить MAC обов'язковою для деяких організацій.

Недоліки полягають у складності впровадження та підтримки через необхідність класифікації всіх даних. Менша гнучкість порівняно з DAC обмежує можливості адаптації до специфічних потреб. Потенційно надмірні обмеження можуть ускладнити легітимну роботу користувачів.

### Рольова модель контролю доступу (RBAC)

Рольова модель організовує права доступу навколо ролей, які відображають посади або функції в організації. Користувачі отримують ролі, а ролі мають набори прав.

#### Принципи RBAC

Основні компоненти включають користувачів, які є суб'єктами системи. Ролі представляють набори прав, пов'язані з певними функціями. Права визначають дозволені операції з об'єктами. Сесії встановлюють активні ролі для користувача в даний момент.

Реалізація RBAC в PostgreSQL:

```sql
-- Створення функціональних ролей
CREATE ROLE sales_role;
CREATE ROLE marketing_role;
CREATE ROLE finance_role;
CREATE ROLE hr_role;
CREATE ROLE management_role;

-- Створення схеми для різних відділів
CREATE SCHEMA sales;
CREATE SCHEMA marketing;
CREATE SCHEMA finance;
CREATE SCHEMA hr;

-- Створення таблиць
CREATE TABLE sales.customers (
    customer_id SERIAL PRIMARY KEY,
    company_name VARCHAR(200) NOT NULL,
    contact_person VARCHAR(100),
    email VARCHAR(100),
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sales.orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES sales.customers(customer_id),
    order_date DATE NOT NULL,
    total_amount DECIMAL(12, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending'
);

CREATE TABLE finance.invoices (
    invoice_id SERIAL PRIMARY KEY,
    order_id INTEGER,
    invoice_date DATE NOT NULL,
    due_date DATE NOT NULL,
    amount DECIMAL(12, 2) NOT NULL,
    paid BOOLEAN DEFAULT false
);

CREATE TABLE hr.employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    department VARCHAR(50),
    salary DECIMAL(10, 2),
    hire_date DATE NOT NULL
);

-- Надання прав ролям
-- Sales роль
GRANT USAGE ON SCHEMA sales TO sales_role;
GRANT SELECT, INSERT, UPDATE ON sales.customers TO sales_role;
GRANT SELECT, INSERT, UPDATE ON sales.orders TO sales_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA sales TO sales_role;

-- Finance роль
GRANT USAGE ON SCHEMA sales, finance TO finance_role;
GRANT SELECT ON sales.customers, sales.orders TO finance_role;
GRANT SELECT, INSERT, UPDATE ON finance.invoices TO finance_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA finance TO finance_role;

-- HR роль
GRANT USAGE ON SCHEMA hr TO hr_role;
GRANT SELECT, INSERT, UPDATE ON hr.employees TO hr_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA hr TO hr_role;

-- Management роль має доступ до всього
GRANT USAGE ON SCHEMA sales, marketing, finance, hr TO management_role;
GRANT SELECT ON ALL TABLES IN SCHEMA sales, finance, hr TO management_role;

-- Створення користувачів та призначення ролей
CREATE ROLE john_sales LOGIN PASSWORD 'password123';
CREATE ROLE mary_finance LOGIN PASSWORD 'password123';
CREATE ROLE bob_hr LOGIN PASSWORD 'password123';
CREATE ROLE alice_manager LOGIN PASSWORD 'password123';

-- Призначення ролей користувачам
GRANT sales_role TO john_sales;
GRANT finance_role TO mary_finance;
GRANT hr_role TO bob_hr;
GRANT management_role, sales_role, finance_role TO alice_manager;
```

Ієрархія ролей та наслідування прав:

```sql
-- Створення ієрархії ролей
CREATE ROLE employee_base;
CREATE ROLE supervisor INHERIT;
CREATE ROLE department_head INHERIT;
CREATE ROLE executive INHERIT;

-- Базові права для всіх співробітників
GRANT CONNECT ON DATABASE application_db TO employee_base;
GRANT USAGE ON SCHEMA public TO employee_base;

-- Наслідування прав
GRANT employee_base TO supervisor;
GRANT supervisor TO department_head;
GRANT department_head TO executive;

-- Додаткові права для кожного рівня
-- Supervisor може змінювати свої записи
GRANT UPDATE ON sales.orders TO supervisor;

-- Department head може переглядати фінансову інформацію
GRANT SELECT ON finance.invoices TO department_head;

-- Executive має повний доступ
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA sales, finance, hr TO executive;

-- Створення користувача з кількома ролями
CREATE ROLE senior_manager LOGIN PASSWORD 'secure_password';
GRANT department_head TO senior_manager;

-- Користувач автоматично отримує всі права з ієрархії:
-- executive -> department_head -> supervisor -> employee_base
```

Динамічне перемикання ролей у сесії:

```sql
-- Підключення як користувач з кількома ролями
-- psql -U alice_manager -d application_db

-- Перегляд поточної ролі
SELECT current_role, current_user;

-- Перегляд всіх доступних ролей
SELECT * FROM pg_roles WHERE pg_has_role(current_user, oid, 'member');

-- Перемикання на конкретну роль
SET ROLE sales_role;

-- Тепер активні тільки права sales_role
SELECT current_role; -- Результат: sales_role

-- Виконання операції як sales
INSERT INTO sales.customers (company_name, contact_person, email)
VALUES ('ACME Corp', 'John Doe', 'john@acme.com');

-- Спроба доступу до HR даних буде заблокована
SELECT * FROM hr.employees; -- ERROR: permission denied

-- Перемикання на іншу роль
SET ROLE finance_role;

-- Тепер можна працювати з фінансами
SELECT * FROM finance.invoices;

-- Повернення до початкової ролі
RESET ROLE;

SELECT current_role; -- Результат: alice_manager
```

Аудит використання ролей:

```sql
-- Створення розширеної таблиці аудиту
CREATE TABLE rbac_audit (
    audit_id SERIAL PRIMARY KEY,
    session_id TEXT,
    username VARCHAR(50),
    current_role VARCHAR(50),
    action VARCHAR(50),
    object_schema VARCHAR(50),
    object_name VARCHAR(100),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    query TEXT,
    success BOOLEAN
);

-- Функція логування ролей
CREATE OR REPLACE FUNCTION log_role_usage()
RETURNS EVENT_TRIGGER AS $$
DECLARE
    audit_query TEXT;
BEGIN
    INSERT INTO rbac_audit (
        session_id,
        username,
        current_role,
        action,
        query
    ) VALUES (
        pg_backend_pid()::TEXT,
        current_user,
        current_role,
        tg_tag,
        current_query()
    );
END;
$$ LANGUAGE plpgsql;

-- Створення event trigger для логування
CREATE EVENT TRIGGER log_ddl_commands
ON ddl_command_end
EXECUTE FUNCTION log_role_usage();

-- Аналіз використання ролей
SELECT
    current_role,
    COUNT(*) as operation_count,
    COUNT(DISTINCT username) as unique_users,
    array_agg(DISTINCT action) as actions
FROM rbac_audit
WHERE timestamp > CURRENT_DATE - INTERVAL '7 days'
GROUP BY current_role
ORDER BY operation_count DESC;

-- Виявлення підозрілої активності
SELECT
    username,
    current_role,
    COUNT(*) as failed_attempts,
    array_agg(DISTINCT object_name) as attempted_objects
FROM rbac_audit
WHERE success = false
    AND timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
GROUP BY username, current_role
HAVING COUNT(*) > 5
ORDER BY failed_attempts DESC;
```

#### Переваги та недоліки RBAC

Переваги рольової моделі включають простоту адміністрування через управління ролями замість окремих користувачів. Відповідність організаційній структурі природно відображає бізнес-процеси. Легкість аудиту через чіткий зв'язок між ролями та правами. Зменшення помилок завдяки стандартизованим наборам прав.

Недоліки полягають у можливій надмірності прав, коли ролі занадто широкі. Складності при нестандартних вимогах, які не вписуються в рольову модель. Потребі періодичного перегляду та оновлення ролей у міру змін організації.

## Загрози безпеці баз даних

Розуміння потенційних загроз є першим кроком до побудови ефективної системи захисту. Розглянемо найпоширеніші типи атак на бази даних.

### SQL ін'єкції

SQL ін'єкція є однією з найнебезпечніших вразливостей веб-застосунків, що дозволяє зловмисникам виконувати довільний SQL код через недостатню валідацію користувацького вводу.

#### Механізм SQL ін'єкції

Вразливий код дозволяє зловмисникам вставляти власний SQL код у запити застосунку.

Приклад вразливого коду на Python:

```python
import psycopg2

# НЕБЕЗПЕЧНИЙ КОД - НЕ ВИКОРИСТОВУВАТИ
def get_user_vulnerable(username, password):
    conn = psycopg2.connect("dbname=app user=app_user")
    cursor = conn.cursor()

    # Вразливий запит - конкатенація рядків
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"

    cursor.execute(query)
    result = cursor.fetchone()

    conn.close()
    return result

# Нормальне використання
user = get_user_vulnerable('john_doe', 'password123')

# Атака SQL ін'єкцією
# Зловмисник вводить: username = "admin' --"
# Результуючий запит:
# SELECT * FROM users WHERE username = 'admin' --' AND password = 'password123'
# Частина після -- коментується, пароль не перевіряється!

attacker_input = "admin' --"
user = get_user_vulnerable(attacker_input, 'any_password')
# Зловмисник отримує доступ без знання пароля!
```

Типи SQL ін'єкцій:

```python
# 1. Union-based SQL injection
# Зловмисник може витягти дані з інших таблиць
username = "' UNION SELECT credit_card_number, cvv, expiry FROM payment_info --"

# Результуючий запит:
# SELECT * FROM users WHERE username = ''
# UNION SELECT credit_card_number, cvv, expiry FROM payment_info --' AND password = '...'

# 2. Boolean-based blind SQL injection
# Витягування даних по одному біту через true/false відповіді
username = "admin' AND SUBSTRING(password, 1, 1) = 'a' --"

# Зловмисник перебирає символи, визначаючи пароль

# 3. Time-based blind SQL injection
# Використання затримок для витягування інформації
username = "admin' AND IF(SUBSTRING(password,1,1)='a', SLEEP(5), 0) --"

# Якщо пароль починається з 'a', відповідь затримається на 5 секунд

# 4. Second-order SQL injection
# Шкідливий код зберігається в БД і виконується пізніше
registration_name = "admin'--"
# Це ім'я зберігається в БД
# Пізніше, коли воно використовується в іншому запиті:
query = f"UPDATE users SET email = 'new@email.com' WHERE name = '{stored_name}'"
# Виникає ін'єкція
```

#### Захист від SQL ін'єкцій

Використання параметризованих запитів є найефективнішим методом захисту:

```python
import psycopg2
from psycopg2 import sql

# БЕЗПЕЧНИЙ КОД - використання параметрів
def get_user_secure(username, password):
    conn = psycopg2.connect("dbname=app user=app_user")
    cursor = conn.cursor()

    # Параметризований запит - використання %s placeholder
    query = "SELECT * FROM users WHERE username = %s AND password = %s"

    # Параметри передаються окремо
    cursor.execute(query, (username, password))
    result = cursor.fetchone()

    conn.close()
    return result

# Тепер спроба ін'єкції не спрацює
attacker_input = "admin' --"
user = get_user_secure(attacker_input, 'any_password')
# Запит шукатиме користувача з іменем "admin' --" (буквально)
# Ін'єкція не виконається!
```

Використання ORM для додаткового захисту:

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True)
    email = Column(String(100))
    password_hash = Column(String(255))

engine = create_engine('postgresql://user:pass@localhost/db')
Session = sessionmaker(bind=engine)

# БЕЗПЕЧНИЙ КОД - використання ORM
def get_user_orm(username):
    session = Session()

    # ORM автоматично параметризує запити
    user = session.query(User).filter(User.username == username).first()

    session.close()
    return user

# Спроба ін'єкції неможлива
attacker_input = "admin' --"
user = get_user_orm(attacker_input)
# ORM перетворить це в безпечний параметризований запит
```

Додаткові методи захисту:

```python
import re
from html import escape

def sanitize_input(user_input, input_type='text'):
    """Додаткова валідація та санітизація вводу"""

    # 1. Обмеження довжини
    if len(user_input) > 100:
        raise ValueError("Input too long")

    # 2. Whitelist валідація для різних типів
    if input_type == 'username':
        # Тільки літери, цифри, підкреслення
        if not re.match(r'^[a-zA-Z0-9_]+$', user_input):
            raise ValueError("Invalid username format")

    elif input_type == 'email':
        # Базова валідація email
        if not re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', user_input):
            raise ValueError("Invalid email format")

    elif input_type == 'integer':
        # Перевірка що це дійсно число
        try:
            int(user_input)
        except ValueError:
            raise ValueError("Invalid integer")

    # 3. Екранування HTML (для захисту від XSS)
    sanitized = escape(user_input)

    return sanitized

# Використання
def safe_get_user(username):
    try:
        # Валідація перед використанням
        clean_username = sanitize_input(username, 'username')

        # Параметризований запит
        user = get_user_secure(clean_username, password)
        return user
    except ValueError as e:
        print(f"Invalid input: {e}")
        return None
```

Налаштування безпеки на рівні бази даних:

```sql
-- Обмеження прав додатка в БД
CREATE ROLE app_user LOGIN PASSWORD 'secure_password';

-- Надання тільки необхідних прав
GRANT CONNECT ON DATABASE app_db TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE ON users, orders TO app_user;

-- Заборона DROP, TRUNCATE, та інших небезпечних операцій
REVOKE CREATE ON SCHEMA public FROM app_user;
REVOKE ALL ON pg_catalog.pg_authid FROM app_user;

-- Використання представлень замість прямого доступу до таблиць
CREATE VIEW user_safe_view AS
SELECT user_id, username, email, created_at
FROM users;

GRANT SELECT ON user_safe_view TO app_user;
REVOKE SELECT ON users FROM app_user;

-- Функція безпечного пошуку замість прямих запитів
CREATE OR REPLACE FUNCTION safe_find_user(p_username TEXT)
RETURNS TABLE(user_id INT, username VARCHAR, email VARCHAR) AS $$
BEGIN
    -- Валідація параметра
    IF length(p_username) > 50 OR p_username ~ '[^a-zA-Z0-9_]' THEN
        RAISE EXCEPTION 'Invalid username format';
    END IF;

    -- Безпечний запит
    RETURN QUERY
    SELECT u.user_id, u.username, u.email
    FROM users u
    WHERE u.username = p_username;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

GRANT EXECUTE ON FUNCTION safe_find_user(TEXT) TO app_user;
```

### Несанкціонований доступ

Несанкціонований доступ може відбуватися через слабкі паролі, викрадені облікові дані, неправильно налаштовані права доступу або exploitation вразливостей.

#### Методи захисту від несанкціонованого доступу

Багатофакторна автентифікація додає додатковий рівень захисту:

```python
import pyotp
import qrcode
from datetime import datetime, timedelta
import hashlib

class TwoFactorAuth:
    def __init__(self, secret_key=None):
        # Генерація унікального ключа для користувача
        self.secret_key = secret_key or pyotp.random_base32()
        self.totp = pyotp.TOTP(self.secret_key)

    def get_provisioning_uri(self, username, issuer='MyApp'):
        """Генерація URI для QR коду"""
        return self.totp.provisioning_uri(
            name=username,
            issuer_name=issuer
        )

    def generate_qr_code(self, username):
        """Створення QR коду для Google Authenticator"""
        uri = self.get_provisioning_uri(username)
        qr = qrcode.make(uri)
        return qr

    def verify_token(self, token):
        """Перевірка 2FA токену"""
        return self.totp.verify(token, valid_window=1)

    def get_backup_codes(self, count=10):
        """Генерація резервних кодів"""
        codes = []
        for i in range(count):
            code = hashlib.sha256(
                f"{self.secret_key}{i}{datetime.now()}".encode()
            ).hexdigest()[:8]
            codes.append(code)
        return codes

# Використання в додатку
def setup_2fa_for_user(user_id, username):
    """Налаштування 2FA для користувача"""
    tfa = TwoFactorAuth()

    # Зберігання секретного ключа в БД
    save_2fa_secret(user_id, tfa.secret_key)

    # Генерація резервних кодів
    backup_codes = tfa.get_backup_codes()
    save_backup_codes(user_id, backup_codes)

    # Повернення QR коду користувачу
    qr_code = tfa.generate_qr_code(username)
    return qr_code, backup_codes

def login_with_2fa(username, password, token):
    """Вхід з двофакторною автентифікацією"""
    # Перевірка пароля
    user = authenticate_user(username, password)
    if not user:
        return False, "Invalid credentials"

    # Отримання секретного ключа користувача
    secret_key = get_user_2fa_secret(user['id'])
    if not secret_key:
        return False, "2FA not configured"

    # Перевірка 2FA токену
    tfa = TwoFactorAuth(secret_key)
    if tfa.verify_token(token):
        return True, "Login successful"

    # Перевірка резервного коду
    if verify_backup_code(user['id'], token):
        return True, "Login successful with backup code"

    return False, "Invalid 2FA token"
```

Обмеження спроб входу та виявлення атак:

```python
from datetime import datetime, timedelta
from collections import defaultdict
import redis

class LoginRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.max_attempts = 5
        self.lockout_duration = timedelta(minutes=15)
        self.attempt_window = timedelta(minutes=5)

    def get_attempt_key(self, identifier, attempt_type='login'):
        """Генерація ключа для відстеження спроб"""
        return f"{attempt_type}:{identifier}:attempts"

    def get_lockout_key(self, identifier):
        """Генерація ключа для блокування"""
        return f"lockout:{identifier}"

    def is_locked_out(self, identifier):
        """Перевірка чи заблокований користувач"""
        lockout_key = self.get_lockout_key(identifier)
        return self.redis.exists(lockout_key)

    def record_failed_attempt(self, identifier):
        """Реєстрація невдалої спроби входу"""
        attempt_key = self.get_attempt_key(identifier)

        # Інкремент лічильника з TTL
        pipeline = self.redis.pipeline()
        pipeline.incr(attempt_key)
        pipeline.expire(attempt_key, int(self.attempt_window.total_seconds()))
        attempts = pipeline.execute()[0]

        # Блокування при перевищенні ліміту
        if attempts >= self.max_attempts:
            lockout_key = self.get_lockout_key(identifier)
            self.redis.setex(
                lockout_key,
                int(self.lockout_duration.total_seconds()),
                '1'
            )

            # Логування підозрілої активності
            self.log_suspicious_activity(identifier, attempts)

            return True, self.lockout_duration

        return False, self.max_attempts - attempts

    def reset_attempts(self, identifier):
        """Скидання лічильника при успішному вході"""
        attempt_key = self.get_attempt_key(identifier)
        self.redis.delete(attempt_key)

    def log_suspicious_activity(self, identifier, attempts):
        """Логування підозрілої активності"""
        log_entry = {
            'identifier': identifier,
            'attempts': attempts,
            'timestamp': datetime.now().isoformat(),
            'action': 'account_locked'
        }

        # Збереження в Redis список для аналізу
        self.redis.lpush('security:suspicious_activity', str(log_entry))
        self.redis.ltrim('security:suspicious_activity', 0, 999)

    def get_recent_suspicious_activity(self, limit=100):
        """Отримання останньої підозрілої активності"""
        activities = self.redis.lrange('security:suspicious_activity', 0, limit-1)
        return [eval(a.decode()) for a in activities]

# Використання в додатку
redis_client = redis.Redis(host='localhost', port=6379, db=0)
rate_limiter = LoginRateLimiter(redis_client)

def secure_login(username, password, ip_address):
    """Безпечний вхід з rate limiting"""

    # Перевірка блокування по IP та username
    for identifier in [ip_address, username]:
        if rate_limiter.is_locked_out(identifier):
            return {
                'success': False,
                'error': 'Account temporarily locked due to multiple failed attempts',
                'locked_until': rate_limiter.lockout_duration
            }

    # Спроба автентифікації
    user = authenticate_user(username, password)

    if user:
        # Успішний вхід - скидання лічильників
        rate_limiter.reset_attempts(ip_address)
        rate_limiter.reset_attempts(username)

        return {
            'success': True,
            'user': user,
            'session_token': generate_session_token(user)
        }
    else:
        # Невдала спроба - реєстрація
        locked_ip, remaining_ip = rate_limiter.record_failed_attempt(ip_address)
        locked_user, remaining_user = rate_limiter.record_failed_attempt(username)

        if locked_ip or locked_user:
            return {
                'success': False,
                'error': 'Too many failed attempts. Account locked.',
                'locked_duration': rate_limiter.lockout_duration
            }

        return {
            'success': False,
            'error': 'Invalid credentials',
            'remaining_attempts': min(remaining_ip, remaining_user)
        }
```

Моніторинг та виявлення аномальної активності:

```sql
-- Таблиця логів автентифікації
CREATE TABLE authentication_logs (
    log_id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    ip_address INET,
    user_agent TEXT,
    success BOOLEAN,
    failure_reason VARCHAR(100),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    session_id VARCHAR(100),
    geolocation JSONB
);

CREATE INDEX idx_auth_username_time ON authentication_logs(username, timestamp DESC);
CREATE INDEX idx_auth_ip_time ON authentication_logs(ip_address, timestamp DESC);
CREATE INDEX idx_auth_success ON authentication_logs(success, timestamp DESC);

-- Функція виявлення аномалій
CREATE OR REPLACE FUNCTION detect_login_anomalies()
RETURNS TABLE(
    alert_type VARCHAR,
    username VARCHAR,
    details JSONB,
    risk_score INTEGER
) AS $$
BEGIN
    -- 1. Множинні невдалі спроби з різних IP
    RETURN QUERY
    SELECT
        'multiple_ips_failed_login'::VARCHAR,
        a.username,
        jsonb_build_object(
            'distinct_ips', COUNT(DISTINCT a.ip_address),
            'failed_attempts', COUNT(*),
            'time_window', '15 minutes'
        ),
        80 as risk_score
    FROM authentication_logs a
    WHERE a.timestamp > CURRENT_TIMESTAMP - INTERVAL '15 minutes'
        AND a.success = false
    GROUP BY a.username
    HAVING COUNT(DISTINCT a.ip_address) >= 5;

    -- 2. Успішний вхід після багатьох невдалих спроб
    RETURN QUERY
    SELECT
        'successful_after_failures'::VARCHAR,
        a.username,
        jsonb_build_object(
            'failed_attempts', COUNT(*) FILTER (WHERE NOT a.success),
            'success_ip', MAX(a.ip_address) FILTER (WHERE a.success)
        ),
        60 as risk_score
    FROM authentication_logs a
    WHERE a.timestamp > CURRENT_TIMESTAMP - INTERVAL '30 minutes'
    GROUP BY a.username
    HAVING COUNT(*) FILTER (WHERE NOT a.success) >= 3
        AND COUNT(*) FILTER (WHERE a.success) > 0;

    -- 3. Вхід з нетипової локації
    RETURN QUERY
    SELECT
        'unusual_location'::VARCHAR,
        current_login.username,
        jsonb_build_object(
            'usual_country', usual_location.country,
            'current_country', current_login.country,
            'distance_km', current_login.distance
        ),
        70 as risk_score
    FROM (
        SELECT
            username,
            geolocation->>'country' as country,
            1000 as distance -- Спрощено, в реальності треба розраховувати
        FROM authentication_logs
        WHERE timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
            AND success = true
    ) current_login
    JOIN (
        SELECT
            username,
            mode() WITHIN GROUP (ORDER BY geolocation->>'country') as country
        FROM authentication_logs
        WHERE timestamp > CURRENT_TIMESTAMP - INTERVAL '30 days'
            AND timestamp < CURRENT_TIMESTAMP - INTERVAL '1 day'
            AND success = true
        GROUP BY username
    ) usual_location USING (username)
    WHERE current_login.country != usual_location.country;

    -- 4. Одночасні входи з різних локацій (неможливе для однієї людини)
    RETURN QUERY
    SELECT
        'impossible_travel'::VARCHAR,
        a1.username,
        jsonb_build_object(
            'location1', a1.geolocation,
            'location2', a2.geolocation,
            'time_diff_minutes', EXTRACT(EPOCH FROM (a2.timestamp - a1.timestamp))/60
        ),
        90 as risk_score
    FROM authentication_logs a1
    JOIN authentication_logs a2 ON a1.username = a2.username
    WHERE a1.success AND a2.success
        AND a1.timestamp < a2.timestamp
        AND a2.timestamp - a1.timestamp < INTERVAL '2 hours'
        AND a1.geolocation->>'country' != a2.geolocation->>'country'
        AND a1.timestamp > CURRENT_TIMESTAMP - INTERVAL '6 hours';
END;
$$ LANGUAGE plpgsql;

-- Автоматичне виявлення та сповіщення
CREATE TABLE security_alerts (
    alert_id SERIAL PRIMARY KEY,
    alert_type VARCHAR(100),
    username VARCHAR(50),
    details JSONB,
    risk_score INTEGER,
    status VARCHAR(20) DEFAULT 'new',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP,
    resolved_by VARCHAR(50),
    notes TEXT
);

-- Процедура обробки аномалій
CREATE OR REPLACE PROCEDURE process_security_alerts()
LANGUAGE plpgsql
AS $$
DECLARE
    anomaly RECORD;
BEGIN
    FOR anomaly IN SELECT * FROM detect_login_anomalies()
    LOOP
        -- Збереження алерту
        INSERT INTO security_alerts (alert_type, username, details, risk_score)
        VALUES (anomaly.alert_type, anomaly.username, anomaly.details, anomaly.risk_score);

        -- Автоматичні дії на основі risk_score
        IF anomaly.risk_score >= 80 THEN
            -- Блокування облікового запису
            PERFORM lock_user_account(anomaly.username, 'Automated lock due to high-risk activity');

            -- Відправка сповіщення адміністраторам
            PERFORM send_security_notification(
                'High Risk Alert',
                format('User %s - %s (Risk: %s)',
                    anomaly.username, anomaly.alert_type, anomaly.risk_score)
            );
        ELSIF anomaly.risk_score >= 60 THEN
            -- Запит додаткової автентифікації
            PERFORM require_additional_auth(anomaly.username);
        END IF;
    END LOOP;
END;
$$;
```

### Витоки даних

Витоки даних можуть відбуватися через зовнішні атаки, внутрішні загрози, неправильні налаштування безпеки або помилки в коді.

#### Методи запобігання витокам даних

Data Loss Prevention стратегії:

```sql
-- 1. Маскування чутливих даних
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Функція маскування номерів кредитних карток
CREATE OR REPLACE FUNCTION mask_credit_card(card_number TEXT)
RETURNS TEXT AS $$
BEGIN
    IF card_number IS NULL OR length(card_number) < 4 THEN
        RETURN '****';
    END IF;

    RETURN '****-****-****-' || right(card_number, 4);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Функція маскування email
CREATE OR REPLACE FUNCTION mask_email(email TEXT)
RETURNS TEXT AS $$
DECLARE
    parts TEXT[];
    username TEXT;
    domain TEXT;
BEGIN
    IF email IS NULL OR email !~ '@' THEN
        RETURN '***@***.***';
    END IF;

    parts := string_to_array(email, '@');
    username := parts[1];
    domain := parts[2];

    IF length(username) <= 2 THEN
        RETURN '**@' || domain;
    ELSE
        RETURN left(username, 2) || '***@' || domain;
    END IF;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Представлення з маскованими даними для аналітики
CREATE VIEW customers_masked AS
SELECT
    customer_id,
    first_name,
    last_name,
    mask_email(email) as email,
    mask_credit_card(credit_card_number) as credit_card,
    phone,
    created_at
FROM customers;

-- Надання доступу до маскованих даних аналітикам
GRANT SELECT ON customers_masked TO analyst_role;
REVOKE ALL ON customers FROM analyst_role;
```

Аудит доступу до чутливих даних:

```sql
-- Таблиця аудиту доступу до даних
CREATE TABLE sensitive_data_access_log (
    log_id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    table_name VARCHAR(100),
    column_names TEXT[],
    row_id INTEGER,
    operation VARCHAR(20),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,
    application_context JSONB
);

-- Функція логування доступу
CREATE OR REPLACE FUNCTION log_sensitive_access()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO sensitive_data_access_log (
        username,
        table_name,
        column_names,
        row_id,
        operation,
        ip_address
    ) VALUES (
        current_user,
        TG_TABLE_NAME,
        TG_ARGV,
        COALESCE(NEW.id, OLD.id),
        TG_OP,
        inet_client_addr()
    );

    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Тригери для критичних таблиць
CREATE TRIGGER log_customer_access
AFTER SELECT OR INSERT OR UPDATE OR DELETE ON customers
FOR EACH ROW EXECUTE FUNCTION log_sensitive_access('email', 'credit_card_number');

CREATE TRIGGER log_payment_access
AFTER SELECT OR INSERT OR UPDATE OR DELETE ON payment_methods
FOR EACH ROW EXECUTE FUNCTION log_sensitive_access('card_number', 'cvv');

-- Аналіз підозрілого доступу
CREATE OR REPLACE FUNCTION detect_data_exfiltration()
RETURNS TABLE(
    username VARCHAR,
    suspicious_activity VARCHAR,
    record_count BIGINT,
    time_window INTERVAL
) AS $$
BEGIN
    -- Масове читання даних
    RETURN QUERY
    SELECT
        l.username,
        'mass_data_access'::VARCHAR,
        COUNT(*),
        MAX(l.timestamp) - MIN(l.timestamp)
    FROM sensitive_data_access_log l
    WHERE l.timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
        AND l.operation = 'SELECT'
    GROUP BY l.username
    HAVING COUNT(*) > 1000;

    -- Доступ у незвичний час
    RETURN QUERY
    SELECT
        l.username,
        'unusual_time_access'::VARCHAR,
        COUNT(*),
        INTERVAL '1 hour'
    FROM sensitive_data_access_log l
    WHERE EXTRACT(HOUR FROM l.timestamp) NOT BETWEEN 8 AND 18
        AND EXTRACT(DOW FROM l.timestamp) BETWEEN 1 AND 5
        AND l.timestamp > CURRENT_TIMESTAMP - INTERVAL '24 hours'
    GROUP BY l.username
    HAVING COUNT(*) > 100;

    -- Доступ до незвичних таблиць
    RETURN QUERY
    SELECT
        l.username,
        'unusual_table_access'::VARCHAR,
        COUNT(*),
        INTERVAL '1 hour'
    FROM sensitive_data_access_log l
    LEFT JOIN (
        SELECT username, table_name
        FROM sensitive_data_access_log
        WHERE timestamp > CURRENT_TIMESTAMP - INTERVAL '30 days'
            AND timestamp < CURRENT_TIMESTAMP - INTERVAL '7 days'
        GROUP BY username, table_name
    ) usual ON l.username = usual.username AND l.table_name = usual.table_name
    WHERE l.timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
        AND usual.username IS NULL
    GROUP BY l.username
    HAVING COUNT(*) > 50;
END;
$$ LANGUAGE plpgsql;
```

## Криптографічні методи захисту

Шифрування є фундаментальним методом захисту даних як при зберіганні, так і при передачі. Розглянемо основні підходи до криптографічного захисту баз даних.

### Шифрування даних у спокої (Encryption at Rest)

Шифрування даних у спокої захищає інформацію, збережену на дисках, від несанкціонованого доступу при фізичному викраденні носіїв або отриманні доступу до файлової системи.

#### Шифрування на рівні стовпців

Шифрування окремих стовпців дозволяє захистити найбільш чутливі дані, зберігаючи можливість ефективного пошуку по незашифрованих полях.

Реалізація шифрування в PostgreSQL:

```sql
-- Створення розширення для криптографії
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Таблиця з шифрованими даними
CREATE TABLE secure_customers (
    customer_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) NOT NULL,
    -- Шифровані поля
    ssn_encrypted BYTEA, -- Номер соціального страхування
    credit_card_encrypted BYTEA,
    phone_encrypted BYTEA,
    -- Незашифровані поля для пошуку
    country VARCHAR(2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Функція шифрування даних
CREATE OR REPLACE FUNCTION encrypt_data(
    plain_text TEXT,
    encryption_key TEXT
) RETURNS BYTEA AS $$
BEGIN
    RETURN pgp_sym_encrypt(plain_text, encryption_key);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Функція розшифрування даних
CREATE OR REPLACE FUNCTION decrypt_data(
    encrypted_data BYTEA,
    encryption_key TEXT
) RETURNS TEXT AS $$
BEGIN
    RETURN pgp_sym_decrypt(encrypted_data, encryption_key);
EXCEPTION
    WHEN OTHERS THEN
        -- Логування спроби несанкціонованого розшифрування
        INSERT INTO security_events (event_type, details, timestamp)
        VALUES ('decryption_failed',
                'Failed decryption attempt by ' || current_user,
                CURRENT_TIMESTAMP);
        RETURN NULL;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Вставка даних з шифруванням
INSERT INTO secure_customers (
    username,
    email,
    ssn_encrypted,
    credit_card_encrypted,
    phone_encrypted,
    country
) VALUES (
    'john_doe',
    'john@example.com',
    encrypt_data('123-45-6789', 'master_encryption_key'),
    encrypt_data('4532-1234-5678-9010', 'master_encryption_key'),
    encrypt_data('+1-555-0123', 'master_encryption_key'),
    'US'
);

-- Представлення з розшифрованими даними для авторизованих користувачів
CREATE VIEW customers_decrypted AS
SELECT
    customer_id,
    username,
    email,
    decrypt_data(ssn_encrypted, current_setting('app.encryption_key')) as ssn,
    decrypt_data(credit_card_encrypted, current_setting('app.encryption_key')) as credit_card,
    decrypt_data(phone_encrypted, current_setting('app.encryption_key')) as phone,
    country,
    created_at
FROM secure_customers;

-- Обмеження доступу до розшифрованих даних
REVOKE ALL ON secure_customers FROM PUBLIC;
GRANT SELECT ON customers_decrypted TO privileged_role;
```

Управління ключами шифрування в застосунку:

```python
import psycopg2
from cryptography.fernet import Fernet
import os
from dotenv import load_dotenv

class EncryptionKeyManager:
    """Керування ключами шифрування"""

    def __init__(self):
        load_dotenv()
        self.master_key = os.getenv('MASTER_ENCRYPTION_KEY')
        self.key_rotation_days = 90

    def get_current_key(self):
        """Отримання поточного ключа шифрування"""
        return self.master_key

    def rotate_keys(self):
        """Ротація ключів шифрування"""
        new_key = Fernet.generate_key().decode()

        # Зберігання нового ключа
        self._store_new_key(new_key)

        # Перешифрування всіх даних з старим ключем
        self._reencrypt_all_data(self.master_key, new_key)

        # Оновлення поточного ключа
        self.master_key = new_key

        return new_key

    def _store_new_key(self, new_key):
        """Зберігання нового ключа в безпечному сховищі"""
        # В продакшені використовувати AWS KMS, Azure Key Vault, HashiCorp Vault
        # Тут спрощена версія
        with open('.env', 'a') as f:
            f.write(f'\nARCHIVED_KEY_{int(time.time())}={self.master_key}')
            f.write(f'\nMASTER_ENCRYPTION_KEY={new_key}')

    def _reencrypt_all_data(self, old_key, new_key):
        """Перешифрування всіх даних"""
        conn = psycopg2.connect("dbname=app")
        cur = conn.cursor()

        # Отримання всіх записів
        cur.execute("SELECT customer_id, ssn_encrypted, credit_card_encrypted, phone_encrypted FROM secure_customers")

        for row in cur.fetchall():
            customer_id, ssn_enc, cc_enc, phone_enc = row

            # Розшифрування старим ключем
            cur.execute("SELECT decrypt_data(%s, %s)", (ssn_enc, old_key))
            ssn_plain = cur.fetchone()[0]

            cur.execute("SELECT decrypt_data(%s, %s)", (cc_enc, old_key))
            cc_plain = cur.fetchone()[0]

            cur.execute("SELECT decrypt_data(%s, %s)", (phone_enc, old_key))
            phone_plain = cur.fetchone()[0]

            # Шифрування новим ключем
            cur.execute("""
                UPDATE secure_customers
                SET ssn_encrypted = encrypt_data(%s, %s),
                    credit_card_encrypted = encrypt_data(%s, %s),
                    phone_encrypted = encrypt_data(%s, %s)
                WHERE customer_id = %s
            """, (ssn_plain, new_key, cc_plain, new_key, phone_plain, new_key, customer_id))

        conn.commit()
        cur.close()
        conn.close()

class SecureDataAccess:
    """Безпечний доступ до зашифрованих даних"""

    def __init__(self, encryption_key):
        self.encryption_key = encryption_key
        self.conn = None

    def connect(self):
        """Підключення до БД з встановленням ключа шифрування"""
        self.conn = psycopg2.connect("dbname=app user=app_user")

        # Встановлення ключа шифрування для сесії
        cur = self.conn.cursor()
        cur.execute(f"SET app.encryption_key = '{self.encryption_key}'")
        cur.close()

    def get_customer_data(self, customer_id):
        """Отримання розшифрованих даних клієнта"""
        if not self.conn:
            self.connect()

        cur = self.conn.cursor()
        cur.execute("""
            SELECT customer_id, username, email, ssn, credit_card, phone
            FROM customers_decrypted
            WHERE customer_id = %s
        """, (customer_id,))

        result = cur.fetchone()
        cur.close()

        return result

    def insert_customer(self, username, email, ssn, credit_card, phone, country):
        """Вставка даних з автоматичним шифруванням"""
        if not self.conn:
            self.connect()

        cur = self.conn.cursor()
        cur.execute("""
            INSERT INTO secure_customers (
                username, email, ssn_encrypted,
                credit_card_encrypted, phone_encrypted, country
            ) VALUES (
                %s, %s,
                encrypt_data(%s, %s),
                encrypt_data(%s, %s),
                encrypt_data(%s, %s),
                %s
            )
            RETURNING customer_id
        """, (username, email, ssn, self.encryption_key,
              credit_card, self.encryption_key,
              phone, self.encryption_key, country))

        customer_id = cur.fetchone()[0]
        self.conn.commit()
        cur.close()

        return customer_id

    def close(self):
        if self.conn:
            self.conn.close()

# Використання
key_manager = EncryptionKeyManager()
data_access = SecureDataAccess(key_manager.get_current_key())

# Вставка зашифрованих даних
customer_id = data_access.insert_customer(
    username='alice_smith',
    email='alice@example.com',
    ssn='987-65-4321',
    credit_card='5500-0000-0000-0004',
    phone='+1-555-9999',
    country='US'
)

# Отримання розшифрованих даних
customer_data = data_access.get_customer_data(customer_id)
print(customer_data)

data_access.close()
```

#### Прозоре шифрування даних (TDE)

Transparent Data Encryption шифрує всю базу даних на рівні файлової системи без необхідності змін у коді застосунку.

Налаштування TDE в PostgreSQL (приклад з pgcrypto):

```bash
#!/bin/bash

# Налаштування шифрування на рівні файлової системи

# 1. Створення зашифрованого розділу
cryptsetup luksFormat /dev/sdb1
cryptsetup luksOpen /dev/sdb1 encrypted_db

# 2. Створення файлової системи
mkfs.ext4 /dev/mapper/encrypted_db

# 3. Монтування
mkdir /var/lib/postgresql/encrypted
mount /dev/mapper/encrypted_db /var/lib/postgresql/encrypted

# 4. Переміщення даних PostgreSQL
systemctl stop postgresql
rsync -av /var/lib/postgresql/14/main/ /var/lib/postgresql/encrypted/main/

# 5. Оновлення конфігурації PostgreSQL
echo "data_directory = '/var/lib/postgresql/encrypted/main'" >> /etc/postgresql/14/main/postgresql.conf

# 6. Запуск PostgreSQL
systemctl start postgresql
```

Налаштування шифрування в MySQL:

```sql
-- Увімкнення шифрування таблиць в MySQL
SET GLOBAL innodb_encryption_threads = 4;

-- Створення таблиці з шифруванням
CREATE TABLE encrypted_orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(20)
) ENCRYPTION='Y';

-- Шифрування існуючої таблиці
ALTER TABLE orders ENCRYPTION='Y';

-- Перевірка статусу шифрування
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    CREATE_OPTIONS
FROM information_schema.TABLES
WHERE CREATE_OPTIONS LIKE '%ENCRYPTION%';
```

### Шифрування даних при передачі (Encryption in Transit)

Шифрування даних при передачі захищає інформацію від перехоплення під час передачі по мережі.

#### Налаштування SSL/TLS з'єднань

Конфігурація PostgreSQL для SSL:

```bash
# Генерація сертифікатів
openssl req -new -x509 -days 365 -nodes -text \
    -out server.crt \
    -keyout server.key \
    -subj "/CN=database.example.com"

chmod 0600 server.key
chown postgres:postgres server.key server.crt

# Переміщення сертифікатів
mv server.crt server.key /var/lib/postgresql/14/main/
```

Налаштування postgresql.conf:

```conf
# SSL параметри
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_prefer_server_ciphers = on
ssl_min_protocol_version = 'TLSv1.2'

# Вимагати SSL для віддалених з'єднань
```

Налаштування pg_hba.conf для вимагання SSL:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Локальні з'єднання
local   all             all                                     peer

# Віддалені з'єднання тільки через SSL
hostssl all             all             0.0.0.0/0               md5
hostssl all             all             ::/0                    md5

# Заборона незашифрованих з'єднань
hostnossl all           all             0.0.0.0/0               reject
```

Підключення до БД з перевіркою SSL:

```python
import psycopg2
import ssl

class SecureDBConnection:
    def __init__(self, host, database, user, password,
                 ca_cert_path=None, client_cert_path=None, client_key_path=None):
        self.conn_params = {
            'host': host,
            'database': database,
            'user': user,
            'password': password,
            'sslmode': 'verify-full',  # Вимагає перевірку сертифіката
        }

        if ca_cert_path:
            self.conn_params['sslrootcert'] = ca_cert_path

        if client_cert_path and client_key_path:
            self.conn_params['sslcert'] = client_cert_path
            self.conn_params['sslkey'] = client_key_path

    def connect(self):
        """Безпечне підключення з SSL"""
        try:
            conn = psycopg2.connect(**self.conn_params)

            # Перевірка SSL з'єднання
            cur = conn.cursor()
            cur.execute("SELECT ssl_is_used()")
            ssl_used = cur.fetchone()[0]

            if not ssl_used:
                raise Exception("SSL connection not established")

            cur.execute("SELECT ssl_version()")
            ssl_version = cur.fetchone()[0]
            print(f"Connected with SSL version: {ssl_version}")

            cur.close()
            return conn

        except Exception as e:
            print(f"Connection failed: {e}")
            raise

    def test_connection(self):
        """Тестування безпечного з'єднання"""
        conn = self.connect()

        cur = conn.cursor()

        # Отримання інформації про SSL
        cur.execute("""
            SELECT
                ssl_is_used() AS ssl_enabled,
                ssl_version() AS ssl_version,
                ssl_cipher() AS ssl_cipher,
                ssl_client_dn() AS client_dn
        """)

        ssl_info = cur.fetchone()
        print(f"""
        SSL Connection Info:
        - Enabled: {ssl_info[0]}
        - Version: {ssl_info[1]}
        - Cipher: {ssl_info[2]}
        - Client DN: {ssl_info[3]}
        """)

        cur.close()
        conn.close()

# Використання
secure_conn = SecureDBConnection(
    host='database.example.com',
    database='production',
    user='app_user',
    password='secure_password',
    ca_cert_path='/path/to/root.crt',
    client_cert_path='/path/to/client.crt',
    client_key_path='/path/to/client.key'
)

connection = secure_conn.connect()
```

## Аудит та моніторинг

Систематичний аудит та моніторинг є критично важливими для виявлення та реагування на інциденти безпеки.

### Система аудиту на рівні бази даних

Комплексна система логування всіх операцій:

```sql
-- Розширена таблиця аудиту
CREATE TABLE comprehensive_audit_log (
    audit_id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Інформація про користувача
    database_user VARCHAR(50),
    application_user VARCHAR(50),
    session_id TEXT,

    -- Інформація про операцію
    operation_type VARCHAR(20), -- SELECT, INSERT, UPDATE, DELETE, DDL
    object_schema VARCHAR(50),
    object_name VARCHAR(100),
    object_type VARCHAR(20), -- TABLE, VIEW, FUNCTION, etc.

    -- Деталі операції
    old_values JSONB,
    new_values JSONB,
    affected_rows INTEGER,

    -- Мережева інформація
    client_ip INET,
    client_port INTEGER,
    server_ip INET,

    -- SQL інформація
    query_text TEXT,
    query_plan TEXT,
    execution_time_ms NUMERIC,

    -- Контекст застосунку
    application_name VARCHAR(100),
    application_context JSONB,

    -- Результат
    success BOOLEAN,
    error_message TEXT
);

-- Індекси для швидкого пошуку
CREATE INDEX idx_audit_timestamp ON comprehensive_audit_log(timestamp DESC);
CREATE INDEX idx_audit_user ON comprehensive_audit_log(database_user, application_user);
CREATE INDEX idx_audit_object ON comprehensive_audit_log(object_schema, object_name);
CREATE INDEX idx_audit_operation ON comprehensive_audit_log(operation_type, success);
CREATE INDEX idx_audit_client_ip ON comprehensive_audit_log(client_ip);

-- Партиціонування для великих обсягів
CREATE TABLE comprehensive_audit_log_2024_01 PARTITION OF comprehensive_audit_log
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE comprehensive_audit_log_2024_02 PARTITION OF comprehensive_audit_log
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... інші партиції

-- Функція комплексного аудиту
CREATE OR REPLACE FUNCTION comprehensive_audit()
RETURNS TRIGGER AS $$
DECLARE
    old_data JSONB;
    new_data JSONB;
    app_user VARCHAR(50);
    app_context JSONB;
BEGIN
    -- Отримання контексту застосунку
    BEGIN
        app_user := current_setting('app.current_user');
        app_context := current_setting('app.context')::JSONB;
    EXCEPTION WHEN OTHERS THEN
        app_user := NULL;
        app_context := NULL;
    END;

    -- Формування JSON об'єктів зі старими та новими значеннями
    IF TG_OP IN ('UPDATE', 'DELETE') THEN
        old_data := row_to_json(OLD)::JSONB;
    END IF;

    IF TG_OP IN ('INSERT', 'UPDATE') THEN
        new_data := row_to_json(NEW)::JSONB;
    END IF;

    -- Запис у лог
    INSERT INTO comprehensive_audit_log (
        database_user,
        application_user,
        session_id,
        operation_type,
        object_schema,
        object_name,
        object_type,
        old_values,
        new_values,
        client_ip,
        application_name,
        application_context,
        success
    ) VALUES (
        current_user,
        app_user,
        pg_backend_pid()::TEXT,
        TG_OP,
        TG_TABLE_SCHEMA,
        TG_TABLE_NAME,
        'TABLE',
        old_data,
        new_data,
        inet_client_addr(),
        current_setting('application_name'),
        app_context,
        true
    );

    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Застосування аудиту до критичних таблиць
CREATE TRIGGER audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION comprehensive_audit();

CREATE TRIGGER audit_orders
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION comprehensive_audit();

CREATE TRIGGER audit_payments
AFTER INSERT OR UPDATE OR DELETE ON payment_methods
FOR EACH ROW EXECUTE FUNCTION comprehensive_audit();
```

Аналітичні запити для виявлення аномалій:

```sql
-- 1. Топ користувачів за кількістю операцій
SELECT
    database_user,
    application_user,
    COUNT(*) as operation_count,
    COUNT(*) FILTER (WHERE NOT success) as failed_operations,
    array_agg(DISTINCT operation_type) as operations,
    MIN(timestamp) as first_seen,
    MAX(timestamp) as last_seen
FROM comprehensive_audit_log
WHERE timestamp > CURRENT_TIMESTAMP - INTERVAL '24 hours'
GROUP BY database_user, application_user
ORDER BY operation_count DESC
LIMIT 20;

-- 2. Незвичайна активність вночі
SELECT
    database_user,
    COUNT(*) as night_operations,
    array_agg(DISTINCT object_name) as accessed_objects,
    array_agg(DISTINCT client_ip::TEXT) as source_ips
FROM comprehensive_audit_log
WHERE EXTRACT(HOUR FROM timestamp) BETWEEN 0 AND 6
    AND timestamp > CURRENT_TIMESTAMP - INTERVAL '7 days'
GROUP BY database_user
HAVING COUNT(*) > 10
ORDER BY night_operations DESC;

-- 3. Масові видалення
SELECT
    timestamp,
    database_user,
    application_user,
    object_name,
    affected_rows,
    query_text,
    client_ip
FROM comprehensive_audit_log
WHERE operation_type = 'DELETE'
    AND affected_rows > 100
    AND timestamp > CURRENT_TIMESTAMP - INTERVAL '7 days'
ORDER BY timestamp DESC;

-- 4. Невдалі операції
SELECT
    database_user,
    operation_type,
    object_name,
    COUNT(*) as failure_count,
    array_agg(DISTINCT error_message) as errors,
    MAX(timestamp) as last_failure
FROM comprehensive_audit_log
WHERE NOT success
    AND timestamp > CURRENT_TIMESTAMP - INTERVAL '24 hours'
GROUP BY database_user, operation_type, object_name
HAVING COUNT(*) > 5
ORDER BY failure_count DESC;

-- 5. Доступ до чутливих даних
SELECT
    a.timestamp,
    a.database_user,
    a.application_user,
    a.object_name,
    a.operation_type,
    a.client_ip,
    a.query_text
FROM comprehensive_audit_log a
WHERE a.object_name IN ('users', 'payment_methods', 'customers')
    AND a.operation_type = 'SELECT'
    AND a.timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
ORDER BY a.timestamp DESC;

-- 6. Зміни схеми бази даних (DDL операції)
SELECT
    timestamp,
    database_user,
    object_type,
    object_name,
    query_text,
    client_ip
FROM comprehensive_audit_log
WHERE operation_type = 'DDL'
    AND timestamp > CURRENT_TIMESTAMP - INTERVAL '30 days'
ORDER BY timestamp DESC;

-- 7. Підозріла зміна привілеїв
SELECT
    timestamp,
    database_user,
    query_text,
    client_ip,
    success
FROM comprehensive_audit_log
WHERE query_text ILIKE '%GRANT%'
    OR query_text ILIKE '%REVOKE%'
    OR query_text ILIKE '%ALTER ROLE%'
ORDER BY timestamp DESC;
```

## Стратегії резервного копіювання та відновлення

Надійна стратегія резервного копіювання є останньою лінією захисту від втрати даних.

### Типи резервних копій

Комплексна стратегія включає різні типи бекапів:

```bash
#!/bin/bash

# Скрипт комплексного резервного копіювання PostgreSQL

# Конфігурація
DB_HOST="localhost"
DB_PORT="5432"
DB_NAME="production"
DB_USER="backup_user"
BACKUP_DIR="/var/backups/postgresql"
RETENTION_DAYS=30
S3_BUCKET="s3://company-db-backups"

# Створення директорій
mkdir -p "$BACKUP_DIR"/{full,incremental,wal}

# 1. Повне резервне копіювання (щоденно о 2:00)
function full_backup() {
    echo "Starting full backup..."

    TIMESTAMP=$(date +%Y%m%d_%H%M%S)
    BACKUP_FILE="$BACKUP_DIR/full/full_backup_$TIMESTAMP.dump"

    # Створення дампу
    pg_dump -h $DB_HOST -p $DB_PORT -U $DB_USER \
        -Fc -f "$BACKUP_FILE" $DB_NAME

    # Стиснення
    gzip "$BACKUP_FILE"

    # Вивантаження до S3
    aws s3 cp "$BACKUP_FILE.gz" \
        "$S3_BUCKET/full/full_backup_$TIMESTAMP.dump.gz"

    # Перевірка цілісності
    if [ $? -eq 0 ]; then
        echo "Full backup completed: $BACKUP_FILE.gz"

        # Видалення старих локальних копій
        find "$BACKUP_DIR/full" -name "*.gz" -mtime +$RETENTION_DAYS -delete
    else
        echo "ERROR: Full backup failed!"
        exit 1
    fi
}

# 2. Інкрементальне резервне копіювання (щогодини)
function incremental_backup() {
    echo "Starting incremental backup..."

    # Використання pg_basebackup для базового бекапу
    TIMESTAMP=$(date +%Y%m%d_%H%M%S)
    BACKUP_DIR_INC="$BACKUP_DIR/incremental/$TIMESTAMP"

    pg_basebackup -h $DB_HOST -p $DB_PORT -U $DB_USER \
        -D "$BACKUP_DIR_INC" -Ft -z -P

    if [ $? -eq 0 ]; then
        echo "Incremental backup completed: $BACKUP_DIR_INC"

        # Синхронізація з S3
        aws s3 sync "$BACKUP_DIR_INC" \
            "$S3_BUCKET/incremental/$TIMESTAMP/"
    fi
}

# 3. Безперервне архівування WAL (Write-Ahead Log)
function archive_wal() {
    # Ця функція викликається PostgreSQL автоматично
    # Конфігурація в postgresql.conf:
    # archive_mode = on
    # archive_command = '/path/to/script/archive_wal.sh %p %f'

    WAL_FILE=$1
    WAL_NAME=$2

    # Копіювання WAL файлу
    cp "$WAL_FILE" "$BACKUP_DIR/wal/$WAL_NAME"

    # Вивантаження до S3
    aws s3 cp "$BACKUP_DIR/wal/$WAL_NAME" \
        "$S3_BUCKET/wal/$WAL_NAME"

    # Видалення старих WAL файлів (зберігаємо 7 днів)
    find "$BACKUP_DIR/wal" -name "*.gz" -mtime +7 -delete
}

# 4. Моментальний знімок (Snapshot) для критичних операцій
function create_snapshot() {
    echo "Creating database snapshot..."

    SNAPSHOT_NAME="snapshot_$(date +%Y%m%d_%H%M%S)"

    # В PostgreSQL використовуємо pg_export_snapshot
    psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
        -c "SELECT pg_export_snapshot();"

    echo "Snapshot created: $SNAPSHOT_NAME"
}

# 5. Експорт окремих таблиць для швидкого відновлення
function export_critical_tables() {
    TIMESTAMP=$(date +%Y%m%d_%H%M%S)

    CRITICAL_TABLES=("users" "orders" "payments" "products")

    for table in "${CRITICAL_TABLES[@]}"; do
        pg_dump -h $DB_HOST -p $DB_PORT -U $DB_USER \
            -t $table -Fc -f "$BACKUP_DIR/tables/${table}_$TIMESTAMP.dump" \
            $DB_NAME

        gzip "$BACKUP_DIR/tables/${table}_$TIMESTAMP.dump"
    done
}

# Головна функція
case "$1" in
    full)
        full_backup
        ;;
    incremental)
        incremental_backup
        ;;
    snapshot)
        create_snapshot
        ;;
    tables)
        export_critical_tables
        ;;
    wal)
        archive_wal "$2" "$3"
        ;;
    *)
        echo "Usage: $0 {full|incremental|snapshot|tables|wal}"
        exit 1
        ;;
esac
```

Автоматизація через cron:

```cron
# Резервне копіювання PostgreSQL

# Повне щоденне резервне копіювання о 2:00
0 2 * * * /usr/local/bin/backup_postgresql.sh full >> /var/log/postgresql/backup.log 2>&1

# Інкрементальне резервне копіювання кожні 6 годин
0 */6 * * * /usr/local/bin/backup_postgresql.sh incremental >> /var/log/postgresql/backup.log 2>&1

# Експорт критичних таблиць кожні 2 години
0 */2 * * * /usr/local/bin/backup_postgresql.sh tables >> /var/log/postgresql/backup.log 2>&1

# Перевірка цілісності щотижня
0 3 * * 0 /usr/local/bin/verify_backups.sh >> /var/log/postgresql/verify.log 2>&1
```

### Відновлення після збоїв

Процедури відновлення для різних сценаріїв:

```bash
#!/bin/bash

# Скрипт відновлення PostgreSQL

DB_HOST="localhost"
DB_PORT="5432"
DB_NAME="production"
DB_USER="postgres"
BACKUP_DIR="/var/backups/postgresql"
S3_BUCKET="s3://company-db-backups"

# 1. Повне відновлення з останнього дампу
function restore_full() {
    echo "Starting full restore..."

    # Завантаження останнього бекапу з S3
    LATEST_BACKUP=$(aws s3 ls "$S3_BUCKET/full/" | sort | tail -n 1 | awk '{print $4}')

    aws s3 cp "$S3_BUCKET/full/$LATEST_BACKUP" "$BACKUP_DIR/restore/"

    # Розпакування
    gunzip "$BACKUP_DIR/restore/$LATEST_BACKUP"

    # Видалення існуючої бази
    dropdb -h $DB_HOST -p $DB_PORT -U $DB_USER $DB_NAME

    # Створення нової бази
    createdb -h $DB_HOST -p $DB_PORT -U $DB_USER $DB_NAME

    # Відновлення
    pg_restore -h $DB_HOST -p $DB_PORT -U $DB_USER \
        -d $DB_NAME -Fc "${BACKUP_DIR}/restore/${LATEST_BACKUP%.gz}"

    if [ $? -eq 0 ]; then
        echo "Full restore completed successfully"
    else
        echo "ERROR: Restore failed!"
        exit 1
    fi
}

# 2. Point-in-Time Recovery (PITR)
function restore_pitr() {
    TARGET_TIME=$1

    if [ -z "$TARGET_TIME" ]; then
        echo "ERROR: Target time required (format: 'YYYY-MM-DD HH:MM:SS')"
        exit 1
    fi

    echo "Starting PITR to $TARGET_TIME..."

    # Зупинка PostgreSQL
    systemctl stop postgresql

    # Видалення старих даних
    rm -rf /var/lib/postgresql/14/main/*

    # Відновлення останнього базового бекапу
    LATEST_BASE=$(aws s3 ls "$S3_BUCKET/incremental/" | sort | tail -n 1 | awk '{print $2}')

    aws s3 sync "$S3_BUCKET/incremental/$LATEST_BASE" \
        /var/lib/postgresql/14/main/

    # Завантаження WAL файлів
    aws s3 sync "$S3_BUCKET/wal/" /var/lib/postgresql/14/wal_archive/

    # Створення recovery.conf
    cat > /var/lib/postgresql/14/main/recovery.signal << EOF
restore_command = 'cp /var/lib/postgresql/14/wal_archive/%f %p'
recovery_target_time = '$TARGET_TIME'
recovery_target_action = 'promote'
EOF

    # Налаштування прав
    chown -R postgres:postgres /var/lib/postgresql/14/main/

    # Запуск PostgreSQL (він автоматично виконає відновлення)
    systemctl start postgresql

    # Моніторинг процесу відновлення
    while systemctl is-active --quiet postgresql; do
        if psql -U postgres -c "SELECT pg_is_in_recovery();" | grep -q "f"; then
            echo "PITR completed successfully to $TARGET_TIME"
            break
        fi
        sleep 5
    done
}

# 3. Відновлення окремої таблиці
function restore_table() {
    TABLE_NAME=$1

    if [ -z "$TABLE_NAME" ]; then
        echo "ERROR: Table name required"
        exit 1
    fi

    echo "Restoring table: $TABLE_NAME..."

    # Знаходження останнього бекапу таблиці
    LATEST_TABLE=$(aws s3 ls "$S3_BUCKET/tables/" | grep "$TABLE_NAME" | sort | tail -n 1 | awk '{print $4}')

    aws s3 cp "$S3_BUCKET/tables/$LATEST_TABLE" "$BACKUP_DIR/restore/"

    # Розпакування
    gunzip "$BACKUP_DIR/restore/$LATEST_TABLE"

    # Відновлення таблиці
    pg_restore -h $DB_HOST -p $DB_PORT -U $DB_USER \
        -d $DB_NAME -t $TABLE_NAME \
        "${BACKUP_DIR}/restore/${LATEST_TABLE%.gz}"

    echo "Table $TABLE_NAME restored successfully"
}

# 4. Тестове відновлення (валідація бекапів)
function test_restore() {
    echo "Starting test restore validation..."

    TEST_DB="test_restore_$(date +%Y%m%d_%H%M%S)"

    # Завантаження останнього бекапу
    LATEST_BACKUP=$(aws s3 ls "$S3_BUCKET/full/" | sort | tail -n 1 | awk '{print $4}')
    aws s3 cp "$S3_BUCKET/full/$LATEST_BACKUP" "$BACKUP_DIR/test/"

    gunzip "$BACKUP_DIR/test/$LATEST_BACKUP"

    # Створення тестової бази
    createdb -h $DB_HOST -p $DB_PORT -U $DB_USER $TEST_DB

    # Відновлення
    pg_restore -h $DB_HOST -p $DB_PORT -U $DB_USER \
        -d $TEST_DB -Fc "${BACKUP_DIR}/test/${LATEST_BACKUP%.gz}"

    if [ $? -eq 0 ]; then
        # Перевірка цілісності
        psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $TEST_DB \
            -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public';"

        echo "Test restore successful. Backup is valid."

        # Видалення тестової бази
        dropdb -h $DB_HOST -p $DB_PORT -U $DB_USER $TEST_DB
    else
        echo "ERROR: Test restore failed! Backup may be corrupted."
        exit 1
    fi
}

# Головна функція
case "$1" in
    full)
        restore_full
        ;;
    pitr)
        restore_pitr "$2"
        ;;
    table)
        restore_table "$2"
        ;;
    test)
        test_restore
        ;;
    *)
        echo "Usage: $0 {full|pitr|table|test} [parameters]"
        echo "Examples:"
        echo "  $0 full"
        echo "  $0 pitr '2024-01-15 14:30:00'"
        echo "  $0 table users"
        echo "  $0 test"
        exit 1
        ;;
esac
```

## Висновки

Безпека баз даних є багатошаровою системою захисту, що включає контроль доступу, шифрування, аудит та резервне копіювання. Вибір моделі контролю доступу залежить від специфічних вимог організації: дискреційна модель надає гнучкість, мандатна забезпечує суворий централізований контроль, а рольова модель пропонує баланс між ними.

Захист від загроз вимагає комплексного підходу: використання параметризованих запитів для запобігання SQL ін'єкціям, впровадження багатофакторної автентифікації, моніторинг аномальної активності та застосування криптографічних методів захисту даних.

Шифрування даних як у спокої, так і при передачі є обов'язковим для захисту конфіденційної інформації. Управління ключами шифрування та їх регулярна ротація є критично важливими для підтримки безпеки.

Систематичний аудит та моніторинг дозволяють виявляти інциденти безпеки на ранніх стадіях та швидко реагувати на загрози. Аналітика логів допомагає ідентифікувати підозрілі патерни поведінки та потенційні атаки.

Надійна стратегія резервного копіювання є останньою лінією захисту від втрати даних. Комбінація повних, інкрементальних бекапів та безперервного архівування WAL забезпечує можливість відновлення даних на будь-який момент часу з мінімальними втратами.

Ефективне адміністрування баз даних вимагає постійної уваги до безпеки, регулярного тестування процедур відновлення, моніторингу продуктивності та проактивного виявлення потенційних проблем до того, як вони вплинуть на роботу системи.
