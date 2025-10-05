# Лекція 18. Моніторинг продуктивності та налагодження

## Вступ

Продуктивність бази даних безпосередньо впливає на користувацький досвід та бізнес-показники організації. Повільні запити призводять до затримок у роботі застосунків, зниження конверсії та незадоволення користувачів. Ефективний моніторинг та налагодження дозволяють виявляти проблеми на ранніх стадіях та підтримувати оптимальну продуктивність системи.

Сучасні бази даних обробляють величезні обсяги даних та тисячі одночасних запитів. Без систематичного моніторингу неможливо зрозуміти, як система поводиться під навантаженням, де виникають вузькі місця та як оптимізувати роботу для досягнення максимальної ефективності.

Розуміння метрик продуктивності, вміння профілювати запити та використовувати інструменти моніторингу є критично важливими навичками для адміністраторів баз даних та розробників, які прагнуть створювати високопродуктивні системи.

## Метрики продуктивності СУБД

Систематичний збір та аналіз метрик дозволяє об'єктивно оцінити стан системи та приймати обґрунтовані рішення щодо оптимізації.

### Пропускна здатність (Throughput)

Пропускна здатність вимірює кількість операцій, які система може обробити за одиницю часу. Це одна з найважливіших метрик для оцінки загальної продуктивності.

#### Вимірювання пропускної здатності

Основні показники включають транзакції за секунду, запити за секунду, обсяг даних, оброблених за одиницю часу.

Збір метрик пропускної здатності в PostgreSQL:

```sql
-- Створення таблиці для збереження статистики
CREATE TABLE performance_metrics (
    metric_id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metric_name VARCHAR(100),
    metric_value NUMERIC,
    metric_unit VARCHAR(50),
    database_name VARCHAR(100),
    additional_info JSONB
);

-- Функція збору метрик пропускної здатності
CREATE OR REPLACE FUNCTION collect_throughput_metrics()
RETURNS void AS $$
DECLARE
    current_stats RECORD;
    previous_stats RECORD;
    time_diff NUMERIC;
    tps NUMERIC;
    qps NUMERIC;
BEGIN
    -- Отримання поточної статистики
    SELECT
        xact_commit + xact_rollback as total_transactions,
        tup_returned + tup_fetched + tup_inserted + tup_updated + tup_deleted as total_operations,
        blks_read + blks_hit as total_blocks,
        now() as current_time
    INTO current_stats
    FROM pg_stat_database
    WHERE datname = current_database();

    -- Отримання попередньої статистики
    SELECT
        metric_value,
        timestamp
    INTO previous_stats
    FROM performance_metrics
    WHERE metric_name = 'total_transactions'
        AND database_name = current_database()
    ORDER BY timestamp DESC
    LIMIT 1;

    IF previous_stats IS NOT NULL THEN
        -- Розрахунок різниці в часі
        time_diff := EXTRACT(EPOCH FROM (current_stats.current_time - previous_stats.timestamp));

        -- Розрахунок TPS (Transactions Per Second)
        tps := (current_stats.total_transactions - previous_stats.metric_value) / time_diff;

        -- Збереження метрик
        INSERT INTO performance_metrics (metric_name, metric_value, metric_unit, database_name)
        VALUES ('transactions_per_second', tps, 'tps', current_database());
    END IF;

    -- Збереження поточних значень для наступного виміру
    INSERT INTO performance_metrics (metric_name, metric_value, metric_unit, database_name)
    VALUES
        ('total_transactions', current_stats.total_transactions, 'count', current_database()),
        ('total_operations', current_stats.total_operations, 'count', current_database()),
        ('total_blocks', current_stats.total_blocks, 'blocks', current_database());
END;
$$ LANGUAGE plpgsql;

-- Автоматичний збір метрик кожні 60 секунд
-- Використовуючи pg_cron розширення
CREATE EXTENSION IF NOT EXISTS pg_cron;

SELECT cron.schedule('collect-metrics', '60 seconds', 'SELECT collect_throughput_metrics()');
```

Аналіз трендів пропускної здатності:

```sql
-- Аналіз пропускної здатності за останню добу
SELECT
    date_trunc('hour', timestamp) as hour,
    AVG(metric_value) as avg_tps,
    MAX(metric_value) as max_tps,
    MIN(metric_value) as min_tps,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY metric_value) as p95_tps,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY metric_value) as p99_tps
FROM performance_metrics
WHERE metric_name = 'transactions_per_second'
    AND timestamp > CURRENT_TIMESTAMP - INTERVAL '24 hours'
GROUP BY date_trunc('hour', timestamp)
ORDER BY hour;

-- Порівняння пропускної здатності по днях тижня
SELECT
    EXTRACT(DOW FROM timestamp) as day_of_week,
    to_char(timestamp, 'Day') as day_name,
    AVG(metric_value) as avg_tps,
    MAX(metric_value) as max_tps
FROM performance_metrics
WHERE metric_name = 'transactions_per_second'
    AND timestamp > CURRENT_TIMESTAMP - INTERVAL '7 days'
GROUP BY EXTRACT(DOW FROM timestamp), to_char(timestamp, 'Day')
ORDER BY day_of_week;

-- Виявлення аномалій у пропускній здатності
WITH stats AS (
    SELECT
        AVG(metric_value) as mean,
        STDDEV(metric_value) as stddev
    FROM performance_metrics
    WHERE metric_name = 'transactions_per_second'
        AND timestamp > CURRENT_TIMESTAMP - INTERVAL '7 days'
)
SELECT
    m.timestamp,
    m.metric_value as tps,
    s.mean,
    s.stddev,
    CASE
        WHEN m.metric_value > s.mean + 3 * s.stddev THEN 'Unusually High'
        WHEN m.metric_value < s.mean - 3 * s.stddev THEN 'Unusually Low'
        ELSE 'Normal'
    END as status
FROM performance_metrics m
CROSS JOIN stats s
WHERE m.metric_name = 'transactions_per_second'
    AND m.timestamp > CURRENT_TIMESTAMP - INTERVAL '24 hours'
    AND (m.metric_value > s.mean + 3 * s.stddev OR m.metric_value < s.mean - 3 * s.stddev)
ORDER BY m.timestamp DESC;
```

Моніторинг пропускної здатності на рівні застосунку:

