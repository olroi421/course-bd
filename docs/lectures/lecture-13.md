# Лекція 13. Проєктування NoSQL схем

## Вступ

Проєктування схем даних у документо-орієнтованих СУБД кардинально відрізняється від проєктування реляційних баз даних. Якщо в реляційних системах ми прагнемо нормалізації та уникнення дублювання даних, то в NoSQL системах, зокрема в MongoDB, ми часто свідомо денормалізуємо дані для досягнення оптимальної продуктивності. Ця зміна парадигми вимагає нового підходу до моделювання даних та розуміння компромісів між різними стратегіями проєктування.

Ключовою відмінністю є те, що в реляційних базах даних ми проєктуємо схему незалежно від запитів додатка, керуючись принципами нормалізації. У NoSQL системах схема проєктується виходячи з того, як додаток буде використовувати дані. Це означає, що одна й та сама предметна область може мати різні схеми в залежності від специфіки застосування.

У цій лекції ми розглянемо фундаментальні принципи моделювання даних у документо-орієнтованих СУБД, детально проаналізуємо підходи вбудовування та посилань, вивчимо стратегії денормалізації та практичні паттерни проєктування для типових сценаріїв використання.

## Принципи моделювання даних у документо-орієнтованих СУБД

### Відмінності від реляційного моделювання

Реляційне моделювання базується на математичній теорії відношень та нормальних формах, які мінімізують надмірність даних. Документо-орієнтоване моделювання керується іншими принципами, де головним критерієм є ефективність доступу до даних для конкретних сценаріїв використання.

У реляційних базах даних типова структура для системи управління студентами виглядала б наступним чином:

```sql
-- Реляційна модель з нормалізацією
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100)
);

CREATE TABLE addresses (
    address_id INT PRIMARY KEY,
    student_id INT,
    street VARCHAR(100),
    city VARCHAR(50),
    postal_code VARCHAR(10),
    FOREIGN KEY (student_id) REFERENCES students(student_id)
);

CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    course_name VARCHAR(100),
    credits INT
);

CREATE TABLE enrollments (
    enrollment_id INT PRIMARY KEY,
    student_id INT,
    course_id INT,
    enrollment_date DATE,
    grade VARCHAR(2),
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

Та сама структура в MongoDB може бути організована по-різному в залежності від патернів доступу:

```javascript
// Варіант 1: Максимальне вбудовування для швидкого читання
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "student_id": "S2024001",
    "first_name": "Іван",
    "last_name": "Петров",
    "email": "ivan.petrov@university.edu.ua",
    "address": {
        "street": "вул. Хрещатик, 15",
        "city": "Київ",
        "postal_code": "01001"
    },
    "enrollments": [
        {
            "course_id": "CS301",
            "course_name": "Бази даних",
            "credits": 6,
            "enrollment_date": ISODate("2024-09-01"),
            "grade": "A",
            "instructor": {
                "name": "Проф. Іваненко",
                "email": "ivanenko@university.edu.ua"
            }
        },
        {
            "course_id": "CS302",
            "course_name": "Алгоритми",
            "credits": 5,
            "enrollment_date": ISODate("2024-09-01"),
            "grade": "B+",
            "instructor": {
                "name": "Доц. Коваленко",
                "email": "kovalenko@university.edu.ua"
            }
        }
    ]
}
```

### Проєктування орієнтоване на запити

Фундаментальний принцип проєктування NoSQL схем полягає в тому, що структура даних має відповідати патернам доступу додатка. Перш ніж проєктувати схему, необхідно детально проаналізувати, як додаток буде використовувати дані.

Процес проєктування починається з ідентифікації ключових запитів та операцій:

```javascript
// Типові операції для системи управління студентами

// 1. Отримання повної інформації про студента (часто)
db.students.findOne({ student_id: "S2024001" })

// 2. Отримання списку курсів студента (часто)
db.students.findOne(
    { student_id: "S2024001" },
    { enrollments: 1 }
)

// 3. Додавання нового зарахування (рідко)
db.students.updateOne(
    { student_id: "S2024001" },
    {
        $push: {
            enrollments: {
                course_id: "CS303",
                enrollment_date: new Date()
            }
        }
    }
)

// 4. Пошук всіх студентів на курсі (рідко)
db.students.find({
    "enrollments.course_id": "CS301"
})

// 5. Оновлення оцінки (рідко)
db.students.updateOne(
    {
        student_id: "S2024001",
        "enrollments.course_id": "CS301"
    },
    {
        $set: {
            "enrollments.$.grade": "A"
        }
    }
)
```

На основі аналізу частоти та характеру операцій приймається рішення про структуру даних. Якщо інформація про студента та його курси завжди використовується разом, доцільно вбудувати курси в документ студента. Якщо ж часто потрібна інформація про курси незалежно від студентів, краще винести курси в окрему колекцію.

### Компроміси між продуктивністю та консистентністю

Кожне рішення щодо структури даних передбачає певні компроміси. Розглянемо основні фактори, які впливають на вибір стратегії моделювання.

Швидкість читання проти швидкості запису є одним з головних компромісів. Вбудовування даних прискорює читання, оскільки вся інформація отримується одним запитом, але ускладнює оновлення, особливо якщо треба оновити дані в багатьох місцях.

```javascript
// Вбудовування: швидке читання, повільне оновлення
{
    "_id": ObjectId("..."),
    "student_id": "S2024001",
    "name": "Іван Петров",
    "courses": [
        {
            "course_id": "CS301",
            "name": "Бази даних",
            "instructor": "Проф. Іваненко",
            "instructor_email": "ivanenko@university.edu.ua"
        }
    ]
}

// Якщо викладач змінює email, треба оновити всі документи студентів
db.students.updateMany(
    { "courses.instructor": "Проф. Іваненко" },
    {
        $set: {
            "courses.$[elem].instructor_email": "new.email@university.edu.ua"
        }
    },
    {
        arrayFilters: [{ "elem.instructor": "Проф. Іваненко" }]
    }
)
```

Розмір документа також є важливим фактором. MongoDB має обмеження на максимальний розмір документа у 16 МБ. Якщо вбудовувати необмежену кількість елементів, можна досягти цього ліміту.

```javascript
// Потенційна проблема: необмежене зростання масиву
{
    "_id": ObjectId("..."),
    "course_id": "CS301",
    "name": "Бази даних",
    "reviews": [
        // Якщо курс популярний, тут може бути тисячі відгуків
        { "student": "S001", "rating": 5, "comment": "Excellent course" },
        { "student": "S002", "rating": 4, "comment": "Very good" },
        // ... тисячі інших відгуків
    ]
}

// Краще рішення: винести відгуки в окрему колекцію
// Колекція courses
{
    "_id": ObjectId("..."),
    "course_id": "CS301",
    "name": "Бази даних",
    "avg_rating": 4.5,
    "reviews_count": 1247
}

// Колекція reviews
{
    "_id": ObjectId("..."),
    "course_id": "CS301",
    "student_id": "S001",
    "rating": 5,
    "comment": "Excellent course",
    "date": ISODate("2024-10-15")
}
```

Атомарність операцій є ще одним критерієм вибору. MongoDB гарантує атомарність операцій на рівні окремого документа, тому якщо операція має бути атомарною, краще зберігати всі пов'язані дані в одному документі.

```javascript
// Атомарне оновлення балансу та історії транзакцій
db.accounts.updateOne(
    { account_id: "ACC001" },
    {
        $inc: { balance: -100 },
        $push: {
            transactions: {
                amount: -100,
                type: "withdrawal",
                date: new Date(),
                description: "ATM withdrawal"
            }
        }
    }
)

