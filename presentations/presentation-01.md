# Еволюція та архітектура систем управління базами даних

## План лекції

1. Історичний розвиток систем обробки даних
2. Проблеми файлово-орієнтованих систем
3. Концепція баз даних
4. Класифікація СУБД
5. Трирівнева архітектура ANSI/SPARC
6. Незалежність даних
7. Компонентна архітектура СУБД
8. Каталог системи (метабаза)
9. Сучасні архітектури та тенденції

## **📚 Основні поняття:**

**База даних (БД)** — організована колекція взаємопов'язаних даних, що зберігаються в комп'ютерній системі та служать для задоволення інформаційних потреб організації.

**Система управління базами даних (СУБД)** — комплекс програмних засобів, призначених для створення структури нової бази даних, наповнення її змістом, редагування змісту та візуалізації інформації.

**Дані** — факти, що можуть бути записані та мають неявне значення.

**Інформація** — оброблені дані, що мають значення та цінність для користувача.

## **1. Історичний розвиток**

```mermaid
timeline
    title Розвиток систем обробки даних
    1890 : Перфокарти Холлеріта
         : Механічна табуляція
    1950 : Перші комп'ютери
         : Файлові системи
    1960 : Ієрархічні та мережеві моделі
         : IBM IMS, IDMS
    1970 : Реляційна модель Кодда
         : System R, SQL
    1990 : Об'єктно-орієнтовані СУБД
         : ObjectDB, Versant
    2000 : NoSQL революція
         : MongoDB, Cassandra
    2010 : Хмарні СУБД
         : AWS RDS, Google Cloud
```

## Доісторія: Перфокарти (1890)

## Ера файлових систем (1950-1960)

### 📁 **Характеристики періоду:**

- 🖥️ **Централізована обробка** на мейнфреймах
- 📋 **Орієнтованість на застосування** — кожна програма мала власні файли
- ⏭️ **Послідовний доступ** — магнітні стрічки
- 🔄 **Пакетна обробка** — накопичення та нічна обробка

### Структура файлової системи 1960-х

```
PAYROLL_SYSTEM/
├── EMPLOYEES.DAT    (співробітники)
├── DEPARTMENTS.DAT  (відділи)
└── SALARIES.DAT     (зарплати)

INVENTORY_SYSTEM/
├── PRODUCTS.DAT     (товари)
├── SUPPLIERS.DAT    (постачальники)
└── ORDERS.DAT       (замовлення)
```

## Перехід до дискового зберігання

### 💽 **Революція жорстких дисків (1960-ті):**

- ⚡ **Швидкий доступ** до довільного запису
- 🔄 **Оновлення на місці** замість перезапису
- 💬 **Інтерактивна робота** з даними
- 📊 **Перші системи управління файлами:** IBM ISAM

### Ієрархічна модель (1960-1970)

```mermaid
graph TD
    A[🏢 УНІВЕРСИТЕТ] --> B[🎓 ФАКУЛЬТЕТ CS]
    A --> C[📊 ФАКУЛЬТЕТ МАТЕМАТИКИ]

    B --> D[💻 КАФЕДРА ПРОГРАМУВАННЯ]
    B --> E[📈 КАФЕДРА СИСТЕМНОГО АНАЛІЗУ]

    D --> F[👨‍🎓 Іван Петров]
    D --> G[👩‍🎓 Марія Коваленко]

    E --> H[👨‍🎓 Олександр Сидоров]
    E --> I[👩‍🎓 Анна Мельник]
```

**✅ Переваги:** Швидкий доступ, природна структура
**❌ Недоліки:** Жорстка структура, дублювання даних

## Реляційна революція (1970)

### 🎯 **Едгар Кодд — IBM Research**

> **"A Relational Model of Data for Large Shared Data Banks"**

### Ключові ідеї:

- 🔢 **Математична основа** — реляційна алгебра
- 🔗 **Логічна незалежність** від фізичної структури
- 📝 **Декларативний підхід** — що потрібно, а не як отримати
- 📋 **Таблична організація** даних

## 12 правил Кодда (1985)

### 🏛️ **Фундамент реляційних СУБД:**

1. **Правило інформації** — дані тільки в таблицях
2. **Гарантований доступ** — через ключ + назву стовпця
3. **Null-значення** — систематична обробка
4. **Динамічний каталог** — метадані як звичайні дані
5. **Повнота мови** — DDL, DML, безпека
6. **Оновлення представлень**

...і ще 6 правил для справжньої реляційної СУБД

## **2. Проблеми файлових систем**

## Основні проблеми файлових систем

### 1. 📊 **Надмірність даних**

```
ФАЙЛ: EMPLOYEES.DAT
+--------+-----------+------------+
| emp_id | dept_name | dept_phone |
+--------+-----------+------------+
| E001   | ІТ Відділ | 555-1234   |
| E002   | ІТ Відділ | 555-1234   | ← Дублювання
+--------+-----------+------------+

ФАЙЛ: DEPARTMENTS.DAT
+-----------+------------+--------+
| dept_name | dept_phone | budget |
+-----------+------------+--------+
| ІТ Відділ | 555-1234   | 500000 | ← Дублювання
+-----------+------------+--------+
```

**❌ Наслідки:** Марнотратство місця, проблеми оновлення, неузгодженість

## 2. Залежність від структури

### 💻 **Приклад COBOL коду 1960-х:**