```python
import psycopg2
import time
from datetime import datetime, timedelta
from collections import deque
import statistics

class ThroughputMonitor:
    """Моніторинг пропускної здатності застосунку"""

    def __init__(self, window_size=60):
        self.window_size = window_size  # Розмір вікна в секундах
        self.operations = deque()  # Черга операцій
        self.start_time = time.time()

    def record_operation(self, operation_type='query'):
        """Реєстрація операції"""
        current_time = time.time()

        # Додавання операції
        self.operations.append({
            'timestamp': current_time,
            'type': operation_type
        })

        # Видалення старих операцій за межами вікна
        cutoff_time = current_time - self.window_size
        while self.operations and self.operations[0]['timestamp'] < cutoff_time:
            self.operations.popleft()

    def get_current_throughput(self):
        """Отримання поточної пропускної здатності"""
        if not self.operations:
            return 0.0

        current_time = time.time()
        time_window = current_time - self.operations[0]['timestamp']

        if time_window == 0:
            return 0.0

        return len(self.operations) / time_window

    def get_statistics(self):
        """Отримання статистики пропускної здатності"""
        if not self.operations:
            return {
                'current_ops_per_sec': 0,
                'total_operations': 0,
                'uptime_seconds': time.time() - self.start_time
            }

        throughput = self.get_current_throughput()
        total_ops = len(self.operations)
        uptime = time.time() - self.start_time

        return {
            'current_ops_per_sec': round(throughput, 2),
            'total_operations': total_ops,
            'uptime_seconds': round(uptime, 2),
            'average_ops_per_sec': round(total_ops / uptime, 2) if uptime > 0 else 0
        }

class DatabaseConnectionPool:
    """Пул з'єднань з моніторингом"""

    def __init__(self, connection_string, pool_size=10):
        self.connection_string = connection_string
        self.pool_size = pool_size
        self.monitor = ThroughputMonitor(window_size=60)
        self.connections = []
        self._initialize_pool()

    def _initialize_pool(self):
        """Ініціалізація пулу з'єднань"""
        for _ in range(self.pool_size):
            conn = psycopg2.connect(self.connection_string)
            self.connections.append(conn)

    def execute_query(self, query, params=None):
        """Виконання запиту з моніторингом"""
        start_time = time.time()

        # Отримання з'єднання з пулу
        conn = self.connections[0] if self.connections else None

        if not conn:
            raise Exception("No available connections")

        try:
            with conn.cursor() as cur:
                cur.execute(query, params)
                result = cur.fetchall()
                conn.commit()

            # Реєстрація успішної операції
            self.monitor.record_operation('query')

            execution_time = time.time() - start_time
            return result, execution_time

        except Exception as e:
            conn.rollback()
            raise e

    def get_performance_stats(self):
        """Отримання статистики продуктивності"""
        return self.monitor.get_statistics()

# Використання
pool = DatabaseConnectionPool(
    "postgresql://user:password@localhost/dbname",
    pool_size=20
)

# Симуляція навантаження
import random

for i in range(1000):
    try:
        result, exec_time = pool.execute_query(
            "SELECT * FROM users WHERE user_id = %s",
            (random.randint(1, 10000),)
        )

        # Періодичний вивід статистики
        if i % 100 == 0:
            stats = pool.get_performance_stats()
            print(f"Iteration {i}:")
            print(f"  Current throughput: {stats['current_ops_per_sec']} ops/sec")
            print(f"  Average throughput: {stats['average_ops_per_sec']} ops/sec")
            print(f"  Total operations: {stats['total_operations']}")
            print()

    except Exception as e:
        print(f"Error: {e}")

    # Невелика затримка
    time.sleep(0.01)
```

### Час відгуку (Response Time)

Час відгуку вимірює, скільки часу потрібно для виконання операції від початку до завершення. Це критична метрика для користувацького досвіду.

#### Вимірювання та аналіз часу відгуку

Збір детальної статистики часу виконання запитів:

```sql
-- Встановлення розширення для статистики запитів
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Налаштування параметрів
ALTER SYSTEM SET pg_stat_statements.track = 'all';
ALTER SYSTEM SET pg_stat_statements.max = 10000;
SELECT pg_reload_conf();

-- Аналіз найповільніших запитів
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    min_exec_time,
    max_exec_time,
    stddev_exec_time,
    rows,
    100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS cache_hit_ratio
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Запити з найбільшим розкидом часу виконання
SELECT
    query,
    calls,
    mean_exec_time,
    stddev_exec_time,
    max_exec_time,
    min_exec_time,
    max_exec_time - min_exec_time AS time_range,
    CASE
        WHEN mean_exec_time > 0
        THEN stddev_exec_time / mean_exec_time
        ELSE 0
    END AS coefficient_of_variation
FROM pg_stat_statements
WHERE calls > 100
    AND query NOT LIKE '%pg_stat_statements%'
ORDER BY coefficient_of_variation DESC
LIMIT 20;

-- Запити, що споживають найбільше загального часу
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    ROUND(100.0 * total_exec_time / SUM(total_exec_time) OVER (), 2) AS percent_total_time
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY total_exec_time DESC
LIMIT 20;
```

Створення таблиці для історії метрик часу відгуку:

```sql
-- Таблиця для збереження історії метрик
CREATE TABLE query_response_time_history (
    history_id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    query_hash BIGINT,
    query_text TEXT,
    calls BIGINT,
    total_time NUMERIC,
    mean_time NUMERIC,
    min_time NUMERIC,
    max_time NUMERIC,
    stddev_time NUMERIC,
    p50_time NUMERIC,
    p95_time NUMERIC,
    p99_time NUMERIC
);

-- Функція збору історичних метрик
CREATE OR REPLACE FUNCTION collect_query_metrics()
RETURNS void AS $$
BEGIN
    INSERT INTO query_response_time_history (
        query_hash,
        query_text,
        calls,
        total_time,
        mean_time,
        min_time,
        max_time,
        stddev_time
    )
    SELECT
        queryid,
        query,
        calls,
        total_exec_time,
        mean_exec_time,
        min_exec_time,
        max_exec_time,
        stddev_exec_time
    FROM pg_stat_statements
    WHERE query NOT LIKE '%pg_stat_statements%';
END;
$$ LANGUAGE plpgsql;

-- Планування регулярного збору (кожні 5 хвилин)
SELECT cron.schedule('collect-query-metrics', '*/5 * * * *', 'SELECT collect_query_metrics()');

-- Аналіз деградації продуктивності
WITH current_metrics AS (
    SELECT
        query_hash,
        mean_time as current_mean
    FROM query_response_time_history
    WHERE timestamp > CURRENT_TIMESTAMP - INTERVAL '1 hour'
    GROUP BY query_hash, mean_time
),
historical_metrics AS (
    SELECT
        query_hash,
        AVG(mean_time) as historical_mean,
        STDDEV(mean_time) as historical_stddev
    FROM query_response_time_history
    WHERE timestamp BETWEEN CURRENT_TIMESTAMP - INTERVAL '7 days'
        AND CURRENT_TIMESTAMP - INTERVAL '1 day'
    GROUP BY query_hash
)
SELECT
    h.query_hash,
    qh.query_text,
    c.current_mean,
    h.historical_mean,
    h.historical_stddev,
    ROUND((c.current_mean - h.historical_mean) / h.historical_mean * 100, 2) as percent_change,
    CASE
        WHEN c.current_mean > h.historical_mean + 2 * h.historical_stddev
        THEN 'DEGRADED'
        ELSE 'NORMAL'
    END as status
FROM current_metrics c
JOIN historical_metrics h USING (query_hash)
JOIN query_response_time_history qh ON qh.query_hash = c.query_hash
WHERE c.current_mean > h.historical_mean + 2 * h.historical_stddev
ORDER BY percent_change DESC;
```

Моніторинг часу відгуку на рівні застосунку:

