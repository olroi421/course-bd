# Лекція 9. Управління транзакціями та паралельним доступом

## Вступ

Сучасні бази даних одночасно обслуговують сотні або навіть тисячі користувачів, які виконують операції читання та запису даних. Без належних механізмів контролю така одночасна робота призвела б до хаосу: дані втрачалися б, перезаписувалися некоректно, а результати запитів ставали б непередбачуваними. Управління транзакціями та паралельним доступом є фундаментальною частиною будь-якої сучасної СУБД, що забезпечує коректність і надійність роботи з даними в багатокористувацьких середовищах.

У цій лекції ми розглянемо концепцію транзакцій, їхні властивості, рівні ізоляції, механізми контролю паралельності та методи вирішення конфліктних ситуацій. Розуміння цих концепцій критично важливе для розробки надійних інформаційних систем, що працюють під високим навантаженням.

## Концепція транзакції

Транзакція є основною одиницею роботи в системах баз даних. Це логічна одиниця обробки, яка може складатися з однієї або кількох операцій з базою даних.

### Визначення транзакції

Транзакція (англ. transaction) - це послідовність операцій з базою даних, які виконуються як єдине ціле. Транзакція або виконується повністю (фіксується, commit), або повністю скасовується (відкочується, rollback), якщо виникає помилка.

Формально транзакція складається з наступних компонентів:

- BEGIN TRANSACTION - початок транзакції
- Набір операцій читання (READ) та запису (WRITE)
- COMMIT - успішне завершення транзакції
- ROLLBACK - скасування всіх змін транзакції

Приклад простої транзакції переведення коштів між рахунками:

```sql
BEGIN TRANSACTION;

-- Перевірка балансу на рахунку відправника
SELECT balance FROM accounts WHERE account_id = 'ACC001';

-- Зняття коштів з рахунку відправника
UPDATE accounts
SET balance = balance - 1000
WHERE account_id = 'ACC001';

-- Додавання коштів на рахунок отримувача
UPDATE accounts
SET balance = balance + 1000
WHERE account_id = 'ACC002';

-- Реєстрація операції в журналі
INSERT INTO transaction_log (from_account, to_account, amount, timestamp)
VALUES ('ACC001', 'ACC002', 1000, CURRENT_TIMESTAMP);

COMMIT;
```

Якщо на будь-якому етапі виникає помилка (наприклад, недостатньо коштів на рахунку або збій системи), всі зміни скасовуються:

```sql
BEGIN TRANSACTION;

UPDATE accounts
SET balance = balance - 1000
WHERE account_id = 'ACC001';

-- Перевірка достатності коштів
IF (SELECT balance FROM accounts WHERE account_id = 'ACC001') < 0 THEN
    ROLLBACK;
    -- Всі зміни скасовані, баланс повернуто до початкового стану
ELSE
    UPDATE accounts
    SET balance = balance + 1000
    WHERE account_id = 'ACC002';

    COMMIT;
END IF;
```

### Операції транзакції

У контексті управління транзакціями розрізняють два типи операцій:

Операції читання (READ або SELECT):
- Отримання значення з бази даних
- Не змінюють стан бази даних
- Позначаються як read(X), де X - елемент даних

Операції запису (WRITE або UPDATE/INSERT/DELETE):
- Змінюють значення в базі даних
- Впливають на стан системи
- Позначаються як write(X), де X - елемент даних

Приклад нотації операцій транзакції:

```
T1: BEGIN
T1: read(A)
T1: A := A - 100
T1: write(A)
T1: read(B)
T1: B := B + 100
T1: write(B)
T1: COMMIT
```

## ACID властивості транзакцій

ACID - це абревіатура чотирьох ключових властивостей, які повинна гарантувати будь-яка надійна транзакційна система. Ці властивості були вперше сформульовані Джимом Греєм у 1981 році і стали фундаментом теорії транзакцій.

### Атомарність (Atomicity)

Атомарність гарантує, що транзакція виконується повністю або не виконується зовсім. Не може бути часткового виконання транзакції.

Принцип "все або нічого":
- Якщо транзакція завершилася успішно (COMMIT), всі її зміни застосовуються до бази даних
- Якщо транзакція перервана (ROLLBACK або збій), жодна зі змін не застосовується
- Проміжні стани транзакції невидимі іншим користувачам

Приклад порушення атомарності у файловій системі без підтримки транзакцій:

```
Операція 1: Зняти 1000 з рахунку A ✓ (виконано)
Операція 2: Додати 1000 на рахунок B ✗ (збій системи)

Результат: гроші зникли з рахунку A, але не з'явились на рахунку B
```

У системі з підтримкою атомарності:

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 1000 WHERE account_id = 'A';
-- Якщо тут трапляється збій системи...
UPDATE accounts SET balance = balance + 1000 WHERE account_id = 'B';

