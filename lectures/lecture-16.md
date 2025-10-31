# Лекція 16. Масштабування та розподілені архітектури

## Вступ

Сучасні інформаційні системи повинні обробляти постійно зростаючі обсяги даних та навантаження від мільйонів користувачів по всьому світу. Традиційні підходи з єдиним сервером баз даних швидко досягають своїх меж, що робить масштабування критично важливим аспектом проєктування систем управління даними.

Розподілені архітектури баз даних дозволяють досягти рівнів продуктивності, надійності та доступності, які неможливі в межах однієї машини. Водночас, розподіленість вносить складність у забезпечення консистентності даних, координацію транзакцій та відмовостійкість системи.

Розуміння принципів масштабування та розподілених архітектур є фундаментальним для розробників, які проєктують системи, здатні рости разом з бізнесом та забезпечувати безперебійну роботу навіть при відмовах окремих компонентів.

## Вертикальне та горизонтальне масштабування

### Вертикальне масштабування (Scale Up)

Вертикальне масштабування передбачає збільшення потужності окремого сервера шляхом додавання процесорів, оперативної пам'яті, дискового простору або заміни компонентів на більш продуктивні.

#### Характеристики вертикального масштабування

Принцип роботи полягає в тому, що замість додавання нових серверів система покращується через модернізацію існуючого обладнання. Наприклад, сервер з 16 ГБ оперативної пам'яті замінюється на конфігурацію з 64 ГБ, або процесор з чотирма ядрами оновлюється до варіанту з шістнадцятьма ядрами.

Типові сценарії застосування включають реляційні бази даних, які важко розподілити горизонтально, застосунки з високими вимогами до консистентності даних, системи з складними транзакціями, що охоплюють багато таблиць, legacy застосунки, не призначені для розподіленого виконання.

Приклад вертикального масштабування PostgreSQL:

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

#### Переваги вертикального масштабування

Простота реалізації є головною перевагою, оскільки не потрібно переписувати код застосунку або змінювати архітектуру системи. Розробники можуть продовжувати використовувати звичні підходи до роботи з базою даних.

Збереження ACID гарантій означає, що всі властивості транзакцій залишаються в силі без додаткових зусиль. Атомарність, консистентність, ізольованість та довговічність забезпечуються природним чином у межах однієї машини.

Відсутність мережевої латентності усуває затримки, пов'язані з передачею даних між вузлами. Всі операції виконуються локально, що забезпечує мінімальний час відгуку.

Простіше адміністрування вимагає управління лише одним сервером замість кластера машин. Резервне копіювання, моніторинг та оновлення виконуються централізовано.

#### Недоліки вертикального масштабування

Обмеження масштабованості є фундаментальною проблемою, оскільки існує фізична межа потужності одного сервера. Навіть найпотужніші комерційні сервери мають кінцеву продуктивність.

Приклад розрахунку меж вертикального масштабування:

```
Сучасний high-end сервер (2024):
- Процесор: AMD EPYC 9654 (96 ядер, 192 потоки)
- Оперативна пам'ять: до 6 TB DDR5
- Дискова система: до 24 NVMe SSD дисків
- Мережа: 100 Gbps

Обмеження:
- Максимальна пропускна здатність процесора
- Пропускна здатність шини пам'яті
- Швидкість дискової підсистеми
- Можливості охолодження

Коли досягнуті ці межі, єдиний варіант - горизонтальне масштабування
```

Єдина точка відмови створює критичний ризик, оскільки відмова сервера призводить до повної недоступності системи. Навіть з резервним копіюванням відновлення може зайняти години.

Висока вартість зростає експоненціально зі збільшенням потужності. Подвоєння продуктивності часто коштує в чотири рази більше, а найпотужніші конфігурації можуть коштувати сотні тисяч доларів.

Необхідність простою при оновленні означає, що більшість операцій з модернізації вимагають зупинки сервера, що призводить до недоступності сервісу.

### Горизонтальне масштабування (Scale Out)

Горизонтальне масштабування передбачає додавання нових серверів до існуючого кластера для розподілу навантаження та даних між множиною машин.