```python
import psycopg2
import time
import numpy as np
from collections import defaultdict
from datetime import datetime

class ResponseTimeTracker:
    """Відстеження часу відгуку запитів"""

    def __init__(self):
        self.query_times = defaultdict(list)
        self.max_samples = 10000

    def record_query(self, query_hash, execution_time):
        """Реєстрація часу виконання запиту"""
        times = self.query_times[query_hash]
        times.append(execution_time)

        # Обмеження кількості зразків
        if len(times) > self.max_samples:
            times.pop(0)

    def get_percentiles(self, query_hash, percentiles=[50, 95, 99]):
        """Розрахунок персентилів часу виконання"""
        times = self.query_times.get(query_hash, [])

        if not times:
            return None

        return {
            f'p{p}': np.percentile(times, p)
            for p in percentiles
        }

    def get_statistics(self, query_hash):
        """Отримання статистики для конкретного запиту"""
        times = self.query_times.get(query_hash, [])

        if not times:
            return None

        return {
            'count': len(times),
            'mean': np.mean(times),
            'median': np.median(times),
            'std': np.std(times),
            'min': min(times),
            'max': max(times),
            'p50': np.percentile(times, 50),
            'p95': np.percentile(times, 95),
            'p99': np.percentile(times, 99)
        }

    def get_slow_queries(self, threshold_ms=1000):
        """Виявлення повільних запитів"""
        slow_queries = []

        for query_hash, times in self.query_times.items():
            if not times:
                continue

            stats = self.get_statistics(query_hash)

            if stats['p95'] > threshold_ms:
                slow_queries.append({
                    'query_hash': query_hash,
                    'p95_time': stats['p95'],
                    'mean_time': stats['mean'],
                    'count': stats['count']
                })

        return sorted(slow_queries, key=lambda x: x['p95_time'], reverse=True)

class MonitoredConnection:
    """З'єднання з БД з моніторингом"""

    def __init__(self, connection_string):
        self.conn = psycopg2.connect(connection_string)
        self.tracker = ResponseTimeTracker()

    def execute_with_monitoring(self, query, params=None):
        """Виконання запиту з моніторингом часу"""
        # Генерація хешу запиту
        query_hash = hash(query.strip())

        start_time = time.time()

        try:
            with self.conn.cursor() as cur:
                cur.execute(query, params)
                result = cur.fetchall()
                self.conn.commit()

            execution_time = (time.time() - start_time) * 1000  # Переведення в мілісекунди

            # Запис часу виконання
            self.tracker.record_query(query_hash, execution_time)

            return result, execution_time

        except Exception as e:
            self.conn.rollback()
            raise e

    def get_query_stats(self, query):
        """Отримання статистики для запиту"""
        query_hash = hash(query.strip())
        return self.tracker.get_statistics(query_hash)

    def report_slow_queries(self, threshold_ms=1000):
        """Звіт про повільні запити"""
        slow_queries = self.tracker.get_slow_queries(threshold_ms)

        print(f"\nSlow Queries Report (threshold: {threshold_ms}ms)")
        print("=" * 80)

        for i, query in enumerate(slow_queries, 1):
            print(f"\n{i}. Query Hash: {query['query_hash']}")
            print(f"   P95 Time: {query['p95_time']:.2f}ms")
            print(f"   Mean Time: {query['mean_time']:.2f}ms")
            print(f"   Executions: {query['count']}")

    def close(self):
        """Закриття з'єднання"""
        self.conn.close()

# Використання
conn = MonitoredConnection("postgresql://user:password@localhost/dbname")

# Виконання запитів
queries = [
    "SELECT * FROM users WHERE user_id = %s",
    "SELECT * FROM orders WHERE created_at > %s",
    "SELECT u.*, COUNT(o.order_id) FROM users u LEFT JOIN orders o ON u.user_id = o.user_id GROUP BY u.user_id"
]

import random
from datetime import datetime, timedelta

for _ in range(1000):
    query = random.choice(queries)

    if "user_id" in query:
        params = (random.randint(1, 10000),)
    elif "created_at" in query:
        params = (datetime.now() - timedelta(days=random.randint(1, 30)),)
    else:
        params = None

    result, exec_time = conn.execute_with_monitoring(query, params)

    # Вивід статистики кожні 100 запитів
    if _ % 100 == 0:
        stats = conn.get_query_stats(query)
        if stats:
            print(f"\nQuery: {query[:50]}...")
            print(f"  Mean: {stats['mean']:.2f}ms")
            print(f"  P95: {stats['p95']:.2f}ms")
            print(f"  P99: {stats['p99']:.2f}ms")

# Звіт про повільні запити
conn.report_slow_queries(threshold_ms=100)

conn.close()
```

### Утилізація ресурсів

Моніторинг використання системних ресурсів допомагає виявити вузькі місця та оптимізувати конфігурацію.

#### Використання процесора

Аналіз навантаження на CPU:

```sql
-- Запити, що споживають найбільше CPU
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    state_change,
    query,
    EXTRACT(EPOCH FROM (now() - query_start)) as runtime_seconds
FROM pg_stat_activity
WHERE state != 'idle'
    AND pid != pg_backend_pid()
ORDER BY query_start
LIMIT 20;

-- Аналіз CPU часу по запитах
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    ROUND(100.0 * total_exec_time / SUM(total_exec_time) OVER (), 2) AS cpu_percent
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Моніторинг CPU на рівні системи:

```python
import psutil
import time
from collections import deque

class CPUMonitor:
    """Моніторинг використання CPU"""

    def __init__(self, window_size=60):
        self.window_size = window_size
        self.cpu_percentages = deque(maxlen=window_size)
        self.process = psutil.Process()

    def collect_metrics(self):
        """Збір метрик CPU"""
        # Загальне використання CPU системою
        system_cpu = psutil.cpu_percent(interval=1)

        # Використання CPU процесом PostgreSQL
        process_cpu = self.process.cpu_percent(interval=1)

        # CPU по ядрам
        cpu_per_core = psutil.cpu_percent(interval=1, percpu=True)

        self.cpu_percentages.append({
            'timestamp': time.time(),
            'system_cpu': system_cpu,
            'process_cpu': process_cpu,
            'cpu_per_core': cpu_per_core
        })

        return {
            'system_cpu': system_cpu,
            'process_cpu': process_cpu,
            'cpu_cores': len(cpu_per_core),
            'cpu_per_core': cpu_per_core
        }

    def get_statistics(self):
        """Отримання статистики CPU"""
        if not self.cpu_percentages:
            return None

        system_cpus = [m['system_cpu'] for m in self.cpu_percentages]
        process_cpus = [m['process_cpu'] for m in self.cpu_percentages]

        return {
            'system_cpu_avg': sum(system_cpus) / len(system_cpus),
            'system_cpu_max': max(system_cpus),
            'process_cpu_avg': sum(process_cpus) / len(process_cpus),
            'process_cpu_max': max(process_cpus),
            'samples': len(self.cpu_percentages)
        }

    def detect_cpu_spike(self, threshold=80):
        """Виявлення сплесків CPU"""
        if not self.cpu_percentages:
            return False

        recent_cpu = self.cpu_percentages[-1]['system_cpu']
        return recent_cpu > threshold

# Використання
cpu_monitor = CPUMonitor()

for _ in range(60):
    metrics = cpu_monitor.collect_metrics()

    print(f"System CPU: {metrics['system_cpu']:.1f}%")
    print(f"PostgreSQL CPU: {metrics['process_cpu']:.1f}%")

    if cpu_monitor.detect_cpu_spike(threshold=80):
        print("⚠️ WARNING: High CPU usage detected!")
        stats = cpu_monitor.get_statistics()
        print(f"Average CPU: {stats['system_cpu_avg']:.1f}%")
        print(f"Max CPU: {stats['system_cpu_max']:.1f}%")

    time.sleep(1)
```

#### Використання пам'яті

Моніторинг використання оперативної пам'яті:

```sql
-- Налаштування пам'яті PostgreSQL
SHOW shared_buffers;
SHOW effective_cache_size;
SHOW work_mem;
SHOW maintenance_work_mem;