// Цю операцію не можна атомарно виконати, якщо баланс
// і транзакції зберігаються в різних колекціях
```

### Аналіз патернів доступу

Перед проєктуванням схеми необхідно детально проаналізувати, як додаток працює з даними. Це включає визначення типових запитів, їх частоти та критичності для продуктивності системи.

Створимо матрицю операцій для типової системи управління курсами:

```javascript
// Матриця операцій (частота виконання)
const operationsMatrix = {
    // Критичні для продуктивності (виконуються постійно)
    critical: [
        {
            operation: "Отримати інформацію про студента з курсами",
            frequency: "1000+ запитів/хвилину",
            pattern: "read-heavy",
            data: ["student", "enrollments", "courses"]
        },
        {
            operation: "Перегляд списку курсів з викладачами",
            frequency: "500+ запитів/хвилину",
            pattern: "read-heavy",
            data: ["courses", "instructors"]
        }
    ],

    // Часті але менш критичні
    frequent: [
        {
            operation: "Зарахування студента на курс",
            frequency: "100 запитів/хвилину",
            pattern: "write",
            data: ["enrollments", "courses"]
        },
        {
            operation: "Оновлення оцінки",
            frequency: "50 запитів/хвилину",
            pattern: "write",
            data: ["enrollments"]
        }
    ],

    // Рідкісні операції
    rare: [
        {
            operation: "Створення нового курсу",
            frequency: "1-2 запити/день",
            pattern: "write",
            data: ["courses", "instructors"]
        },
        {
            operation: "Видалення студента",
            frequency: "1-5 запитів/день",
            pattern: "write",
            data: ["students", "enrollments"]
        }
    ]
}
```

На основі цього аналізу можна прийняти рішення про структуру даних. Якщо читання інформації про студента з курсами є критичним, доцільно вбудувати базову інформацію про курси в документ студента:

```javascript
// Оптимізована структура для частого читання
{
    "_id": ObjectId("..."),
    "student_id": "S2024001",
    "name": "Іван Петров",
    "email": "ivan@university.edu.ua",

    // Вбудовуємо мінімальну інформацію про курси для швидкого доступу
    "current_courses": [
        {
            "course_id": "CS301",
            "name": "Бази даних",
            "credits": 6,
            "instructor_name": "Проф. Іваненко"
        }
    ],

    // Посилання на повну інформацію про курси для рідкісних запитів
    "course_ids": ["CS301", "CS302", "MATH201"],

    // Підсумкова інформація
    "total_credits": 17,
    "gpa": 3.85
}
```

## Embedded vs Referenced підходи

### Концепція вбудовування документів

Вбудовування (embedding) означає зберігання пов'язаних даних всередині одного документа. Це природний підхід для документо-орієнтованих баз даних, який забезпечує найкращу продуктивність читання.

Розглянемо детальний приклад вбудовування для системи блогу:

```javascript
// Повне вбудовування: пост з коментарями та автором
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "title": "Введення в NoSQL бази даних",
    "slug": "intro-to-nosql",
    "created_at": ISODate("2024-10-01T10:00:00Z"),
    "updated_at": ISODate("2024-10-15T14:30:00Z"),

    // Вбудована інформація про автора
    "author": {
        "user_id": "U001",
        "name": "Іван Петров",
        "email": "ivan@example.com",
        "avatar_url": "https://example.com/avatars/ivan.jpg"
    },

    // Основний контент
    "content": "Детальний текст статті...",
    "excerpt": "Короткий опис...",

    // Метадані
    "tags": ["NoSQL", "MongoDB", "Databases"],
    "category": "Technology",
    "views_count": 1547,
    "likes_count": 89,

    // Вбудовані коментарі
    "comments": [
        {
            "comment_id": "C001",
            "author": {
                "user_id": "U002",
                "name": "Марія Коваленко",
                "avatar_url": "https://example.com/avatars/maria.jpg"
            },
            "content": "Відмінна стаття! Дуже корисна інформація.",
            "created_at": ISODate("2024-10-02T09:15:00Z"),
            "likes_count": 5,

            // Вкладені відповіді на коментар
            "replies": [
                {
                    "reply_id": "R001",
                    "author": {
                        "user_id": "U001",
                        "name": "Іван Петров"
                    },
                    "content": "Дякую за відгук!",
                    "created_at": ISODate("2024-10-02T10:30:00Z")
                }
            ]
        },
        {
            "comment_id": "C002",
            "author": {
                "user_id": "U003",
                "name": "Олександр Сидоров",
                "avatar_url": "https://example.com/avatars/alex.jpg"
            },
            "content": "Чи можна більше деталей про шардинг?",
            "created_at": ISODate("2024-10-03T14:20:00Z"),
            "likes_count": 2,
            "replies": []
        }
    ]
}
```

Переваги такого підходу очевидні для операції отримання повної інформації про пост:

```javascript
// Один запит отримує всю інформацію
const post = db.posts.findOne({ slug: "intro-to-nosql" })

// У реляційній БД знадобилося б кілька запитів або складний JOIN
// SELECT * FROM posts WHERE slug = 'intro-to-nosql'
// SELECT * FROM users WHERE user_id = (author з попереднього запиту)
// SELECT * FROM comments WHERE post_id = ...
// SELECT * FROM users WHERE user_id IN (автори коментарів)
```

Однак цей підхід має обмеження. Якщо користувач змінює своє ім'я або аватар, треба оновити всі пости та коментарі:

```javascript
// Проблема: оновлення профілю користувача
db.posts.updateMany(
    { "author.user_id": "U001" },
    {
        $set: {
            "author.name": "Іван Володимирович Петров",
            "author.avatar_url": "https://example.com/avatars/ivan_new.jpg"
        }
    }
)

db.posts.updateMany(
    { "comments.author.user_id": "U001" },
    {
        $set: {
            "comments.$[elem].author.name": "Іван Володимирович Петров",
            "comments.$[elem].author.avatar_url": "https://example.com/avatars/ivan_new.jpg"
        }
    },
    {
        arrayFilters: [{ "elem.author.user_id": "U001" }]
    }
)
```

### Концепція використання посилань

Посилання (referencing) означає зберігання тільки ідентифікатора пов'язаної сутності, подібно до зовнішніх ключів у реляційних базах даних. Це зменшує дублювання даних, але вимагає додаткових запитів для отримання повної інформації.

Розглянемо ту саму систему блогу з використанням посилань:

```javascript
// Колекція users
{
    "_id": ObjectId("507f1f77bcf86cd799439020"),
    "user_id": "U001",
    "name": "Іван Петров",
    "email": "ivan@example.com",
    "avatar_url": "https://example.com/avatars/ivan.jpg",
    "bio": "Software engineer and technical writer",
    "created_at": ISODate("2023-01-15T00:00:00Z"),
    "stats": {
        "posts_count": 47,
        "comments_count": 234,
        "followers_count": 1523
    }
}

// Колекція posts
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "title": "Введення в NoSQL бази даних",
    "slug": "intro-to-nosql",
    "author_id": "U001",  // Посилання замість вбудовування
    "content": "Детальний текст статті...",
    "created_at": ISODate("2024-10-01T10:00:00Z"),
    "tags": ["NoSQL", "MongoDB", "Databases"],
    "stats": {
        "views_count": 1547,
        "likes_count": 89,
        "comments_count": 2
    }
}

// Колекція comments
{
    "_id": ObjectId("507f1f77bcf86cd799439030"),
    "comment_id": "C001",
    "post_id": "507f1f77bcf86cd799439011",  // Посилання на пост
    "author_id": "U002",  // Посилання на користувача
    "content": "Відмінна стаття! Дуже корисна інформація.",
    "created_at": ISODate("2024-10-02T09:15:00Z"),
    "likes_count": 5,
    "parent_comment_id": null  // Для вкладених коментарів
}
```

Для отримання повної інформації про пост потрібно кілька запитів або використання aggregation pipeline:

```javascript
// Варіант 1: Послідовні запити
const post = db.posts.findOne({ slug: "intro-to-nosql" })
const author = db.users.findOne({ user_id: post.author_id })
const comments = db.comments.find({ post_id: post._id }).toArray()

// Отримання авторів коментарів
const commentAuthorIds = comments.map(c => c.author_id)
const commentAuthors = db.users.find({
    user_id: { $in: commentAuthorIds }
}).toArray()

// Варіант 2: Aggregation pipeline з $lookup
db.posts.aggregate([
    { $match: { slug: "intro-to-nosql" } },

    // Приєднання інформації про автора
    {
        $lookup: {
            from: "users",
            localField: "author_id",
            foreignField: "user_id",
            as: "author"
        }
    },
    { $unwind: "$author" },

    // Приєднання коментарів
    {
        $lookup: {
            from: "comments",
            localField: "_id",
            foreignField: "post_id",
            as: "comments"
        }
    },

    // Приєднання авторів коментарів
    {
        $lookup: {
            from: "users",
            localField: "comments.author_id",
            foreignField: "user_id",
            as: "comment_authors"
        }
    },

    // Форматування результату
    {
        $project: {
            title: 1,
            content: 1,
            created_at: 1,
            author: {
                name: 1,
                avatar_url: 1
            },
            comments: {
                $map: {
                    input: "$comments",
                    as: "comment",
                    in: {
                        content: "$$comment.content",
                        created_at: "$$comment.created_at",
                        author: {
                            $arrayElemAt: [
                                {
                                    $filter: {
                                        input: "$comment_authors",
                                        cond: {
                                            $eq: ["$$this.user_id", "$$comment.author_id"]
                                        }
                                    }
                                },
                                0
                            ]
                        }
                    }
                }
            }
        }
    }
])
```

Перевага підходу з посиланнями проявляється при оновленні даних користувача:

```javascript
// Просте оновлення в одному місці
db.users.updateOne(
    { user_id: "U001" },
    {
        $set: {
            name: "Іван Володимирович Петров",
            avatar_url: "https://example.com/avatars/ivan_new.jpg"
        }
    }
)