COMMIT;
-- При відновленні система автоматично відкочує всю транзакцію
-- Баланс рахунку A повернеться до початкового значення
```

Забезпечення атомарності:
- Журнал транзакцій (transaction log) записує всі операції
- При збої система використовує журнал для відкату незавершених транзакцій
- Операція ROLLBACK явно скасовує зміни транзакції

### Консистентність (Consistency)

Консистентність гарантує, що транзакція переводить базу даних з одного коректного стану в інший коректний стан, зберігаючи всі визначені правила цілісності.

Правила цілісності включають:
- Обмеження домену (типи даних, діапазони значень)
- Обмеження ключів (унікальність, обов'язковість)
- Посилальна цілісність (зовнішні ключі)
- Бізнес-правила (наприклад, баланс не може бути від'ємним)

Приклад забезпечення консистентності через обмеження:

```sql
CREATE TABLE accounts (
    account_id VARCHAR(10) PRIMARY KEY,
    customer_id INT NOT NULL,
    balance DECIMAL(15,2) NOT NULL CHECK (balance >= 0),
    account_type VARCHAR(20) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Транзакція, яка порушує консистентність
BEGIN TRANSACTION;

UPDATE accounts
SET balance = balance - 5000
WHERE account_id = 'ACC001' AND balance = 3000;
-- Помилка: порушено обмеження CHECK (balance >= 0)
-- Транзакція автоматично відкочується

ROLLBACK;
```

Приклад бізнес-правила консистентності:

```sql
BEGIN TRANSACTION;

-- Бізнес-правило: сума переведення не може перевищувати денний ліміт
DECLARE @daily_limit DECIMAL(15,2) = 10000;
DECLARE @today_total DECIMAL(15,2);

SELECT @today_total = SUM(amount)
FROM transactions
WHERE from_account = 'ACC001'
  AND DATE(timestamp) = CURRENT_DATE;

IF (@today_total + 5000) > @daily_limit THEN
    ROLLBACK;
    -- Повідомлення: "Перевищено денний ліміт переказів"
ELSE
    -- Виконати переказ
    UPDATE accounts SET balance = balance - 5000 WHERE account_id = 'ACC001';
    UPDATE accounts SET balance = balance + 5000 WHERE account_id = 'ACC002';
    COMMIT;
END IF;
```

### Ізольованість (Isolation)

Ізольованість гарантує, що паралельне виконання транзакцій дає такий самий результат, як і їх послідовне виконання. Кожна транзакція виконується так, ніби вона єдина в системі.

Проблеми відсутності ізоляції:

Втрата оновлення (Lost Update):

```
Час    Транзакція T1              Транзакція T2              Значення X
------ -------------------------- -------------------------- -----------
t1     READ(X)  // X = 100                                   100
t2                                 READ(X)  // X = 100       100
t3     X = X + 50                                            100
t4                                 X = X + 30                100
t5     WRITE(X) // X = 150                                   150
t6                                 WRITE(X) // X = 130       130
t7     COMMIT                                                130
t8                                 COMMIT                    130

Очікуваний результат: 100 + 50 + 30 = 180
Фактичний результат: 130 (втрачено +50)
```

Брудне читання (Dirty Read):

```
Час    Транзакція T1              Транзакція T2              Значення X
------ -------------------------- -------------------------- -----------
t1     READ(X)  // X = 100                                   100
t2     X = X + 50                                            100
t3     WRITE(X) // X = 150                                   150
t4                                 READ(X)  // X = 150       150
t5     ROLLBACK                                              100
t6                                 // Використовує X = 150
t7                                 COMMIT

T2 прочитала значення 150, яке було відкочено
```

Неповторюване читання (Non-repeatable Read):

```
Час    Транзакція T1              Транзакція T2              Значення X
------ -------------------------- -------------------------- -----------
t1     READ(X)  // X = 100                                   100
t2                                 READ(X)  // X = 100       100
t3                                 X = X + 50                100
t4                                 WRITE(X)                  150
t5                                 COMMIT                    150
t6     READ(X)  // X = 150                                   150
t7     // X змінилось під час транзакції!

T1 прочитала X двічі і отримала різні значення
```

Фантомне читання (Phantom Read):

```sql
-- Транзакція T1: підрахунок студентів групи КН-21
BEGIN TRANSACTION;

SELECT COUNT(*) FROM students WHERE group_name = 'КН-21';
-- Результат: 25 студентів

-- Транзакція T2: додає нового студента
BEGIN TRANSACTION;
INSERT INTO students (name, group_name) VALUES ('Іванов', 'КН-21');
COMMIT;

-- Транзакція T1: повторний підрахунок
SELECT COUNT(*) FROM students WHERE group_name = 'КН-21';
-- Результат: 26 студентів (з'явився "фантом")

COMMIT;
```

Забезпечення ізоляції досягається через механізми блокування та управління версіями даних, які ми розглянемо далі.

### Довговічність (Durability)

Довговічність гарантує, що після успішного завершення транзакції (COMMIT) її результати збережуться назавжди, навіть у разі збою системи.

Механізми забезпечення довговічності:

Журнал передзапису (Write-Ahead Logging, WAL):

```
Процес фіксації транзакції:
1. Запис операцій у журнал транзакцій на диск
2. Застосування змін до бази даних (може бути відкладено)
3. Повернення підтвердження користувачу

При відновленні після збою:
1. Читання журналу транзакцій
2. Повторне виконання (REDO) зафіксованих транзакцій
3. Відкат (UNDO) незавершених транзакцій
```

Приклад запису в журнал транзакцій:

```
[LSN: 1001] BEGIN TRANSACTION T1
[LSN: 1002] T1: OLD_VALUE(accounts.ACC001.balance) = 5000
[LSN: 1003] T1: NEW_VALUE(accounts.ACC001.balance) = 4000
[LSN: 1004] T1: OLD_VALUE(accounts.ACC002.balance) = 2000
[LSN: 1005] T1: NEW_VALUE(accounts.ACC002.balance) = 3000
[LSN: 1006] COMMIT TRANSACTION T1
```

При збої системи між записами LSN 1005 та 1006:

```
Відновлення:
- T1 не має запису COMMIT
- Система виконує UNDO, повертаючи OLD_VALUE для всіх змін T1
- Баланси повертаються: ACC001 = 5000, ACC002 = 2000
```

Контрольні точки (Checkpoints):

```
Періодично СУБД створює контрольну точку:
1. Запис всіх змін з буферів у базу даних на диск
2. Запис інформації про активні транзакції
3. Позначка в журналі: CHECKPOINT

При відновленні:
- Система починає з останньої контрольної точки
- Обробляє лише транзакції після цієї точки
- Значно скорочує час відновлення
```

Приклад використання контрольної точки:

```
Часова шкала журналу:
[CP: 5000] CHECKPOINT
[LSN: 5001] BEGIN T1
[LSN: 5002] T1: UPDATE accounts...
[LSN: 5003] BEGIN T2
[LSN: 5004] T2: INSERT INTO transactions...
[LSN: 5005] COMMIT T1
[LSN: 5006] BEGIN T3
[LSN: 5007] T3: UPDATE accounts...
[ЗБІЙ СИСТЕМИ]

Відновлення:
1. Знайти останню контрольну точку (CP: 5000)
2. Активні транзакції: T1 (завершена), T2 (не завершена), T3 (не завершена)
3. REDO T1 (зафіксована після контрольної точки)
4. UNDO T2 та T3 (не зафіксовані)
```

## Рівні ізоляції транзакцій

Стандарт SQL визначає чотири рівні ізоляції транзакцій, які представляють компроміс між коректністю та продуктивністю. Чим вищий рівень ізоляції, тим більше гарантій коректності, але нижча паралельність та продуктивність системи.

### Огляд рівнів ізоляції

Рівні ізоляції та проблеми, які вони дозволяють або запобігають:

| Рівень ізоляції | Брудне читання | Неповторюване читання | Фантомне читання |
|-----------------|----------------|-----------------------|------------------|
| READ UNCOMMITTED | Можливо | Можливо | Можливо |
| READ COMMITTED | Неможливо | Можливо | Можливо |
| REPEATABLE READ | Неможливо | Неможливо | Можливо |
| SERIALIZABLE | Неможливо | Неможливо | Неможливо |

### READ UNCOMMITTED

Найнижчий рівень ізоляції, який не гарантує захисту від жодної з проблем паралельного доступу.

Характеристики:
- Транзакції можуть читати незафіксовані зміни інших транзакцій
- Відсутні блокування на читання
- Максимальна продуктивність, мінімальна коректність

Приклад використання:

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

BEGIN TRANSACTION;

-- Може прочитати незафіксовані дані інших транзакцій
SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Значення може бути з незавершеної транзакції!

COMMIT;
```

Сценарій брудного читання:

```sql
-- Сесія 1
BEGIN TRANSACTION;
UPDATE products SET price = 100 WHERE product_id = 'P001';
-- Транзакція не завершена, зміна не зафіксована

-- Сесія 2 (READ UNCOMMITTED)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN TRANSACTION;
SELECT price FROM products WHERE product_id = 'P001';
-- Повертає 100 (незафіксоване значення)
COMMIT;

-- Сесія 1
ROLLBACK; -- Скасування зміни
-- Тепер price знову 90, але Сесія 2 вже використала значення 100!
```

Використання READ UNCOMMITTED:
- Звіти, де не критична точність даних
- Моніторинг системи
- Аналітичні запити на копіях баз даних

### READ COMMITTED

Стандартний рівень ізоляції в більшості СУБД (PostgreSQL, Oracle, SQL Server).

Характеристики:
- Транзакція бачить лише зафіксовані зміни
- Запобігає брудному читанню
- Дозволяє неповторюване та фантомне читання
- Блокування на запис, але не на читання

Приклад використання:

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

BEGIN TRANSACTION;

-- Бачить лише зафіксовані дані
SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Результат: 5000

-- Інша транзакція змінює та фіксує баланс до 6000

-- Повторне читання
SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Результат: 6000 (неповторюване читання)

COMMIT;
```

Запобігання брудному читанню:

```sql
-- Сесія 1
BEGIN TRANSACTION;
UPDATE products SET price = 100 WHERE product_id = 'P001';
-- Зміна не зафіксована

-- Сесія 2 (READ COMMITTED)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION;
SELECT price FROM products WHERE product_id = 'P001';
-- Очікує на фіксацію Сесії 1 або повертає старе значення 90
COMMIT;

-- Сесія 1
COMMIT; -- Тепер Сесія 2 може побачити нове значення 100
```

Неповторюване читання при READ COMMITTED:

```sql
-- Транзакція T1
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION;

SELECT salary FROM employees WHERE employee_id = 'E001';
-- Результат: 50000

-- Виконується деяка бізнес-логіка...
-- В цей час інша транзакція змінює зарплату на 55000 і фіксує

SELECT salary FROM employees WHERE employee_id = 'E001';
-- Результат: 55000 (значення змінилось!)

-- Якщо бізнес-логіка залежить від стабільності значення, це проблема
COMMIT;
```

### REPEATABLE READ

Забезпечує консистентне читання протягом всієї транзакції.

Характеристики:
- Запобігає брудному та неповторюваному читанню
- Дозволяє фантомне читання
- Блокування на читання та запис
- Транзакція бачить снапшот даних на момент свого початку

Приклад використання:

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN TRANSACTION;

SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Результат: 5000

-- Інша транзакція намагається змінити баланс
-- Блокується до завершення поточної транзакції

SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Результат: 5000 (те саме значення, повторюване читання)

COMMIT;
```

Запобігання неповторюваному читанню:

```sql
-- Сесія 1 (REPEATABLE READ)
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;

SELECT price FROM products WHERE product_id = 'P001';
-- Результат: 90

-- Сесія 2 намагається оновити
BEGIN TRANSACTION;
UPDATE products SET price = 100 WHERE product_id = 'P001';
-- ОЧІКУЄ завершення Сесії 1 (блокується)

-- Сесія 1 продовжує
SELECT price FROM products WHERE product_id = 'P001';
-- Результат: 90 (стабільне значення)

COMMIT;
-- Тепер Сесія 2 може виконати оновлення
```

Фантомне читання при REPEATABLE READ:

```sql
-- Транзакція T1 (REPEATABLE READ)
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;

SELECT COUNT(*) FROM orders WHERE customer_id = 'C001';
-- Результат: 5 замовлень

-- Транзакція T2 вставляє нове замовлення
BEGIN TRANSACTION;
INSERT INTO orders (customer_id, order_date, amount)
VALUES ('C001', CURRENT_DATE, 500);
COMMIT;

-- Транзакція T1 повторює запит
SELECT COUNT(*) FROM orders WHERE customer_id = 'C001';
-- Результат може бути 6 (фантомний запис)
-- Залежить від конкретної реалізації СУБД

COMMIT;
```

### SERIALIZABLE

Найвищий рівень ізоляції, який гарантує повну ізоляцію транзакцій.

Характеристики:
- Запобігає всім проблемам паралельності
- Транзакції виконуються так, ніби вони послідовні
- Найнижча продуктивність
- Максимальна коректність

Приклад використання:

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN TRANSACTION;

-- Всі операції повністю ізольовані
SELECT SUM(amount) FROM orders WHERE customer_id = 'C001';

-- Жодна інша транзакція не може змінити відповідні дані
-- до завершення поточної транзакції

UPDATE customer_statistics
SET total_orders = (SELECT SUM(amount) FROM orders WHERE customer_id = 'C001')
WHERE customer_id = 'C001';

COMMIT;
```

Запобігання фантомному читанню:

```sql
-- Транзакція T1 (SERIALIZABLE)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;

SELECT COUNT(*) FROM products WHERE category = 'Electronics';
-- Результат: 50

-- Транзакція T2 намагається вставити новий товар
BEGIN TRANSACTION;
INSERT INTO products (name, category, price)
VALUES ('Laptop', 'Electronics', 1000);
-- БЛОКУЄТЬСЯ або призводить до помилки серіалізації

-- Транзакція T1 завершує роботу
SELECT COUNT(*) FROM products WHERE category = 'Electronics';
-- Результат: 50 (те саме значення, фантоми неможливі)

COMMIT;

-- Тепер T2 може виконати вставку
```

Конфлікт серіалізації:

```sql
-- Транзакція T1
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Баланс: 1000

-- Транзакція T2
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Баланс: 1000

-- T1 намагається оновити
UPDATE accounts SET balance = balance - 200 WHERE account_id = 'ACC001';
COMMIT;

-- T2 намагається оновити
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'ACC001';
-- ПОМИЛКА: serialization failure
-- Потрібно повторити транзакцію T2
```

Вибір рівня ізоляції залежить від вимог конкретного застосунку:
- READ COMMITTED - для більшості вебзастосунків
- REPEATABLE READ - для фінансових операцій середньої критичності
- SERIALIZABLE - для критичних фінансових транзакцій, інвентаризації

## Протоколи управління паралельністю

Для забезпечення ізоляції транзакцій СУБД використовують різні протоколи управління паралельністю. Розглянемо три основні підходи: блокування, позначки часу та валідацію.

### Протокол блокування

Блокування є найпоширенішим механізмом контролю паралельності. Транзакція отримує блокування на об'єкт даних перед доступом до нього.

Типи блокувань:

Спільне блокування (Shared Lock, S-lock):
- Використовується для операцій читання
- Кілька транзакцій можуть одночасно мати S-lock на об'єкт
- Запобігає отриманню X-lock іншими транзакціями

Виключне блокування (Exclusive Lock, X-lock):
- Використовується для операцій запису
- Лише одна транзакція може мати X-lock на об'єкт
- Запобігає отриманню будь-яких блокувань іншими транзакціями

Матриця сумісності блокувань:

```
        | S-lock | X-lock |
--------|--------|--------|
S-lock  |   ✓    |   ✗    |
X-lock  |   ✗    |   ✗    |

✓ - сумісно (дозволено)
✗ - несумісно (заблоковано)
```

Приклад використання блокувань:

```sql
-- Транзакція T1
BEGIN TRANSACTION;

-- Отримання S-lock на запис
SELECT balance FROM accounts WHERE account_id = 'ACC001';

-- Інша транзакція може також отримати S-lock
-- але не може отримати X-lock

COMMIT; -- Звільнення S-lock
```

```sql
-- Транзакція T2
BEGIN TRANSACTION;

-- Отримання X-lock на запис
UPDATE accounts
SET balance = balance + 1000
WHERE account_id = 'ACC001';

-- Жодна інша транзакція не може отримати жодного блокування
-- до завершення T2

COMMIT; -- Звільнення X-lock
```

Двофазний протокол блокування (Two-Phase Locking, 2PL):

Протокол 2PL гарантує серіалізовність виконання транзакцій і складається з двох фаз:

Фаза зростання (Growing Phase):
- Транзакція може отримувати нові блокування
- Транзакція не може звільняти блокування
- Триває від початку транзакції до першого звільнення блокування

Фаза скорочення (Shrinking Phase):
- Транзакція може звільняти блокування
- Транзакція не може отримувати нові блокування
- Триває від першого звільнення до кінця транзакції

Приклад 2PL:

```
Транзакція T1:
BEGIN
    LOCK-S(A)      // Фаза зростання
    READ(A)
    LOCK-X(B)      // Фаза зростання
    READ(B)
    B = B + A
    WRITE(B)
    UNLOCK(A)      // Фаза скорочення починається
    UNLOCK(B)      // Фаза скорочення
COMMIT
```

Строгий двофазний протокол (Strict 2PL):
- Всі виключні блокування звільняються лише при COMMIT або ROLLBACK
- Запобігає каскадним відкатам
- Використовується в більшості комерційних СУБД

Приклад Strict 2PL:

```
Транзакція T1:
BEGIN
    LOCK-S(A)
    READ(A)
    LOCK-X(B)
    READ(B)
    B = B + A
    WRITE(B)
    // Блокування НЕ звільняються
COMMIT           // ВСІ блокування звільняються тут
```

Гранулярність блокування:

СУБД може блокувати об'єкти на різних рівнях:

```
База даних
    ├── Таблиця
    │   ├── Сторінка (Page)
    │   │   └── Запис (Row)
    │   └── Індекс
```

Ієрархія блокувань:

```sql
-- Блокування на рівні рядка (найдрібніша гранулярність)
BEGIN TRANSACTION;
SELECT * FROM accounts WHERE account_id = 'ACC001' FOR UPDATE;
-- Блокується лише один рядок
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'ACC001';
COMMIT;

-- Блокування на рівні таблиці (найгрубіша гранулярність)
BEGIN TRANSACTION;
LOCK TABLE accounts IN EXCLUSIVE MODE;
-- Блокується вся таблиця
UPDATE accounts SET balance = balance * 1.05;
COMMIT;
```

Протокол інтенційних блокувань (Intention Locks):

Для ефективної роботи з ієрархією об'єктів використовуються інтенційні блокування:

- IS (Intention Shared) - намір отримати S-lock на нижчому рівні
- IX (Intention Exclusive) - намір отримати X-lock на нижчому рівні
- SIX (Shared + Intention Exclusive) - S-lock на поточному рівні та намір X-lock на нижчому

Матриця сумісності інтенційних блокувань:

```
      | IS  | IX  | S   | SIX | X   |
------|-----|-----|-----|-----|-----|
IS    |  ✓  |  ✓  |  ✓  |  ✓  |  ✗  |
IX    |  ✓  |  ✓  |  ✗  |  ✗  |  ✗  |
S     |  ✓  |  ✗  |  ✓  |  ✗  |  ✗  |
SIX   |  ✓  |  ✗  |  ✗  |  ✗  |  ✗  |
X     |  ✗  |  ✗  |  ✗  |  ✗  |  ✗  |
```

Приклад використання інтенційних блокувань:

```
Транзакція хоче оновити рядок у таблиці:
1. Отримати IS-lock на базу даних
2. Отримати IX-lock на таблицю
3. Отримати X-lock на рядок
4. Виконати оновлення
5. Звільнити блокування при COMMIT
```

### Протокол позначок часу

Альтернативний підхід до управління паралельністю без використання блокувань. Кожній транзакції призначається унікальна позначка часу.

Основні принципи:

Кожна транзакція T отримує позначку часу TS(T):
- TS(T1) < TS(T2) означає, що T1 розпочалась раніше T2
- Система гарантує, що результат відповідає послідовному виконанню в порядку позначок часу

Для кожного елемента даних X зберігаються:
- read_TS(X) - позначка часу останньої транзакції, яка читала X
- write_TS(X) - позначка часу останньої транзакції, яка записувала X

Правила протоколу позначок часу:

Операція READ(X) від транзакції T з TS(T):

```
Якщо TS(T) < write_TS(X):
    // T намагається читати застаріле значення
    ROLLBACK T
Інакше:
    Виконати READ(X)
    read_TS(X) = max(read_TS(X), TS(T))
```

Операція WRITE(X) від транзакції T з TS(T):

```
Якщо TS(T) < read_TS(X):
    // Інша транзакція вже прочитала більш пізнє значення
    ROLLBACK T
Якщо TS(T) < write_TS(X):
    // Запис застарів, ігноруємо (Thomas Write Rule)
    Пропустити запис
Інакше:
    Виконати WRITE(X)
    write_TS(X) = TS(T)
```

Приклад роботи протоколу позначок часу:

```
Стан системи:
X = 50, read_TS(X) = 10, write_TS(X) = 5

Транзакція T1 з TS(T1) = 15:
READ(X):
    TS(15) >= write_TS(5) ✓
    Виконується: прочитано X = 50
    read_TS(X) = 15

Транзакція T2 з TS(T2) = 12:
WRITE(X, нове_значення=60):
    TS(12) < read_TS(15) ✗
    T2 намагається записати значення, яке вже було прочитано пізнішою транзакцією
    ROLLBACK T2

Транзакція T3 з TS(T3) = 20:
WRITE(X, нове_значення=70):
    TS(20) >= read_TS(15) ✓
    TS(20) >= write_TS(5) ✓
    Виконується: X = 70
    write_TS(X) = 20
```

Багатоверсійний протокол позначок часу (Multiversion Timestamp Ordering):

Замість зберігання одного значення, система зберігає кілька версій:

```
Для кожного X зберігаються версії:
X₁ з write_TS(X₁) = 5
X₂ з write_TS(X₂) = 10
X₃ з write_TS(X₃) = 15

Транзакція T з TS(T) = 12 читає X:
Вибирається версія X₂ (write_TS = 10 <= 12 < 15)
Транзакція бачить консистентний знімок даних
```

Переваги протоколу позначок часу:
- Відсутність взаємоблокувань
- Операції читання ніколи не чекають
- Підходить для розподілених систем

Недоліки:
- Більше відкатів порівняно з блокуванням
- Накладні витрати на управління версіями
- Складність реалізації

### Протокол оптимістичної валідації

Оптимістичний підхід припускає, що конфлікти між транзакціями трапляються рідко. Транзакції виконуються без контролю, а перевірка конфліктів відбувається перед фіксацією.

Три фази виконання транзакції:

Фаза читання (Read Phase):
- Транзакція читає дані з бази
- Всі зміни записуються у локальний буфер (не в базу даних)
- Створюється read_set та write_set транзакції

Фаза валідації (Validation Phase):
- Перевірка конфліктів з іншими транзакціями
- Якщо конфліктів немає - перехід до фази запису
- Якщо є конфлікти - ROLLBACK

Фаза запису (Write Phase):
- Застосування змін з локального буфера до бази даних
- COMMIT транзакції

Правила валідації:

Для транзакції Ti, що проходить валідацію, та транзакції Tj, що вже завершилась:

```
Умова 1: Ti завершує читання до початку запису Tj
    finish(Ti_read) < start(Tj_write) ✓

Умова 2: Ti починає запис після завершення Tj
    start(Ti_write) > finish(Tj_write) ✓

Умова 3: Множини не перетинаються
    write_set(Tj) ∩ read_set(Ti) = ∅ ✓
    write_set(Tj) ∩ write_set(Ti) = ∅ ✓
```

Приклад оптимістичної валідації:

```sql
-- Транзакція T1 (TS = 10)
BEGIN TRANSACTION;
-- Фаза читання
SELECT balance FROM accounts WHERE account_id = 'ACC001'; -- Баланс: 1000
-- read_set = {accounts.ACC001}

-- Локальні обчислення
local_balance = 1000 - 200; -- = 800

-- Зміна у локальному буфері (НЕ в БД)
-- write_set = {accounts.ACC001}

-- Фаза валідації
VALIDATE:
    Перевірка конфліктів з завершеними транзакціями
    Немає конфліктів ✓

-- Фаза запису
UPDATE accounts SET balance = 800 WHERE account_id = 'ACC001';
COMMIT;
```

Приклад конфлікту при валідації:

```
Транзакція T1 (TS = 10):
READ(X) = 100
write_set(T1) = {X}

Транзакція T2 (TS = 15):
READ(X) = 100
WRITE(X) = 150
COMMIT успішний

Транзакція T1 продовжує:
WRITE(X) = 120
VALIDATE:
    write_set(T2) = {X}
    write_set(T1) = {X}
    write_set(T2) ∩ write_set(T1) = {X} ≠ ∅
    КОНФЛІКТ! ROLLBACK T1
```

Застосування оптимістичної валідації:

```sql
-- Типовий сценарій для вебзастосунків
-- Транзакція редагування профілю користувача

BEGIN TRANSACTION;

-- Фаза читання: отримання даних для форми редагування
SELECT name, email, phone FROM users WHERE user_id = 123;
-- Користувач отримує форму, заповнює її (можливо, кілька хвилин)

-- Фаза валідації + запису: збереження змін
-- Перевірка, чи не змінив хтось інший ці дані
IF (SELECT version FROM users WHERE user_id = 123) = original_version THEN
    UPDATE users
    SET name = 'Нове ім''я',
        email = 'new@email.com',
        version = version + 1
    WHERE user_id = 123;
    COMMIT;
ELSE
    ROLLBACK;
    -- Повідомлення: "Дані були змінені іншим користувачем"
END IF;
```

Переваги оптимістичної валідації:
- Відсутність блокувань під час читання
- Високий рівень паралельності при низькій конфліктності
- Ефективність для транзакцій, що переважно читають

Недоліки:
- Високі накладні витрати при частих конфліктах
- Потреба у повторному виконанні відкочених транзакцій
- Складність реалізації для складних транзакцій

## Виявлення та вирішення взаємоблокувань

Взаємоблокування (deadlock) виникає, коли кілька транзакцій чекають одна на одну у циклічній залежності, і жодна не може продовжити виконання.

### Класичний приклад взаємоблокування

```
Транзакція T1:                  Транзакція T2:
LOCK-X(A) ✓                     LOCK-X(B) ✓
READ(A)                         READ(B)
A = A - 100                     B = B - 200
LOCK-X(B) ⏳ чекає             LOCK-X(A) ⏳ чекає
...                             ...

Граф очікування:
T1 → T2
T2 → T1
(цикл = взаємоблокування)
```

Детальний сценарій взаємоблокування:

```sql
-- Час t1: Транзакція T1
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'ACC001';
-- T1 отримує X-lock на рядок ACC001

-- Час t2: Транзакція T2
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 200 WHERE account_id = 'ACC002';
-- T2 отримує X-lock на рядок ACC002

-- Час t3: T1 продовжує
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'ACC002';
-- T1 чекає на X-lock для ACC002 (утримується T2)

-- Час t4: T2 продовжує
UPDATE accounts SET balance = balance + 200 WHERE account_id = 'ACC001';
-- T2 чекає на X-lock для ACC001 (утримується T1)

-- Взаємоблокування: T1 чекає T2, T2 чекає T1
```

### Методи боротьби з взаємоблокуванням

Існує три основні підходи до боротьби з взаємоблокуванням: запобігання, уникнення та виявлення.

Запобігання взаємоблокуванню:

Упорядкування ресурсів:

```sql
-- Правило: завжди блокувати рахунки в порядку зростання account_id

-- Правильна реалізація переказу
CREATE PROCEDURE transfer_money(from_acc VARCHAR(10), to_acc VARCHAR(10), amount DECIMAL)
BEGIN
    DECLARE first_acc VARCHAR(10);
    DECLARE second_acc VARCHAR(10);

    -- Визначення порядку блокування
    IF from_acc < to_acc THEN
        SET first_acc = from_acc;
        SET second_acc = to_acc;
    ELSE
        SET first_acc = to_acc;
        SET second_acc = from_acc;
    END IF;

    BEGIN TRANSACTION;

    -- Блокування в упорядкованому порядку
    SELECT balance FROM accounts WHERE account_id = first_acc FOR UPDATE;
    SELECT balance FROM accounts WHERE account_id = second_acc FOR UPDATE;

    -- Виконання переказу
    UPDATE accounts SET balance = balance - amount WHERE account_id = from_acc;
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_acc;

    COMMIT;
END;
```

Отримання всіх блокувань одночасно:

```sql
BEGIN TRANSACTION;

-- Отримати всі необхідні блокування відразу
LOCK TABLE accounts IN EXCLUSIVE MODE;
LOCK TABLE transaction_log IN EXCLUSIVE MODE;

-- Виконати всі операції
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 'ACC001';
UPDATE accounts SET balance = balance + 1000 WHERE account_id = 'ACC002';
INSERT INTO transaction_log VALUES (...);

COMMIT;
```

Тайм-аут на блокування:

```sql
-- PostgreSQL
SET lock_timeout = '5s';

BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 'ACC001';
-- Якщо блокування не отримано протягом 5 секунд - помилка
UPDATE accounts SET balance = balance + 1000 WHERE account_id = 'ACC002';
COMMIT;
```

Уникнення взаємоблокування за часовими позначками:

Wait-Die схема (старіші транзакції мають пріоритет):

```
Якщо Ti запитує ресурс, утримуваний Tj:
    Якщо TS(Ti) < TS(Tj):  // Ti старіша
        Ti чекає (wait)
    Інакше:                 // Ti молодша
        Ti відкочується (die)
```

Wound-Wait схема (молодші транзакції мають пріоритет):

```
Якщо Ti запитує ресурс, утримуваний Tj:
    Якщо TS(Ti) < TS(Tj):  // Ti старіша
        Tj відкочується (wound)
    Інакше:                 // Ti молодша
        Ti чекає (wait)
```

Виявлення взаємоблокування:

Граф очікування (Wait-For Graph):

```
Вузли: активні транзакції
Ребра: Ti → Tj означає "Ti чекає на ресурс від Tj"

Взаємоблокування існує ⟺ У графі є цикл
```

Приклад побудови графу очікування:

```
Стан блокувань:
T1 утримує X-lock(A), чекає X-lock(B)
T2 утримує X-lock(B), чекає X-lock(C)
T3 утримує X-lock(C), чекає X-lock(A)

Граф очікування:
T1 → T2 (T1 чекає на B від T2)
T2 → T3 (T2 чекає на C від T3)
T3 → T1 (T3 чекає на A від T1)

Виявлено цикл: T1 → T2 → T3 → T1
DEADLOCK!
```

Алгоритм виявлення циклів:

```python
def detect_deadlock(wait_for_graph):
    """
    Виявлення циклів у графі очікування
    """
    def has_cycle(node, visited, rec_stack):
        visited[node] = True
        rec_stack[node] = True

        for neighbor in wait_for_graph[node]:
            if not visited[neighbor]:
                if has_cycle(neighbor, visited, rec_stack):
                    return True
            elif rec_stack[neighbor]:
                return True

        rec_stack[node] = False
        return False

    visited = {node: False for node in wait_for_graph}
    rec_stack = {node: False for node in wait_for_graph}

    for node in wait_for_graph:
        if not visited[node]:
            if has_cycle(node, visited, rec_stack):
                return True

    return False
```

Вибір транзакції для відкату:

Критерії вибору жертви:
- Вік транзакції (відкотити найновішу)
- Кількість виконаної роботи (відкотити ту, що зробила менше)
- Кількість блокувань (відкотити ту, що утримує більше)
- Кількість попередніх відкатів (уникати голодування)

```python
def select_victim(deadlock_cycle):
    """
    Вибір транзакції для відкату
    """
    min_cost = float('inf')
    victim = None

    for transaction in deadlock_cycle:
        cost = calculate_rollback_cost(transaction)
        # Враховуємо:
        # - час виконання
        # - кількість змін
        # - кількість попередніх відкатів

        if cost < min_cost:
            min_cost = cost
            victim = transaction

    return victim

def calculate_rollback_cost(transaction):
    """
    Обчислення вартості відкату транзакції
    """
    execution_time = transaction.current_time - transaction.start_time
    num_updates = len(transaction.write_set)
    rollback_count = transaction.previous_rollbacks

    # Формула вартості (приклад)
    cost = execution_time * 0.5 + num_updates * 10 + rollback_count * 100

    return cost
```

Приклад роботи детектора взаємоблокувань у PostgreSQL:

```sql
-- Перегляд поточних блокувань
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
JOIN pg_catalog.pg_stat_activity blocking_activity
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

Параметри виявлення взаємоблокувань у різних СУБД:

```sql
-- PostgreSQL
SHOW deadlock_timeout; -- За замовчуванням 1 секунда

-- MySQL/InnoDB
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout'; -- За замовчуванням 50 секунд

-- SQL Server
-- Автоматичне виявлення кожні 5 секунд
```

## Багатоверсійний контроль паралельності

Багатоверсійний контроль паралельності (Multiversion Concurrency Control, MVCC) - це сучасний підхід до управління паралельністю, який зберігає кілька версій кожного об'єкта даних.

### Основні принципи MVCC

Замість блокування читання, MVCC дозволяє транзакціям читати старі версії даних:

```
Традиційний підхід (блокування):
T1: WRITE(X) → X заблоковано
T2: READ(X) → чекає на T1

MVCC підхід:
T1: WRITE(X) → створює нову версію X₂
T2: READ(X) → читає стару версію X₁
```

Структура версій:

```
Кожна версія об'єкта X містить:
- Значення даних
- Позначку часу створення (створено транзакцією T_create)
- Позначку часу видалення (видалено транзакцією T_delete)
- Покажчик на попередню/наступну версію
```

Приклад ланцюга версій:

```
X₁: value=100, created=T1, deleted=T3, next→X₂
X₂: value=150, created=T3, deleted=T5, next→X₃
X₃: value=200, created=T5, deleted=NULL, next→NULL
```

### Правила читання версій

Транзакція T з позначкою часу TS(T) може прочитати версію Xᵢ, якщо:

```
1. created(Xᵢ) <= TS(T)
   // Версія створена до або в момент початку T

2. deleted(Xᵢ) > TS(T) АБО deleted(Xᵢ) = NULL
   // Версія не видалена або видалена після початку T
```

Приклад вибору версії для читання:

```
Версії об'єкта X:
X₁: value=100, created=5, deleted=15
X₂: value=150, created=15, deleted=25
X₃: value=200, created=25, deleted=NULL

Транзакція T з TS(T) = 20:
Перевірка X₁: created(5) <= 20 ✓, deleted(15) <= 20 ✗ → НЕ ПІДХОДИТЬ
Перевірка X₂: created(15) <= 20 ✓, deleted(25) > 20 ✓ → ПІДХОДИТЬ
Результат: T читає X₂ = 150
```

### Снапшот ізоляція

Снапшот ізоляція (Snapshot Isolation) - найпоширеніша реалізація MVCC.

Принципи снапшот ізоляції:

```
При початку транзакції T:
1. Фіксується поточний стан бази даних (снапшот)
2. T бачить лише дані, зафіксовані до моменту її початку
3. T не бачить змін інших паралельних транзакцій
4. При COMMIT перевіряються конфлікти з іншими транзакціями
```

Приклад снапшот ізоляції в PostgreSQL:

```sql
-- Транзакція T1 (розпочата в момент часу 10)
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- PostgreSQL використовує MVCC для REPEATABLE READ

SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Результат: 1000 (значення на момент часу 10)

-- Транзакція T2 (розпочата в момент часу 15)
BEGIN TRANSACTION;
UPDATE accounts SET balance = 2000 WHERE account_id = 'ACC001';
COMMIT;
-- Зафіксовано в момент часу 20

-- Транзакція T1 продовжує
SELECT balance FROM accounts WHERE account_id = 'ACC001';
-- Результат: 1000 (все ще бачить снапшот на момент часу 10)

COMMIT;
```

Внутрішня реалізація снапшот ізоляції:

```
PostgreSQL використовує номери транзакцій (XID):

Кожен кортеж має:
- xmin: XID транзакції, яка створила кортеж
- xmax: XID транзакції, яка видалила кортеж (NULL якщо активний)

Транзакція T з XID = 150 бачить кортеж, якщо:
1. xmin < 150 AND (xmax = NULL OR xmax >= 150)
   // Створений раніше і не видалений або видалений пізніше
2. xmin зафіксована (COMMITTED)
3. xmax не зафіксована АБО xmax = NULL
```

Приклад структури кортежу в PostgreSQL:

```
Таблиця accounts після кількох оновлень:

Кортеж 1 (старий):
account_id = 'ACC001'
balance = 1000
xmin = 100 (створений транзакцією 100)
xmax = 150 (видалений транзакцією 150)
ctid = (0,1) → вказує на (0,2)

Кортеж 2 (новий):
account_id = 'ACC001'
balance = 1500
xmin = 150 (створений транзакцією 150)
xmax = NULL (активний)
ctid = (0,2)

Транзакція з XID = 140:
Читає кортеж 1 (xmin=100 < 140, xmax=150 > 140)

Транзакція з XID = 160:
Читає кортеж 2 (xmin=150 < 160, xmax=NULL)
```

### Проблема аномалії запису

Снапшот ізоляція не запобігає аномалії запису (write skew):

```sql
-- Правило: сума балансів двох рахунків >= 0

-- Транзакція T1
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE account_id IN ('ACC001', 'ACC002');
-- Результат: ACC001 = 100, ACC002 = 100 (сума = 200)

-- T1 перевіряє умову і знімає з ACC001
UPDATE accounts SET balance = balance - 150 WHERE account_id = 'ACC001';
-- ACC001 = -50, але ACC001 + ACC002 = 50 > 0 ✓

-- Транзакція T2 (паралельно)
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE account_id IN ('ACC001', 'ACC002');
-- Результат: ACC001 = 100, ACC002 = 100 (сума = 200)

-- T2 перевіряє умову і знімає з ACC002
UPDATE accounts SET balance = balance - 150 WHERE account_id = 'ACC002';
-- ACC002 = -50, але ACC001 + ACC002 = 50 > 0 ✓

COMMIT; -- T1
COMMIT; -- T2

-- Фінальний стан: ACC001 = -50, ACC002 = -50
-- Сума = -100 < 0 ✗ ПОРУШЕНО ПРАВИЛО!
```

Вирішення через серіалізовний рівень ізоляції:

```sql
-- PostgreSQL Serializable Snapshot Isolation (SSI)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT balance FROM accounts WHERE account_id IN ('ACC001', 'ACC002');

UPDATE accounts SET balance = balance - 150 WHERE account_id = 'ACC001';

COMMIT;
-- PostgreSQL виявляє конфлікт серіалізації
-- ERROR: could not serialize access due to read/write dependencies
```

### Очищення старих версій

MVCC створює багато застарілих версій, які потрібно періодично видаляти.

Процес очищення у PostgreSQL (VACUUM):

```sql
-- Ручне очищення
VACUUM accounts;

-- Аналіз та очищення з статистикою
VACUUM ANALYZE accounts;

-- Повне очищення з блокуванням таблиці
VACUUM FULL accounts;

-- Автоматичне очищення (autovacuum)
SHOW autovacuum;
```

Критерії видалення версій:

```
Версія Xᵢ може бути видалена, якщо:
1. deleted(Xᵢ) != NULL
   // Версія помічена як видалена
2. deleted(Xᵢ) < min(TS всіх активних транзакцій)
   // Жодна активна транзакція не може її побачити
3. Транзакція, що видалила Xᵢ, зафіксована
   // Видалення завершено успішно
```

Приклад процесу очищення:

```
Стан до очищення:
X₁: value=100, created=5, deleted=15, зафіксовано
X₂: value=150, created=15, deleted=25, зафіксовано
X₃: value=200, created=25, deleted=NULL

Активні транзакції: T₁(TS=30), T₂(TS=35)
min(TS) = 30

Аналіз версій:
X₁: deleted=15 < 30 ✓ → МОЖНА ВИДАЛИТИ
X₂: deleted=25 < 30 ✓ → МОЖНА ВИДАЛИТИ
X₃: deleted=NULL ✗ → ЗАЛИШАЄТЬСЯ

Стан після очищення:
X₃: value=200, created=25, deleted=NULL
```

Параметри автоматичного очищення:

```sql
-- PostgreSQL
ALTER TABLE accounts SET (
    autovacuum_vacuum_threshold = 50,
    autovacuum_vacuum_scale_factor = 0.2,
    autovacuum_vacuum_cost_delay = 20
);

-- Формула запуску autovacuum:
-- vacuum_threshold + vacuum_scale_factor * число_кортежів
```

### Переваги та недоліки MVCC

Переваги MVCC:

Висока паралельність:
- Читання ніколи не блокує запис
- Запис ніколи не блокує читання
- Кілька транзакцій можуть працювати одночасно

```sql
-- Традиційне блокування
T1: UPDATE accounts SET balance = 1000; -- Блокує таблицю
T2: SELECT * FROM accounts; -- ЧЕКАЄ на T1

-- MVCC
T1: UPDATE accounts SET balance = 1000; -- Створює нову версію
T2: SELECT * FROM accounts; -- Читає стару версію, НЕ чекає
```

Консистентність читання:
- Транзакція бачить консистентний снапшот
- Відсутність фантомних читань (на рівні REPEATABLE READ)

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT COUNT(*) FROM products; -- 100 товарів

-- Інша транзакція додає 10 товарів і фіксує

SELECT COUNT(*) FROM products; -- Все одно 100 товарів

COMMIT;
```

Швидке відновлення:
- Відкат транзакції не потребує фізичного відкату змін
- Достатньо не робити версії видимими

```
Відкат транзакції T:
1. Позначити всі версії з created=T як "відкочені"
2. Жодних фізичних змін у базі даних
3. Версії будуть видалені при наступному VACUUM
```

Недоліки MVCC:

Накладні витрати на зберігання:

```
Приклад росту таблиці:
Початковий розмір: 1 GB (1 млн рядків)

Після 1000 оновлень кожного рядка:
Фізичний розмір: ~1000 GB (1 млрд версій)
Активні дані: 1 GB
"Мертві" версії: 999 GB

Необхідне регулярне очищення!
```

Вартість очищення:

```sql
-- VACUUM може бути ресурсовитратним
-- Впливає на продуктивність в пікові години

-- Моніторинг необхідності VACUUM
SELECT
    schemaname,
    tablename,
    n_dead_tup,
    n_live_tup,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_ratio DESC;
```

Проблема роздування індексів:

```
Індекси також зберігають вказівники на всі версії:

Індекс на balance:
100 → [версія₁, версія₂, версія₃, ..., версіяₙ]

Після VACUUM:
100 → [поточна_версія]

Але індекс не зменшується автоматично!
Потрібен REINDEX для повного звільнення простору.
```

```sql
-- Перебудова індексу
REINDEX INDEX accounts_balance_idx;

-- Перебудова всіх індексів таблиці
REINDEX TABLE accounts;
```

### Порівняння MVCC з блокуванням

Сценарій 1: Інтенсивне читання з рідкісним записом

```
Блокування:
- Запис блокує всі читання
- Низька паралельність
- Довгі черги очікування

MVCC:
- Читання не блокуються
- Висока паралельність
- Читачі отримують консистентні дані ✓
```

Сценарій 2: Інтенсивний запис з рідкісним читанням

```
Блокування:
- Короткі блокування
- Висока швидкість
- Мінімальні накладні витрати ✓

MVCC:
- Створення багатьох версій
- Накладні витрати на зберігання
- Необхідність частого VACUUM
```

Сценарій 3: Довгі транзакції

```
Блокування:
- Блокування утримуються довго
- Велика черга очікування
- Ризик взаємоблокувань

MVCC:
- Накопичення великої кількості версій
- Неможливість очищення старих версій
- Роздування бази даних
- Погіршення продуктивності
```

## Практичні рекомендації

### Вибір рівня ізоляції

Критерії вибору рівня ізоляції для різних типів застосунків:

Вебзастосунки загального призначення:

```sql
-- Рекомендація: READ COMMITTED
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

Причини:
- Баланс між коректністю та продуктивністю
- Запобігає брудному читанню
- Мінімальні блокування
- Підходить для більшості CRUD операцій
```

Фінансові системи:

```sql
-- Рекомендація: REPEATABLE READ або SERIALIZABLE
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

Причини:
- Критична важливість консистентності
- Запобігання неповторюваному читанню
- Можливість аудиту транзакцій
- Прийнятні накладні витрати для критичних операцій
```

Аналітичні системи (звіти):

```sql
-- Рекомендація: READ COMMITTED або READ UNCOMMITTED
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

Причини:
- Перевага продуктивності над точністю
- Допустимі незначні неточності
- Мінімальний вплив на операційні транзакції
- Швидке виконання складних запитів
```

### Оптимізація транзакцій

Мінімізація розміру транзакцій:

```sql
-- Погано: довга транзакція
BEGIN TRANSACTION;
-- Обробка 1 000 000 записів в одній транзакції
FOR each_record IN (SELECT * FROM large_table) LOOP
    UPDATE processed_data SET status = 'done' WHERE id = each_record.id;
END LOOP;
COMMIT; -- Утримує блокування дуже довго

-- Добре: пакетна обробка
FOR batch_num IN 1..1000 LOOP
    BEGIN TRANSACTION;
    -- Обробка 1000 записів за раз
    UPDATE processed_data
    SET status = 'done'
    WHERE id IN (
        SELECT id FROM large_table
        WHERE batch = batch_num
    );
    COMMIT; -- Швидке звільнення блокувань
END LOOP;
```

Правильний порядок операцій:

```sql
-- Погано: блокування в непередбачуваному порядку
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'ACC002';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'ACC001';
-- Ризик взаємоблокування з транзакцією, що працює в протилежному порядку
COMMIT;

-- Добре: консистентний порядок блокування
BEGIN TRANSACTION;
-- Завжди блокувати в порядку зростання ID
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'ACC001';
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'ACC002';
COMMIT;
```

Використання явних блокувань при необхідності:

```sql
-- Оптимістичне блокування (для рідкісних конфліктів)
BEGIN TRANSACTION;
SELECT balance, version FROM accounts WHERE account_id = 'ACC001';
-- Користувач працює з даними...
UPDATE accounts
SET balance = new_balance, version = version + 1
WHERE account_id = 'ACC001' AND version = old_version;
IF ROW_COUNT = 0 THEN
    ROLLBACK;
    -- Дані були змінені іншою транзакцією
ELSE
    COMMIT;
END IF;

-- Песимістичне блокування (для частих конфліктів)
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 'ACC001' FOR UPDATE;
-- Отримано виключне блокування
UPDATE accounts SET balance = new_balance WHERE account_id = 'ACC001';
COMMIT;
```

### Моніторинг та налагодження

Моніторинг блокувань у PostgreSQL:

```sql
-- Поточні блокування
SELECT
    l.pid,
    l.locktype,
    l.mode,
    l.granted,
    a.query,
    a.state,
    a.wait_event_type,
    a.wait_event
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted;

-- Статистика по таблицях
SELECT
    schemaname,
    tablename,
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    n_live_tup AS live_tuples,
    n_dead_tup AS dead_tuples,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Виявлення повільних транзакцій:

```sql
-- Транзакції, що виконуються довго
SELECT
    pid,
    now() - xact_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
    AND now() - xact_start > interval '5 minutes'
ORDER BY duration DESC;

-- Налаштування попереджень про довгі транзакції
SET log_min_duration_statement = 1000; -- мс
```

Аналіз взаємоблокувань:

```sql
-- PostgreSQL логує взаємоблокування
-- У файлі postgresql.log шукаємо:
/*
ERROR: deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 678;
        blocked by process 12346.
        Process 12346 waits for ShareLock on transaction 679;
        blocked by process 12345.
HINT: See server log for query details.
CONTEXT: while updating tuple (0,42) in relation "accounts"
*/

-- Увімкнення детального логування взаємоблокувань
SET log_lock_waits = on;
SET deadlock_timeout = '1s';
```

### Оптимізація MVCC

Налаштування автоматичного очищення:

```sql
-- Агресивне очищення для таблиць з частими оновленнями
ALTER TABLE accounts SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 50,
    autovacuum_analyze_scale_factor = 0.05,
    autovacuum_analyze_threshold = 50
);

-- Відкладене очищення для великих таблиць з рідкісними оновленнями
ALTER TABLE audit_log SET (
    autovacuum_vacuum_scale_factor = 0.4,
    autovacuum_vacuum_threshold = 1000
);
```

Моніторинг роздування таблиць:

```sql
-- Розрахунок роздування (bloat)
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_live_tup,
    n_dead_tup,
    ROUND((n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0)), 2) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY dead_ratio DESC;
```

Стратегії зменшення роздування:

```sql
-- Регулярне обслуговування
-- Можна автоматизувати через cron або pg_cron

-- Щоденне очищення критичних таблиць
VACUUM ANALYZE accounts;

-- Щотижнева перебудова індексів
REINDEX TABLE CONCURRENTLY accounts;

-- Щомісячне повне очищення (у вікно обслуговування)
VACUUM FULL accounts; -- Блокує таблицю!
```

## Висновки

Управління транзакціями та паралельним доступом є критично важливою частиною будь-якої сучасної СУБД. Основні висновки цієї лекції:

ACID властивості забезпечують надійність транзакцій:
- Атомарність гарантує, що транзакція виконується повністю або не виконується зовсім
- Консистентність підтримує цілісність даних згідно з бізнес-правилами
- Ізольованість запобігає конфліктам між паралельними транзакціями
- Довговічність гарантує збереження результатів після фіксації

Рівні ізоляції представляють компроміс між коректністю та продуктивністю:
- READ UNCOMMITTED - максимальна продуктивність, мінімальні гарантії
- READ COMMITTED - стандартний вибір для більшості застосунків
- REPEATABLE READ - для застосунків з підвищеними вимогами до консистентності
- SERIALIZABLE - максимальні гарантії коректності, можливі проблеми з продуктивністю

Протоколи управління паралельністю мають різні характеристики:
- Блокування - традиційний підхід з явним контролем доступу
- Позначки часу - детерміністичне упорядкування без взаємоблокувань
- Оптимістична валідація - ефективна при низькій конфліктності

Взаємоблокування потребують уваги:
- Запобігання через упорядкування ресурсів та тайм-аути
- Виявлення через аналіз графу очікування
- Вирішення через вибір жертви для відкату

MVCC забезпечує високу паралельність:
- Читання не блокує запис, запис не блокує читання
- Консистентні снапшоти для транзакцій
- Потребує регулярного очищення застарілих версій

Розуміння цих концепцій дозволяє розробникам створювати надійні та ефективні застосунки, що коректно працюють під високим навантаженням і забезпечують цілісність даних навіть у складних багатокористувацьких середовищах.