-- Аналіз використання буферів
SELECT
    CASE
        WHEN class = 'relation' THEN 'Tables/Indexes'
        WHEN class = 'visibility map' THEN 'Visibility Map'
        WHEN class = 'free space map' THEN 'Free Space Map'
        ELSE class
    END as buffer_type,
    COUNT(*) as buffers,
    pg_size_pretty(COUNT(*) * 8192) as size
FROM pg_buffercache
GROUP BY class
ORDER BY COUNT(*) DESC;

-- Таблиці з найбільшою кількістю буферів у пам'яті
SELECT
    c.relname as table_name,
    COUNT(*) as buffers,
    pg_size_pretty(COUNT(*) * 8192) as cached_size,
    pg_size_pretty(pg_total_relation_size(c.oid)) as total_size,
    ROUND(100.0 * COUNT(*) * 8192 / pg_total_relation_size(c.oid), 2) as percent_cached
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname, c.oid
ORDER BY COUNT(*) DESC
LIMIT 20;

-- Ефективність кешування
SELECT
    datname,
    blks_hit,
    blks_read,
    ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio
FROM pg_stat_database
WHERE datname = current_database();
```

Системний моніторинг пам'яті:

```python
import psutil

class MemoryMonitor:
    """Моніторинг використання пам'яті"""

    def __init__(self):
        self.process = psutil.Process()

    def collect_metrics(self):
        """Збір метрик пам'яті"""
        # Загальна системна пам'ять
        system_memory = psutil.virtual_memory()

        # Пам'ять процесу PostgreSQL
        process_memory = self.process.memory_info()

        # Пам'ять по процесам PostgreSQL
        postgres_processes = []
        for proc in psutil.process_iter(['pid', 'name', 'memory_info']):
            try:
                if 'postgres' in proc.info['name'].lower():
                    postgres_processes.append({
                        'pid': proc.info['pid'],
                        'name': proc.info['name'],
                        'memory_mb': proc.info['memory_info'].rss / 1024 / 1024
                    })
            except (psutil.NoSuchProcess, psutil.AccessDenied):
                continue

        total_postgres_memory = sum(p['memory_mb'] for p in postgres_processes)

        return {
            'system': {
                'total_gb': system_memory.total / 1024 / 1024 / 1024,
                'available_gb': system_memory.available / 1024 / 1024 / 1024,
                'used_percent': system_memory.percent
            },
            'postgres': {
                'processes': len(postgres_processes),
                'total_memory_mb': total_postgres_memory,
                'total_memory_gb': total_postgres_memory / 1024,
                'percent_of_system': (total_postgres_memory * 1024 * 1024 / system_memory.total) * 100
            }
        }

    def check_memory_pressure(self, threshold=90):
        """Перевірка тиску на пам'ять"""
        metrics = self.collect_metrics()

        if metrics['system']['used_percent'] > threshold:
            return True, metrics

        return False, metrics

    def print_report(self):
        """Вивід звіту про пам'ять"""
        metrics = self.collect_metrics()

        print("\nMemory Usage Report")
        print("=" * 60)
        print("\nSystem Memory:")
        print(f"  Total: {metrics['system']['total_gb']:.2f} GB")
        print(f"  Available: {metrics['system']['available_gb']:.2f} GB")
        print(f"  Used: {metrics['system']['used_percent']:.1f}%")

        print("\nPostgreSQL Memory:")
        print(f"  Processes: {metrics['postgres']['processes']}")
        print(f"  Total Memory: {metrics['postgres']['total_memory_gb']:.2f} GB")
        print(f"  % of System: {metrics['postgres']['percent_of_system']:.1f}%")

        # Попередження
        pressure, _ = self.check_memory_pressure()
        if pressure:
            print("\n⚠️ WARNING: High memory usage detected!")

# Використання
memory_monitor = MemoryMonitor()
memory_monitor.print_report()

# Періодичний моніторинг
import time

for _ in range(10):
    pressure, metrics = memory_monitor.check_memory_pressure(threshold=85)

    if pressure:
        print(f"\n⚠️ Memory Pressure Alert!")
        print(f"System memory usage: {metrics['system']['used_percent']:.1f}%")
        print(f"PostgreSQL memory: {metrics['postgres']['total_memory_gb']:.2f} GB")

    time.sleep(60)
```

#### Використання дискового простору та вводу-виводу

Моніторинг дискової активності:

```sql
-- Розмір бази даних та таблиць
SELECT
    pg_database.datname as database_name,
    pg_size_pretty(pg_database_size(pg_database.datname)) as size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;

-- Найбільші таблиці
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as indexes_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- Статистика вводу-виводу
SELECT
    relname as table_name,
    heap_blks_read,
    heap_blks_hit,
    ROUND(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) as cache_hit_ratio,
    idx_blks_read,
    idx_blks_hit,
    ROUND(100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0), 2) as index_cache_hit_ratio
FROM pg_statio_user_tables
ORDER BY heap_blks_read + idx_blks_read DESC
LIMIT 20;

-- Таблиці з найбільшою кількістю sequential scans
SELECT
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    CASE
        WHEN seq_scan > 0 THEN ROUND(seq_tup_read::NUMERIC / seq_scan, 2)
        ELSE 0
    END as avg_rows_per_seq_scan,
    n_live_tup as estimated_rows
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_scan DESC
LIMIT 20;
```

Моніторинг дискового простору на рівні системи:

```python
import psutil
import os
from datetime import datetime