// Зміни автоматично відображаються у всіх постах і коментарях
// при наступному запиті через $lookup
```

### Гібридний підхід

На практиці найефективнішим часто є гібридний підхід, який поєднує переваги обох стратегій. Ідея полягає в тому, щоб вбудовувати тільки ту інформацію, яка рідко змінюється і часто потрібна, а для решти використовувати посилання.

```javascript
// Гібридна структура для блогу
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "title": "Введення в NoSQL бази даних",
    "slug": "intro-to-nosql",

    // Вбудовуємо базову інформацію про автора (рідко змінюється)
    "author": {
        "user_id": "U001",
        "name": "Іван Петров",
        "avatar_url": "https://example.com/avatars/ivan.jpg"
    },

    // Повна інформація про автора за посиланням
    "author_id": "U001",

    "content": "Детальний текст статті...",
    "created_at": ISODate("2024-10-01T10:00:00Z"),

    // Вбудовуємо останні коментарі для швидкого доступу
    "recent_comments": [
        {
            "comment_id": "C002",
            "author_name": "Олександр Сидоров",
            "author_avatar": "https://example.com/avatars/alex.jpg",
            "content_preview": "Чи можна більше деталей...",
            "created_at": ISODate("2024-10-03T14:20:00Z")
        }
    ],

    // Посилання для отримання всіх коментарів
    "comments_count": 47,

    // Кешуємо підрахунки
    "stats": {
        "views_count": 1547,
        "likes_count": 89,
        "shares_count": 23
    }
}

// Окрема колекція для всіх коментарів
{
    "_id": ObjectId("..."),
    "post_id": "507f1f77bcf86cd799439011",
    "author_id": "U003",
    "content": "Чи можна більше деталей про шардинг?",
    "created_at": ISODate("2024-10-03T14:20:00Z"),
    "likes_count": 2
}
```

Стратегія оновлення для гібридного підходу:

```javascript
// При зміні імені користувача оновлюємо тільки вбудовані копії
async function updateUserName(userId, newName, newAvatarUrl) {
    // 1. Оновлюємо головний документ користувача
    await db.users.updateOne(
        { user_id: userId },
        {
            $set: {
                name: newName,
                avatar_url: newAvatarUrl
            }
        }
    )

    // 2. Оновлюємо вбудовані копії в постах (асинхронно)
    await db.posts.updateMany(
        { "author.user_id": userId },
        {
            $set: {
                "author.name": newName,
                "author.avatar_url": newAvatarUrl
            }
        }
    )

    // 3. Оновлюємо в останніх коментарях
    await db.posts.updateMany(
        { "recent_comments.author_id": userId },
        {
            $set: {
                "recent_comments.$[elem].author_name": newName,
                "recent_comments.$[elem].author_avatar": newAvatarUrl
            }
        },
        {
            arrayFilters: [{ "elem.author_id": userId }]
        }
    )
}
```

### Критерії вибору між підходами

Вибір між вбудовуванням та посиланнями базується на аналізі кількох ключових факторів.

Розмір пов'язаних даних є першим критерієм. Якщо дані невеликі та мають обмежений розмір, вбудовування безпечне. Якщо дані можуть рости необмежено, краще використовувати посилання.

```javascript
// Добре для вбудовування: обмежена кількість елементів
{
    "student_id": "S001",
    "name": "Іван Петров",
    "contact_phones": [  // Максимум 2-3 телефони
        { "type": "mobile", "number": "+380501234567" },
        { "type": "home", "number": "+380441234567" }
    ]
}

// Погано для вбудовування: необмежена кількість
{
    "course_id": "CS301",
    "name": "Бази даних",
    "all_student_submissions": [  // Може бути тисячі
        { "student_id": "S001", "assignment": "HW1", "file": "..." },
        { "student_id": "S002", "assignment": "HW1", "file": "..." },
        // ... потенційно тисячі елементів
    ]
}

// Краще рішення: окрема колекція
{
    "_id": ObjectId("..."),
    "course_id": "CS301",
    "student_id": "S001",
    "assignment": "HW1",
    "submission_file": "...",
    "submitted_at": ISODate("...")
}
```

Частота оновлень є другим критерієм. Дані, що рідко змінюються, добре підходять для вбудовування. Дані, що часто оновлюються, краще зберігати окремо.

```javascript
// Добре для вбудовування: статична інформація
{
    "order_id": "ORD001",
    "customer": {
        // Інформація фіксується на момент замовлення і не змінюється
        "name": "Іван Петров",
        "address": "вул. Хрещатик, 15",
        "phone": "+380501234567"
    },
    "items": [
        {
            // Ціна фіксується на момент замовлення
            "product_name": "Ноутбук",
            "price": 25000,
            "quantity": 1
        }
    ]
}

// Погано для вбудовування: часто оновлювана інформація
{
    "product_id": "PROD001",
    "name": "Ноутбук",
    "current_price": 25000,  // Змінюється щодня
    "stock_quantity": 15,     // Змінюється при кожному продажу
    "all_reviews": [...]      // Постійно додаються нові
}
```

Патерни доступу визначають оптимальну структуру. Якщо дані завжди використовуються разом, вбудовування ефективне. Якщо дані часто потрібні окремо, краще використовувати посилання.

```javascript
// Сценарій 1: Дані завжди використовуються разом
// Профіль користувача завжди показується з адресою
{
    "user_id": "U001",
    "name": "Іван Петров",
    "address": {  // Вбудовуємо, бо завжди показуємо разом
        "street": "вул. Хрещатик, 15",
        "city": "Київ"
    }
}

// Сценарій 2: Дані використовуються незалежно
// Курси часто показуються без студентів, і навпаки
// Колекція courses
{
    "course_id": "CS301",
    "name": "Бази даних",
    "credits": 6
}

// Колекція students
{
    "student_id": "S001",
    "name": "Іван Петров"
}

// Колекція enrollments (зв'язок)
{
    "student_id": "S001",
    "course_id": "CS301",
    "grade": "A"
}
```

## Денормалізація як проектна парадигма NoSQL

### Філософія денормалізації

Якщо в реляційних базах даних нормалізація є gold standard, то в документо-орієнтованих системах денормалізація часто є оптимальним підходом. Денормалізація означає свідоме дублювання даних для оптимізації продуктивності читання.

Основний принцип денормалізації у NoSQL полягає в тому, що дисковий простір дешевший за час процесора та мережевий трафік. Сучасні диски великі та недорогі, тому зберігання кількох копій даних економічно виправдане, якщо це прискорює доступ.

Розглянемо еволюцію схеми від нормалізованої до денормалізованої:

```javascript
// Етап 1: Повністю нормалізована структура (як у SQL)
// Колекція users
{
    "_id": ObjectId("..."),
    "user_id": "U001",
    "username": "ivanpetrov",
    "email": "ivan@example.com"
}

// Колекція posts
{
    "_id": ObjectId("..."),
    "post_id": "P001",
    "author_id": "U001",
    "title": "My First Post",
    "content": "..."
}

// Колекція comments
{
    "_id": ObjectId("..."),
    "comment_id": "C001",
    "post_id": "P001",
    "author_id": "U001",
    "text": "Great post!"
}

// Проблема: Для відображення поста з коментарями потрібно 3+ запити
```

```javascript
// Етап 2: Часткова денормалізація
// Колекція users (без змін)
{
    "_id": ObjectId("..."),
    "user_id": "U001",
    "username": "ivanpetrov",
    "email": "ivan@example.com",
    "full_name": "Іван Петров",
    "avatar_url": "https://example.com/avatar.jpg"
}

// Колекція posts з денормалізованою інформацією про автора
{
    "_id": ObjectId("..."),
    "post_id": "P001",
    "title": "My First Post",
    "content": "...",

    // Денормалізація: дублюємо базову інформацію про автора
    "author": {
        "user_id": "U001",
        "username": "ivanpetrov",
        "full_name": "Іван Петров",
        "avatar_url": "https://example.com/avatar.jpg"
    },

    // Зберігаємо посилання для повної інформації
    "author_id": "U001",

    "created_at": ISODate("2024-10-15"),
    "stats": {
        "views": 150,
        "likes": 23,
        "comments_count": 5
    }
}

