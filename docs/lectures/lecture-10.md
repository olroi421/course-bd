# Лекція 10. Збережені процедури та активні бази даних

## Вступ

У попередніх лекціях ми розглядали системи управління базами даних переважно як пасивні сховища інформації, де вся логіка обробки даних знаходилася на стороні клієнтських застосунків. Однак сучасні СУБД пропонують значно більше можливостей, дозволяючи розміщувати логіку обробки даних безпосередньо в базі даних через збережені процедури, функції та тригери. Цей підхід створює те, що називається активними базами даних, де база даних може самостійно реагувати на події та виконувати складні операції без втручання клієнтських застосунків.

Серверне програмування в контексті баз даних стало критично важливим у сучасній розробці програмного забезпечення. Воно дозволяє централізувати бізнес-логіку, підвищити продуктивність через зменшення мережевого трафіку, забезпечити кращу безпеку та цілісність даних, а також спростити супровід складних систем. У цій лекції ми детально розглянемо архітектуру серверного програмування, механізми тригерів та представлень, а також проаналізуємо компроміси між розміщенням логіки на сервері та клієнті.

## Архітектура серверного програмування

### Процедурні розширення SQL

Стандарт SQL визначає декларативну мову запитів, яка чудово підходить для операцій з множинами даних, але має обмеження в реалізації складної процедурної логіки. Для вирішення цієї проблеми різні виробники СУБД розробили процедурні розширення SQL, які додають до мови конструкції традиційних мов програмування.

#### Огляд процедурних розширень

Кожна велика СУБД має власне процедурне розширення SQL з унікальним синтаксисом та можливостями.

**PL/pgSQL (PostgreSQL):**

PL/pgSQL є процедурною мовою для PostgreSQL, що тісно інтегрована з SQL та надає структури управління потоком виконання, обробку помилок та можливість створення складних функцій.

```sql
-- Приклад функції на PL/pgSQL
CREATE OR REPLACE FUNCTION calculate_order_total(order_id_param INTEGER)
RETURNS NUMERIC AS $$
DECLARE
    total_amount NUMERIC := 0;
    item_record RECORD;
BEGIN
    -- Обчислення суми замовлення
    FOR item_record IN
        SELECT quantity, unit_price
        FROM order_items
        WHERE order_id = order_id_param
    LOOP
        total_amount := total_amount + (item_record.quantity * item_record.unit_price);
    END LOOP;

    RETURN total_amount;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE NOTICE 'Замовлення % не знайдено', order_id_param;
        RETURN 0;
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Помилка обчислення: %', SQLERRM;
END;
$$ LANGUAGE plpgsql;
```

**PL/SQL (Oracle):**

PL/SQL від Oracle є одним із найстаріших та найпотужніших процедурних розширень SQL, що підтримує об'єктно-орієнтоване програмування, колекції та розширені можливості обробки помилок.

```sql
-- Приклад пакету на PL/SQL
CREATE OR REPLACE PACKAGE order_management AS
    -- Оголошення типів
    TYPE order_summary_type IS RECORD (
        order_id NUMBER,
        customer_name VARCHAR2(100),
        total_amount NUMBER
    );

    -- Оголошення процедур
    PROCEDURE process_order(p_order_id IN NUMBER);
    FUNCTION get_order_status(p_order_id IN NUMBER) RETURN VARCHAR2;
END order_management;
/

CREATE OR REPLACE PACKAGE BODY order_management AS
    PROCEDURE process_order(p_order_id IN NUMBER) IS
        v_status VARCHAR2(20);
    BEGIN
        SELECT status INTO v_status
        FROM orders
        WHERE order_id = p_order_id;

        IF v_status = 'PENDING' THEN
            UPDATE orders
            SET status = 'PROCESSING',
                processing_date = SYSDATE
            WHERE order_id = p_order_id;

            COMMIT;
        END IF;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE_APPLICATION_ERROR(-20001, 'Замовлення не знайдено');
    END process_order;

    FUNCTION get_order_status(p_order_id IN NUMBER) RETURN VARCHAR2 IS
        v_status VARCHAR2(20);
    BEGIN
        SELECT status INTO v_status
        FROM orders
        WHERE order_id = p_order_id;

        RETURN v_status;
    END get_order_status;
END order_management;
/
```

**T-SQL (Microsoft SQL Server):**

Transact-SQL від Microsoft пропонує процедурні конструкції з акцентом на продуктивність та інтеграцію з екосистемою Microsoft.

```sql
-- Приклад збереженої процедури на T-SQL
CREATE OR ALTER PROCEDURE UpdateInventory
    @ProductID INT,
    @QuantityChange INT,
    @OperationType CHAR(1) -- 'A' для додавання, 'S' для віднімання
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        DECLARE @CurrentStock INT;
        DECLARE @NewStock INT;

        -- Отримання поточного залишку з блокуванням
        SELECT @CurrentStock = stock_quantity
        FROM inventory WITH (UPDLOCK)
        WHERE product_id = @ProductID;

        IF @CurrentStock IS NULL
        BEGIN
            RAISERROR('Товар не знайдено', 16, 1);
            RETURN;
        END

        -- Обчислення нового залишку
        IF @OperationType = 'A'
            SET @NewStock = @CurrentStock + @QuantityChange;
        ELSE IF @OperationType = 'S'
        BEGIN
            IF @CurrentStock < @QuantityChange
            BEGIN
                RAISERROR('Недостатньо товару на складі', 16, 1);
                RETURN;
            END
            SET @NewStock = @CurrentStock - @QuantityChange;
        END
        ELSE
        BEGIN
            RAISERROR('Невірний тип операції', 16, 1);
            RETURN;
        END

        -- Оновлення залишку
        UPDATE inventory
        SET stock_quantity = @NewStock,
            last_updated = GETDATE()
        WHERE product_id = @ProductID;

        COMMIT TRANSACTION;

        PRINT 'Залишок успішно оновлено';
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();

        RAISERROR(@ErrorMessage, @ErrorSeverity, 1);
    END CATCH
END;
```

#### Основні елементи процедурних розширень

Незалежно від конкретної реалізації, процедурні розширення SQL зазвичай підтримують наступні конструкції.

**Змінні та типи даних:**

Процедурні мови дозволяють оголошувати змінні для зберігання проміжних результатів та стану виконання.

```sql
-- PostgreSQL
DECLARE
    total_sales NUMERIC(10,2);
    customer_name VARCHAR(100);
    order_date DATE := CURRENT_DATE;
    is_premium BOOLEAN := FALSE;

-- Oracle
DECLARE
    v_employee_count NUMBER;
    v_department_name departments.name%TYPE; -- тип з таблиці
    v_salary_record employees%ROWTYPE; -- тип запису

-- SQL Server
DECLARE
    @TotalRevenue DECIMAL(18,2),
    @CustomerCount INT = 0,
    @ProcessDate DATETIME = GETDATE();
```

**Структури управління потоком:**

Умовні оператори та цикли дозволяють реалізувати складну логіку прийняття рішень.

```sql
-- Умовні конструкції (PostgreSQL)
IF total_amount > 1000 THEN
    discount_rate := 0.10;
ELSIF total_amount > 500 THEN
    discount_rate := 0.05;
ELSE
    discount_rate := 0;
END IF;

-- Цикли (PostgreSQL)
FOR counter IN 1..10 LOOP
    INSERT INTO logs (message) VALUES ('Ітерація ' || counter);
END LOOP;

WHILE stock_level < minimum_level LOOP
    CALL reorder_product(product_id);
    SELECT quantity INTO stock_level
    FROM inventory
    WHERE product_id = current_product;
END LOOP;

-- Курсори для обробки множин
DECLARE
    order_cursor CURSOR FOR
        SELECT order_id, customer_id, total
        FROM orders
        WHERE status = 'PENDING';
    order_rec RECORD;
BEGIN
    OPEN order_cursor;
    LOOP
        FETCH order_cursor INTO order_rec;
        EXIT WHEN NOT FOUND;

        -- Обробка кожного замовлення
        PERFORM process_order(order_rec.order_id);
    END LOOP;
    CLOSE order_cursor;
END;
```

**Обробка винятків:**

Механізми обробки помилок дозволяють коректно реагувати на виняткові ситуації.