#### Характеристики горизонтального масштабування

Принцип роботи базується на розподілі даних та обчислень між багатьма серверами. Замість одного потужного сервера система складається з множини відносно недорогих машин, що працюють координовано.

Типові сценарії застосування включають веб-застосунки з великою кількістю користувачів, NoSQL бази даних з масивними обсягами даних, системи аналітики великих даних, мікросервісні архітектури з незалежними компонентами.

Приклад горизонтального масштабування з PostgreSQL та PgBouncer:

```bash
#!/bin/bash

# Конфігурація master-slave реплікації

# На master сервері
cat > /etc/postgresql/15/main/postgresql.conf << EOF
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
sudo -u postgres pg_basebackup -h master.example.com -D /var/lib/postgresql/15/main -U replication_user -P -v -R -X stream -C -S replica_1

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

Використання connection pooler для розподілу читання:

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

#### Переваги горизонтального масштабування

Практично необмежена масштабованість дозволяє додавати нові сервери в міру зростання навантаження. Теоретично можна розширювати систему до тисяч вузлів.

Відмовостійкість забезпечується через дублювання даних на кількох серверах. Відмова одного вузла не призводить до недоступності системи.

Економічна ефективність досягається через використання звичайного обладнання. Замість одного дорогого сервера можна використовувати багато дешевших машин.

Географічний розподіл дозволяє розміщувати сервери близько до користувачів у різних регіонах світу, зменшуючи латентність.

#### Недоліки горизонтального масштабування

Складність архітектури вимагає ретельного проєктування системи з урахуванням розподіленості. Треба вирішувати проблеми консистентності, синхронізації та координації.

Складніше забезпечення консистентності даних через те, що інформація розподілена між кількома серверами. Транзакції, що охоплюють кілька вузлів, потребують спеціальних протоколів.

Мережева латентність додає затримки при передачі даних між вузлами. Операції, що вимагають координації кількох серверів, виконуються повільніше.

Складніше адміністрування потребує управління кластером серверів, моніторингу стану кожного вузла, координації оновлень.

### Компроміси та обмеження

#### CAP теорема

CAP теорема стверджує, що розподілена система не може одночасно гарантувати всі три властивості: консистентність, доступність та стійкість до розділення мережі.

Консистентність означає, що всі вузли бачать однакові дані в один і той самий момент часу. Будь-яке читання отримує найновіші записані дані.

Доступність гарантує, що кожен запит отримує відповідь без гарантії, що ця відповідь містить найновіші дані. Система продовжує працювати навіть при відмові частини вузлів.

Стійкість до розділення означає, що система продовжує працювати навіть при втраті або затримці повідомлень між вузлами. Мережеві проблеми не зупиняють систему.

Практичні наслідки CAP теорії:

```
CP системи (Consistency + Partition tolerance):
- Приклади: MongoDB (з majority write concern), HBase, Redis Cluster
- Вибір: Консистентність важливіша за доступність
- Поведінка: При мережевому розділенні частина вузлів стає недоступною
- Сценарії: Фінансові транзакції, інвентарізація товарів

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

## Стратегії шардингу

Шардинг є методом горизонтального розподілу даних між кількома базами даних або серверами. Кожен шард містить підмножину загальних даних.

### Діапазонний шардинг (Range-based Sharding)

Діапазонний шардинг розподіляє дані на основі діапазонів значень ключа шардингу. Наприклад, користувачі з прізвищами від А до М на одному сервері, від Н до Я на іншому.

#### Реалізація діапазонного шардингу

Приклад розподілу користувачів за датою реєстрації:

```sql
-- Шард 1: користувачі зареєстровані у 2022 році
CREATE TABLE users_2022 (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    registration_date DATE NOT NULL CHECK (
        registration_date >= '2022-01-01' AND
        registration_date < '2023-01-01'
    ),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Шард 2: користувачі зареєстровані у 2023 році
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

-- Шард 3: користувачі зареєстровані у 2024 році
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
```

Логіка маршрутизації запитів до відповідного шарду:

```python
from datetime import datetime
import psycopg2

class ShardRouter:
    def __init__(self):
        self.shards = {
            2022: {
                'host': 'shard1.example.com',
                'database': 'users_db',
                'table': 'users_2022'
            },
            2023: {
                'host': 'shard2.example.com',
                'database': 'users_db',
                'table': 'users_2023'
            },
            2024: {
                'host': 'shard3.example.com',
                'database': 'users_db',
                'table': 'users_2024'
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

    def get_user_by_email(self, email, registration_year=None):
        if registration_year:
            shards_to_check = [self.shards[registration_year]]
        else:
            shards_to_check = self.shards.values()

        for shard in shards_to_check:
            conn = psycopg2.connect(
                host=shard['host'],
                database=shard['database'],
                user='app_user',
                password='secure_password'
            )

            try:
                with conn.cursor() as cur:
                    query = f"""
                        SELECT user_id, username, email, registration_date
                        FROM {shard['table']}
                        WHERE email = %s
                    """
                    cur.execute(query, (email,))
                    result = cur.fetchone()
                    if result:
                        return result
            finally:
                conn.close()

        return None

router = ShardRouter()
user_id = router.insert_user(
    'john_doe',
    'john@example.com',
    datetime(2024, 3, 15)
)
```

#### Переваги діапазонного шардингу

Простота концепції робить цей підхід інтуїтивно зрозумілим. Діапазони значень легко визначити та підтримувати.

Ефективні діапазонні запити дозволяють швидко отримати всі дані в певному діапазоні, оскільки вони знаходяться на одному шарді.

Легкість додавання нових шардів для нових діапазонів дозволяє природно розширювати систему в міру зростання даних.

#### Недоліки діапазонного шардингу

Нерівномірний розподіл даних може призвести до ситуації, коли один шард містить набагато більше записів, ніж інші. Це створює hotspot та знижує ефективність.

Складність перебалансування вимагає перерозподілу даних між шардами при зміні діапазонів, що є трудомістким процесом.

### Хешований шардинг (Hash-based Sharding)

Хешований шардинг використовує хеш-функцію для визначення, на якому шарді зберігати кожен запис. Результат хешування ключа визначає номер шарду.

#### Реалізація хешованого шардингу

Приклад використання consistent hashing для розподілу користувачів:

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

#### Переваги хешованого шардингу

Рівномірний розподіл даних досягається через властивості хеш-функцій, які розподіляють значення випадковим чином.

Передбачуваність розміщення даних дозволяє швидко знайти потрібний шард за ключем без додаткових запитів.

Добра масштабованість для операцій читання та запису за ключем забезпечує високу продуктивність.

#### Недоліки хешованого шардингу

Складність діапазонних запитів вимагає звернення до всіх шардів, оскільки дані з одного діапазону розподілені по різних серверах.

Перехешування при додаванні шардів може призвести до необхідності перерозподілу значної частини даних.

### Директорний шардинг (Directory-based Sharding)

Директорний шардинг використовує окрему таблицю-довідник, яка містить інформацію про розміщення даних на шардах.

#### Реалізація директорного шардингу

Приклад використання lookup таблиці для визначення шарду:

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

Логіка роботи з директорним шардингом:

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
        # Отримання поточної інформації
        old_shard = self.get_shard_info(tenant_id)

        # Копіювання даних на новий шард
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

            # Оновлення довідника
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

            # Очищення кешу
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

# Виконання запиту для конкретного tenant
results = sharding.execute_query(
    tenant_id=1001,
    query="SELECT * FROM orders WHERE order_date > %s",
    params=('2024-01-01',)
)