```cobol
01 EMPLOYEE-RECORD.
   05 EMP-ID      PIC X(5).
   05 EMP-NAME    PIC X(30).
   05 EMP-SALARY  PIC 9(7)V99.

PROCEDURE DIVISION.
READ-EMPLOYEE.
    READ EMPLOYEE-FILE
        AT END MOVE 'Y' TO EOF-FLAG.
```

### 🔄 **Зміна структури = модифікація всіх програм**

- Додати поле "дата прийняття" → переписати ВСІ програми
- Змінити довжину поля → перекомпілювати ВСЕ
- Конвертувати існуючі файли

## 3. Складності інтеграції

### 🏝️ **"Острови інформації"**

```
КАДРИ:              ЗАРПЛАТА:           ПРОЄКТИ:
employee.dat        payroll.dat         projects.dat
├── emp_code        ├── employee_num    ├── worker_id
├── full_name       ├── emp_name        ├── person_name
└── position        └── salary          └── role
```

**❓ Проблема:** Як зрозуміти, що `emp_code = employee_num = worker_id`?

**❌ Результат:**

- Ручне встановлення зв'язків
- Спеціальні програми інтеграції
- Конфлікти в даних

## 4. Проблеми паралельного доступу

### ⚠️ **Втрата оновлення (Lost Update)**

| Час | Програма A | Програма B | Баланс |
|-----|------------|------------|---------|
| T1  | READ 1000  |            | 1000    |
| T2  |            | READ 1000  | 1000    |
| T3  | -100       |            | 1000    |
| T4  |            | -50        | 1000    |
| T5  | WRITE 900  |            | 900     |
| T6  |            | WRITE 950  | **950** |

**💸 Результат:** Втрачено операцію -100! Замість 850 маємо 950.

## 5. Безпека та відновлення

### 🔒 **Обмежені можливості:**

- Контроль тільки на рівні **цілого файлу**
- Відсутність **аудиту операцій**
- Немає **транзакційної моделі**
- Складне **відновлення після збоїв**

### ⚡ **При збої системи:**

- Часткові оновлення
- Невідомий стан даних
- Ручне відновлення цілісності
- Довгі простої

## **3. Концепція баз даних**

## Фундаментальні принципи БД

### 🎯 **Центральна ідея: "Єдине джерело істини"**

```mermaid
graph TD
    A[🗃️ ЦЕНТРАЛІЗОВАНА БД] --> B[👥 Система кадрів]
    A --> C[💰 Система зарплати]
    A --> D[📋 Система проєктів]
    A --> E[📊 Система звітності]

    F[Співробітники] --> A
    G[Відділи] --> A
    H[Зарплати] --> A
    I[Проєкти] --> A
```

### ✅ **Результати:**
- Ліквідація дублювання
- Гарантія узгодженості
- Централізоване адміністрування
- Спрощення оновлень

## Незалежність даних

### 🔄 **Два типи незалежності:**

**Логічна незалежність:**
- Додавання таблиць не впливає на існуючі програми
- Зміна зв'язків не торкається незалежних операцій

**Фізична незалежність:**
- Зміна індексів прозора для користувачів
- Перенесення на інші диски непомітне
- Зміна алгоритмів зберігання

```sql
-- Логічне представлення незмінне
SELECT employee_name, department, salary
FROM employee_view;

-- Фізична реалізація може змінюватися:
-- ✓ Додавання індексів
-- ✓ Розбиття таблиць
-- ✓ Зміна дискового розміщення
```
## Архітектура та компоненти СУБД: огляд

### 🏗️ **Що розглянемо детальніше:**

- **Трирівнева архітектура ANSI/SPARC** (1975): зовнішній → концептуальний → внутрішній рівень
- **Компоненти СУБД:** обробник запитів, менеджер транзакцій, менеджер буферів, каталог системи
- Детальний розгляд — у розділах 5–8, одразу після класифікації типів СУБД

## **4. Класифікація СУБД**

## За моделями даних

### 📊 **Основні типи:**

1. **🌳 Ієрархічні** — дерево записів
2. **🕸️ Мережеві** — складні зв'язки
3. **📋 Реляційні** — таблиці + SQL
4. **🎯 Об'єктно-орієнтовані** — об'єкти + методи
5. **📄 NoSQL** — документи, графи, ключ-значення

### Порівняння моделей

| Модель | Структура | Переваги | Недоліки |
|--------|-----------|----------|----------|
| 🌳 Ієрархічна | Дерево | Швидкість | Жорсткість |
| 🕸️ Мережева | Граф | Складні зв'язки | Складність |
| 📋 Реляційна | Таблиці | Гнучкість, SQL | Об'єктно-реляційний розрив |
| 🎯 ОО | Об'єкти | Природність | Нестандартність |
| 📄 NoSQL | Різна | Масштабованість | Консистентність |

## Реляційна модель детально

### 📋 **Приклад структури:**

```sql
-- Таблиця студентів
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    group_name VARCHAR(10),
    email VARCHAR(100) UNIQUE
);

-- Таблиця курсів
CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    course_name VARCHAR(100) NOT NULL,
    credits INT CHECK (credits > 0),
    teacher_id INT
);

-- Зв'язок багато-до-багатьох
CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    grade CHAR(2),
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

## NoSQL різновиди

### 1. 📄 **Document-орієнтовані (MongoDB)**

```json
{
  "student_id": "S001",
  "name": {
    "first": "Іван",
    "last": "Петров"
  },
  "group": "КН-21",
  "courses": [
    {
      "course_id": "C001",
      "name": "Бази даних",
      "grade": "A"
    }
  ]
}
```

### 2. 🗝️ **Key-Value (Redis)**

```python
# Простий ключ-значення
r.set("user:1001:name", "Іван Петров")