```sql
-- PostgreSQL
BEGIN
    -- Критична операція
    UPDATE accounts SET balance = balance - amount WHERE account_id = source_account;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Рахунок % не знайдено', source_account;
    END IF;
EXCEPTION
    WHEN insufficient_funds THEN
        RAISE NOTICE 'Недостатньо коштів на рахунку';
        ROLLBACK;
    WHEN deadlock_detected THEN
        RAISE WARNING 'Виявлено взаємне блокування, повторна спроба';
        -- Логіка повторної спроби
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Непередбачена помилка: %', SQLERRM;
END;
```

### Збережені процедури та функції

Збережені процедури та функції є іменованими блоками коду, що зберігаються в базі даних та можуть викликатися повторно.

#### Визначення та відмінності

Збережена процедура є підпрограмою, що виконує певні дії, але не обов'язково повертає значення. Процедури можуть змінювати стан бази даних, виконувати транзакції та повертати результати через вихідні параметри або набори даних.

Функція завжди повертає значення і призначена для використання у виразах SQL. Функції зазвичай не повинні змінювати стан бази даних, хоча деякі СУБД дозволяють це робити.

```sql
-- Приклад функції (PostgreSQL)
CREATE OR REPLACE FUNCTION get_customer_discount(customer_id_param INTEGER)
RETURNS NUMERIC AS $$
DECLARE
    total_purchases NUMERIC;
    discount_rate NUMERIC;
BEGIN
    -- Обчислення загальної суми покупок клієнта
    SELECT COALESCE(SUM(total_amount), 0)
    INTO total_purchases
    FROM orders
    WHERE customer_id = customer_id_param
        AND order_date >= CURRENT_DATE - INTERVAL '1 year';

    -- Визначення розміру знижки на основі покупок
    CASE
        WHEN total_purchases > 10000 THEN discount_rate := 0.15;
        WHEN total_purchases > 5000 THEN discount_rate := 0.10;
        WHEN total_purchases > 1000 THEN discount_rate := 0.05;
        ELSE discount_rate := 0;
    END CASE;

    RETURN discount_rate;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Використання функції в запиті
SELECT
    customer_id,
    customer_name,
    get_customer_discount(customer_id) as discount_rate,
    total_amount,
    total_amount * (1 - get_customer_discount(customer_id)) as final_amount
FROM orders o
JOIN customers c USING (customer_id)
WHERE order_date = CURRENT_DATE;
```

```sql
-- Приклад процедури (PostgreSQL)
CREATE OR REPLACE PROCEDURE process_monthly_billing()
LANGUAGE plpgsql
AS $$
DECLARE
    customer_rec RECORD;
    invoice_id INTEGER;
BEGIN
    -- Обробка кожного активного клієнта
    FOR customer_rec IN
        SELECT customer_id, customer_name, email
        FROM customers
        WHERE status = 'ACTIVE'
    LOOP
        -- Створення рахунку
        INSERT INTO invoices (customer_id, invoice_date, status)
        VALUES (customer_rec.customer_id, CURRENT_DATE, 'PENDING')
        RETURNING invoice_id INTO invoice_id;

        -- Додавання позицій до рахунку з підписок
        INSERT INTO invoice_items (invoice_id, description, amount)
        SELECT
            invoice_id,
            subscription_name,
            monthly_fee
        FROM subscriptions
        WHERE customer_id = customer_rec.customer_id
            AND status = 'ACTIVE';

        -- Відправка повідомлення
        PERFORM send_email(
            customer_rec.email,
            'Новий рахунок',
            'Створено рахунок #' || invoice_id
        );

        RAISE NOTICE 'Оброблено клієнта: %', customer_rec.customer_name;
    END LOOP;

    COMMIT;
END;
$$;
```

#### Параметри процедур та функцій

Процедури та функції можуть приймати параметри різних типів та режимів.

```sql
-- PostgreSQL: різні типи параметрів
CREATE OR REPLACE FUNCTION calculate_shipping_cost(
    -- Вхідні параметри
    weight_kg NUMERIC,
    distance_km NUMERIC,
    is_express BOOLEAN DEFAULT FALSE,
    -- Вихідні параметри
    OUT base_cost NUMERIC,
    OUT express_fee NUMERIC,
    OUT total_cost NUMERIC
) AS $$
BEGIN
    -- Базова вартість залежить від ваги та відстані
    base_cost := (weight_kg * 0.5) + (distance_km * 0.1);

    -- Додаткова плата за експрес-доставку
    IF is_express THEN
        express_fee := base_cost * 0.5;
    ELSE
        express_fee := 0;
    END IF;

    total_cost := base_cost + express_fee;
END;
$$ LANGUAGE plpgsql;

-- Виклик функції з вихідними параметрами
SELECT * FROM calculate_shipping_cost(15.5, 250, TRUE);
```

```sql
-- SQL Server: процедура з різними типами параметрів
CREATE PROCEDURE TransferFunds
    @SourceAccount INT,
    @TargetAccount INT,
    @Amount DECIMAL(18,2),
    @TransactionID INT OUTPUT, -- вихідний параметр
    @ErrorMessage NVARCHAR(500) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @ErrorMessage = NULL;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Перевірка балансу
        DECLARE @SourceBalance DECIMAL(18,2);
        SELECT @SourceBalance = balance
        FROM accounts WITH (UPDLOCK)
        WHERE account_id = @SourceAccount;

        IF @SourceBalance < @Amount
        BEGIN
            SET @ErrorMessage = 'Недостатньо коштів';
            ROLLBACK TRANSACTION;
            RETURN 1;
        END

        -- Виконання переказу
        UPDATE accounts
        SET balance = balance - @Amount
        WHERE account_id = @SourceAccount;

        UPDATE accounts
        SET balance = balance + @Amount
        WHERE account_id = @TargetAccount;

        -- Створення запису транзакції
        INSERT INTO transactions (source_account, target_account, amount, transaction_date)
        VALUES (@SourceAccount, @TargetAccount, @Amount, GETDATE());

        SET @TransactionID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
        RETURN 0; -- успіх
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        SET @ErrorMessage = ERROR_MESSAGE();
        RETURN 2; -- помилка
    END CATCH
END;

-- Виклик процедури
DECLARE @TxnID INT, @ErrMsg NVARCHAR(500), @Result INT;

EXEC @Result = TransferFunds
    @SourceAccount = 1001,
    @TargetAccount = 1002,
    @Amount = 500.00,
    @TransactionID = @TxnID OUTPUT,
    @ErrorMessage = @ErrMsg OUTPUT;

IF @Result = 0
    PRINT 'Переказ успішний, ID транзакції: ' + CAST(@TxnID AS VARCHAR);
ELSE
    PRINT 'Помилка: ' + @ErrMsg;
```

## Тригери: семантика, класифікація, області застосування

### Концепція тригерів

Тригер є спеціальним типом збереженої процедури, яка автоматично виконується у відповідь на певні події в базі даних, такі як вставка, оновлення або видалення даних у таблиці. Тригери реалізують концепцію активних баз даних, де база даних може самостійно реагувати на зміни без явних команд від застосунку.

#### Базова структура тригера

```sql
-- PostgreSQL: синтаксис створення тригера
CREATE TRIGGER trigger_name
    {BEFORE | AFTER | INSTEAD OF} {INSERT | UPDATE | DELETE}
    ON table_name
    [FOR EACH {ROW | STATEMENT}]
    [WHEN (condition)]
    EXECUTE FUNCTION function_name();
```

Тригер складається з декількох ключових елементів:

Час виконання визначає, коли тригер спрацьовує відносно події, що його викликала. Тригери BEFORE виконуються до модифікації даних і можуть змінювати значення або навіть скасувати операцію. Тригери AFTER виконуються після успішного завершення операції і зазвичай використовуються для каскадних змін або логування. Тригери INSTEAD OF замінюють стандартну операцію власною логікою і часто використовуються для оновлюваних представлень.

Подія визначає операцію, що викликає тригер. Це може бути INSERT для вставки нових записів, UPDATE для зміни існуючих або DELETE для видалення. Один тригер може реагувати на кілька подій одночасно.

Рівень гранулярності визначає, чи виконується тригер один раз для кожного модифікованого рядка, чи один раз для всієї операції незалежно від кількості змінених рядків.