# Міграція tenant на інший шард
sharding.migrate_tenant(
    tenant_id=1002,
    new_shard_name='shard_premium',
    new_shard_host='premium-shard.example.com',
    new_shard_database='tenant_db'
)
```

#### Переваги директорного шардингу

Максимальна гнучкість у розподілі даних дозволяє розміщувати кожного клієнта на оптимальному шарді незалежно від інших.

Легкість міграції даних між шардами потребує лише оновлення lookup таблиці без зміни архітектури.

Можливість балансування навантаження шляхом переміщення найбільш активних клієнтів на окремі потужні сервери.

#### Недоліки директорного шардингу

Додаткова точка відмови у вигляді lookup сервера може паралізувати всю систему при його недоступності.

Затримка на lookup запит додає додаткову мережеву операцію перед кожним запитом до даних.

Складність підтримки консистентності між довідником та фактичним розміщенням даних.

## Реплікація даних

Реплікація передбачає створення та підтримку копій даних на кількох серверах для забезпечення відмовостійкості та розподілу навантаження читання.

### Синхронна реплікація

Синхронна реплікація гарантує, що дані записані на всі репліки перед підтвердженням транзакції клієнту.

#### Характеристики синхронної реплікації

Процес запису включає наступні кроки: клієнт відправляє запит на запис, master сервер записує дані локально, master чекає підтвердження від усіх синхронних реплік, тільки після цього транзакція вважається завершеною.

Приклад налаштування синхронної реплікації в PostgreSQL:

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

Конфігурація з гарантованим збереженням даних:

```bash
# postgresql.conf на master
synchronous_commit = remote_apply
synchronous_standby_names = 'FIRST 2 (replica1, replica2, replica3)'

# Це означає, що master чекає на підтвердження від перших двох доступних реплік
# Транзакція завершується тільки після застосування змін на репліках
```

Переваги синхронної реплікації:

Нульова втрата даних гарантується тим, що дані присутні на кількох серверах перед підтвердженням.

Консистентність читання забезпечує актуальні дані на всіх репліках без затримки.

Автоматичне відновлення після збоїв можливе без втрати транзакцій.

Недоліки синхронної реплікації:

Підвищена латентність запису через необхідність чекати на підтвердження від реплік.

Зниження доступності при недоступності реплік може заблокувати операції запису.

Обмеження продуктивності через послідовне підтвердження транзакцій.

### Асинхронна реплікація

Асинхронна реплікація не чекає на підтвердження від реплік перед завершенням транзакції.

#### Характеристики асинхронної реплікації

Процес запису виконується так: клієнт відправляє запит на запис, master записує дані локально, master відразу підтверджує транзакцію клієнту, зміни асинхронно передаються на репліки.

Приклад налаштування асинхронної реплікації:

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

Налаштування алертів на затримку реплікації:

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
            if lag > 60:  # Затримка більше 60 секунд
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

Переваги асинхронної реплікації:

Низька латентність запису забезпечує швидке підтвердження транзакцій клієнту.

Висока доступність для запису підтримується навіть при недоступності реплік.

Краща продуктивність через відсутність очікування підтвердження.

Недоліки асинхронної реплікації:

Можлива втрата даних при збої master між записом та реплікацією.

Незначна затримка читання на репліках може показувати дещо застарілі дані.

Складність автоматичного failover через потенційну розбіжність даних.

### Master-slave архітектура

Master-slave це класична архітектура реплікації, де один сервер приймає запити на запис, а інші тільки реплікують дані.

#### Характеристики master-slave

Розподіл відповідальності чітко визначений: master обробляє всі операції запису та частину читання, slave сервери обробляють операції читання та зберігають копію даних.

Типова конфігурація з автоматичним failover:

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
  data_dir: /var/lib/postgresql/15/main
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

Приклад застосунку з розділенням читання та запису:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import NullPool
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
        self.SlaveSession = sessionmaker(bind=random.choice(self.slave_engines))

    def get_write_session(self):
        return self.MasterSession()

    def get_read_session(self):
        engine = random.choice(self.slave_engines)
        Session = sessionmaker(bind=engine)
        return Session()

# Використання
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

Переваги master-slave:

Простота архітектури полегшує розуміння та підтримку системи.

Розподіл навантаження читання між кількома slaves підвищує продуктивність.

Захист даних через наявність кількох копій.

Недоліки master-slave:

Єдина точка відмови для запису створює вразливість системи.

Обмеження масштабування запису неможливо подолати додаванням slaves.

Складність автоматичного promocion slave до master при збої.

### Master-master архітектура

Master-master або multi-master архітектура дозволяє кільком серверам приймати операції запису одночасно.

#### Характеристики master-master

Двонаправлена реплікація означає, що кожен master реплікує свої зміни на інший master та приймає зміни від нього.

Приклад налаштування multi-master replication з BDR PostgreSQL:

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

Розв'язання конфліктів при одночасних змінах:

```sql
-- Налаштування стратегії розв'язання конфліктів
ALTER TABLE users SET (
    bdr.conflict_detection = 'row_origin',
    bdr.conflict_resolution = 'last_update_wins'
);