// Тепер один запит отримує пост з інформацією про автора
```

```javascript
// Етап 3: Агресивна денормалізація
{
    "_id": ObjectId("..."),
    "post_id": "P001",
    "title": "My First Post",
    "content": "...",

    "author": {
        "user_id": "U001",
        "username": "ivanpetrov",
        "full_name": "Іван Петров",
        "avatar_url": "https://example.com/avatar.jpg"
    },

    // Вбудовуємо останні коментарі
    "recent_comments": [
        {
            "comment_id": "C001",
            "author": {
                "user_id": "U002",
                "username": "mariakoval",
                "full_name": "Марія Коваленко",
                "avatar_url": "https://example.com/avatar2.jpg"
            },
            "text": "Great post!",
            "created_at": ISODate("2024-10-16"),
            "likes": 3
        },
        {
            "comment_id": "C002",
            "author": {
                "user_id": "U003",
                "username": "alexsid",
                "full_name": "Олександр Сидоров",
                "avatar_url": "https://example.com/avatar3.jpg"
            },
            "text": "Very informative",
            "created_at": ISODate("2024-10-17"),
            "likes": 1
        }
    ],

    // Зберігаємо лінк на всі коментарі
    "total_comments": 47,

    "created_at": ISODate("2024-10-15"),
    "stats": {
        "views": 150,
        "likes": 23
    }
}

// Тепер ОДИН запит отримує пост, автора та останні коментарі
const post = db.posts.findOne({ post_id: "P001" })
```

### Стратегії підтримки консистентності при денормалізації

Денормалізація створює проблему підтримки консистентності даних. Коли одна і та сама інформація зберігається в кількох місцях, необхідно забезпечити синхронізацію при оновленнях.

Існує кілька стратегій управління консистентністю денормалізованих даних:

**Стратегія 1: Eventual Consistency (Остаточна консистентність)**

Система дозволяє тимчасову неконсистентність, але гарантує, що врешті-решт всі копії будуть оновлені.

```javascript
// Приклад: Оновлення профілю користувача
async function updateUserProfile(userId, updates) {
    // 1. Оновлюємо головний документ
    await db.users.updateOne(
        { user_id: userId },
        { $set: updates }
    )

    // 2. Запускаємо асинхронне оновлення денормалізованих копій
    // Це може виконуватися в background job
    updateDenormalizedUserData(userId, updates)

    return { success: true }
}

async function updateDenormalizedUserData(userId, updates) {
    const updateFields = {}

    // Формуємо оновлення для вкладених структур
    if (updates.full_name) {
        updateFields["author.full_name"] = updates.full_name
    }
    if (updates.avatar_url) {
        updateFields["author.avatar_url"] = updates.avatar_url
    }

    // Оновлюємо всі пости
    await db.posts.updateMany(
        { "author.user_id": userId },
        { $set: updateFields }
    )

    // Оновлюємо коментарі
    await db.posts.updateMany(
        { "recent_comments.author.user_id": userId },
        {
            $set: {
                "recent_comments.$[elem].author.full_name": updates.full_name,
                "recent_comments.$[elem].author.avatar_url": updates.avatar_url
            }
        },
        {
            arrayFilters: [{ "elem.author.user_id": userId }]
        }
    )
}
```

**Стратегія 2: Версіонування даних**

Кожна копія даних має версію, що дозволяє виявити застарілі дані.

```javascript
// Додаємо версію до денормалізованих даних
{
    "post_id": "P001",
    "title": "My Post",

    "author": {
        "user_id": "U001",
        "username": "ivanpetrov",
        "full_name": "Іван Петров",
        "avatar_url": "https://example.com/avatar.jpg",
        "version": 5  // Версія даних користувача
    }
}

// При читанні перевіряємо актуальність
async function getPost(postId) {
    const post = await db.posts.findOne({ post_id: postId })

    // Перевіряємо версію автора
    const currentAuthor = await db.users.findOne(
        { user_id: post.author.user_id },
        { projection: { version: 1, username: 1, full_name: 1, avatar_url: 1 } }
    )

    // Якщо версії не співпадають, оновлюємо дані
    if (post.author.version < currentAuthor.version) {
        await db.posts.updateOne(
            { post_id: postId },
            {
                $set: {
                    "author.username": currentAuthor.username,
                    "author.full_name": currentAuthor.full_name,
                    "author.avatar_url": currentAuthor.avatar_url,
                    "author.version": currentAuthor.version
                }
            }
        )

        post.author = currentAuthor
    }

    return post
}
```

**Стратегія 3: Тільки додавання (Append-Only)**

Замість оновлення існуючих даних, додаємо нові записи з позначкою часу.

```javascript
// Замість оновлення зберігаємо історію змін
{
    "user_id": "U001",
    "profile_history": [
        {
            "version": 1,
            "full_name": "Іван Петров",
            "avatar_url": "https://example.com/avatar1.jpg",
            "valid_from": ISODate("2024-01-01"),
            "valid_to": ISODate("2024-06-01")
        },
        {
            "version": 2,
            "full_name": "Іван Володимирович Петров",
            "avatar_url": "https://example.com/avatar2.jpg",
            "valid_from": ISODate("2024-06-01"),
            "valid_to": null  // Поточна версія
        }
    ]
}

// При запиті беремо актуальну версію
db.users.aggregate([
    { $match: { user_id: "U001" } },
    {
        $project: {
            current_profile: {
                $arrayElemAt: [
                    {
                        $filter: {
                            input: "$profile_history",
                            cond: { $eq: ["$$this.valid_to", null] }
                        }
                    },
                    0
                ]
            }
        }
    }
])
```

**Стратегія 4: Змішаний підхід**

Денормалізуємо тільки дані, що рідко змінюються, для частозмінних даних використовуємо посилання.

```javascript
{
    "post_id": "P001",
    "title": "My Post",

    // Денормалізуємо стабільні дані
    "author": {
        "user_id": "U001",
        "username": "ivanpetrov"  // Рідко змінюється
    },

    // Посилання для частозмінних даних
    "author_id": "U001",  // Для отримання avatar_url, bio і т.д.

    // Кешуємо підрахунки
    "cached_stats": {
        "likes_count": 23,
        "views_count": 150,
        "last_updated": ISODate("2024-10-17T10:00:00Z")
    }
}

// При відображенні поста
async function displayPost(postId) {
    const post = await db.posts.findOne({ post_id: postId })

    // Для стабільних даних використовуємо денормалізовану копію
    const authorUsername = post.author.username

    // Для змінних даних робимо додатковий запит
    const authorDetails = await db.users.findOne(
        { user_id: post.author_id },
        { projection: { avatar_url: 1, bio: 1, verified: 1 } }
    )

    return {
        ...post,
        author: {
            ...post.author,
            ...authorDetails
        }
    }
}
```

### Розрахунок вартості денормалізації

При прийнятті рішення про денормалізацію необхідно порівняти вартість додаткового дискового простору з вартістю додаткових запитів та складністю підтримки консистентності.

```javascript
// Приклад розрахунку для системи соціальної мережі

// Варіант 1: Нормалізована структура
// users: 1,000,000 записів × 1 KB = 1 GB
// posts: 10,000,000 записів × 2 KB = 20 GB
// При перегляді стрічки (100 постів):
// - 1 запит постів
// - До 100 запитів користувачів (якщо немає кешу)
// Або складний JOIN через $lookup

// Варіант 2: Денормалізована структура
// users: 1,000,000 × 1 KB = 1 GB
// posts: 10,000,000 × 2.5 KB = 25 GB (додатково 500 bytes на автора)
// При перегляді стрічки:
// - 1 запит постів (отримує всю інформацію)

// Додаткові витрати: 5 GB (25%)
// Виграш: Усунення 100 додаткових запитів або складного JOIN
// При 1000 переглядів стрічки на секунду економимо ~100,000 запитів/сек
```

Критерії вибору денормалізації:

```javascript
// Денормалізуємо, якщо:
const shouldDenormalize = {
    // 1. Дані читаються набагато частіше, ніж оновлюються
    readWriteRatio: 100,  // 100:1

    // 2. Розмір денормалізованих даних невеликий
    denormalizedDataSize: "< 1KB per document",

    // 3. Дані відносно стабільні
    updateFrequency: "раз на місяць або рідше",

    // 4. Продуктивність критична
    performanceRequirement: "< 100ms response time",

    // 5. Можемо забезпечити консистентність
    consistencyStrategy: "eventual consistency acceptable"
}