### Класифікація тригерів

#### За часом виконання

**BEFORE тригери:**

Тригери BEFORE виконуються перед виконанням операції та мають доступ до нових значень через спеціальні змінні. Вони можуть модифікувати дані перед їх збереженням або скасувати операцію.

```sql
-- PostgreSQL: BEFORE тригер для валідації та модифікації даних
CREATE OR REPLACE FUNCTION validate_employee_salary()
RETURNS TRIGGER AS $$
BEGIN
    -- Валідація: зарплата не може бути від'ємною
    IF NEW.salary < 0 THEN
        RAISE EXCEPTION 'Зарплата не може бути від''ємною';
    END IF;

    -- Автоматична нормалізація: округлення до копійок
    NEW.salary := ROUND(NEW.salary, 2);

    -- Автоматичне встановлення дати модифікації
    NEW.last_modified := CURRENT_TIMESTAMP;

    -- Валідація: зарплата не може перевищувати максимум для посади
    IF NEW.salary > (SELECT max_salary
                     FROM positions
                     WHERE position_id = NEW.position_id) THEN
        RAISE EXCEPTION 'Зарплата перевищує максимум для посади';
    END IF;

    -- Повернення модифікованого запису
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_employee_salary
    BEFORE INSERT OR UPDATE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION validate_employee_salary();
```

**AFTER тригери:**

Тригери AFTER виконуються після успішного завершення операції та використовуються для каскадних змін, аудиту та підтримки денормалізованих даних.

```sql
-- PostgreSQL: AFTER тригер для аудиту
CREATE OR REPLACE FUNCTION audit_salary_changes()
RETURNS TRIGGER AS $$
BEGIN
    -- Логування змін зарплати
    INSERT INTO salary_audit (
        employee_id,
        old_salary,
        new_salary,
        changed_by,
        changed_at,
        operation_type
    ) VALUES (
        COALESCE(NEW.employee_id, OLD.employee_id),
        OLD.salary,
        NEW.salary,
        current_user,
        CURRENT_TIMESTAMP,
        TG_OP
    );

    -- Для UPDATE перевіряємо значну зміну
    IF TG_OP = 'UPDATE' AND
       ABS(NEW.salary - OLD.salary) > OLD.salary * 0.10 THEN
        -- Повідомлення для значних змін (>10%)
        INSERT INTO notifications (message, created_at)
        VALUES (
            format('Значна зміна зарплати для співробітника %s: %s -> %s',
                   NEW.employee_id, OLD.salary, NEW.salary),
            CURRENT_TIMESTAMP
        );
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER log_salary_changes
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION audit_salary_changes();
```

**INSTEAD OF тригери:**

Ці тригери замінюють стандартну операцію і найчастіше використовуються для створення оновлюваних представлень.

```sql
-- PostgreSQL: INSTEAD OF тригер для оновлюваного представлення
CREATE VIEW employee_details AS
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    e.email,
    d.department_name,
    p.position_name,
    e.salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN positions p ON e.position_id = p.position_id;

CREATE OR REPLACE FUNCTION update_employee_details()
RETURNS TRIGGER AS $$
BEGIN
    -- Оновлення базової таблиці співробітників
    UPDATE employees
    SET
        first_name = NEW.first_name,
        last_name = NEW.last_name,
        email = NEW.email,
        salary = NEW.salary,
        department_id = (
            SELECT department_id
            FROM departments
            WHERE department_name = NEW.department_name
        ),
        position_id = (
            SELECT position_id
            FROM positions
            WHERE position_name = NEW.position_name
        )
    WHERE employee_id = NEW.employee_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_employee_view
    INSTEAD OF UPDATE ON employee_details
    FOR EACH ROW
    EXECUTE FUNCTION update_employee_details();
```

#### За рівнем гранулярності

**Тригери на рівні рядка (ROW-level):**

Виконуються один раз для кожного модифікованого рядка і мають доступ до значень конкретного рядка через змінні NEW та OLD.

```sql
-- Приклад: підтримка історії змін на рівні рядків
CREATE OR REPLACE FUNCTION maintain_product_history()
RETURNS TRIGGER AS $$
BEGIN
    -- Для кожного зміненого рядка створюємо запис історії
    INSERT INTO product_history (
        product_id,
        old_price,
        new_price,
        old_stock,
        new_stock,
        change_date,
        change_type
    ) VALUES (
        NEW.product_id,
        OLD.price,
        NEW.price,
        OLD.stock_quantity,
        NEW.stock_quantity,
        CURRENT_TIMESTAMP,
        TG_OP
    );

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER track_product_changes
    AFTER UPDATE ON products
    FOR EACH ROW
    WHEN (OLD.price IS DISTINCT FROM NEW.price OR
          OLD.stock_quantity IS DISTINCT FROM NEW.stock_quantity)
    EXECUTE FUNCTION maintain_product_history();
```

**Тригери на рівні оператора (STATEMENT-level):**

Виконуються один раз для всієї операції незалежно від кількості змінених рядків.

```sql
-- Приклад: логування операцій на рівні оператора
CREATE OR REPLACE FUNCTION log_bulk_operations()
RETURNS TRIGGER AS $$
DECLARE
    affected_rows INTEGER;
BEGIN
    -- Отримання кількості змінених рядків
    GET DIAGNOSTICS affected_rows = ROW_COUNT;

    -- Логування операції
    INSERT INTO operation_log (
        table_name,
        operation_type,
        rows_affected,
        executed_by,
        executed_at
    ) VALUES (
        TG_TABLE_NAME,
        TG_OP,
        affected_rows,
        current_user,
        CURRENT_TIMESTAMP
    );

    -- Перевірка на масові небезпечні операції
    IF TG_OP = 'DELETE' AND affected_rows > 1000 THEN
        RAISE WARNING 'Масове видалення % рядків з таблиці %',
                      affected_rows, TG_TABLE_NAME;
    END IF;

    RETURN NULL; -- для AFTER STATEMENT тригерів
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER monitor_bulk_changes
    AFTER INSERT OR UPDATE OR DELETE ON critical_data
    FOR EACH STATEMENT
    EXECUTE FUNCTION log_bulk_operations();
```

### Області застосування тригерів

#### Підтримка цілісності даних

Тригери можуть забезпечувати складні обмеження цілісності, які неможливо виразити через стандартні обмеження CHECK або FOREIGN KEY.

```sql
-- Забезпечення складного бізнес-правила
CREATE OR REPLACE FUNCTION enforce_project_constraints()
RETURNS TRIGGER AS $$
DECLARE
    active_projects INTEGER;
    employee_capacity INTEGER;
BEGIN
    -- Правило: співробітник не може бути призначений на більше 3 активних проєкти
    SELECT COUNT(*) INTO active_projects
    FROM project_assignments
    WHERE employee_id = NEW.employee_id
        AND status = 'ACTIVE';

    IF active_projects >= 3 THEN
        RAISE EXCEPTION 'Співробітник вже працює над максимальною кількістю проєктів';
    END IF;

    -- Правило: загальна завантаженість не може перевищувати 100%
    SELECT COALESCE(SUM(allocation_percentage), 0) INTO employee_capacity
    FROM project_assignments
    WHERE employee_id = NEW.employee_id
        AND status = 'ACTIVE';

    IF employee_capacity + NEW.allocation_percentage > 100 THEN
        RAISE EXCEPTION 'Перевищено допустиму завантаженість співробітника';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_project_assignment
    BEFORE INSERT OR UPDATE ON project_assignments
    FOR EACH ROW
    EXECUTE FUNCTION enforce_project_constraints();
```

#### Аудит та логування

Тригери ідеально підходять для автоматичного ведення журналів змін.