-- Створення кастомної функції розв'язання конфліктів
CREATE OR REPLACE FUNCTION custom_conflict_resolver()
RETURNS TRIGGER AS $$
BEGIN
    -- Логіка вибору версії при конфлікті
    IF NEW.priority > OLD.priority THEN
        RETURN NEW;
    ELSE
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

Переваги master-master:

Відсутність єдиної точки відмови підвищує надійність системи.

Розподіл навантаження запису між кількома серверами.

Географічний розподіл з можливістю локального запису в кожному регіоні.

Недоліки master-master:

Складність розв'язання конфліктів при одночасних змінах одних даних.

Потенційні проблеми консистентності через затримку реплікації.

Висока складність налаштування та підтримки.

## Консенсус у розподілених системах

Консенсус це процес досягнення згоди між вузлами розподіленої системи про стан даних або послідовність операцій.

### Paxos алгоритм

Paxos є одним з перших формалізованих алгоритмів консенсусу, розроблених Леслі Лампортом у 1989 році.

#### Основні ролі в Paxos

Proposer пропонує значення для досягнення консенсусу та ініціює процес голосування.

Acceptor приймає або відхиляє пропозиції на основі встановлених правил.

Learner дізнається про досягнутий консенсус та застосовує прийняте рішення.

Фази алгоритму Paxos:

Фаза підготовки передбачає, що proposer відправляє prepare запит з номером пропозиції, acceptor обіцяє не приймати пропозиції з меншим номером.

Фаза прийняття включає відправку accept запиту з значенням, acceptor приймає пропозицію, якщо не обіцяв підтримувати вищу.

Фаза фіксації означає, що більшість acceptors прийняла пропозицію, learners дізнаються про консенсус.

Спрощена реалізація Paxos концепцій:

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional, Set

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

        # Перевірка кворуму
        if len(promises) <= len(self.acceptors) // 2:
            return None, "Не досягнуто кворуму на prepare"

        # Вибір значення
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

        # Перевірка кворуму
        if len(accepted) > len(self.acceptors) // 2:
            return proposed_value, "Консенсус досягнуто"

        return None, "Не досягнуто кворуму на accept"

# Використання
acceptors = [Acceptor(i) for i in range(5)]
proposer = Proposer(1, acceptors)

value, status = proposer.propose("initial_value")
print(f"Результат: {value}, Статус: {status}")
```

### Raft алгоритм

Raft розроблений як більш зрозуміла альтернатива Paxos з акцентом на практичність та освітню цінність.

#### Основні концепції Raft

Ролі вузлів у Raft:

Leader приймає запити від клієнтів та реплікує зміни на followers.

Follower пасивно приймає оновлення від leader.

Candidate намагається стати leader під час виборів.

Фази роботи Raft:

Вибори leader відбуваються коли follower не отримує heartbeat протягом таймауту, candidate запитує голоси у інших вузлів, вузол з більшістю голосів стає leader.

Реплікація логу передбачає, що leader отримує запит від клієнта, додає запис до свого лог, реплікує на followers, підтверджує клієнту після реплікації на більшість.

Спрощена імплементація Raft:

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

            # Запит голосів у інших вузлів
            for peer in self.peers:
                if peer.request_vote(
                    self.current_term,
                    self.node_id,
                    len(self.log),
                    self.log[-1]['term'] if self.log else 0
                ):
                    votes += 1

            # Перевірка більшості
            if votes > (len(self.peers) + 1) // 2:
                self.become_leader()

    def become_leader(self):
        print(f"Node {self.node_id} став leader для терміну {self.current_term}")
        self.state = NodeState.LEADER

        # Ініціалізація індексів для followers
        for peer in self.peers:
            self.next_index[peer.node_id] = len(self.log)
            self.match_index[peer.node_id] = 0

        # Відправка heartbeat
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

            # Перевірка попереднього запису
            if prev_log_index > 0:
                if len(self.log) < prev_log_index or \
                   self.log[prev_log_index - 1]['term'] != prev_log_term:
                    return False

            # Додавання нових записів
            for i, entry in enumerate(entries):
                index = prev_log_index + i
                if len(self.log) > index:
                    if self.log[index]['term'] != entry['term']:
                        self.log = self.log[:index]
                        self.log.append(entry)
                else:
                    self.log.append(entry)

            # Оновлення commit_index
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

# Запуск вузлів
for node in nodes:
    thread = Thread(target=node.run, daemon=True)
    thread.start()
```