# Структуровані дані
r.hset("user:1001", {
    "name": "Іван Петров",
    "group": "КН-21",
    "status": "active"
})
```

## Векторні бази даних 🧭

### 🤖 **Новий клас через поширення LLM/RAG:**

- Зберігають **embedding-вектори** (числове відображення змісту)
- Пошук за **семантичною подібністю**, а не точним значенням
- Спеціалізовані: Pinecone, Qdrant, Weaviate, Milvus
- Розширення наявних СУБД: **pgvector** для PostgreSQL

```sql
-- pgvector: пошук найближчих за змістом документів
SELECT id, content
FROM documents
ORDER BY embedding <-> '[0.01, -0.03, ...]'
LIMIT 3;
```

## За архітектурою

### 1. 🏢 **Централізовані СУБД**

```mermaid
graph TB
    A[💻 КЛІЄНТ 1] --> D[🖥️ ЦЕНТРАЛЬНИЙ СЕРВЕР]
    B[💻 КЛІЄНТ 2] --> D
    C[💻 КЛІЄНТ 3] --> D

    D --> E[(📊 БАЗА ДАНИХ)]
    D --> F[СУБД]
```

**✅ Переваги:** Простота, централізоване управління
**❌ Недоліки:** Єдина точка відмови, обмежена масштабованість

### 2. 🌐 **Розподілені СУБД**

**Горизонтальне розбиття (Sharding):**
```
Вузол 1: студенти КН-21, КН-22
Вузол 2: студенти КН-23, КН-24
Вузол 3: студенти КН-25, КН-26

-- Локальний запит
SELECT * FROM students WHERE group_name = 'КН-21'; -- Тільки Вузол 1

-- Розподілений запит
SELECT COUNT(*) FROM students; -- Всі вузли → агрегація
```

## Паралельні архітектури

### 🔄 **Три моделі:**

**1. Shared-Nothing:**
```mermaid
graph LR
    A[CPU 1<br/>RAM 1<br/>DISK 1] -.-> D[МЕРЕЖА]
    B[CPU 2<br/>RAM 2<br/>DISK 2] -.-> D
    C[CPU N<br/>RAM N<br/>DISK N] -.-> D
```
**Переваги:** Лінійна масштабованість

**2. Shared-Disk:**
```mermaid
graph TB
    A[CPU 1 + RAM 1] --> D[СПІЛЬНІ ДИСКИ]
    B[CPU 2 + RAM 2] --> D
    C[CPU N + RAM N] --> D