```sql
-- Універсальна система аудиту
CREATE OR REPLACE FUNCTION universal_audit()
RETURNS TRIGGER AS $$
DECLARE
    old_data JSONB;
    new_data JSONB;
BEGIN
    -- Формування JSON представлення старих та нових даних
    IF TG_OP = 'DELETE' THEN
        old_data := to_jsonb(OLD);
        new_data := NULL;
    ELSIF TG_OP = 'INSERT' THEN
        old_data := NULL;
        new_data := to_jsonb(NEW);
    ELSE -- UPDATE
        old_data := to_jsonb(OLD);
        new_data := to_jsonb(NEW);
    END IF;

    -- Запис в таблицю аудиту
    INSERT INTO audit_log (
        table_name,
        operation,
        old_values,
        new_values,
        changed_by,
        changed_at,
        client_address
    ) VALUES (
        TG_TABLE_NAME,
        TG_OP,
        old_data,
        new_data,
        current_user,
        CURRENT_TIMESTAMP,
        inet_client_addr()
    );

    RETURN COALESCE(NEW, OLD);
END;
$ LANGUAGE plpgsql;

-- Застосування до критичних таблиць
CREATE TRIGGER audit_employees
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION universal_audit();

CREATE TRIGGER audit_salaries
    AFTER INSERT OR UPDATE OR DELETE ON salaries
    FOR EACH ROW
    EXECUTE FUNCTION universal_audit();
```

#### Підтримка денормалізації

Тригери можуть автоматично оновлювати денормалізовані дані для покращення продуктивності запитів.

```sql
-- Підтримка агрегованих даних
CREATE OR REPLACE FUNCTION update_order_statistics()
RETURNS TRIGGER AS $
BEGIN
    -- При додаванні або зміні позиції замовлення
    IF TG_OP IN ('INSERT', 'UPDATE') THEN
        UPDATE orders
        SET
            total_items = (
                SELECT COUNT(*)
                FROM order_items
                WHERE order_id = NEW.order_id
            ),
            total_amount = (
                SELECT COALESCE(SUM(quantity * unit_price), 0)
                FROM order_items
                WHERE order_id = NEW.order_id
            ),
            last_modified = CURRENT_TIMESTAMP
        WHERE order_id = NEW.order_id;
    END IF;

    -- При видаленні позиції
    IF TG_OP = 'DELETE' THEN
        UPDATE orders
        SET
            total_items = (
                SELECT COUNT(*)
                FROM order_items
                WHERE order_id = OLD.order_id
            ),
            total_amount = (
                SELECT COALESCE(SUM(quantity * unit_price), 0)
                FROM order_items
                WHERE order_id = OLD.order_id
            ),
            last_modified = CURRENT_TIMESTAMP
        WHERE order_id = OLD.order_id;
    END IF;

    RETURN COALESCE(NEW, OLD);
END;
$ LANGUAGE plpgsql;

CREATE TRIGGER maintain_order_totals
    AFTER INSERT OR UPDATE OR DELETE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION update_order_statistics();
```

#### Каскадні операції та автоматизація

Тригери можуть виконувати складні каскадні операції, які виходять за межі стандартних можливостей FOREIGN KEY.

```sql
-- Каскадне оновлення при зміні статусу замовлення
CREATE OR REPLACE FUNCTION cascade_order_status()
RETURNS TRIGGER AS $
BEGIN
    -- При переході замовлення в статус "Доставлено"
    IF NEW.status = 'DELIVERED' AND OLD.status != 'DELIVERED' THEN
        -- Оновлення статистики клієнта
        UPDATE customers
        SET
            total_orders = total_orders + 1,
            total_spent = total_spent + NEW.total_amount,
            last_order_date = CURRENT_DATE
        WHERE customer_id = NEW.customer_id;

        -- Нарахування бонусних балів
        INSERT INTO loyalty_points (customer_id, points, source, created_at)
        VALUES (
            NEW.customer_id,
            FLOOR(NEW.total_amount / 10), -- 1 бал за кожні 10 грн
            'ORDER_' || NEW.order_id,
            CURRENT_TIMESTAMP
        );

        -- Автоматичне створення запиту на відгук
        INSERT INTO review_requests (order_id, customer_id, requested_at)
        VALUES (NEW.order_id, NEW.customer_id, CURRENT_TIMESTAMP);
    END IF;

    -- При скасуванні замовлення
    IF NEW.status = 'CANCELLED' AND OLD.status != 'CANCELLED' THEN
        -- Повернення товару на склад
        UPDATE inventory
        SET stock_quantity = stock_quantity + oi.quantity
        FROM order_items oi
        WHERE inventory.product_id = oi.product_id
            AND oi.order_id = NEW.order_id;

        -- Повернення бонусних балів, якщо були використані
        IF NEW.points_used > 0 THEN
            INSERT INTO loyalty_points (customer_id, points, source, created_at)
            VALUES (
                NEW.customer_id,
                NEW.points_used,
                'REFUND_ORDER_' || NEW.order_id,
                CURRENT_TIMESTAMP
            );
        END IF;
    END IF;

    RETURN NEW;
END;
$ LANGUAGE plpgsql;

CREATE TRIGGER handle_order_status_change
    AFTER UPDATE ON orders
    FOR EACH ROW
    WHEN (OLD.status IS DISTINCT FROM NEW.status)
    EXECUTE FUNCTION cascade_order_status();
```

## Управління подіями та правилами в активних БД

### Концепція активних баз даних

Активні бази даних розширюють традиційну пасивну модель, де операції виконуються тільки у відповідь на явні запити користувачів. В активній базі даних система може самостійно ініціювати дії у відповідь на певні події або умови.

#### Модель ECA (Event-Condition-Action)

Активні бази даних зазвичай базуються на моделі ECA, де кожне правило складається з трьох компонентів.

Подія визначає, що саме викликає правило. Це може бути операція модифікації даних, досягнення певного часу або настання зовнішньої події.

Умова перевіряється після виникнення події. Якщо умова виконується, активується дія.

Дія визначає, що має статися, якщо подія відбулася та умова виконана.

```sql
-- Реалізація ECA моделі через тригер
CREATE OR REPLACE FUNCTION eca_low_stock_alert()
RETURNS TRIGGER AS $
BEGIN
    -- Event: UPDATE операції на таблиці inventory
    -- Condition: перевірка рівня запасів
    IF NEW.stock_quantity < (
        SELECT reorder_level
        FROM products
        WHERE product_id = NEW.product_id
    ) AND NEW.stock_quantity < OLD.stock_quantity THEN
        -- Action: створення замовлення постачальнику
        INSERT INTO purchase_orders (
            product_id,
            supplier_id,
            quantity,
            order_date,
            status
        )
        SELECT
            NEW.product_id,
            p.preferred_supplier_id,
            p.optimal_order_quantity,
            CURRENT_DATE,
            'PENDING'
        FROM products p
        WHERE p.product_id = NEW.product_id;

        -- Action: відправка повідомлення
        INSERT INTO notifications (
            type,
            message,
            priority,
            created_at
        ) VALUES (
            'LOW_STOCK_ALERT',
            format('Низький рівень запасів для товару %s: %s одиниць',
                   NEW.product_id, NEW.stock_quantity),
            'HIGH',
            CURRENT_TIMESTAMP
        );
    END IF;

    RETURN NEW;
END;
$ LANGUAGE plpgsql;

CREATE TRIGGER monitor_stock_levels
    AFTER UPDATE ON inventory
    FOR EACH ROW
    EXECUTE FUNCTION eca_low_stock_alert();
```

### Складні правила та умови

Активні бази даних можуть підтримувати складні правила з множинними умовами та залежностями між подіями.