class DiskMonitor:
    """Моніторинг дискової активності"""

    def __init__(self, postgres_data_dir='/var/lib/postgresql/14/main'):
        self.postgres_data_dir = postgres_data_dir
        self.previous_io = None

    def get_disk_usage(self):
        """Отримання інформації про використання диску"""
        # Знаходження точки монтування для PostgreSQL
        mount_point = self._get_mount_point(self.postgres_data_dir)

        # Статистика використання
        usage = psutil.disk_usage(mount_point)

        return {
            'mount_point': mount_point,
            'total_gb': usage.total / 1024 / 1024 / 1024,
            'used_gb': usage.used / 1024 / 1024 / 1024,
            'free_gb': usage.free / 1024 / 1024 / 1024,
            'percent_used': usage.percent
        }

    def get_io_statistics(self):
        """Отримання статистики вводу-виводу"""
        # Дискова активність
        io_counters = psutil.disk_io_counters()

        current_io = {
            'read_count': io_counters.read_count,
            'write_count': io_counters.write_count,
            'read_bytes': io_counters.read_bytes,
            'write_bytes': io_counters.write_bytes,
            'read_time': io_counters.read_time,
            'write_time': io_counters.write_time,
            'timestamp': datetime.now()
        }

        # Розрахунок швидкості
        if self.previous_io:
            time_delta = (current_io['timestamp'] - self.previous_io['timestamp']).total_seconds()

            if time_delta > 0:
                read_mb_per_sec = (current_io['read_bytes'] - self.previous_io['read_bytes']) / 1024 / 1024 / time_delta
                write_mb_per_sec = (current_io['write_bytes'] - self.previous_io['write_bytes']) / 1024 / 1024 / time_delta

                current_io['read_mb_per_sec'] = read_mb_per_sec
                current_io['write_mb_per_sec'] = write_mb_per_sec

        self.previous_io = current_io
        return current_io

    def _get_mount_point(self, path):
        """Визначення точки монтування для шляху"""
        path = os.path.abspath(path)
        while not os.path.ismount(path):
            path = os.path.dirname(path)
        return path

    def check_disk_space(self, threshold=90):
        """Перевірка залишку дискового простору"""
        usage = self.get_disk_usage()

        if usage['percent_used'] > threshold:
            return True, usage

        return False, usage

    def get_directory_size(self, directory=None):
        """Розрахунок розміру директорії"""
        if directory is None:
            directory = self.postgres_data_dir

        total_size = 0
        for dirpath, dirnames, filenames in os.walk(directory):
            for filename in filenames:
                filepath = os.path.join(dirpath, filename)
                try:
                    total_size += os.path.getsize(filepath)
                except OSError:
                    continue

        return total_size / 1024 / 1024 / 1024  # Переведення в GB

    def print_report(self):
        """Вивід звіту про дискову активність"""
        usage = self.get_disk_usage()
        io_stats = self.get_io_statistics()
        pg_size = self.get_directory_size()

        print("\nDisk Usage Report")
        print("=" * 60)
        print(f"\nMount Point: {usage['mount_point']}")
        print(f"Total Space: {usage['total_gb']:.2f} GB")
        print(f"Used Space: {usage['used_gb']:.2f} GB ({usage['percent_used']:.1f}%)")
        print(f"Free Space: {usage['free_gb']:.2f} GB")

        print(f"\nPostgreSQL Data Directory Size: {pg_size:.2f} GB")

        print("\nI/O Statistics:")
        print(f"  Read Operations: {io_stats['read_count']:,}")
        print(f"  Write Operations: {io_stats['write_count']:,}")
        print(f"  Total Read: {io_stats['read_bytes'] / 1024 / 1024 / 1024:.2f} GB")
        print(f"  Total Write: {io_stats['write_bytes'] / 1024 / 1024 / 1024:.2f} GB")

        if 'read_mb_per_sec' in io_stats:
            print(f"  Read Speed: {io_stats['read_mb_per_sec']:.2f} MB/s")
            print(f"  Write Speed: {io_stats['write_mb_per_sec']:.2f} MB/s")

        # Попередження
        alert, _ = self.check_disk_space(threshold=85)
        if alert:
            print(f"\n⚠️ WARNING: Disk space usage is high!")

# Використання
disk_monitor = DiskMonitor()
disk_monitor.print_report()

# Періодичний моніторинг
import time

print("\nMonitoring disk I/O (press Ctrl+C to stop)...")
try:
    while True:
        io_stats = disk_monitor.get_io_statistics()

        if 'read_mb_per_sec' in io_stats:
            print(f"{datetime.now().strftime('%H:%M:%S')} - "
                  f"Read: {io_stats['read_mb_per_sec']:6.2f} MB/s, "
                  f"Write: {io_stats['write_mb_per_sec']:6.2f} MB/s")

        time.sleep(5)
except KeyboardInterrupt:
    print("\nMonitoring stopped.")
```

## Профілювання запитів та виявлення вузьких місць

Профілювання дозволяє детально проаналізувати виконання запитів та виявити проблемні місця.

### Використання EXPLAIN та EXPLAIN ANALYZE

EXPLAIN показує план виконання запиту, а EXPLAIN ANALYZE фактично виконує запит та надає реальну статистику.

#### Базове використання EXPLAIN

Аналіз простого запиту:

```sql
-- Створення тестових даних
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100) UNIQUE,
    city VARCHAR(50),
    country VARCHAR(50),
    registration_date DATE,
    is_active BOOLEAN DEFAULT true
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(20)
);

-- Генерація тестових даних
INSERT INTO customers (first_name, last_name, email, city, country, registration_date)
SELECT
    'First_' || i,
    'Last_' || i,
    'user' || i || '@example.com',
    CASE (i % 10)
        WHEN 0 THEN 'Київ'
        WHEN 1 THEN 'Львів'
        WHEN 2 THEN 'Одеса'
        WHEN 3 THEN 'Харків'
        WHEN 4 THEN 'Дніпро'
        ELSE 'Запоріжжя'
    END,
    'Україна',
    CURRENT_DATE - (random() * 365 * 3)::INTEGER,
    random() > 0.1
FROM generate_series(1, 100000) i;

INSERT INTO orders (customer_id, order_date, total_amount, status)
SELECT
    (random() * 100000)::INTEGER + 1,
    CURRENT_TIMESTAMP - (random() * 365)::INTEGER * INTERVAL '1 day',
    (random() * 1000)::NUMERIC(10,2),
    CASE (random() * 3)::INTEGER
        WHEN 0 THEN 'pending'
        WHEN 1 THEN 'completed'
        ELSE 'cancelled'
    END
FROM generate_series(1, 500000) i;

-- Базовий EXPLAIN
EXPLAIN
SELECT * FROM customers WHERE city = 'Київ';

/*
Результат (приблизно):
Seq Scan on customers  (cost=0.00..2084.00 rows=10000 width=...)
  Filter: ((city)::text = 'Київ'::text)
*/

-- EXPLAIN ANALYZE - виконує запит та показує реальні дані
EXPLAIN ANALYZE
SELECT * FROM customers WHERE city = 'Київ';

/*
Результат (приблизно):
Seq Scan on customers  (cost=0.00..2084.00 rows=10000 width=...) (actual time=0.015..15.234 rows=10000 loops=1)
  Filter: ((city)::text = 'Київ'::text)
  Rows Removed by Filter: 90000
Planning Time: 0.123 ms
Execution Time: 16.456 ms
*/
```

Розширений аналіз з різними опціями:

```sql
-- EXPLAIN з додатковими деталями
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT
    c.first_name,
    c.last_name,
    COUNT(o.order_id) as order_count,
    SUM(o.total_amount) as total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.is_active = true
    AND c.registration_date > CURRENT_DATE - INTERVAL '1 year'
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING COUNT(o.order_id) > 5
ORDER BY total_spent DESC
LIMIT 100;

-- Детальний аналіз з BUFFERS
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    c.city,
    COUNT(*) as customer_count,
    AVG(o.total_amount) as avg_order_amount
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date > CURRENT_DATE - INTERVAL '90 days'
GROUP BY c.city
ORDER BY customer_count DESC;
```

Інтерпретація плану виконання:

```python
import psycopg2
import json
from typing import Dict, List