```
**Переваги:** Простота розширення

**3. Shared-Memory:**
```mermaid
graph TB
    A[CPU 1] --> D[СПІЛЬНА ПАМ'ЯТЬ]
    B[CPU 2] --> D
    C[CPU N] --> D
    D --> E[ДИСКИ]
```
**Переваги:** Швидка комунікація
## Архітектура сучасної СУБД зсередини

### 🔍 **Від «що обрати» до «як влаштовано»**

- Класифікація відповіла на питання, **яку** СУБД обрати
- Далі — **як** обрана СУБД організована зсередини: рівні представлення, незалежність даних, компоненти, каталог

## **🎯 Основні поняття:**

**Трирівнева архітектура ANSI/SPARC** — концептуальна модель організації СУБД у вигляді трьох логічно незалежних рівнів: зовнішнього, концептуального та внутрішнього.

**Логічна незалежність даних** — можливість модифікувати концептуальну схему без зміни зовнішніх представлень та програм користувачів.

**Фізична незалежність даних** — можливість змінювати внутрішню схему без впливу на концептуальну схему та програми користувачів.

**Каталог системи** — сукупність системних таблиць з метаданими про структуру бази даних.

## **5. Трирівнева архітектура ANSI/SPARC**

## Історичний контекст (1975)

### 🏛️ **Передумови створення:**

- 🔗 **Тісний зв'язок** програм з фізичною структурою
- 🔄 **Складність модифікації** без впливу на програми
- 📏 **Відсутність стандартів** організації СУБД
- 🔒 **Проблеми безпеки** на різних рівнях

### Рішення: Трирівнева модель

```mermaid
graph TB
    subgraph "🔍 ЗОВНІШНІЙ РІВЕНЬ"
        A[👨‍💼 HR View]
        B[👨‍💻 Dev View]
        C[📊 Manager View]
        D[🔧 Admin View]
    end

    subgraph "🧠 КОНЦЕПТУАЛЬНИЙ РІВЕНЬ"
        E[📋 Глобальна логічна схема<br/>Повний опис БД]
    end

    subgraph "💾 ВНУТРІШНІЙ РІВЕНЬ"
        F[🗂️ Фізична схема]
        G[📇 Індекси та методи доступу]
        H[💽 Розміщення на носіях]
    end

    A --> E
    B --> E
    C --> E
    D --> E
    E --> F
    E --> G
    E --> H
```

## Зовнішній рівень (External Level)

### 👥 **Рівень індивідуальних користувачів**

**Основні характеристики:**

- 🎯 **Персоналізація** — кожен бачить тільки потрібні дані
- 🔒 **Безпека** — обмеження доступу до конфіденційної інформації
- 🎭 **Спрощення** — приховування складності повної структури
- 🧮 **Обчислення** — агреговані та похідні поля

### Приклад представлення для HR

```sql
-- Представлення для відділу кадрів
CREATE VIEW hr_employee_info AS
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    e.position,
    e.hire_date,
    d.department_name,
    e.phone,
    e.email
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.status = 'ACTIVE';
-- Зарплата прихована від HR!
```

## Типи зовнішніх представлень

### 📊 **1. Селективні представлення**

```sql
-- Тільки активні ІТ співробітники
CREATE VIEW it_active_employees AS
SELECT employee_id, first_name, last_name, position
FROM employees
WHERE department_id = 1 AND status = 'ACTIVE';
```

### 🔗 **2. З'єднувальні представлення**

```sql
-- Інформація про проєкти з командою
CREATE VIEW project_team_details AS
SELECT
    p.project_name,
    CONCAT(e.first_name, ' ', e.last_name) AS team_member,
    pa.role_in_project,
    pa.allocation_percentage
FROM projects p
JOIN project_assignments pa ON p.project_id = pa.project_id
JOIN employees e ON pa.employee_id = e.employee_id;
```

### 📈 **3. Агрегатні представлення**

```sql
-- Статистика по відділах
CREATE VIEW department_statistics AS
SELECT
    d.department_name,
    COUNT(e.employee_id) AS employee_count,
    AVG(DATEDIFF(CURRENT_DATE, e.hire_date) / 365.25) AS avg_years,
    COUNT(CASE WHEN e.status = 'ACTIVE' THEN 1 END) AS active_count
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name;
```

## Концептуальний рівень (Conceptual Level)

### 🌐 **Рівень спільноти користувачів**

**Ключові особливості:**

- 📚 **Повнота** — опис усіх сутностей, атрибутів, зв'язків
- 🔗 **Зв'язки** — визначення всіх типів відношень між сутностями
- ⚖️ **Бізнес-правила** — всі обмеження цілісності
- 📊 **Метадані** — інформація про структуру БД

### Приклад концептуальної схеми

```sql
-- Повна структура з усіма обмеженнями
CREATE TABLE employees (
    employee_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    birth_date DATE NOT NULL,
    hire_date DATE DEFAULT (CURRENT_DATE),
    position_id INT NOT NULL,
    department_id INT NOT NULL,
    salary DECIMAL(10,2) CHECK (salary > 0),
    email VARCHAR(100) UNIQUE,

    FOREIGN KEY (position_id) REFERENCES positions(position_id),
    FOREIGN KEY (department_id) REFERENCES departments(department_id),
    CONSTRAINT chk_age CHECK (birth_date <= DATE_SUB(CURRENT_DATE, INTERVAL 16 YEAR))
);
```

## Бізнес-правила на концептуальному рівні

### ⚙️ **Складні тригери та обмеження**

```sql
-- Тригер перевірки навантаження співробітника
DELIMITER //
CREATE TRIGGER trg_check_workload
BEFORE INSERT ON project_assignments
FOR EACH ROW
BEGIN
    DECLARE total_allocation DECIMAL(5,2);

    SELECT COALESCE(SUM(allocation_percentage), 0) INTO total_allocation
    FROM project_assignments
    WHERE employee_id = NEW.employee_id AND status = 'ACTIVE';

    IF (total_allocation + NEW.allocation_percentage) > 100 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Навантаження не може перевищувати 100%';
    END IF;
END //
DELIMITER ;
```

## Внутрішній рівень (Internal Level)

### 💾 **Рівень фізичного зберігання**

**Основні компоненти:**

- 🗂️ **Фізичні структури** — способи зберігання на дисках
- 📇 **Індекси** — методи швидкого доступу до даних
- 💾 **Буферизація** — управління оперативною пам'яттю
- 🗄️ **Партиціювання** — розбиття великих таблиць

### Приклади фізичних оптимізацій

```sql
-- Партиціювання по роках
CREATE TABLE project_logs (
    log_id BIGINT AUTO_INCREMENT,
    project_id INT NOT NULL,
    log_date DATE NOT NULL,
    log_message TEXT,
    PRIMARY KEY (log_id, log_date)
)
PARTITION BY RANGE (YEAR(log_date)) (
    PARTITION p_2022 VALUES LESS THAN (2023),
    PARTITION p_2023 VALUES LESS THAN (2024),
    PARTITION p_2024 VALUES LESS THAN (2025)
);
```

## Стратегії індексування

### 📇 **Типи індексів для різних потреб**

```sql
-- B-Tree для загального використання
CREATE INDEX idx_employee_department ON employees(department_id);

-- Композитний для складних запитів
CREATE INDEX idx_dept_status_hire ON employees(department_id, status, hire_date);

-- Часткові для специфічних умов
CREATE INDEX idx_active_employees ON employees(hire_date)
WHERE status = 'ACTIVE';

-- Повнотекстові для пошуку
CREATE FULLTEXT INDEX idx_employee_search
ON employees(first_name, last_name);
```

## **6. Незалежність даних**

## Логічна незалежність

### 🔄 **Зміни концептуальної схеми без впливу на програми**

**✅ Що можна робити безпечно:**

- ➕ **Додавання нових таблиць**
- ➕ **Додавання нових стовпців**
- ➕ **Створення нових зв'язків**
- ➕ **Додавання індексів**

```sql
-- Додавання нового стовпця
ALTER TABLE employees
ADD COLUMN middle_name VARCHAR(50),
ADD COLUMN phone VARCHAR(20);

-- Існуючі представлення продовжують працювати!
SELECT * FROM hr_employee_info; -- ✅ Працює
```

### Приклад додавання функціональності

```sql
-- Додавання таблиці навичок
CREATE TABLE employee_skills (
    skill_id INT PRIMARY KEY AUTO_INCREMENT,
    employee_id INT,
    skill_name VARCHAR(100),
    skill_level ENUM('BEGINNER', 'INTERMEDIATE', 'ADVANCED'),
    FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
);

-- Старі програми працюють без змін
-- Нові програми можуть використовувати розширену функціональність
```

## Обмеження логічної незалежності

### ⚠️ **Що може зламати існуючі програми:**

```sql
-- ❌ НЕБЕЗПЕЧНО: Видалення стовпців
-- ALTER TABLE employees DROP COLUMN department_id;

-- ❌ НЕБЕЗПЕЧНО: Зміна типів даних
-- ALTER TABLE employees MODIFY COLUMN salary VARCHAR(20);

-- ✅ БЕЗПЕЧНО: Поетапна міграція
ALTER TABLE employees ADD COLUMN salary_new DECIMAL(10,2);
-- 1. Міграція даних
-- 2. Оновлення представлень
-- 3. Видалення старого стовпця
-- 4. Перейменування нового
```

## Фізична незалежність

### 💽 **Зміни зберігання без впливу на логіку**

**Приклади фізичних змін:**

```sql
-- Перенесення на SSD диск
CREATE TABLESPACE ssd_space LOCATION '/ssd/mysql/data';
ALTER TABLE employees TABLESPACE ssd_space;

-- Додавання індексів для швидкодії
CREATE INDEX idx_employee_hire_date ON employees(hire_date);

-- Зміна механізму зберігання
ALTER TABLE employees ENGINE=InnoDB;

-- Програми користувачів працюють без змін! ✅
SELECT * FROM employees WHERE department_id = 5;
```

## Переваги фізичної незалежності

### 🚀 **Оптимізація без простоїв**

```sql
-- Моніторинг використання індексів
SELECT
    TABLE_NAME,
    INDEX_NAME,
    CARDINALITY,
    INDEX_TYPE
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'company_db'
ORDER BY CARDINALITY DESC;

-- Оптимізація на основі статистики
DROP INDEX idx_rarely_used ON employees;
CREATE INDEX idx_frequently_used ON employees(department_id, status);
```

**Результат:** Програми працюють швидше без жодних змін у коді! 🎯

## **7. Компонентна архітектура СУБД**

## Загальна структура

```mermaid
graph TB
    subgraph "👥 РІВЕНЬ КОРИСТУВАЧІВ"
        A[🧑‍💼 Кінцеві користувачі]
        B[👨‍💻 Програмісти]
        C[👨‍💼 Адміністратори]
    end

    subgraph "🖥️ РІВЕНЬ ІНТЕРФЕЙСІВ"
        D[📱 Застосунки]
        E[🔧 СУБД утиліти]
        F[📊 Звітні системи]
    end

    subgraph "🔍 РІВЕНЬ ОБРОБКИ ЗАПИТІВ"
        G[📝 DDL Компілятор]
        H[🔍 DML Процесор]
        I[⚡ Оптимізатор запитів]
        J[🚀 Виконавчий рушій]
    end

    subgraph "🔄 РІВЕНЬ УПРАВЛІННЯ ДАНИМИ"
        K[🔄 Менеджер транзакцій]
        L[🔒 Менеджер блокувань]
        M[💾 Менеджер буферів]
    end

    subgraph "💽 ФІЗИЧНИЙ РІВЕНЬ"
        N[🗂️ Менеджер файлів]
        O[💾 Система зберігання]
        P[📋 Каталог системи]
    end

    A --> D
    B --> D
    C --> E
    D --> G
    D --> H
    G --> I
    H --> I
    I --> J
    J --> K
    K --> L
    K --> M
    L --> N
    M --> N
    N --> O
    N --> P
```

## DDL/DML Компілятори

### 📝 **DDL Компілятор - Визначення структури**

```sql
-- Обробка DDL команди
CREATE TABLE employees (
    employee_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    salary DECIMAL(10,2) CHECK (salary > 0)
);

-- DDL компілятор виконує:
-- 1. ✅ Перевірку синтаксису
-- 2. 🔍 Валідацію типів даних
-- 3. ⚖️ Перевірку обмежень
-- 4. 🔗 Перевірку посилань
-- 5. 📋 Генерацію метаданих
```

### 🔍 **DML Процесор - Маніпулювання даними**

```sql
-- Складний запит для обробки
SELECT
    d.department_name,
    COUNT(e.employee_id) as employee_count,
    AVG(e.salary) as avg_salary
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
WHERE d.budget > 100000
GROUP BY d.department_id, d.department_name
HAVING COUNT(e.employee_id) > 5
ORDER BY avg_salary DESC;
```

**DML процесор створює дерево запиту:**

```
ORDER BY(avg_salary DESC)
  └── HAVING(COUNT > 5)
      └── GROUP BY(dept_id, dept_name)
          └── AGGREGATE(COUNT, AVG)
              └── SELECTION(budget > 100000)
                  └── JOIN(LEFT JOIN)
                      ├── SCAN(departments)
                      └── SCAN(employees)
```

## Оптимізатор запитів

### ⚡ **Вибір найкращого плану виконання**

```sql
-- Запит з багатьма можливими планами
SELECT e.first_name, d.department_name, p.project_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN project_assignments pa ON e.employee_id = pa.employee_id
JOIN projects p ON pa.project_id = p.project_id
WHERE d.location = 'Київ' AND p.status = 'ACTIVE';
```

### Типи оптимізації

**1. 📊 Статистична оптимізація**

```sql
-- Збір статистики для прийняття рішень
ANALYZE TABLE employees UPDATE HISTOGRAM ON department_id, salary;

-- Оптимізатор знає:
-- - В departments 5 записів з 'Київ' (10% таблиці)
-- - В projects 50 'ACTIVE' проєктів (25% таблиці)
-- - В employees 1000 записів загалом
```

**2. 🧮 Алгебраїчна оптимізація**

```sql
-- Проштовхування селекції (selection pushdown)
-- Замість: JOIN всіх таблиць → потім WHERE
-- Краще: WHERE спочатку → потім JOIN менших результатів

-- План 1 (неоптимальний):
-- JOIN(employees, departments, projects) → WHERE(location='Київ')

-- План 2 (оптимальний):
-- WHERE departments(location='Київ') → JOIN → WHERE projects(status='ACTIVE') → JOIN
```

**3. 🔗 Вибір алгоритму JOIN**

| Алгоритм | Найкраще для | Складність |
|----------|-------------|-------------|
| 🔄 **Nested Loop** | Малі таблиці | O(n×m) |
| # **Hash Join** | Середні таблиці | O(n+m) |
| 📊 **Sort-Merge** | Великі сортовані | O(n log n + m log m) |

## Менеджер буферів

### 💾 **Управління оперативною пам'яттю**

```sql
-- Налаштування буферного пулу
SET GLOBAL innodb_buffer_pool_size = 8589934592; -- 8GB
SET GLOBAL innodb_buffer_pool_instances = 8;     -- 8 екземплярів

-- Моніторинг ефективності
SELECT
    POOL_SIZE,
    FREE_BUFFERS,
    DATABASE_PAGES,
    MODIFIED_DATABASE_PAGES
FROM INFORMATION_SCHEMA.INNODB_BUFFER_POOL_STATS;
```

### Алгоритми заміщення сторінок

**🔄 LRU з модифікаціями (InnoDB):**

```mermaid
graph LR
    A[New Page] --> B{Hot Access?}
    B -->|Yes| C[Young Sublist<br/>63%]
    B -->|No| D[Old Sublist<br/>37%]

    C --> E[Recently Used]
    D --> F{Access Again<br/>Within 1s?}
    F -->|Yes| C
    F -->|No| G[Evicted]
```

**Переваги:**
- 🔥 **Гарячі дані** залишаються в пам'яті
- 🧹 **Сканування** не витискає корисні дані
- ⚡ **Адаптивність** до патернів доступу

## Менеджер транзакцій

### 🔄 **Забезпечення властивостей ACID**

```sql
-- Приклад атомарної транзакції
START TRANSACTION;

-- Зменшення балансу відправника
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 'ACC001';

-- Перевірка достатності коштів
IF (SELECT balance FROM accounts WHERE account_id = 'ACC001') < 0 THEN
    ROLLBACK; -- ❌ Скасування всієї транзакції
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Недостатньо коштів';
END IF;

-- Збільшення балансу отримувача
UPDATE accounts SET balance = balance + 1000 WHERE account_id = 'ACC002';

COMMIT; -- ✅ Підтвердження всіх змін
```

### Рівні ізоляції транзакцій

| Рівень | Брудне читання | Неповторювані читання | Фантомні записи |
|--------|----------------|----------------------|-----------------|
| 📖 **READ UNCOMMITTED** | ❌ Можливо | ❌ Можливо | ❌ Можливо |
| 📗 **READ COMMITTED** | ✅ Неможливо | ❌ Можливо | ❌ Можливо |
| 📘 **REPEATABLE READ** | ✅ Неможливо | ✅ Неможливо | ❌ Можливо |
| 📕 **SERIALIZABLE** | ✅ Неможливо | ✅ Неможливо | ✅ Неможливо |

### Приклад проблеми Phantom Reads

```sql
-- Транзакція A:
START TRANSACTION;
SELECT COUNT(*) FROM employees WHERE salary > 50000; -- Результат: 150
-- ... інші операції ...
SELECT COUNT(*) FROM employees WHERE salary > 50000; -- Результат: 151 👻
COMMIT;

-- Транзакція B (паралельно):
START TRANSACTION;
INSERT INTO employees (first_name, last_name, salary, department_id)
VALUES ('Новий', 'Співробітник', 55000, 1); -- 👻 Phantom!
COMMIT;
```

## **8. Каталог системи (метабаза)**

## Структура каталогу

### 📋 **Метадані про структуру БД**

```sql
-- Інформація про таблиці
SELECT
    TABLE_NAME,
    ENGINE,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb,
    CREATE_TIME,
    UPDATE_TIME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'company_db'
ORDER BY size_mb DESC;
```

### Опис стовпців

```sql
-- Детальна інформація про структуру таблиці
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    IS_NULLABLE,
    COLUMN_DEFAULT,
    COLUMN_KEY,
    EXTRA,
    COLUMN_COMMENT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'company_db'
  AND TABLE_NAME = 'employees'
ORDER BY ORDINAL_POSITION;
```

## Практичне використання каталогу

### 🔍 **Аналіз розміру БД**

```sql
-- ТОП-5 найбільших таблиць
SELECT
    TABLE_NAME,
    ROUND(((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024), 2) AS 'Size (MB)',
    ROUND((INDEX_LENGTH / DATA_LENGTH), 2) AS 'Index/Data Ratio',
    TABLE_ROWS
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'company_db'
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC
LIMIT 5;
```

### 📊 **Аналіз ефективності індексів**

```sql
-- Індекси з низькою селективністю
SELECT
    TABLE_NAME,
    INDEX_NAME,
    ROUND(CARDINALITY / t.TABLE_ROWS * 100, 2) AS selectivity_percent
FROM INFORMATION_SCHEMA.STATISTICS s
JOIN INFORMATION_SCHEMA.TABLES t USING (TABLE_SCHEMA, TABLE_NAME)
WHERE s.TABLE_SCHEMA = 'company_db'
  AND s.INDEX_NAME != 'PRIMARY'
HAVING selectivity_percent < 10 -- Низька селективність
ORDER BY selectivity_percent ASC;
```

### 🔗 **Аналіз зв'язків між таблицями**

```sql
-- Карта зовнішніх ключів
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME,
    DELETE_RULE,
    UPDATE_RULE
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE CONSTRAINT_SCHEMA = 'company_db'
  AND REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY TABLE_NAME;
```

## Автоматизація через каталог

### 🤖 **Генерація адміністративних команд**

```sql
-- Генерація команд оптимізації
SELECT
    CONCAT('OPTIMIZE TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ';')
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'company_db'
  AND ENGINE = 'InnoDB'
  AND UPDATE_TIME < DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Результат:
-- OPTIMIZE TABLE company_db.old_logs;
-- OPTIMIZE TABLE company_db.archived_data;
```

### 📋 **Генерація документації**

```sql
-- Автоматична документація схеми
SELECT
    CONCAT('## Таблиця: ', TABLE_NAME, '\n',
           '- **Записів:** ', IFNULL(TABLE_ROWS, 0), '\n',
           '- **Розмір:** ', ROUND((DATA_LENGTH + INDEX_LENGTH)/1024/1024, 2), ' MB\n',
           '- **Створено:** ', CREATE_TIME, '\n',
           '- **Оновлено:** ', UPDATE_TIME, '\n'
    ) as documentation
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'company_db'
  AND TABLE_TYPE = 'BASE TABLE';
```
## **9. Сучасні архітектури та тенденції**

## Еволюція архітектурних підходів

### 📈 **Від монолітних до розподілених**

```mermaid
timeline
    title Еволюція архітектур СУБД
    1970s : Монолітні СУБД
          : Oracle, IBM DB2
    1990s : Клієнт-сервер
          : Розподіл навантаження
    2000s : Веб-орієнтовані
          : MySQL, PostgreSQL
    2005s : NoSQL революція
          : MongoDB, Cassandra
    2010s : Cloud-native
          : AWS RDS, Google Cloud
    2015s : Serverless
          : Aurora Serverless
    2020s : HTAP системи
          : TiDB, CockroachDB
```

## Типи сучасних архітектур

### 1. 🏢 **Монолітна архітектура**

**✅ Переваги:**
- 🎯 Простота розгортання
- ⚡ Ефективна взаємодія компонентів
- 🔒 Легкість забезпечення ACID
- 🎛️ Централізований контроль

**❌ Недоліки:**
- ❌ Єдина точка відмови
- 📈 Складність масштабування
- 🔧 Обмежена гнучкість

**Приклад: Oracle Database**

### 2. 🧩 **Мікросервісна архітектура**

```mermaid
graph TB
    A[⚖️ Load Balancer] --> B[🔍 Query Service]
    A --> C[💾 Storage Service]
    A --> D[🔄 Transaction Service]
    A --> E[📊 Analytics Service]

    B --> F[📋 Metadata Store]
    C --> G[🗂️ Distributed Storage]
    D --> H[📝 Transaction Log]
    E --> I[📊 Data Warehouse]
```

**Приклад: Amazon Aurora**

### 3. ☁️ **Serverless архітектура**

```javascript
// AWS Aurora Serverless - автомасштабування
const params = {
    resourceArn: 'aurora-cluster-arn',
    secretArn: 'secret-arn',
    database: 'company_db',
    sql: 'SELECT COUNT(*) FROM employees'
};

// Не потрібно керувати з'єднаннями або серверами!
rdsData.executeStatement(params);
```

**✅ Переваги serverless:**
- 📈 **Автомасштабування** на основі навантаження
- 💰 **Оплата за використання**
- 🔧 **Нульове адміністрування**
- ⚡ **Миттєве масштабування**

**🔎 Інші приклади:** Neon (Postgres, branching бази як git), PlanetScale (MySQL/Vitess, з 2025 — і Postgres)

## Спеціалізовані архітектури

### 📊 **Колонкова архітектура (ClickHouse)**

```sql
-- Оптимізація для аналітики
CREATE TABLE sales_analytics (
    date Date,
    product_id UInt32,
    sales_amount Decimal(15,2),
    quantity UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, product_id);

-- Швидкий аналітичний запит
SELECT
    toMonth(date) as month,
    sum(sales_amount) as total_sales
FROM sales_analytics
WHERE date >= '2024-01-01'
GROUP BY month;
```

### 📄 **Document-орієнтована (MongoDB)**

```javascript
// Гнучка схема для веб-додатків
db.employees.insertOne({
    _id: ObjectId(),
    name: "Іван Петров",
    department: {
        id: 1,
        name: "ІТ Відділ",
        location: "Київ"
    },
    skills: ["JavaScript", "Python", "MongoDB"],
    projects: [
        {
            name: "Вебдодаток",
            role: "Lead Developer",
            status: "active"
        }
    ]
});

// Запит з nested структурами
db.employees.find({
    "department.name": "ІТ Відділ",
    "skills": { $in: ["Python", "JavaScript"] }
});
```

### ⏰ **Time-Series (InfluxDB)**

```sql
-- Оптимізація для IoT та моніторингу
-- Зберігання метрик серверів
server_metrics,host=web01,region=kyiv cpu=75.2,memory=60.5,disk_io=1250

-- Аналіз трендів
SELECT
    MEAN("cpu") AS avg_cpu,
    PERCENTILE("cpu", 95) AS p95_cpu
FROM "server_metrics"
WHERE time >= now() - 1h
GROUP BY time(5m), "host";
```

### 🌐 **Графова архітектура (Neo4j)**

```cypher
// Моделювання складних зв'язків
CREATE (emp:Employee {name: 'Іван Петров'})
CREATE (dept:Department {name: 'ІТ Відділ'})
CREATE (proj:Project {name: 'Новий сайт'})

CREATE (emp)-[:WORKS_IN]->(dept)
CREATE (emp)-[:ASSIGNED_TO {role: 'Lead Developer'}]->(proj)

// Пошук шляхів в організаційній мережі
MATCH path = (emp:Employee)-[:WORKS_IN*..3]-(colleague:Employee)
WHERE emp.name = 'Іван Петров'
RETURN colleague.name, length(path)
```

### 🧭 **Векторна архітектура (pgvector)**

```sql
-- Гібридний пошук: звичайна умова + семантична близькість
SELECT d.id, d.title, d.embedding <-> :query_vector AS distance
FROM documents d
JOIN projects p ON d.project_id = p.project_id
WHERE p.status = 'ACTIVE'
ORDER BY distance
LIMIT 5;
```

- Пошук за **семантичною подібністю**, а не точним значенням
- HNSW-індекси замість B-дерева
- Спеціалізовані (Pinecone, Qdrant) або розширення наявних СУБД (pgvector)

### 🔒 **Реєстрові (ledger) бази даних**

- Незмінність записів + криптографічна верифікація (immudb)
- Ніша значно вужча, ніж очікувалося на піку хайпу блокчейну
- Використання: аудиторський слід, фінансові реєстри, supply chain

### 📋 **Матриця вибору**

| Сценарій | Рекомендована архітектура | СУБД |
|----------|--------------------------|------|
| 💼 **Бізнес-додаток** | Монолітна реляційна | PostgreSQL, MySQL |
| 📊 **Аналітика** | Колонкова | ClickHouse, BigQuery |
| 🌐 **Веб-додаток** | Document-oriented | MongoDB, Firestore |
| 🏪 **E-commerce** | HTAP гібридна | TiDB, CockroachDB |
| 📱 **IoT/Моніторинг** | Time-Series | InfluxDB, TimescaleDB |
| 🌍 **Глобальний сервіс** | Розподілена | Cassandra, DynamoDB |
| 🤖 **RAG/AI-пошук** | Векторна | pgvector, Qdrant, Pinecone |
| 🔑 **Кеш** | In-Memory | Redis, Memcached |
| 🕸️ **Графові зв'язки** | Графова | Neo4j, Amazon Neptune |
## Хмарні СУБД (DBaaS)

### ☁️ **Database as a Service:**

- **AWS:** RDS, Aurora, DynamoDB
- **Google:** Cloud SQL, Firestore, BigQuery
- **Azure:** SQL Database, Cosmos DB
- **MongoDB:** Atlas

### ✅ **Переваги хмарних БД:**

- 📈 **Автоматичне масштабування**
- 🔧 **Керована інфраструктура**
- 🌍 **Глобальна доступність**
- 💰 **Оплата за використання**
- 🛡️ **Вбудована відмовостійкість**


## Тенденції розвитку

### 🚀 **Сучасні напрямки:**

1. **🤖 AI/ML інтеграція**
    - Автоматична оптимізація запитів
    - Прогнозування навантаження
    - Векторний пошук та embedding-и (pgvector, Pinecone)

2. **🔗 Multi-model бази**
    - Azure Cosmos DB
    - ArangoDB
    - OrientDB

3. **📊 Реального часу**
    - Apache Kafka + ksqlDB
    - Apache Pulsar
    - Event sourcing
## **Висновки**

### 🎯 **Ключові ідеї лекції**

1. **📈 Еволюція СУБД** — від файлових систем через ієрархічні/мережеві моделі до реляційної революції Кодда, NoSQL, хмарних і векторних СУБД
2. **🏗️ Трирівнева архітектура ANSI/SPARC** — фундамент сучасних СУБД
    - Зовнішній рівень забезпечує персоналізацію
    - Концептуальний рівень містить бізнес-логіку
    - Внутрішній рівень оптимізує продуктивність
3. **🔄 Незалежність даних** — ключ до гнучкості
    - Логічна незалежність дозволяє еволюцію схеми
    - Фізична незалежність забезпечує оптимізацію без простоїв
4. **🧩 Компонентна архітектура** — розподіл відповідальності між обробником запитів, менеджером транзакцій, менеджером буферів і каталогом системи
5. **🌐 Різноманітність сучасних архітектур** — від монолітних і мікросервісних до serverless, HTAP і векторних рішень для RAG/AI-пошуку

### 📋 **Практичний висновок**

Вибір типу та архітектури СУБД залежить від моделі даних, характеру навантаження (OLTP/OLAP), обсягів даних, вимог до консистентності та бюджету проєкту.