```sql
-- Складне правило з множинними умовами
CREATE OR REPLACE FUNCTION complex_pricing_rule()
RETURNS TRIGGER AS $
DECLARE
    customer_tier VARCHAR(20);
    order_count INTEGER;
    seasonal_discount NUMERIC;
BEGIN
    -- Визначення категорії клієнта
    SELECT
        CASE
            WHEN total_spent > 50000 THEN 'PLATINUM'
            WHEN total_spent > 20000 THEN 'GOLD'
            WHEN total_spent > 5000 THEN 'SILVER'
            ELSE 'STANDARD'
        END,
        total_orders
    INTO customer_tier, order_count
    FROM customers
    WHERE customer_id = NEW.customer_id;

    -- Перевірка сезонних знижок
    SELECT discount_rate INTO seasonal_discount
    FROM seasonal_promotions
    WHERE product_id = NEW.product_id
        AND CURRENT_DATE BETWEEN start_date AND end_date
        AND is_active = TRUE
    LIMIT 1;

    -- Застосування складної логіки ціноутворення
    NEW.unit_price := (
        SELECT base_price
        FROM products
        WHERE product_id = NEW.product_id
    );

    -- Знижка за категорією клієнта
    NEW.unit_price := NEW.unit_price * (
        CASE customer_tier
            WHEN 'PLATINUM' THEN 0.85  -- 15% знижка
            WHEN 'GOLD' THEN 0.90      -- 10% знижка
            WHEN 'SILVER' THEN 0.95    -- 5% знижка
            ELSE 1.0
        END
    );

    -- Додаткова знижка за обсяг в поточному замовленні
    IF NEW.quantity >= 100 THEN
        NEW.unit_price := NEW.unit_price * 0.90;
    ELSIF NEW.quantity >= 50 THEN
        NEW.unit_price := NEW.unit_price * 0.95;
    END IF;

    -- Застосування сезонної знижки
    IF seasonal_discount IS NOT NULL THEN
        NEW.unit_price := NEW.unit_price * (1 - seasonal_discount);
    END IF;

    -- Спеціальна пропозиція для нових клієнтів
    IF order_count = 0 THEN
        NEW.unit_price := NEW.unit_price * 0.90;
        INSERT INTO promotion_tracking (customer_id, promotion_type, discount_amount)
        VALUES (NEW.customer_id, 'FIRST_ORDER', NEW.unit_price * 0.10 * NEW.quantity);
    END IF;

    -- Округлення до копійок
    NEW.unit_price := ROUND(NEW.unit_price, 2);

    RETURN NEW;
END;
$ LANGUAGE plpgsql;

CREATE TRIGGER apply_dynamic_pricing
    BEFORE INSERT OR UPDATE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION complex_pricing_rule();
```

### Управління ланцюжками тригерів

Коли один тригер викликає зміни, які активують інші тригери, утворюється ланцюжок тригерів. Важливо правильно керувати такими ланцюжками, щоб уникнути нескінченних циклів або непередбачуваної поведінки.

```sql
-- Механізм запобігання циклічним викликам
CREATE TABLE trigger_execution_context (
    context_id SERIAL PRIMARY KEY,
    trigger_name VARCHAR(100),
    started_at TIMESTAMP,
    depth INTEGER
);

CREATE OR REPLACE FUNCTION prevent_trigger_recursion()
RETURNS TRIGGER AS $
DECLARE
    current_depth INTEGER;
    max_depth CONSTANT INTEGER := 5; -- максимальна глибина вкладеності
BEGIN
    -- Перевірка поточної глибини виконання
    SELECT COALESCE(MAX(depth), 0) INTO current_depth
    FROM trigger_execution_context
    WHERE trigger_name = TG_NAME
        AND started_at > CURRENT_TIMESTAMP - INTERVAL '1 second';

    IF current_depth >= max_depth THEN
        RAISE EXCEPTION 'Виявлено можливий циклічний виклик тригера %', TG_NAME;
    END IF;

    -- Реєстрація виконання
    INSERT INTO trigger_execution_context (trigger_name, started_at, depth)
    VALUES (TG_NAME, CURRENT_TIMESTAMP, current_depth + 1);

    -- Основна логіка тригера
    -- ...

    -- Очищення контексту
    DELETE FROM trigger_execution_context
    WHERE trigger_name = TG_NAME
        AND depth = current_depth + 1;

    RETURN NEW;
END;
$ LANGUAGE plpgsql;
```

## Представлення (VIEW): визначення, оновлювані представлення, матеріалізація

### Базові концепції представлень

Представлення, або view, є віртуальною таблицею, що базується на результаті SQL-запиту. Представлення не зберігають дані фізично, а динамічно обчислюються під час звернення до них.

#### Створення та використання простих представлень

```sql
-- Просте представлення для спрощення запитів
CREATE VIEW employee_summary AS
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS full_name,
    e.email,
    d.department_name,
    p.position_name,
    e.salary,
    e.hire_date,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, e.hire_date)) AS years_employed
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN positions p ON e.position_id = p.position_id
WHERE e.status = 'ACTIVE';

-- Використання представлення
SELECT full_name, department_name, salary
FROM employee_summary
WHERE years_employed > 5
ORDER BY salary DESC;
```

#### Переваги використання представлень

Представлення надають кілька важливих переваг у проєктуванні баз даних.

Абстракція складності дозволяє приховати складні JOIN-операції та обчислення за простим інтерфейсом.

```sql
-- Складний запит інкапсульований у представлення
CREATE VIEW quarterly_sales_report AS
SELECT
    p.product_id,
    p.product_name,
    p.category,
    DATE_TRUNC('quarter', o.order_date) AS quarter,
    COUNT(DISTINCT o.order_id) AS orders_count,
    SUM(oi.quantity) AS total_units_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    AVG(oi.unit_price) AS average_price,
    DENSE_RANK() OVER (
        PARTITION BY DATE_TRUNC('quarter', o.order_date)
        ORDER BY SUM(oi.quantity * oi.unit_price) DESC
    ) AS revenue_rank
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'COMPLETED'
GROUP BY p.product_id, p.product_name, p.category, DATE_TRUNC('quarter', o.order_date);

-- Простий запит до представлення
SELECT product_name, quarter, total_revenue, revenue_rank
FROM quarterly_sales_report
WHERE quarter = DATE_TRUNC('quarter', CURRENT_DATE)
    AND revenue_rank <= 10;
```

Безпека та контроль доступу забезпечують можливість надавати доступ до підмножини даних без відкриття базових таблиць.

```sql
-- Представлення для обмеженого доступу HR департаменту
CREATE VIEW hr_employee_view AS
SELECT
    employee_id,
    first_name,
    last_name,
    email,
    phone,
    department_id,
    position_id,
    hire_date,
    status
FROM employees;
-- Не включає зарплату та інші конфіденційні дані

-- Надання доступу тільки до представлення
GRANT SELECT ON hr_employee_view TO hr_staff;
REVOKE SELECT ON employees FROM hr_staff;
```

Логічна незалежність даних дозволяє змінювати структуру базових таблиць без впливу на застосунки.

```sql
-- Представлення забезпечує стабільний інтерфейс
CREATE VIEW customer_info AS
SELECT
    customer_id,
    first_name,
    last_name,
    email,
    phone,
    -- Якщо адреса переміститься в окрему таблицю,
    -- представлення приховує цю зміну
    address_street,
    address_city,
    address_postal_code
FROM customers;
```

### Оновлювані представлення

Деякі представлення підтримують операції INSERT, UPDATE та DELETE, якщо вони відповідають певним критеріям.

#### Критерії оновлюваності

Представлення зазвичай є оновлюваним, якщо воно базується на одній таблиці, не містить агрегатних функцій, DISTINCT, GROUP BY, HAVING, не містить UNION або INTERSECT та включає всі обов'язкові стовпці базової таблиці.

```sql
-- Оновлюване представлення
CREATE VIEW active_employees AS
SELECT
    employee_id,
    first_name,
    last_name,
    email,
    department_id,
    salary
FROM employees
WHERE status = 'ACTIVE';

-- Операції модифікації працюють безпосередньо
UPDATE active_employees
SET salary = salary * 1.05
WHERE department_id = 10;

INSERT INTO active_employees (first_name, last_name, email, department_id, salary)
VALUES ('Іван', 'Петренко', 'ivan.petrenko@company.com', 10, 50000);
```

#### INSTEAD OF тригери для складних представлень

Для представлень, що не є природно оновлюваними, можна використовувати INSTEAD OF тригери для визначення логіки оновлення.

```sql
-- Складне представлення з JOIN
CREATE VIEW employee_department_view AS
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    e.email,
    d.department_id,
    d.department_name,
    d.location,
    e.salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- INSTEAD OF тригер для UPDATE
CREATE OR REPLACE FUNCTION update_employee_department()
RETURNS TRIGGER AS $
BEGIN
    -- Оновлення таблиці співробітників
    UPDATE employees
    SET
        first_name = NEW.first_name,
        last_name = NEW.last_name,
        email = NEW.email,
        salary = NEW.salary
    WHERE employee_id = NEW.employee_id;

    -- Якщо назва департаменту змінилася, оновлюємо таблицю департаментів
    IF OLD.department_name IS DISTINCT FROM NEW.department_name THEN
        UPDATE departments
        SET department_name = NEW.department_name
        WHERE department_id = NEW.department_id;
    END IF;

    -- Якщо змінився сам департамент
    IF OLD.department_id IS DISTINCT FROM NEW.department_id THEN
        UPDATE employees
        SET department_id = NEW.department_id
        WHERE employee_id = NEW.employee_id;
    END IF;

    RETURN NEW;
END;
$ LANGUAGE plpgsql;

CREATE TRIGGER update_employee_dept_view
    INSTEAD OF UPDATE ON employee_department_view
    FOR EACH ROW
    EXECUTE FUNCTION update_employee_department();
```