// НЕ денормалізуємо, якщо:
const shouldNotDenormalize = {
    // 1. Дані часто змінюються
    updateFrequency: "кілька разів на день",

    // 2. Велика кількість копій (складно синхронізувати)
    numberOfCopies: "> 1000 copies",

    // 3. Потрібна сильна консистентність
    consistencyRequirement: "immediate consistency required",

    // 4. Розмір даних великий
    dataSize: "> 10KB per copy",

    // 5. Дані використовуються незалежно
    usagePattern: "accessed separately more often"
}
```

## Міграція реляційних схем до документо-орієнтованих

### Аналіз реляційної схеми

Процес міграції починається з детального аналізу існуючої реляційної схеми та патернів доступу до даних. Розглянемо типову систему електронної комерції.

Реляційна схема:

```sql
-- Таблиця користувачів
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE,
    password_hash VARCHAR(255),
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    created_at TIMESTAMP
);

-- Таблиця адрес
CREATE TABLE addresses (
    address_id INT PRIMARY KEY,
    user_id INT,
    type ENUM('billing', 'shipping'),
    street VARCHAR(200),
    city VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(50),
    is_default BOOLEAN,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Таблиця товарів
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(200),
    description TEXT,
    price DECIMAL(10,2),
    category_id INT,
    created_at TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

-- Таблиця категорій
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(100),
    parent_id INT,
    FOREIGN KEY (parent_id) REFERENCES categories(category_id)
);

-- Таблиця замовлень
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    order_date TIMESTAMP,
    status VARCHAR(50),
    total_amount DECIMAL(10,2),
    shipping_address_id INT,
    billing_address_id INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id),
    FOREIGN KEY (billing_address_id) REFERENCES addresses(address_id)
);