class QueryAnalyzer:
    """Аналізатор планів виконання запитів"""

    def __init__(self, connection_string):
        self.conn = psycopg2.connect(connection_string)

    def analyze_query(self, query, analyze=True):
        """Отримання та аналіз плану виконання"""
        explain_query = f"EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) {query}" if analyze else f"EXPLAIN (FORMAT JSON) {query}"

        with self.conn.cursor() as cur:
            cur.execute(explain_query)
            plan = cur.fetchone()[0]

        return self._parse_plan(plan[0])

    def _parse_plan(self, plan_node):
        """Розбір вузла плану"""
        result = {
            'node_type': plan_node.get('Node Type'),
            'startup_cost': plan_node.get('Startup Cost'),
            'total_cost': plan_node.get('Total Cost'),
            'plan_rows': plan_node.get('Plan Rows'),
            'plan_width': plan_node.get('Plan Width')
        }

        # Додавання фактичних даних якщо є
        if 'Actual Total Time' in plan_node:
            result.update({
                'actual_time': plan_node.get('Actual Total Time'),
                'actual_rows': plan_node.get('Actual Rows'),
                'actual_loops': plan_node.get('Actual Loops')
            })

        # Додавання інформації про буфери
        if 'Shared Hit Blocks' in plan_node:
            result['buffer_stats'] = {
                'shared_hit': plan_node.get('Shared Hit Blocks', 0),
                'shared_read': plan_node.get('Shared Read Blocks', 0),
                'shared_written': plan_node.get('Shared Written Blocks', 0)
            }

        # Рекурсивний розбір дочірніх вузлів
        if 'Plans' in plan_node:
            result['children'] = [self._parse_plan(child) for child in plan_node['Plans']]

        return result

    def identify_issues(self, plan):
        """Виявлення потенційних проблем у плані"""
        issues = []

        def check_node(node, depth=0):
            # Перевірка на Sequential Scan великих таблиць
            if node['node_type'] == 'Seq Scan':
                if node.get('actual_rows', 0) > 10000:
                    issues.append({
                        'severity': 'WARNING',
                        'type': 'Sequential Scan',
                        'message': f"Sequential scan returning {node.get('actual_rows')} rows. Consider adding an index.",
                        'node': node['node_type']
                    })

            # Перевірка на неточні оцінки
            if 'actual_rows' in node and 'plan_rows' in node:
                estimate_ratio = node['actual_rows'] / max(node['plan_rows'], 1)

                if estimate_ratio > 10 or estimate_ratio < 0.1:
                    issues.append({
                        'severity': 'WARNING',
                        'type': 'Poor Row Estimate',
                        'message': f"Estimated {node['plan_rows']} rows but got {node['actual_rows']}. Statistics may be outdated.",
                        'node': node['node_type']
                    })

            # Перевірка на велику кількість дискових читань
            if 'buffer_stats' in node:
                if node['buffer_stats']['shared_read'] > 1000:
                    issues.append({
                        'severity': 'INFO',
                        'type': 'High Disk I/O',
                        'message': f"{node['buffer_stats']['shared_read']} disk blocks read. Data not in cache.",
                        'node': node['node_type']
                    })

            # Перевірка вкладених циклів з великою кількістю ітерацій
            if node['node_type'] == 'Nested Loop':
                if node.get('actual_loops', 1) > 1000:
                    issues.append({
                        'severity': 'CRITICAL',
                        'type': 'Nested Loop',
                        'message': f"Nested loop with {node.get('actual_loops')} iterations. Consider hash or merge join.",
                        'node': node['node_type']
                    })

            # Рекурсивна перевірка дочірніх вузлів
            if 'children' in node:
                for child in node['children']:
                    check_node(child, depth + 1)

        check_node(plan)
        return issues

    def suggest_optimizations(self, query, plan):
        """Пропозиції щодо оптимізації"""
        suggestions = []
        issues = self.identify_issues(plan)

        # Аналіз проблем та формування рекомендацій
        for issue in issues:
            if issue['type'] == 'Sequential Scan':
                suggestions.append({
                    'issue': issue['message'],
                    'suggestion': "Add an appropriate index on the filtered column(s)",
                    'example': "CREATE INDEX idx_column_name ON table_name(column_name);"
                })

            elif issue['type'] == 'Poor Row Estimate':
                suggestions.append({
                    'issue': issue['message'],
                    'suggestion': "Update table statistics",
                    'example': "ANALYZE table_name;"
                })

            elif issue['type'] == 'High Disk I/O':
                suggestions.append({
                    'issue': issue['message'],
                    'suggestion': "Consider increasing shared_buffers or work_mem",
                    'example': "ALTER SYSTEM SET shared_buffers = '4GB';"
                })

            elif issue['type'] == 'Nested Loop':
                suggestions.append({
                    'issue': issue['message'],
                    'suggestion': "Force hash or merge join",
                    'example': "SET enable_nestloop = off; -- For this session only"
                })

        return suggestions

    def print_analysis(self, query):
        """Вивід повного аналізу запиту"""
        print("\nQuery Analysis")
        print("=" * 80)
        print(f"\nQuery:\n{query}\n")

        plan = self.analyze_query(query)

        print("Execution Statistics:")
        print(f"  Total Cost: {plan['total_cost']:.2f}")
        if 'actual_time' in plan:
            print(f"  Actual Time: {plan['actual_time']:.3f} ms")
            print(f"  Actual Rows: {plan['actual_rows']}")

        issues = self.identify_issues(plan)

        if issues:
            print(f"\nIssues Found: {len(issues)}")
            for i, issue in enumerate(issues, 1):
                print(f"\n{i}. [{issue['severity']}] {issue['type']}")
                print(f"   {issue['message']}")

        suggestions = self.suggest_optimizations(query, plan)

        if suggestions:
            print(f"\nOptimization Suggestions:")
            for i, suggestion in enumerate(suggestions, 1):
                print(f"\n{i}. Issue: {suggestion['issue']}")
                print(f"   Suggestion: {suggestion['suggestion']}")
                print(f"   Example: {suggestion['example']}")

    def close(self):
        self.conn.close()

# Використання
analyzer = QueryAnalyzer("postgresql://user:password@localhost/dbname")

# Аналіз запиту
query = """
SELECT
    c.first_name,
    c.last_name,
    COUNT(o.order_id) as order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.city = 'Київ'
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY order_count DESC
LIMIT 10
"""

analyzer.print_analysis(query)
analyzer.close()
```

### Виявлення N+1 проблеми

N+1 проблема виникає, коли виконується один запит для отримання списку об'єктів, а потім по одному запиту для кожного об'єкта.

Приклад N+1 проблеми та її вирішення:

```python
import psycopg2
import time