### Матеріалізовані представлення

Матеріалізовані представлення фізично зберігають результат запиту, на відміну від звичайних представлень, що обчислюються динамічно. Це значно прискорює запити до складних агрегацій, але вимагає періодичного оновлення.

#### Створення та використання

```sql
-- Створення матеріалізованого представлення
CREATE MATERIALIZED VIEW sales_analytics AS
SELECT
    p.category,
    DATE_TRUNC('month', o.order_date) AS month,
    COUNT(DISTINCT o.order_id) AS orders_count,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    AVG(oi.quantity * oi.unit_price) AS avg_order_value,
    SUM(oi.quantity) AS units_sold
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'COMPLETED'
GROUP BY p.category, DATE_TRUNC('month', o.order_date);

-- Створення індексу для швидкого пошуку
CREATE INDEX idx_sales_analytics_category_month
ON sales_analytics(category, month);

-- Запити виконуються дуже швидко, оскільки дані вже агреговані
SELECT category, month, total_revenue
FROM sales_analytics
WHERE month >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '6 months')
ORDER BY month DESC, total_revenue DESC;
```

#### Стратегії оновлення

Матеріалізовані представлення потребують періодичного оновлення для відображення актуальних даних.

```sql
-- Повне оновлення (перебудова всіх даних)
REFRESH MATERIALIZED VIEW sales_analytics;

-- Конкурентне оновлення (не блокує читання)
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_analytics;

-- Автоматизоване оновлення за розкладом
CREATE OR REPLACE FUNCTION refresh_sales_analytics()
RETURNS void AS $
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY sales_analytics;

    INSERT INTO materialized_view_refresh_log (
        view_name,
        refreshed_at,
        duration
    ) VALUES (
        'sales_analytics',
        CURRENT_TIMESTAMP,
        EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - transaction_timestamp()))
    );
END;
$ LANGUAGE plpgsql;

-- Використання планувальника для щоденного оновлення
-- (через pg_cron або зовнішній scheduler)
```

#### Інкрементальне оновлення

Для великих матеріалізованих представлень часто ефективніше використовувати інкрементальне оновлення через тригери.

```sql
-- Таблиця для відстеження змін
CREATE TABLE sales_changes (
    change_id SERIAL PRIMARY KEY,
    order_id INTEGER,
    change_type VARCHAR(10),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Тригер для відстеження змін
CREATE OR REPLACE FUNCTION track_sales_changes()
RETURNS TRIGGER AS $
BEGIN
    INSERT INTO sales_changes (order_id, change_type)
    VALUES (
        COALESCE(NEW.order_id, OLD.order_id),
        TG_OP
    );
    RETURN COALESCE(NEW, OLD);
END;
$ LANGUAGE plpgsql;

CREATE TRIGGER track_order_changes
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION track_sales_changes();

-- Процедура інкрементального оновлення
CREATE OR REPLACE PROCEDURE incremental_refresh_sales_analytics()
LANGUAGE plpgsql
AS $
DECLARE
    affected_periods RECORD;
BEGIN
    -- Визначення періодів, що потребують оновлення
    FOR affected_periods IN
        SELECT DISTINCT
            p.category,
            DATE_TRUNC('month', o.order_date) AS month
        FROM sales_changes sc
        JOIN orders o ON sc.order_id = o.order_id
        JOIN order_items oi ON o.order_id = oi.order_id
        JOIN products p ON oi.product_id = p.product_id
        WHERE sc.changed_at > (
            SELECT MAX(refreshed_at)
            FROM materialized_view_refresh_log
            WHERE view_name = 'sales_analytics'
        )
    LOOP
        -- Видалення старих агрегатів для цього періоду
        DELETE FROM sales_analytics
        WHERE category = affected_periods.category
            AND month = affected_periods.month;

        -- Вставка оновлених агрегатів
        INSERT INTO sales_analytics
        SELECT
            p.category,
            DATE_TRUNC('month', o.order_date) AS month,
            COUNT(DISTINCT o.order_id),
            COUNT(DISTINCT o.customer_id),
            SUM(oi.quantity * oi.unit_price),
            AVG(oi.quantity * oi.unit_price),
            SUM(oi.quantity)
        FROM products p
        JOIN order_items oi ON p.product_id = oi.product_id
        JOIN orders o ON oi.order_id = o.order_id
        WHERE o.status = 'COMPLETED'
            AND p.category = affected_periods.category
            AND DATE_TRUNC('month', o.order_date) = affected_periods.month
        GROUP BY p.category, DATE_TRUNC('month', o.order_date);
    END LOOP;

    -- Очищення журналу змін
    DELETE FROM sales_changes
    WHERE changed_at <= CURRENT_TIMESTAMP - INTERVAL '1 day';

    COMMIT;
END;
$;
```

## Компроміси між серверною та клієнтською логікою

### Розподіл відповідальності

Одне з ключових архітектурних рішень при проєктуванні систем баз даних - це вибір між розміщенням бізнес-логіки на сервері бази даних або в клієнтському застосунку. Кожен підхід має свої переваги та недоліки.

#### Логіка на сервері бази даних

**Переваги серверної логіки:**

Централізація бізнес-правил забезпечує єдину точку контролю. Коли логіка знаходиться в базі даних через збережені процедури та тригери, всі застосунки автоматично дотримуються однакових правил, незалежно від мови програмування або платформи.

```sql
-- Централізоване правило валідації в БД
CREATE OR REPLACE FUNCTION validate_credit_limit()
RETURNS TRIGGER AS $
BEGIN
    IF NEW.credit_limit > 100000 AND NEW.credit_rating < 'A' THEN
        RAISE EXCEPTION 'Кредитний ліміт понад 100000 доступний тільки для рейтингу A';
    END IF;

    IF NEW.credit_limit < 0 THEN
        RAISE EXCEPTION 'Кредитний ліміт не може бути від''ємним';
    END IF;

    -- Автоматична перевірка кредитної історії
    IF EXISTS (
        SELECT 1 FROM payment_defaults
        WHERE customer_id = NEW.customer_id
            AND resolved = FALSE
    ) THEN
        NEW.credit_limit := LEAST(NEW.credit_limit, 10000);
        RAISE NOTICE 'Ліміт знижено через неоплачені борги';
    END IF;

    RETURN NEW;
END;
$ LANGUAGE plpgsql;

-- Всі застосунки (веб, мобільні, API) автоматично дотримуються цих правил
```

Зменшення мережевого трафіку досягається через виконання складної обробки безпосередньо біля даних.

```sql
-- Замість передачі великих обсягів даних клієнту
-- обробка відбувається на сервері
CREATE OR REPLACE FUNCTION calculate_customer_metrics(
    customer_id_param INTEGER,
    period_months INTEGER DEFAULT 12
)
RETURNS TABLE (
    total_orders INTEGER,
    total_spent NUMERIC,
    average_order_value NUMERIC,
    loyalty_score NUMERIC,
    recommended_tier VARCHAR(20)
) AS $
BEGIN
    RETURN QUERY
    WITH customer_stats AS (
        SELECT
            COUNT(*) AS orders,
            SUM(total_amount) AS spent,
            AVG(total_amount) AS avg_value
        FROM orders
        WHERE customer_id = customer_id_param
            AND order_date >= CURRENT_DATE - (period_months || ' months')::INTERVAL
    )
    SELECT
        orders,
        spent,
        avg_value,
        -- Складний розрахунок на сервері
        (spent / 1000.0 + orders * 2.0) AS loyalty,
        CASE
            WHEN spent > 50000 THEN 'PLATINUM'
            WHEN spent > 20000 THEN 'GOLD'
            WHEN spent > 5000 THEN 'SILVER'
            ELSE 'STANDARD'
        END AS tier
    FROM customer_stats;
END;
$ LANGUAGE plpgsql STABLE;

-- Клієнт отримує тільки фінальний результат
SELECT * FROM calculate_customer_metrics(1001);
```