-- Таблиця позицій замовлення
CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Таблиця відгуків
CREATE TABLE reviews (
    review_id INT PRIMARY KEY,
    product_id INT,
    user_id INT,
    rating INT,
    comment TEXT,
    created_at TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

Аналіз патернів доступу:

```javascript
// Типові запити та їх частота
const accessPatterns = {
    veryFrequent: [
        "Отримати профіль користувача з адресами",
        "Показати товар з категорією та рейтингом",
        "Отримати замовлення з позиціями та адресами"
    ],
    frequent: [
        "Список товарів категорії",
        "Пошук товарів",
        "Історія замовлень користувача"
    ],
    rare: [
        "Створення нового користувача",
        "Додавання товару",
        "Оновлення статусу замовлення"
    ]
}
```

### Стратегії трансформації схеми

На основі аналізу можна визначити кілька стратегій трансформації.

**Стратегія 1: Один-до-багатьох з вбудовуванням**

Відношення один-до-багатьох, де "багато" має обмежену кількість, трансформуються через вбудовування.

```javascript
// Користувач з адресами (реляційна: 2 таблиці → MongoDB: 1 документ)
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "user_id": "U001",
    "email": "ivan@example.com",
    "password_hash": "...",
    "first_name": "Іван",
    "last_name": "Петров",
    "created_at": ISODate("2024-01-15T00:00:00Z"),

    // Вбудовуємо адреси (зазвичай 2-5 адрес на користувача)
    "addresses": [
        {
            "address_id": "ADDR001",
            "type": "shipping",
            "street": "вул. Хрещатик, 15",
            "city": "Київ",
            "postal_code": "01001",
            "country": "Україна",
            "is_default": true
        },
        {
            "address_id": "ADDR002",
            "type": "billing",
            "street": "вул. Шевченка, 10",
            "city": "Київ",
            "postal_code": "01004",
            "country": "Україна",
            "is_default": false
        }
    ],

    // Метадані
    "account_status": "active",
    "last_login": ISODate("2024-10-17T14:30:00Z")
}
```

**Стратегія 2: Багато-до-багатьох через денормалізацію**

Складні зв'язки багато-до-багатьох трансформуються через часткову денормалізацію.

```javascript
// Замовлення з повною інформацією про товари
{
    "_id": ObjectId("507f1f77bcf86cd799439020"),
    "order_id": "ORD001",
    "user_id": "U001",
    "order_date": ISODate("2024-10-15T10:00:00Z"),
    "status": "shipped",

    // Денормалізуємо інформацію про користувача (на момент замовлення)
    "customer": {
        "user_id": "U001",
        "email": "ivan@example.com",
        "first_name": "Іван",
        "last_name": "Петров"
    },

    // Вбудовуємо позиції замовлення з інформацією про товари
    "items": [
        {
            "item_id": "ITEM001",
            "product_id": "PROD001",
            // Денормалізуємо дані товару (ціна фіксується на момент замовлення)
            "product_name": "Ноутбук Dell XPS 15",
            "product_sku": "DELL-XPS-15-2024",
            "category": "Ноутбуки",
            "unit_price": NumberDecimal("45000.00"),
            "quantity": 1,
            "subtotal": NumberDecimal("45000.00")
        },
        {
            "item_id": "ITEM002",
            "product_id": "PROD002",
            "product_name": "Миша Logitech MX Master 3",
            "product_sku": "LOG-MX-M3",
            "category": "Аксесуари",
            "unit_price": NumberDecimal("2500.00"),
            "quantity": 1,
            "subtotal": NumberDecimal("2500.00")
        }
    ],

    // Вбудовуємо адреси доставки (фіксуються на момент замовлення)
    "shipping_address": {
        "street": "вул. Хрещатик, 15",
        "city": "Київ",
        "postal_code": "01001",
        "country": "Україна"
    },

    "billing_address": {
        "street": "вул. Хрещатик, 15",
        "city": "Київ",
        "postal_code": "01001",
        "country": "Україна"
    },

    // Підсумкові дані
    "total_amount": NumberDecimal("47500.00"),
    "shipping_cost": NumberDecimal("0.00"),
    "tax_amount": NumberDecimal("0.00"),

    // Історія статусів
    "status_history": [
        {
            "status": "pending",
            "timestamp": ISODate("2024-10-15T10:00:00Z"),
            "note": "Замовлення створено"
        },
        {
            "status": "processing",
            "timestamp": ISODate("2024-10-15T11:30:00Z"),
            "note": "Оплата підтверджена"
        },
        {
            "status": "shipped",
            "timestamp": ISODate("2024-10-16T09:00:00Z"),
            "note": "Відправлено, трек-номер: UA123456789"
        }
    ]
}
```

**Стратегія 3: Гібридний підхід для товарів**

Для товарів, які використовуються як у каталозі, так і в замовленнях, використовуємо комбінацію підходів.

```javascript
// Основна колекція products - повна інформація про товар
{
    "_id": ObjectId("507f1f77bcf86cd799439030"),
    "product_id": "PROD001",
    "sku": "DELL-XPS-15-2024",
    "name": "Ноутбук Dell XPS 15",
    "description": "Потужний ноутбук для професіоналів...",

    // Поточна ціна (змінюється)
    "current_price": NumberDecimal("45000.00"),
    "discount_price": NumberDecimal("42000.00"),

    // Категорія з денормалізованою інформацією
    "category": {
        "category_id": "CAT001",
        "name": "Ноутбуки",
        "path": ["Електроніка", "Комп'ютери", "Ноутбуки"]
    },

    // Характеристики
    "specifications": {
        "processor": "Intel Core i7-13700H",
        "ram": "32GB DDR5",
        "storage": "1TB NVMe SSD",
        "display": "15.6\" 4K OLED",
        "graphics": "NVIDIA RTX 4060"
    },

    // Зображення
    "images": [
        {
            "url": "https://cdn.example.com/products/prod001-1.jpg",
            "alt": "Ноутбук Dell XPS 15 - основне фото",
            "is_primary": true
        },
        {
            "url": "https://cdn.example.com/products/prod001-2.jpg",
            "alt": "Ноутбук Dell XPS 15 - вигляд збоку",
            "is_primary": false
        }
    ],

    // Складські дані
    "inventory": {
        "in_stock": 15,
        "reserved": 3,
        "available": 12,
        "warehouse_locations": ["WAREHOUSE-A", "WAREHOUSE-B"]
    },

    // Агрегована інформація про відгуки
    "reviews_summary": {
        "count": 127,
        "average_rating": NumberDecimal("4.6"),
        "rating_distribution": {
            "5": 89,
            "4": 28,
            "3": 7,
            "2": 2,
            "1": 1
        }
    },

    // Оптимізація пошуку
    "search_tags": [
        "ноутбук", "laptop", "dell", "xps", "gaming",
        "професійний", "4k", "oled"
    ],

    "created_at": ISODate("2024-08-01T00:00:00Z"),
    "updated_at": ISODate("2024-10-15T12:00:00Z")
}

// Окрема колекція reviews для детальних відгуків
{
    "_id": ObjectId("..."),
    "review_id": "REV001",
    "product_id": "PROD001",

    // Денормалізація базової інформації про товар
    "product_name": "Ноутбук Dell XPS 15",
    "product_sku": "DELL-XPS-15-2024",

    // Інформація про автора
    "author": {
        "user_id": "U005",
        "name": "Марія К.",
        "verified_purchase": true
    },

    "rating": 5,
    "title": "Відмінний ноутбук для роботи та розваг",
    "comment": "Купила місяць тому, дуже задоволена...",
    "helpful_count": 23,
    "verified_purchase": true,
    "created_at": ISODate("2024-09-15T14:30:00Z"),

    // Відповіді на відгук
    "responses": [
        {
            "response_id": "RESP001",
            "author_type": "seller",
            "text": "Дякуємо за відгук!",
            "created_at": ISODate("2024-09-16T10:00:00Z")
        }
    ]
}
```

### Процес міграції даних

Практична міграція даних вимагає ретельного планування та поетапного виконання.

```javascript
// Етап 1: Експорт даних з реляційної БД
// Скрипт для експорту даних у JSON формат

const mysql = require('mysql2/promise')
const fs = require('fs').promises

async function exportUsers() {
    const connection = await mysql.createConnection({
        host: 'localhost',
        user: 'root',
        database: 'ecommerce'
    })

    // Експортуємо користувачів з адресами
    const [users] = await connection.execute(`
        SELECT u.*,
               JSON_ARRAYAGG(
                   JSON_OBJECT(
                       'address_id', a.address_id,
                       'type', a.type,
                       'street', a.street,
                       'city', a.city,
                       'postal_code', a.postal_code,
                       'country', a.country,
                       'is_default', a.is_default
                   )
               ) as addresses
        FROM users u
        LEFT JOIN addresses a ON u.user_id = a.user_id
        GROUP BY u.user_id
    `)

    await fs.writeFile('users_export.json', JSON.stringify(users, null, 2))
    await connection.end()
}

// Етап 2: Трансформація даних
async function transformUsers() {
    const usersData = JSON.parse(
        await fs.readFile('users_export.json', 'utf8')
    )

    const transformedUsers = usersData.map(user => ({
        user_id: user.user_id,
        email: user.email,
        password_hash: user.password_hash,
        first_name: user.first_name,
        last_name: user.last_name,
        created_at: new Date(user.created_at),
        addresses: JSON.parse(user.addresses).filter(a => a.address_id !== null),
        account_status: 'active',
        preferences: {
            newsletter: true,
            notifications: true
        }
    }))

    await fs.writeFile(
        'users_transformed.json',
        JSON.stringify(transformedUsers, null, 2)
    )
}

// Етап 3: Імпорт у MongoDB
const { MongoClient } = require('mongodb')

async function importToMongoDB() {
    const client = new MongoClient('mongodb://localhost:27017')
    await client.connect()

    const db = client.db('ecommerce_nosql')
    const usersCollection = db.collection('users')

    const transformedUsers = JSON.parse(
        await fs.readFile('users_transformed.json', 'utf8')
    )

    // Створюємо індекси перед імпортом
    await usersCollection.createIndex({ user_id: 1 }, { unique: true })
    await usersCollection.createIndex({ email: 1 }, { unique: true })

    // Імпортуємо дані батчами
    const batchSize = 1000
    for (let i = 0; i < transformedUsers.length; i += batchSize) {
        const batch = transformedUsers.slice(i, i + batchSize)
        await usersCollection.insertMany(batch, { ordered: false })
        console.log(`Imported ${Math.min(i + batchSize, transformedUsers.length)} users`)
    }

    await client.close()
}

// Повний процес міграції
async function migrateData() {
    console.log('Етап 1: Експорт даних...')
    await exportUsers()

    console.log('Етап 2: Трансформація...')
    await transformUsers()

    console.log('Етап 3: Імпорт у MongoDB...')
    await importToMongoDB()

    console.log('Міграція завершена!')
}
```

### Валідація після міграції

Після міграції необхідно перевірити цілісність та повноту даних.

```javascript
// Скрипт валідації
async function validateMigration() {
    const mysqlConnection = await mysql.createConnection({
        host: 'localhost',
        user: 'root',
        database: 'ecommerce'
    })

    const mongoClient = new MongoClient('mongodb://localhost:27017')
    await mongoClient.connect()
    const db = mongoClient.db('ecommerce_nosql')

    // Перевірка 1: Кількість записів
    const [mysqlCount] = await mysqlConnection.execute(
        'SELECT COUNT(*) as count FROM users'
    )
    const mongoCount = await db.collection('users').countDocuments()

    console.log(`MySQL users: ${mysqlCount[0].count}`)
    console.log(`MongoDB users: ${mongoCount}`)

    if (mysqlCount[0].count !== mongoCount) {
        console.error('Кількість користувачів не співпадає!')
    }

    // Перевірка 2: Вибіркова перевірка даних
    const [sampleUsers] = await mysqlConnection.execute(
        'SELECT * FROM users ORDER BY RAND() LIMIT 100'
    )

    for (const mysqlUser of sampleUsers) {
        const mongoUser = await db.collection('users').findOne({
            user_id: mysqlUser.user_id
        })

        if (!mongoUser) {
            console.error(`Користувач ${mysqlUser.user_id} не знайдений у MongoDB`)
            continue
        }

        // Порівнюємо основні поля
        if (mysqlUser.email !== mongoUser.email) {
            console.error(`Email не співпадає для user ${mysqlUser.user_id}`)
        }

        // Перевірка адрес
        const [addresses] = await mysqlConnection.execute(
            'SELECT COUNT(*) as count FROM addresses WHERE user_id = ?',
            [mysqlUser.user_id]
        )

        if (addresses[0].count !== mongoUser.addresses.length) {
            console.error(
                `Кількість адрес не співпадає для user ${mysqlUser.user_id}`
            )
        }
    }

    console.log('Валідація завершена')

    await mysqlConnection.end()
    await mongoClient.close()
}
```

## Паттерни проєктування для специфічних сценаріїв

### Паттерн "Subset"

Паттерн Subset використовується, коли документ містить багато інформації, але більшість запитів потребує тільки її частини.

```javascript
// Проблема: Великий документ з багатьма даними
{
    "product_id": "PROD001",
    "name": "Ноутбук Dell XPS 15",
    "reviews": [
        // Може бути тисячі відгуків, кожен займає кілька KB
        { "review_id": "REV001", "rating": 5, "comment": "..." },
        { "review_id": "REV002", "rating": 4, "comment": "..." },
        // ... 5000 відгуків
    ],
    "specifications": {
        // Детальні характеристики
    },
    "detailed_description": "..." // Великий текст
}

// Рішення: Subset Pattern
// Основний документ з підмножиною даних
{
    "product_id": "PROD001",
    "name": "Ноутбук Dell XPS 15",
    "price": NumberDecimal("45000.00"),
    "category": "Ноутбуки",

    // Тільки останні 10 відгуків для швидкого відображення
    "recent_reviews": [
        { "review_id": "REV5000", "rating": 5, "comment": "Відмінно!", "date": ISODate("2024-10-17") },
        { "review_id": "REV4999", "rating": 4, "comment": "Добре", "date": ISODate("2024-10-16") }
        // ... тільки 10 найсвіжіших
    ],

    // Агрегована інформація
    "reviews_summary": {
        "total_count": 5000,
        "average_rating": 4.6
    },

    // Короткий опис для списків
    "short_description": "Потужний професійний ноутбук...",

    // Посилання на повні дані
    "full_details_available": true
}

// Окремий документ з повними даними (завантажується при потребі)
{
    "product_id": "PROD001",
    "detailed_description": "...", // Повний детальний опис
    "full_specifications": {
        // Всі детальні характеристики
    },
    "installation_guide": "...",
    "warranty_info": "..."
}

// Колекція всіх відгуків
{
    "review_id": "REV001",
    "product_id": "PROD001",
    "rating": 5,
    "comment": "...",
    "created_at": ISODate("...")
}
```

Використання Subset Pattern:

```javascript
// Показ списку товарів - швидко, один запит
const products = await db.products.find(
    { category: "Ноутбуки" },
    {
        projection: {
            name: 1,
            price: 1,
            "reviews_summary.average_rating": 1,
            short_description: 1
        }
    }
).limit(20).toArray()

// Показ сторінки товару - два запити при потребі
const product = await db.products.findOne({ product_id: "PROD001" })

// Повні деталі завантажуються тільки якщо користувач клікає "Докладніше"
if (userClickedMore) {
    const fullDetails = await db.product_details.findOne({
        product_id: "PROD001"
    })
}

// Всі відгуки завантажуються окремо при кліку "Показати всі відгуки"
if (userWantsAllReviews) {
    const allReviews = await db.reviews.find({
        product_id: "PROD001"
    }).sort({ created_at: -1 }).toArray()
}
```

### Паттерн "Computed"

Паттерн Computed передбачає зберігання попередньо обчислених значень для уникнення дорогих обчислень при кожному запиті.

```javascript
// Проблема: Дорогі обчислення при кожному запиті
// Щоразу при перегляді профілю користувача треба рахувати статистику

// Рішення: Computed Pattern
{
    "user_id": "U001",
    "username": "ivanpetrov",
    "email": "ivan@example.com",

    // Попередньо обчислені метрики
    "computed_stats": {
        "total_posts": 127,
        "total_comments": 543,
        "total_likes_received": 1247,
        "followers_count": 523,
        "following_count": 342,

        // Складніші метрики
        "engagement_rate": NumberDecimal("3.45"), // Обчислено на основі лайків/переглядів
        "avg_post_likes": NumberDecimal("9.82"),
        "most_active_hours": [14, 15, 16, 20, 21], // Години найбільшої активності

        // Часова мітка останнього оновлення
        "last_computed": ISODate("2024-10-17T14:30:00Z")
    },

    // Тижнева статистика
    "weekly_stats": {
        "posts_this_week": 12,
        "comments_this_week": 47,
        "new_followers_this_week": 23,
        "week_start": ISODate("2024-10-14T00:00:00Z")
    }
}

// Фонова задача для оновлення обчислених значень
async function updateUserStats(userId) {
    const stats = await db.posts.aggregate([
        { $match: { "author.user_id": userId } },
        {
            $group: {
                _id: null,
                total_posts: { $sum: 1 },
                total_likes: { $sum: "$likes_count" },
                avg_likes: { $avg: "$likes_count" }
            }
        }
    ]).toArray()

    const commentsCount = await db.comments.countDocuments({
        "author.user_id": userId
    })

    await db.users.updateOne(
        { user_id: userId },
        {
            $set: {
                "computed_stats.total_posts": stats[0].total_posts,
                "computed_stats.total_likes_received": stats[0].total_likes,
                "computed_stats.avg_post_likes": stats[0].avg_likes,
                "computed_stats.total_comments": commentsCount,
                "computed_stats.last_computed": new Date()
            }
        }
    )
}
```

Стратегії оновлення обчислених значень:

```javascript
// Стратегія 1: Оновлення при кожній зміні (синхронно)
async function createPost(userId, postData) {
    // Створюємо пост
    await db.posts.insertOne({
        ...postData,
        author_id: userId
    })

    // Одразу оновлюємо лічильник
    await db.users.updateOne(
        { user_id: userId },
        {
            $inc: { "computed_stats.total_posts": 1 },
            $set: { "computed_stats.last_computed": new Date() }
        }
    )
}

// Стратегія 2: Асинхронне оновлення через чергу
async function createPostAsync(userId, postData) {
    // Створюємо пост
    await db.posts.insertOne({
        ...postData,
        author_id: userId
    })

    // Додаємо завдання в чергу для оновлення статистики
    await jobQueue.add('update-user-stats', { userId })
}

// Стратегія 3: Періодичне оновлення (cron job)
// Кожні 5 хвилин оновлюємо статистику для активних користувачів
cron.schedule('*/5 * * * *', async () => {
    const activeUsers = await db.users.find({
        last_activity: { $gte: new Date(Date.now() - 24 * 60 * 60 * 1000) }
    }).toArray()

    for (const user of activeUsers) {
        await updateUserStats(user.user_id)
    }
})

// Стратегія 4: Оновлення на вимогу з кешуванням
async function getUserStats(userId) {
    const user = await db.users.findOne({ user_id: userId })

    const cacheAge = Date.now() - user.computed_stats.last_computed
    const maxCacheAge = 60 * 60 * 1000 // 1 година

    // Якщо кеш застарів, оновлюємо
    if (cacheAge > maxCacheAge) {
        await updateUserStats(userId)
        return await db.users.findOne({ user_id: userId })
    }

    return user
}
```

### Паттерн "Bucket"

Паттерн Bucket використовується для групування пов'язаних документів, зокрема для часових рядів даних.

```javascript
// Проблема: Мільйони окремих вимірювань від IoT пристроїв
// Без Bucket Pattern: 1 документ = 1 вимірювання
{
    "sensor_id": "TEMP001",
    "timestamp": ISODate("2024-10-17T14:30:00Z"),
    "temperature": 22.5,
    "humidity": 45
}
// При 1 вимірюванні кожну секунду = 86,400 документів на день на сенсор
// 1000 сенсорів = 86,400,000 документів на день

// Рішення: Bucket Pattern
// 1 документ = 1 година вимірювань
{
    "_id": ObjectId("..."),
    "sensor_id": "TEMP001",
    "bucket_start": ISODate("2024-10-17T14:00:00Z"),
    "bucket_end": ISODate("2024-10-17T15:00:00Z"),

    // Масив вимірювань за годину (3600 значень)
    "measurements": [
        {
            "offset": 0,     // Секунди від початку години
            "temp": 22.5,
            "humidity": 45
        },
        {
            "offset": 1,
            "temp": 22.6,
            "humidity": 45
        },
        // ... до 3600 вимірювань
    ],

    // Передобчислена статистика для швидких запитів
    "summary": {
        "count": 3600,
        "temp_min": 21.8,
        "temp_max": 23.2,
        "temp_avg": 22.4,
        "humidity_min": 43,
        "humidity_max": 47,
        "humidity_avg": 45.2
    }
}

// Оптимізована вставка даних
async function recordMeasurement(sensorId, temperature, humidity) {
    const now = new Date()
    const bucketStart = new Date(now)
    bucketStart.setMinutes(0, 0, 0) // Округлюємо до початку години

    const bucketEnd = new Date(bucketStart)
    bucketEnd.setHours(bucketEnd.getHours() + 1)

    const offset = (now - bucketStart) / 1000 // Секунди від початку години

    await db.sensor_data.updateOne(
        {
            sensor_id: sensorId,
            bucket_start: bucketStart
        },
        {
            $setOnInsert: {
                sensor_id: sensorId,
                bucket_start: bucketStart,
                bucket_end: bucketEnd
            },
            $push: {
                measurements: {
                    offset: offset,
                    temp: temperature,
                    humidity: humidity,
                    timestamp: now
                }
            },
            $min: { "summary.temp_min": temperature },
            $max: { "summary.temp_max": temperature },
            $inc: { "summary.count": 1 }
        },
        { upsert: true }
    )
}

// Швидкий запит середніх значень за день
db.sensor_data.aggregate([
    {
        $match: {
            sensor_id: "TEMP001",
            bucket_start: {
                $gte: ISODate("2024-10-17T00:00:00Z"),
                $lt: ISODate("2024-10-18T00:00:00Z")
            }
        }
    },
    {
        $group: {
            _id: null,
            avg_temp: { $avg: "$summary.temp_avg" },
            min_temp: { $min: "$summary.temp_min" },
            max_temp: { $max: "$summary.temp_max" }
        }
    }
])
```

Варіації Bucket Pattern:

```javascript
// Варіація 1: Часові bucket'и змінного розміру
// Для старих даних використовуємо більші bucket'и

// Свіжі дані: bucket по 1 годині
{
    "sensor_id": "TEMP001",
    "bucket_type": "hourly",
    "bucket_start": ISODate("2024-10-17T14:00:00Z"),
    "measurements": [/* до 3600 вимірювань */]
}

// Дані тижневої давності: агреговані bucket'и по 1 дню
{
    "sensor_id": "TEMP001",
    "bucket_type": "daily",
    "bucket_start": ISODate("2024-10-10T00:00:00Z"),
    "hourly_summaries": [
        {
            "hour": 0,
            "temp_avg": 22.4,
            "temp_min": 21.8,
            "temp_max": 23.2
        },
        // ... 24 години
    ]
}

// Дані місячної давності: bucket'и по тижню
{
    "sensor_id": "TEMP001",
    "bucket_type": "weekly",
    "bucket_start": ISODate("2024-09-16T00:00:00Z"),
    "daily_summaries": [/* 7 днів */]
}

// Варіація 2: Schema Versioning в bucket'ах
{
    "sensor_id": "TEMP001",
    "schema_version": 2,  // Версія структури даних
    "bucket_start": ISODate("2024-10-17T14:00:00Z"),

    // Нова версія може мати додаткові поля
    "measurements": [
        {
            "offset": 0,
            "temp": 22.5,
            "humidity": 45,
            "pressure": 1013,  // Нове поле у версії 2
            "air_quality": 85  // Нове поле у версії 2
        }
    ]
}
```

### Паттерн "Schema Versioning"

Паттерн Schema Versioning дозволяє еволюціонувати схему даних без необхідності одноча сної міграції всіх документів.

```javascript
// Версія 1 схеми користувача (початкова)
{
    "user_id": "U001",
    "schema_version": 1,
    "username": "ivanpetrov",
    "email": "ivan@example.com",
    "created_at": ISODate("2024-01-01T00:00:00Z")
}

// Версія 2: Додано структуру для імені
{
    "user_id": "U002",
    "schema_version": 2,
    "username": "mariakoval",
    "email": "maria@example.com",
    "name": {
        "first": "Марія",
        "last": "Коваленко"
    },
    "created_at": ISODate("2024-06-01T00:00:00Z")
}

// Версія 3: Додано налаштування приватності
{
    "user_id": "U003",
    "schema_version": 3,
    "username": "alexsid",
    "email": "alex@example.com",
    "name": {
        "first": "Олександр",
        "last": "Сидоров"
    },
    "privacy_settings": {
        "profile_visibility": "public",
        "show_email": false,
        "allow_messages": true
    },
    "created_at": ISODate("2024-09-01T00:00:00Z")
}

// Обробка різних версій схеми
function normalizeUser(user) {
    // Клонуємо об'єкт
    const normalized = { ...user }

    // Мігруємо версію 1 до версії 3
    if (user.schema_version === 1) {
        // Додаємо структуру name
        normalized.name = {
            first: user.username,
            last: ""
        }

        // Додаємо privacy_settings з дефолтними значеннями
        normalized.privacy_settings = {
            profile_visibility: "public",
            show_email: false,
            allow_messages: true
        }

        normalized.schema_version = 3
    }

    // Мігруємо версію 2 до версії 3
    if (user.schema_version === 2) {
        normalized.privacy_settings = {
            profile_visibility: "public",
            show_email: false,
            allow_messages: true
        }

        normalized.schema_version = 3
    }

    return normalized
}

// Використання при читанні
async function getUser(userId) {
    const user = await db.users.findOne({ user_id: userId })
    return normalizeUser(user)
}

// Поступова міграція (lazy migration)
async function updateUser(userId, updates) {
    const user = await db.users.findOne({ user_id: userId })
    const normalized = normalizeUser(user)

    // Оновлюємо документ до поточної версії схеми
    await db.users.updateOne(
        { user_id: userId },
        {
            $set: {
                ...normalized,
                ...updates,
                schema_version: 3
            }
        }
    )
}

// Пакетна міграція старих документів (background job)
async function migrateOldSchemas() {
    const oldUsers = db.users.find({
        schema_version: { $lt: 3 }
    })

    for await (const user of oldUsers) {
        const normalized = normalizeUser(user)

        await db.users.updateOne(
            { _id: user._id },
            {
                $set: {
                    ...normalized,
                    schema_version: 3
                }
            }
        )
    }
}
```

### Паттерн "Polymorphic"

Паттерн Polymorphic дозволяє зберігати документи різних типів в одній колекції, зберігаючи спільну структуру.

```javascript
// Колекція notifications містить різні типи сповіщень

// Тип 1: Новий коментар
{
    "_id": ObjectId("..."),
    "type": "new_comment",
    "user_id": "U001",
    "created_at": ISODate("2024-10-17T14:30:00Z"),
    "read": false,

    // Специфічні поля для коментаря
    "post_id": "P123",
    "post_title": "Введення в NoSQL",
    "commenter": {
        "user_id": "U002",
        "username": "mariakoval",
        "avatar_url": "..."
    },
    "comment_preview": "Відмінна стаття! Дуже корисна..."
}

// Тип 2: Новий підписник
{
    "_id": ObjectId("..."),
    "type": "new_follower",
    "user_id": "U001",
    "created_at": ISODate("2024-10-17T15:00:00Z"),
    "read": false,

    // Специфічні поля для підписника
    "follower": {
        "user_id": "U003",
        "username": "alexsid",
        "avatar_url": "...",
        "bio": "Software Engineer"
    }
}

// Тип 3: Системне сповіщення
{
    "_id": ObjectId("..."),
    "type": "system_announcement",
    "user_id": "U001",
    "created_at": ISODate("2024-10-17T16:00:00Z"),
    "read": false,

    // Специфічні поля для системного сповіщення
    "title": "Нові функції платформи",
    "message": "Ми додали можливість...",
    "action_url": "/features/new",
    "priority": "medium"
}

// Обробка різних типів сповіщень
function renderNotification(notification) {
    switch (notification.type) {
        case 'new_comment':
            return {
                icon: 'comment',
                title: `${notification.commenter.username} прокоментував ваш пост`,
                description: notification.comment_preview,
                link: `/posts/${notification.post_id}`
            }

        case 'new_follower':
            return {
                icon: 'user-plus',
                title: `${notification.follower.username} підписався на вас`,
                description: notification.follower.bio,
                link: `/users/${notification.follower.username}`
            }

        case 'system_announcement':
            return {
                icon: 'bell',
                title: notification.title,
                description: notification.message,
                link: notification.action_url
            }

        default:
            return {
                icon: 'info',
                title: 'Нове сповіщення',
                description: '',
                link: '/'
            }
    }
}

// Запити для різних типів
// Всі непрочитані сповіщення
db.notifications.find({
    user_id: "U001",
    read: false
}).sort({ created_at: -1 })

// Тільки коментарі
db.notifications.find({
    user_id: "U001",
    type: "new_comment",
    read: false
})

// Створення індексів для поліморфної колекції
db.notifications.createIndex({ user_id: 1, created_at: -1 })
db.notifications.createIndex({ user_id: 1, type: 1, read: 1 })
db.notifications.createIndex({ user_id: 1, read: 1 })
```

### Паттерн "Extended Reference"

Паттерн Extended Reference є розширенням гібридного підходу, де в посилання включається додаткова інформація для уникнення lookup операцій.

```javascript
// Замість простого посилання
{
    "order_id": "ORD001",
    "customer_id": "U001"  // Тільки ID
}

// Extended Reference включає ключову інформацію
{
    "order_id": "ORD001",
    "customer": {
        "user_id": "U001",
        "username": "ivanpetrov",
        "full_name": "Іван Петров",
        "email": "ivan@example.com",
        "verified": true,
        "tier": "premium"
    },
    "created_at": ISODate("2024-10-17T10:00:00Z"),
    "status": "processing"
}

// Переваги:
// 1. Швидке відображення замовлення без додаткового запиту
// 2. Критична інформація про клієнта доступна одразу
// 3. Можна фільтрувати замовлення за tier клієнта

// Знайти всі замовлення преміум клієнтів
db.orders.find({
    "customer.tier": "premium",
    status: "processing"
})

// Для повної інформації про клієнта все ще робимо lookup
const fullCustomer = await db.users.findOne({
    user_id: order.customer.user_id
})
```

## Висновки

Проєктування схем даних для документо-орієнтованих СУБД є фундаментально іншим процесом порівняно з реляційними системами. Замість того, щоб фокусуватися на нормалізації та усуненні дублювання даних, ми оптимізуємо структуру даних для конкретних патернів доступу додатка.

Вибір між вбудовуванням та посиланнями має базуватися на детальному аналізі декількох факторів: як часто дані читаються та оновлюються, який розмір мають пов'язані дані, чи використовуються вони завжди разом, та яка критичність продуктивності для конкретних операцій. Гібридний підхід, що поєднує обидві стратегії, часто є найефективнішим рішенням для реальних застосувань.

Денормалізація в NoSQL системах є не порушенням принципів проєктування, а свідомою стратегією оптимізації. Ключем до успішної денормалізації є ретельне планування стратегій підтримки консистентності та розуміння компромісів між продуктивністю читання та складністю оновлень.

Міграція від реляційних до документо-орієнтованих схем вимагає не просто технічного перенесення даних, а переосмислення моделі даних з урахуванням особливостей NoSQL систем. Процес має включати аналіз існуючих патернів доступу, трансформацію зв'язків між сутностями та ретельну валідацію результатів.

Використання усталених паттернів проєктування, таких як Subset, Computed, Bucket, Schema Versioning, Polymorphic та Extended Reference, допомагає вирішувати типові проблеми ефективно та передбачувано. Ці паттерни є результатом досвіду багатьох проєктів та надають перевірені рішення для конкретних сценаріїв використання.

Успішне проєктування NoSQL схем потребує глибокого розуміння як предметної області, так і технічних можливостей документо-орієнтованих систем. Інвестування часу в ретельне проєктування на початковому етапі окупається значним покращенням продуктивності та зниженням складності підтримки системи в довгостроковій перспективі.