class ORMSimulator:
    """Симуляція ORM для демонстрації N+1 проблеми"""

    def __init__(self, connection_string):
        self.conn = psycopg2.connect(connection_string)

    # НЕПРАВИЛЬНО - N+1 проблема
    def get_customers_with_orders_n_plus_1(self, limit=10):
        """Неефективний підхід з N+1 запитами"""
        start_time = time.time()
        query_count = 0

        # 1 запит для отримання клієнтів
        with self.conn.cursor() as cur:
            cur.execute(f"SELECT customer_id, first_name, last_name FROM customers LIMIT {limit}")
            customers = cur.fetchall()
            query_count += 1

        results = []

        # N запитів для отримання замовлень кожного клієнта
        for customer in customers:
            customer_id, first_name, last_name = customer

            with self.conn.cursor() as cur:
                cur.execute(
                    "SELECT order_id, total_amount FROM orders WHERE customer_id = %s",
                    (customer_id,)
                )
                orders = cur.fetchall()
                query_count += 1

            results.append({
                'customer_id': customer_id,
                'name': f"{first_name} {last_name}",
                'orders': [
                    {'order_id': o[0], 'total_amount': float(o[1])}
                    for o in orders
                ]
            })

        execution_time = time.time() - start_time

        return results, query_count, execution_time

    # ПРАВИЛЬНО - Один запит з JOIN
    def get_customers_with_orders_optimized(self, limit=10):
        """Оптимізований підхід з одним запитом"""
        start_time = time.time()

        # Один запит з JOIN
        with self.conn.cursor() as cur:
            cur.execute(f"""
                SELECT
                    c.customer_id,
                    c.first_name,
                    c.last_name,
                    o.order_id,
                    o.total_amount
                FROM customers c
                LEFT JOIN orders o ON c.customer_id = o.customer_id
                WHERE c.customer_id IN (
                    SELECT customer_id FROM customers LIMIT {limit}
                )
                ORDER BY c.customer_id, o.order_id
            """)
            rows = cur.fetchall()

        # Групування результатів
        customers_dict = {}

        for row in rows:
            customer_id, first_name, last_name, order_id, total_amount = row

            if customer_id not in customers_dict:
                customers_dict[customer_id] = {
                    'customer_id': customer_id,
                    'name': f"{first_name} {last_name}",
                    'orders': []
                }

            if order_id:
                customers_dict[customer_id]['orders'].append({
                    'order_id': order_id,
                    'total_amount': float(total_amount)
                })

        results = list(customers_dict.values())
        execution_time = time.time() - start_time

        return results, 1, execution_time

    # АЛЬТЕРНАТИВА - Eager loading з окремими запитами
    def get_customers_with_orders_eager(self, limit=10):
        """Eager loading - 2 запити"""
        start_time = time.time()

        # Запит 1: Отримання клієнтів
        with self.conn.cursor() as cur:
            cur.execute(f"SELECT customer_id, first_name, last_name FROM customers LIMIT {limit}")
            customers = cur.fetchall()

        customer_ids = [c[0] for c in customers]

        # Запит 2: Отримання всіх замовлень для цих клієнтів
        with self.conn.cursor() as cur:
            cur.execute(
                """
                SELECT customer_id, order_id, total_amount
                FROM orders
                WHERE customer_id = ANY(%s)
                ORDER BY customer_id, order_id
                """,
                (customer_ids,)
            )
            orders = cur.fetchall()

        # Групування замовлень по клієнтам
        orders_by_customer = {}
        for order in orders:
            customer_id, order_id, total_amount = order
            if customer_id not in orders_by_customer:
                orders_by_customer[customer_id] = []
            orders_by_customer[customer_id].append({
                'order_id': order_id,
                'total_amount': float(total_amount)
            })

        # Формування результатів
        results = []
        for customer in customers:
            customer_id, first_name, last_name = customer
            results.append({
                'customer_id': customer_id,
                'name': f"{first_name} {last_name}",
                'orders': orders_by_customer.get(customer_id, [])
            })

        execution_time = time.time() - start_time

        return results, 2, execution_time

    def compare_approaches(self, limit=10):
        """Порівняння різних підходів"""
        print(f"\nComparing query approaches for {limit} customers\n")
        print("=" * 80)

        # N+1 підхід
        print("\n1. N+1 Approach (INEFFICIENT):")
        results1, queries1, time1 = self.get_customers_with_orders_n_plus_1(limit)
        print(f"   Queries executed: {queries1}")
        print(f"   Execution time: {time1*1000:.2f} ms")
        print(f"   Customers returned: {len(results1)}")

        # JOIN підхід
        print("\n2. Single JOIN Approach (EFFICIENT):")
        results2, queries2, time2 = self.get_customers_with_orders_optimized(limit)
        print(f"   Queries executed: {queries2}")
        print(f"   Execution time: {time2*1000:.2f} ms")
        print(f"   Customers returned: {len(results2)}")

        # Eager loading підхід
        print("\n3. Eager Loading Approach (BALANCED):")
        results3, queries3, time3 = self.get_customers_with_orders_eager(limit)
        print(f"   Queries executed: {queries3}")
        print(f"   Execution time: {time3*1000:.2f} ms")
        print(f"   Customers returned: {len(results3)}")

        # Порівняння
        print("\n" + "=" * 80)
        print("\nPerformance Comparison:")
        print(f"   N+1 is {time1/time2:.1f}x slower than JOIN")
        print(f"   N+1 executes {queries1/queries2:.1f}x more queries than JOIN")
        print(f"   Eager loading is {time3/time2:.1f}x {'slower' if time3 > time2 else 'faster'} than JOIN")

    def close(self):
        self.conn.close()

# Використання
orm = ORMSimulator("postgresql://user:password@localhost/dbname")

# Порівняння для різної кількості записів
for limit in [10, 50, 100]:
    orm.compare_approaches(limit)
    print("\n")