Транзакційна цілісність гарантується, оскільки вся пов'язана логіка виконується в одному контексті транзакції.

```sql
-- Атомарна бізнес-операція в БД
CREATE OR REPLACE PROCEDURE process_payment(
    order_id_param INTEGER,
    payment_amount NUMERIC
)
LANGUAGE plpgsql
AS $
DECLARE
    order_total NUMERIC;
    customer_balance NUMERIC;
BEGIN
    -- Все в одній транзакції
    SELECT total_amount INTO order_total
    FROM orders
    WHERE order_id = order_id_param
    FOR UPDATE; -- блокування запису

    IF payment_amount < order_total THEN
        RAISE EXCEPTION 'Недостатня сума платежу';
    END IF;

    -- Створення запису платежу
    INSERT INTO payments (order_id, amount, payment_date, status)
    VALUES (order_id_param, payment_amount, CURRENT_DATE, 'COMPLETED');

    -- Оновлення статусу замовлення
    UPDATE orders
    SET status = 'PAID', paid_date = CURRENT_DATE
    WHERE order_id = order_id_param;

    -- Нарахування бонусів
    INSERT INTO loyalty_points (customer_id, points, source)
    SELECT customer_id, FLOOR(payment_amount / 10), 'PAYMENT'
    FROM orders
    WHERE order_id = order_id_param;

    -- Все або нічого - автоматичний COMMIT/ROLLBACK
    COMMIT;
END;
$;
```

**Недоліки серверної логіки:**

Прив'язка до конкретної СУБД є значним недоліком, оскільки кожна СУБД має власний діалект процедурної мови. Міграція на іншу платформу може вимагати повного переписування коду.

```sql
-- PostgreSQL специфічний код
CREATE OR REPLACE FUNCTION pg_specific_logic()
RETURNS JSON AS $
BEGIN
    RETURN json_build_object(
        'timestamp', to_char(CURRENT_TIMESTAMP, 'YYYY-MM-DD HH24:MI:SS'),
        'data', jsonb_agg(row_to_json(t))
    )
    FROM some_table t;
END;
$ LANGUAGE plpgsql;

-- Цей код не працюватиме в SQL Server або Oracle без змін
```

Складність тестування та налагодження виникає через обмежені інструменти розробки для SQL коду порівняно з традиційними мовами програмування.

Обмеження масштабованості з'являються, коли логіка на сервері БД стає вузьким місцем, оскільки вертикальне масштабування серверів баз даних коштовніше горизонтального масштабування застосунків.

Складність версіонування та розгортання процедур та тригерів потребує додаткових процесів управління змінами в базі даних.

#### Логіка в клієнтському застосунку

**Переваги клієнтської логіки:**

Гнучкість розробки надає доступ до повноцінних мов програмування з багатими екосистемами бібліотек та фреймворків.

```python
# Приклад бізнес-логіки на Python
class OrderProcessor:
    def __init__(self, db_connection):
        self.db = db_connection

    def calculate_discount(self, customer_id: int, order_total: float) -> float:
        """Складна логіка розрахунку знижки"""
        customer = self.db.get_customer(customer_id)

        # Базова знижка за категорією
        discount = customer.tier_discount

        # Додаткові знижки за умовами
        if self._is_birthday_month(customer):
            discount += 0.05

        if self._has_referrals(customer):
            discount += 0.03

        # Сезонні промоції
        seasonal = self._get_seasonal_discount()
        discount = max(discount, seasonal)

        # Обмеження максимальної знижки
        return min(discount, 0.30)

    def _is_birthday_month(self, customer) -> bool:
        return customer.birth_date.month == datetime.now().month

    def _has_referrals(self, customer) -> bool:
        return self.db.count_referrals(customer.id) >= 3

    def _get_seasonal_discount(self) -> float:
        # Інтеграція з зовнішнім сервісом промоцій
        return PromotionService.get_current_discount()
```

Краще тестування стає можливим через використання стандартних інструментів тестування, моків та CI/CD практик.

```python
# Unit тести для бізнес-логіки
import unittest
from unittest.mock import Mock, patch

class TestOrderProcessor(unittest.TestCase):
    def setUp(self):
        self.db_mock = Mock()
        self.processor = OrderProcessor(self.db_mock)

    def test_birthday_discount(self):
        # Мокування клієнта з днем народження цього місяця
        customer = Mock()
        customer.birth_date = datetime(1990, datetime.now().month, 15)
        customer.tier_discount = 0.10

        self.db_mock.get_customer.return_value = customer
        self.db_mock.count_referrals.return_value = 0

        with patch.object(self.processor, '_get_seasonal_discount', return_value=0):
            discount = self.processor.calculate_discount(1, 1000)
            self.assertEqual(discount, 0.15)  # 10% + 5% birthday

    def test_maximum_discount_cap(self):
        customer = Mock()
        customer.tier_discount = 0.25
        # ... тести для граничних випадків
```

Легше версіонування коду застосунку через використання систем контролю версій та стандартних практик розгортання.

Горизонтальне масштабування застосунків простіше та дешевше, ніж масштабування серверів баз даних.

**Недоліки клієнтської логіки:**

Дублювання логіки може виникнути, якщо є кілька застосунків, що працюють з однією базою даних.

Більше мережевих звернень потрібно для виконання складних операцій, що збільшує латентність.

Ризики порушення цілісності зростають, якщо різні застосунки реалізують логіку по-різному або з помилками.

### Гібридний підхід

Найкращою практикою часто є гібридний підхід, де критично важлива логіка цілісності даних знаходиться в базі даних, а бізнес-логіка застосунку - в клієнті.

#### Рекомендації щодо розподілу

**В базі даних варто розміщувати:**

Обмеження цілісності та валідації на рівні даних, які завжди мають бути дотримані.

```sql
-- Критична логіка цілісності в БД
CREATE OR REPLACE FUNCTION ensure_account_integrity()
RETURNS TRIGGER AS $
BEGIN
    -- Фундаментальні правила, що не можуть порушуватися
    IF NEW.balance < 0 AND NEW.account_type != 'CREDIT' THEN
        RAISE EXCEPTION 'Дебетовий рахунок не може мати від''ємний баланс';
    END IF;

    IF NEW.account_type = 'CREDIT' AND NEW.balance < NEW.credit_limit * -1 THEN
        RAISE EXCEPTION 'Перевищено кредитний ліміт';
    END IF;

    RETURN NEW;
END;
$ LANGUAGE plpgsql;
```

Аудит та логування змін критичних даних.

```sql
-- Автоматичний аудит в БД
CREATE TRIGGER audit_financial_transactions
    AFTER INSERT OR UPDATE OR DELETE ON transactions
    FOR EACH ROW
    EXECUTE FUNCTION universal_audit();
```

Складні агрегації та звіти, де обробка великих обсягів даних на сервері ефективніша.

```sql
-- Складна аналітика в БД
CREATE OR REPLACE FUNCTION generate_sales_report(
    start_date DATE,
    end_date DATE
)
RETURNS TABLE (
    product_category VARCHAR,
    total_revenue NUMERIC,
    total_orders INTEGER,
    unique_customers INTEGER,
    growth_rate NUMERIC
) AS $
BEGIN
    RETURN QUERY
    WITH current_period AS (
        SELECT
            p.category,
            SUM(oi.quantity * oi.unit_price) AS revenue,
            COUNT(DISTINCT o.order_id) AS orders,
            COUNT(DISTINCT o.customer_id) AS customers
        FROM orders o
        JOIN order_items oi ON o.order_id = oi.order_id
        JOIN products p ON oi.product_id = p.product_id
        WHERE o.order_date BETWEEN start_date AND end_date
        GROUP BY p.category
    ),
    previous_period AS (
        SELECT
            p.category,
            SUM(oi.quantity * oi.unit_price) AS revenue
        FROM orders o
        JOIN order_items oi ON o.order_id = oi.order_id
        JOIN products p ON oi.product_id = p.product_id
        WHERE o.order_date BETWEEN
            start_date - (end_date - start_date) AND
            start_date - INTERVAL '1 day'
        GROUP BY p.category
    )
    SELECT
        c.category,
        c.revenue,
        c.orders,
        c.customers,
        CASE
            WHEN p.revenue > 0 THEN
                ((c.revenue - p.revenue) / p.revenue * 100)
            ELSE NULL
        END AS growth
    FROM current_period c
    LEFT JOIN previous_period p ON c.category = p.category
    ORDER BY c.revenue DESC;
END;
$ LANGUAGE plpgsql STABLE;
```