### Порівняння Paxos та Raft

Складність розуміння Paxos вважається складнішим для розуміння та імплементації, Raft спроектований для зрозумілості.

Практичність імплементації Paxos має багато варіацій та оптимізацій, Raft має чітку специфікацію та меншу варіативність.

Продуктивність обидва алгоритми мають подібну теоретичну продуктивність, практична продуктивність залежить від імплементації.

Використання в індустрії Paxos використовується в Google Chubby, Apache ZooKeeper базується на подібному алгоритмі Zab, Raft використовується в etcd, Consul, CockroachDB.

## NewSQL системи

NewSQL це клас СУБД, що намагається поєднати ACID гарантії традиційних SQL баз даних з горизонтальною масштабованістю NoSQL систем.

### Характеристики NewSQL

Основні цілі NewSQL включають збереження SQL інтерфейсу та ACID гарантій, підтримку горизонтального масштабування та високої доступності, оптимізацію для сучасного обладнання з багатьма ядрами та великими обсягами RAM.

Приклади NewSQL систем:

Google Cloud Spanner забезпечує глобальну консистентність з горизонтальним масштабуванням, використовує атомарні годинники для синхронізації.

CockroachDB сумісний з PostgreSQL wire protocol, використовує Raft для консенсусу, підтримує geo-distributed deployment.

VoltDB оптимізований для in-memory транзакційної обробки, підтримує горизонтальне масштабування та автоматичний sharding.

### Приклад роботи з CockroachDB

Створення розподіленої таблиці з явним зазначенням розміщення:

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

Моніторинг розподілу даних:

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

## Висновки

Масштабування та розподілені архітектури є критично важливими аспектами сучасних систем управління даними. Вибір між вертикальним та горизонтальним масштабуванням залежить від специфічних вимог проєкту, бюджету та очікуваних темпів зростання.

Вертикальне масштабування залишається простим та ефективним рішенням для середніх навантажень, але має фундаментальні обмеження потужності одного сервера. Горизонтальне масштабування надає практично необмежені можливості зростання, але вносить значну складність у архітектуру та управління системою.

Стратегії шардингу дозволяють розподілити дані між множиною серверів, кожна з них має свої переваги та недоліки. Діапазонний шардинг простий та ефективний для діапазонних запитів, хешований забезпечує рівномірний розподіл, директорний надає максимальну гнучкість.

Реплікація даних є фундаментальним механізмом забезпечення відмовостійкості та розподілу навантаження читання. Вибір між синхронною та асинхронною реплікацією визначає баланс між консистентністю та продуктивністю.

Алгоритми консенсусу, такі як Paxos та Raft, забезпечують узгодженість стану в розподілених системах. Розуміння принципів їх роботи є критично важливим для проєктування надійних розподілених баз даних.

NewSQL системи представляють сучасний підхід до поєднання переваг традиційних SQL баз даних та масштабованості NoSQL систем, відкриваючи нові можливості для побудови глобально розподілених застосунків з строгими гарантіями консистентності.