orm.close()
```

## Інструменти моніторингу

Існує багато інструментів для моніторингу баз даних, від вбудованих до сторонніх рішень.

### Системні інструменти PostgreSQL

Вбудовані представлення для моніторингу:

```sql
-- Створення комплексного представлення для моніторингу
CREATE OR REPLACE VIEW system_monitoring AS
SELECT
    -- Інформація про активність
    (SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active') as active_connections,
    (SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'idle') as idle_connections,
    (SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'idle in transaction') as idle_in_transaction,

    -- Статистика транзакцій
    (SELECT xact_commit FROM pg_stat_database WHERE datname = current_database()) as committed_transactions,
    (SELECT xact_rollback FROM pg_stat_database WHERE datname = current_database()) as rolled_back_transactions,

    -- Статистика блокувань
    (SELECT COUNT(*) FROM pg_locks WHERE granted = false) as blocked_queries,

    -- Розмір бази даних
    pg_size_pretty(pg_database_size(current_database())) as database_size,

    -- Ефективність кешу
    (
        SELECT ROUND(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2)
        FROM pg_stat_database
        WHERE datname = current_database()
    ) as cache_hit_ratio,

    -- Час роботи сервера
    EXTRACT(EPOCH FROM (now() - pg_postmaster_start_time())) / 3600 as uptime_hours;

-- Представлення для моніторингу довгих запитів
CREATE OR REPLACE VIEW long_running_queries AS
SELECT
    pid,
    now() - query_start as duration,
    usename,
    application_name,
    client_addr,
    state,
    LEFT(query, 100) as query_preview,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state != 'idle'
    AND now() - query_start > interval '1 minute'
ORDER BY duration DESC;

-- Представлення для моніторингу блокувань
CREATE OR REPLACE VIEW blocking_queries AS
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement,
    blocked_activity.application_name AS blocked_application
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Використання представлень
SELECT * FROM system_monitoring;
SELECT * FROM long_running_queries;
SELECT * FROM blocking_queries;
```

### Сторонні інструменти моніторингу

Інтеграція з Prometheus та Grafana:

```python
from prometheus_client import start_http_server, Gauge, Counter, Histogram
import psycopg2
import time

class PostgreSQLExporter:
    """Експортер метрик PostgreSQL для Prometheus"""

    def __init__(self, connection_string, port=9187):
        self.conn_string = connection_string
        self.port = port

        # Визначення метрик
        self.active_connections = Gauge(
            'postgresql_active_connections',
            'Number of active connections'
        )

        self.idle_connections = Gauge(
            'postgresql_idle_connections',
            'Number of idle connections'
        )

        self.database_size = Gauge(
            'postgresql_database_size_bytes',
            'Size of the database in bytes',
            ['database']
        )

        self.cache_hit_ratio = Gauge(
            'postgresql_cache_hit_ratio',
            'Cache hit ratio percentage',
            ['database']
        )

        self.transactions_committed = Counter(
            'postgresql_transactions_committed_total',
            'Total number of committed transactions',
            ['database']
        )

        self.transactions_rolled_back = Counter(
            'postgresql_transactions_rolled_back_total',
            'Total number of rolled back transactions',
            ['database']
        )

        self.query_duration = Histogram(
            'postgresql_query_duration_seconds',
            'Query execution time in seconds'
        )

        self.table_size = Gauge(
            'postgresql_table_size_bytes',
            'Size of tables in bytes',
            ['schema', 'table']
        )

    def collect_metrics(self):
        """Збір метрик з PostgreSQL"""
        conn = psycopg2.connect(self.conn_string)
        cur = conn.cursor()

        try:
            # Метрики з'єднань
            cur.execute("""
                SELECT
                    COUNT(*) FILTER (WHERE state = 'active') as active,
                    COUNT(*) FILTER (WHERE state = 'idle') as idle
                FROM pg_stat_activity
            """)
            active, idle = cur.fetchone()
            self.active_connections.set(active)
            self.idle_connections.set(idle)

            # Розмір бази даних
            cur.execute("""
                SELECT datname, pg_database_size(datname)
                FROM pg_database
                WHERE datname = current_database()
            """)
            for db_name, size in cur.fetchall():
                self.database_size.labels(database=db_name).set(size)

            # Ефективність кешу
            cur.execute("""
                SELECT
                    datname,
                    100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0) as ratio
                FROM pg_stat_database
                WHERE datname = current_database()
            """)
            for db_name, ratio in cur.fetchall():
                if ratio:
                    self.cache_hit_ratio.labels(database=db_name).set(ratio)

            # Транзакції
            cur.execute("""
                SELECT datname, xact_commit, xact_rollback
                FROM pg_stat_database
                WHERE datname = current_database()
            """)
            for db_name, committed, rolled_back in cur.fetchall():
                self.transactions_committed.labels(database=db_name).inc(committed)
                self.transactions_rolled_back.labels(database=db_name).inc(rolled_back)

            # Розмір таблиць
            cur.execute("""
                SELECT
                    schemaname,
                    tablename,
                    pg_total_relation_size(schemaname||'.'||tablename)
                FROM pg_tables
                WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
            """)
            for schema, table, size in cur.fetchall():
                self.table_size.labels(schema=schema, table=table).set(size)

        finally:
            cur.close()
            conn.close()

    def start(self, interval=15):
        """Запуск експортера"""
        # Запуск HTTP сервера для Prometheus
        start_http_server(self.port)
        print(f"PostgreSQL Exporter started on port {self.port}")

        # Періодичний збір метрик
        while True:
            try:
                self.collect_metrics()
            except Exception as e:
                print(f"Error collecting metrics: {e}")

            time.sleep(interval)

# Використання
if __name__ == "__main__":
    exporter = PostgreSQLExporter(
        "postgresql://user:password@localhost/dbname",
        port=9187
    )
    exporter.start(interval=15)
```

Конфігурація Prometheus (prometheus.yml):

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'postgresql'
    static_configs:
      - targets: ['localhost:9187']
        labels:
          environment: 'production'
          database: 'main'
```

## Планове обслуговування

Регулярне обслуговування бази даних є критично важливим для підтримки оптимальної продуктивності.

### VACUUM та ANALYZE

VACUUM видаляє мертві рядки та запобігає роздуванню таблиць, ANALYZE оновлює статистику для оптимізатора запитів.

Автоматизація обслуговування:

```sql
-- Налаштування автовакууму
ALTER SYSTEM SET autovacuum = on;
ALTER SYSTEM SET autovacuum_max_workers = 4;
ALTER SYSTEM SET autovacuum_naptime = '1min';
ALTER SYSTEM SET autovacuum_vacuum_threshold = 50;
ALTER SYSTEM SET autovacuum_analyze_threshold = 50;
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.1;
ALTER SYSTEM SET autovacuum_analyze_scale_factor = 0.05;

SELECT pg_reload_conf();

-- Ручне обслуговування критичних таблиць
VACUUM (VERBOSE, ANALYZE) customers;
VACUUM (VERBOSE, ANALYZE) orders;

-- VACUUM FULL для повного очищення (вимагає ексклюзивне блокування)
VACUUM FULL customers;

-- Моніторинг роздування таблиць
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_tuple_percent,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Створення задачі планового обслуговування
SELECT cron.schedule(
    'maintenance-vacuum-analyze',
    '0 2 * * *',  -- Щоденно о 2:00
    $$
    DO $$
    DECLARE
        table_record RECORD;
    BEGIN
        FOR table_record IN
            SELECT schemaname, tablename
            FROM pg_tables
            WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
        LOOP
            EXECUTE format('VACUUM ANALYZE %I.%I',
                table_record.schemaname,
                table_record.tablename);
        END LOOP;
    END $$;
    $$
);
```

### Оновлення статистики

Точна статистика критична для оптимізатора запитів:

```sql
-- Оновлення статистики для всієї бази даних
ANALYZE;

-- Оновлення статистики для конкретної таблиці
ANALYZE customers;

-- Детальне оновлення з вищою точністю
ALTER TABLE customers SET (autovacuum_analyze_scale_factor = 0.02);
ANALYZE VERBOSE customers;

-- Перегляд статистики
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats
WHERE schemaname = 'public'
    AND tablename = 'customers'
    AND attname = 'city';

-- Створення розширеної статистики для кореляції стовпців
CREATE STATISTICS customers_city_country_stats (dependencies)
ON city, country FROM customers;

ANALYZE customers;
```

### Дефрагментація та архівування

Стратегія дефрагментації:

```sql
-- Виявлення роздутих індексів
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Перебудова індексів
REINDEX INDEX CONCURRENTLY idx_customers_city;
REINDEX TABLE CONCURRENTLY customers;

-- Перебудова всіх індексів бази даних
REINDEX DATABASE dbname;

-- Автоматизація архівування старих даних
CREATE OR REPLACE FUNCTION archive_old_orders()
RETURNS void AS $$
DECLARE
    archived_count INTEGER;
BEGIN
    -- Переміщення старих замовлень до архівної таблиці
    WITH moved_rows AS (
        DELETE FROM orders
        WHERE order_date < CURRENT_DATE - INTERVAL '2 years'
        RETURNING *
    )
    INSERT INTO orders_archive
    SELECT * FROM moved_rows;

    GET DIAGNOSTICS archived_count = ROW_COUNT;

    RAISE NOTICE 'Archived % orders', archived_count;

    -- Vacuum після видалення
    EXECUTE 'VACUUM ANALYZE orders';
END;
$$ LANGUAGE plpgsql;

-- Планування архівування
SELECT cron.schedule(
    'archive-old-orders',
    '0 3 1 * *',  -- Першого числа кожного місяця о 3:00
    'SELECT archive_old_orders()'
);
```

## Висновки

Моніторинг продуктивності та налагодження баз даних є безперервним процесом, що вимагає систематичного підходу. Ключові метрики, такі як пропускна здатність, час відгуку та утилізація ресурсів, надають об'єктивну картину стану системи та дозволяють виявляти проблеми на ранніх стадіях.

Профілювання запитів за допомогою EXPLAIN ANALYZE є фундаментальним інструментом для виявлення вузьких місць. Розуміння планів виконання дозволяє оптимізувати запити, додавати необхідні індекси та уникати типових проблем, таких як N+1 запити.

Використання спеціалізованих інструментів моніторингу, від вбудованих представлень PostgreSQL до систем збору метрик як Prometheus, дозволяє автоматизувати виявлення проблем та отримувати сповіщення про критичні події.

Планове обслуговування, включно з VACUUM, ANALYZE та дефрагментацією, є необхідним для підтримки оптимальної продуктивності в довгостроковій перспективі. Автоматизація цих задач через pg_cron забезпечує регулярність виконання без ручного втручання.

Ефективний моніторинг та налагодження вимагає комбінації технічних знань, розуміння специфіки застосунку та проактивного підходу до виявлення потенційних проблем до того, як вони вплинуть на користувачів.