**В застосунку варто розміщувати:**

Бізнес-логіку, специфічну для конкретного застосунку.

```python
# Логіка рекомендаційної системи в застосунку
class RecommendationEngine:
    def get_product_recommendations(self, user_id: int, limit: int = 10):
        # Складні ML алгоритми
        user_profile = self._build_user_profile(user_id)
        similar_users = self._find_similar_users(user_profile)
        products = self._collaborative_filtering(similar_users)

        # Інтеграція з зовнішніми сервісами
        trending = ExternalAPI.get_trending_products()
        products = self._merge_recommendations(products, trending)

        return products[:limit]
```

Логіку презентаційного рівня та форматування даних.

```javascript
// Форматування даних для UI
class OrderFormatter {
    static formatForDisplay(order) {
        return {
            id: order.id,
            displayDate: moment(order.date).format('DD.MM.YYYY'),
            totalFormatted: new Intl.NumberFormat('uk-UA', {
                style: 'currency',
                currency: 'UAH'
            }).format(order.total),
            statusText: this.getStatusText(order.status),
            statusColor: this.getStatusColor(order.status)
        };
    }
}
```

Інтеграцію з зовнішніми сервісами та API.

```python
# Інтеграція з платіжними системами
class PaymentService:
    def process_payment(self, order_id: int, amount: float):
        # Виклик зовнішнього API
        payment_gateway = PaymentGateway(api_key=settings.PAYMENT_API_KEY)

        try:
            result = payment_gateway.charge(
                amount=amount,
                currency='UAH',
                description=f'Payment for order {order_id}'
            )

            # Оновлення БД після успішного платежу
            with self.db.transaction():
                self.db.execute(
                    "UPDATE orders SET status = 'PAID' WHERE id = %s",
                    (order_id,)
                )
                self.db.execute(
                    "INSERT INTO payments (order_id, amount, gateway_id) VALUES (%s, %s, %s)",
                    (order_id, amount, result.transaction_id)
                )

            return result
        except PaymentError as e:
            logger.error(f"Payment failed: {e}")
            raise
```

#### Практичний приклад гібридної архітектури

```sql
-- Шар даних: критична логіка в БД
CREATE OR REPLACE FUNCTION create_order(
    customer_id_param INTEGER,
    items JSONB
)
RETURNS INTEGER AS $
DECLARE
    new_order_id INTEGER;
    item JSONB;
BEGIN
    -- Валідація в БД
    IF NOT EXISTS (SELECT 1 FROM customers WHERE customer_id = customer_id_param AND status = 'ACTIVE') THEN
        RAISE EXCEPTION 'Клієнт не знайдений або неактивний';
    END IF;

    -- Створення замовлення
    INSERT INTO orders (customer_id, order_date, status)
    VALUES (customer_id_param, CURRENT_DATE, 'PENDING')
    RETURNING order_id INTO new_order_id;

    -- Додавання позицій
    FOR item IN SELECT * FROM jsonb_array_elements(items)
    LOOP
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        SELECT
            new_order_id,
            (item->>'product_id')::INTEGER,
            (item->>'quantity')::INTEGER,
            price
        FROM products
        WHERE product_id = (item->>'product_id')::INTEGER;

        -- Зменшення запасів (критична операція в БД)
        UPDATE inventory
        SET stock_quantity = stock_quantity - (item->>'quantity')::INTEGER
        WHERE product_id = (item->>'product_id')::INTEGER;

        IF NOT FOUND OR (SELECT stock_quantity FROM inventory WHERE product_id = (item->>'product_id')::INTEGER) < 0 THEN
            RAISE EXCEPTION 'Недостатньо товару на складі';
        END IF;
    END LOOP;

    RETURN new_order_id;
END;
$ LANGUAGE plpgsql;
```

```python
# Шар застосунку: бізнес-логіка
class OrderService:
    def __init__(self, db, notification_service, payment_service):
        self.db = db
        self.notifications = notification_service
        self.payments = payment_service

    def place_order(self, customer_id: int, items: List[OrderItem]) -> Order:
        """Розміщення замовлення з повною бізнес-логікою"""

        # Валідація на рівні застосунку
        self._validate_items(items)

        # Розрахунок знижок (складна логіка в застосунку)
        discount = self._calculate_discount(customer_id, items)

        # Розрахунок доставки (інтеграція з зовнішнім сервісом)
        shipping_cost = self._calculate_shipping(customer_id, items)

        # Підготовка даних для БД
        items_json = json.dumps([
            {'product_id': item.product_id, 'quantity': item.quantity}
            for item in items
        ])

        try:
            # Виклик процедури БД для створення замовлення
            order_id = self.db.call_function(
                'create_order',
                customer_id,
                items_json
            )

            # Застосування знижки (оновлення в БД)
            total = self._calculate_total(items)
            final_total = total * (1 - discount) + shipping_cost

            self.db.execute(
                "UPDATE orders SET total_amount = %s, discount_amount = %s WHERE order_id = %s",
                (final_total, total * discount, order_id)
            )

            # Асинхронна обробка (не блокує транзакцію БД)
            self._send_notifications_async(order_id, customer_id)

            # Логування в систему аналітики
            analytics.track('order_placed', {
                'order_id': order_id,
                'customer_id': customer_id,
                'total': final_total
            })

            return Order.from_id(order_id)

        except DatabaseError as e:
            logger.error(f"Order creation failed: {e}")
            raise OrderCreationError("Не вдалося створити замовлення")

    def _calculate_discount(self, customer_id: int, items: List[OrderItem]) -> float:
        """Складна логіка розрахунку знижки"""
        # ML модель, промокоди, сезонні акції тощо
        pass

    def _send_notifications_async(self, order_id: int, customer_id: int):
        """Асинхронна відправка повідомлень"""
        task_queue.enqueue(
            'send_order_confirmation',
            order_id=order_id,
            customer_id=customer_id
        )
```

### Критерії вибору архітектури

При прийнятті рішення про розміщення логіки варто враховувати наступні фактори.

Тип операції: транзакційні операції, що вимагають ACID властивостей, краще розміщувати ближче до даних у базі даних. Аналітичні операції та складні обчислення можуть бути ефективнішими в застосунку з використанням спеціалізованих бібліотек.

Частота змін: логіка, що часто змінюється, зручніше розміщувати в застосунку, де процес розгортання швидший. Стабільні правила цілісності даних можна розмістити в базі даних.

Потреба в масштабуванні: якщо очікується високе навантаження, логіка в застосунку дозволяє легше масштабувати систему горизонтально.

Кількість клієнтів: якщо з базою даних працює багато різних застосунків, централізація критичної логіки в БД забезпечує консистентність.

Складність операцій: складні бізнес-процеси з інтеграціями зручніше реалізовувати в застосунку. Операції з великими обсягами даних ефективніше виконувати в БД.

## Висновки

Збережені процедури та активні бази даних є потужними інструментами сучасних СУБД, що дозволяють реалізувати складну логіку безпосередньо на рівні даних. Тригери забезпечують автоматичну реакцію на події та підтримку складних бізнес-правил без втручання клієнтських застосунків. Представлення, як звичайні, так і матеріалізовані, надають гнучкі механізми абстракції та оптимізації доступу до даних.

Вибір між розміщенням логіки на сервері бази даних або в клієнтському застосунку є важливим архітектурним рішенням, що впливає на продуктивність, підтримку та масштабованість системи. Найкращим підходом часто є гібридна архітектура, де критична логіка цілісності даних та складні агрегації виконуються в базі даних, а бізнес-логіка застосунку, інтеграції та презентаційний рівень реалізуються в клієнті.

Розуміння можливостей та обмежень кожного підходу дозволяє приймати обґрунтовані рішення при проєктуванні інформаційних систем, балансуючи між продуктивністю, гнучкістю розробки та підтримки системи.
