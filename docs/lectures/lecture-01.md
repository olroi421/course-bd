# Лекція 1. Вступ до сучасної веброзробки

## Архітектура сучасних вебдодатків

### Визначення вебдодатку

**Вебдодаток** — це програмне забезпечення, яке працює на вебсервері та доступне користувачам через веббраузер без необхідності встановлення на локальному комп'ютері.

### Еволюція вебдодатків

#### 1. Статичні вебсайти (1990-ті)

Характеристики:

- HTML-файли зберігаються на сервері;
- Відсутність інтерактивності;
- Контент однаковий для всіх користувачів.

Приклад структури:
```
website/
├── index.html
├── about.html
├── contact.html
├── css/
│   └── style.css
└── images/
    └── logo.png
```

#### 2. Динамічні вебсайти (2000-ні)

Характеристики:

- Серверні мови програмування (PHP, ASP, JSP);
- Бази даних для зберігання контенту;
- Персоналізований контент.

Приклад технологічного стеку:
```
LAMP Stack:
├── Linux (операційна система)
├── Apache (вебсервер)
├── MySQL (база даних)
└── PHP (серверна мова)
```

#### 3. Вебдодатки (2010-ні)

Характеристики:

- Багата інтерактивність;
- AJAX для асинхронних запитів;
- RESTful API;
- Розділення frontend та backend.

### Сучасна архітектура вебдодатків

```mermaid
graph TB
    subgraph "Client Layer"
        A[🌐 Web Browser]
        B[📱 Mobile App]
        C[💻 Desktop App]
    end

    subgraph "Presentation Layer"
        D[⚡ CDN]
        E[🎨 Frontend Application]
    end

    subgraph "Application Layer"
        F[🔄 Load Balancer]
        G[⚙️ API Gateway]
        H[🛡️ Authentication Service]
        I[📊 Business Logic Services]
    end

    subgraph "Data Layer"
        J[💾 Primary Database]
        K[⚡ Cache Layer]
        L[📈 Analytics DB]
        M[📁 File Storage]
    end

    A --> D
    B --> D
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    G --> I
    I --> J
    I --> K
    I --> L
    I --> M
```

### Детальний аналіз архітектурних компонентів

#### 1. **Client Layer (Клієнтський рівень)**

Включає всі пристрої та додатки, через які користувачі взаємодіють з вебдодатком:

- **Веббраузери**: Chrome, Firefox, Safari, Edge;
- **Мобільні додатки**: нативні iOS/Android додатки;
- **Desktop додатки**: Electron-based додатки (Slack, Discord, VS Code).

#### 2. **Presentation Layer (Рівень представлення)**
Відповідає за відображення інтерфейсу користувача:

- **CDN (Content Delivery Network)**: глобальна мережа серверів для швидкої доставки статичного контенту;
- **Frontend Application**: React, Vue.js, Angular додатки.

Приклад CDN архітектури:
```mermaid
graph LR
    A[👤 Користувач в Україні] --> B[🌍 Київський CDN сервер]
    C[👤 Користувач в США] --> D[🌍 Нью-Йоркський CDN сервер]
    E[👤 Користувач в Японії] --> F[🌍 Токійський CDN сервер]

    B --> G[☁️ Origin Server]
    D --> G
    F --> G
```

#### 3. **Application Layer (Рівень додатку)**
Містить бізнес-логіку та обробляє запити.

Load Balancer (Балансувальник навантаження):

- Розподіляє запити між серверами;
- Забезпечує високу доступність;
- Приклади: NGINX, HAProxy, AWS ELB.

```mermaid
graph LR
    A[🌐 Internet] --> B[⚖️ Load Balancer]
    B --> C[🖥️ Server 1]
    B --> D[🖥️ Server 2]
    B --> E[🖥️ Server 3]

    F[📊 Health Checker] --> C
    F --> D
    F --> E
```

API Gateway (API-шлюз):

- Єдина точка входу для всіх API запитів;
- Автентифікація та авторизація;
- Моніторинг та аналітика;
- Rate limiting (обмеження кількості запитів).


Приклад конфігурації API Gateway:

```yaml
# Приклад конфігурації Kong API Gateway
services:
  - name: user-service
    url: http://user-service:3000
    routes:
      - name: users
        paths: ["/api/users"]
        methods: ["GET", "POST", "PUT", "DELETE"]
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          hour: 1000
```

#### 4. **Data Layer (Рівень даних)**

Зберігає та управляє всіма даними додатку:

- **Primary Database**: основне сховище даних (PostgreSQL, MySQL, MongoDB);
- **Cache Layer**: швидкий доступ до часто запитуваних даних (Redis, Memcached);
- **Analytics Database**: спеціалізовані БД для аналітики (ClickHouse, BigQuery);
- **File Storage**: зберігання файлів (AWS S3, Google Cloud Storage).

### Мікросервісна архітектура

Сучасні великі вебдодатки часто використовують мікросервісну архітектуру:

```mermaid
graph TB
    subgraph "Frontend"
        A[🎨 React SPA]
    end

    subgraph "API Gateway"
        B[🚪 Kong/Zuul]
    end

    subgraph "Microservices"
        C[👤 User Service]
        D[📦 Product Service]
        E[🛒 Cart Service]
        F[💳 Payment Service]
        G[📧 Notification Service]
    end

    subgraph "Data Layer"
        H[(👤 Users DB)]
        I[(📦 Products DB)]
        J[(🛒 Cart Cache)]
        K[💳 Payment API]
        L[📧 Email Queue]
    end

    A --> B
    B --> C
    B --> D
    B --> E
    B --> F
    B --> G

    C --> H
    D --> I
    E --> J
    F --> K
    G --> L
```

Переваги мікросервісної архітектури:

- **Масштабованість**: кожен сервіс можна масштабувати незалежно;
- **Технологічна гнучкість**: різні сервіси можуть використовувати різні технології;
- **Ізольованість**: збій одного сервісу не впливає на інші;
- **Швидкість розробки**: команди працюють незалежно.

Недоліки:
- **Складність**: потребує досвіду в DevOps та розподілених системах;
- **Мережеві затримки**: комунікація між сервісами через мережу;
- **Складність тестування**: інтеграційне тестування стає складнішим.

## Client-Server взаємодія та HTTP протокол

### Основи Client-Server моделі

**Client-Server модель** — це архітектурна модель, де клієнт ініціює запити до сервера, який обробляє ці запити та повертає відповіді.

```mermaid
sequenceDiagram
    participant C as 🌐 Client (Browser)
    participant S as 🖥️ Server

    C->>S: HTTP Request
    Note over C,S: GET /api/users

    S-->>S: Process Request
    Note over S: Database Query,<br/>Business Logic

    S->>C: HTTP Response
    Note over C,S: 200 OK + JSON Data

    C-->>C: Render UI
    Note over C: Update DOM,<br/>Display Data
```

### HTTP протокол детально

#### Структура HTTP запиту

```
GET /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Content-Type: application/json

{
  "include": ["profile", "preferences"]
}
```

Компоненти HTTP запиту:

1. **Request Line**: метод + URL + версія протоколу;
2. **Headers**: додаткова інформація про запит;
3. **Body**: дані запиту (для POST, PUT, PATCH).

#### Структура HTTP відповіді

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 156
Cache-Control: max-age=3600
Set-Cookie: session_id=abc123; Path=/; HttpOnly

{
  "id": 123,
  "name": "Іван Петренко",
  "email": "ivan@example.com",
  "profile": {
    "avatar": "https://cdn.example.com/avatars/123.jpg"
  }
}
```

#### HTTP методи та їх застосування

| Метод | Призначення | Ідемпотентний | Безпечний | Приклад |
|-------|-------------|---------------|-----------|---------|
| **GET** | Отримання даних | ✅ | ✅ | `GET /api/users` |
| **POST** | Створення ресурсу | ❌ | ❌ | `POST /api/users` |
| **PUT** | Повне оновлення | ✅ | ❌ | `PUT /api/users/123` |
| **PATCH** | Часткове оновлення | ❌ | ❌ | `PATCH /api/users/123` |
| **DELETE** | Видалення ресурсу | ✅ | ❌ | `DELETE /api/users/123` |
| **OPTIONS** | Інформація про методи | ✅ | ✅ | `OPTIONS /api/users` |

Пояснення термінів:

- **Ідемпотентний**: повторний виклик дає той самий результат;
- **Безпечний**: не змінює стан сервера.

#### HTTP статус коди

```mermaid
graph LR
    A[📊 HTTP Status Codes] --> B[1xx Informational]
    A --> C[2xx Success]
    A --> D[3xx Redirection]
    A --> E[4xx Client Error]
    A --> F[5xx Server Error]

    C --> C1[200 OK]
    C --> C2[201 Created]
    C --> C3[204 No Content]

    D --> D1[301 Moved Permanently]
    D --> D2[302 Found]
    D --> D3[304 Not Modified]

    E --> E1[400 Bad Request]
    E --> E2[401 Unauthorized]
    E --> E3[403 Forbidden]
    E --> E4[404 Not Found]

    F --> F1[500 Internal Server Error]
    F --> F2[502 Bad Gateway]
    F --> F3[503 Service Unavailable]
```

Детальний розгляд важливих кодів:

**2xx Success:**

- **200 OK**: успішний запит;
- **201 Created**: ресурс створено;
- **204 No Content**: успішно, але немає контенту для повернення.

**4xx Client Error:**

- **400 Bad Request**: некоректний запит;
- **401 Unauthorized**: потрібна автентифікація;
- **403 Forbidden**: доступ заборонено;
- **404 Not Found**: ресурс не знайдено;
- **422 Unprocessable Entity**: помилки валідації.

**5xx Server Error:**

- **500 Internal Server Error**: помилка сервера;
- **502 Bad Gateway**: помилка проксі-сервера;
- **503 Service Unavailable**: сервер тимчасово недоступний.

### Асинхронна взаємодія з сервером

#### AJAX (Asynchronous JavaScript and XML)

**AJAX** дозволяє надсилати запити до сервера без перезавантаження сторінки:

```javascript
// Приклад використання Fetch API
async function fetchUsers() {
    try {
        const response = await fetch('/api/users', {
            method: 'GET',
            headers: {
                'Authorization': 'Bearer ' + token,
                'Content-Type': 'application/json'
            }
        });

        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }

        const users = await response.json();
        displayUsers(users);
    } catch (error) {
        console.error('Помилка завантаження користувачів:', error);
        showErrorMessage('Не вдалося завантажити користувачів');
    }
}
```

#### WebSockets для реального часу

Для додатків, що потребують взаємодії в реальному часі (чати, ігри, біржові дані):

```javascript
// Клієнтська частина
const socket = new WebSocket('ws://localhost:8080');

socket.onopen = function(event) {
    console.log('З\'єднання встановлено');
    socket.send(JSON.stringify({
        type: 'join_room',
        room: 'general'
    }));
};

socket.onmessage = function(event) {
    const message = JSON.parse(event.data);
    displayMessage(message);
};

socket.onclose = function(event) {
    console.log('З\'єднання закрито');
};
```

### Кешування в HTTP

Кешування покращує продуктивність вебдодатків:

```mermaid
sequenceDiagram
    participant B as 🌐 Browser
    participant C as 🗄️ Cache
    participant S as 🖥️ Server

    B->>C: Request /api/data
    alt Cache Hit
        C->>B: Cached Response
    else Cache Miss
        C->>S: Forward Request
        S->>C: Response + Cache Headers
        C->>B: Response
        C-->>C: Store in Cache
    end
```

**HTTP заголовки для кешування:**

```http
# Кешування на 1 годину
Cache-Control: max-age=3600

# Кешування з валідацією
Cache-Control: no-cache
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT

# Заборона кешування
Cache-Control: no-store, no-cache, must-revalidate
```

## REST API vs GraphQL vs gRPC

### REST API (Representational State Transfer)

**REST** — це архітектурний стиль для розробки вебсервісів, заснований на принципах HTTP.

#### Принципи REST:

1. **Stateless (Без стану)**: кожен запит містить всю необхідну інформацію;
2. **Resource-based (Ресурсно-орієнтований)**: URL представляють ресурси;
3. **HTTP методи**: використання стандартних HTTP методів;
4. **Representation (Представлення)**: різні формати даних (JSON, XML).

#### Приклад REST API:

```javascript
// Отримання списку користувачів
GET /api/users
Response: [
    { "id": 1, "name": "Іван", "email": "ivan@example.com" },
    { "id": 2, "name": "Марія", "email": "maria@example.com" }
]

// Отримання конкретного користувача
GET /api/users/1
Response: { "id": 1, "name": "Іван", "email": "ivan@example.com" }

// Створення нового користувача
POST /api/users
Body: { "name": "Петро", "email": "petro@example.com" }
Response: { "id": 3, "name": "Петро", "email": "petro@example.com" }

// Оновлення користувача
PUT /api/users/1
Body: { "name": "Іван Оновлений", "email": "ivan.new@example.com" }

// Видалення користувача
DELETE /api/users/1
Response: 204 No Content
```

#### Структура RESTful URL:

```
https://api.example.com/v1/users          # Колекція користувачів
https://api.example.com/v1/users/123      # Конкретний користувач
https://api.example.com/v1/users/123/posts # Пости користувача
https://api.example.com/v1/posts/456/comments # Коментарі до посту
```

Переваги REST:

- ✅ Простота та зрозумілість;
- ✅ Використання стандартних HTTP методів;
- ✅ Кешування на рівні HTTP;
- ✅ Широка підтримка інструментів.

Недоліки REST:

- ❌ Over-fetching: отримання зайвих даних;
- ❌ Under-fetching: недостатньо даних в одному запиті;
- ❌ Множинні запити для складних даних;
- ❌ Версіонування API може бути складним.

### GraphQL

**GraphQL** — це мова запитів та середовище виконання для API, розроблена Facebook.

#### Основні концепції GraphQL:

```graphql
# Схема GraphQL
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
  profile: Profile
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  createdAt: DateTime!
}

type Query {
  users: [User!]!
  user(id: ID!): User
  posts(limit: Int, offset: Int): [Post!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}
```

#### Приклади запитів GraphQL:

```graphql
# Отримання користувачів з постами
query GetUsersWithPosts {
  users {
    id
    name
    email
    posts {
      id
      title
      createdAt
    }
  }
}

# Отримання конкретного користувача з профілем
query GetUserProfile($userId: ID!) {
  user(id: $userId) {
    name
    email
    profile {
      avatar
      bio
      location
    }
  }
}

# Створення нового користувача
mutation CreateNewUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    email
    createdAt
  }
}
```

#### Архітектура GraphQL:

```mermaid
graph LR
    A[📱 Client] --> B[🔍 GraphQL Query]
    B --> C[⚙️ GraphQL Server]
    C --> D[🔧 Resolvers]
    D --> E[(💾 Database)]
    D --> F[🌐 External API]
    D --> G[📁 File System]

    C --> H[📊 Schema]
    H --> I[📝 Types]
    H --> J[❓ Queries]
    H --> K[✏️ Mutations]
    H --> L[📡 Subscriptions]
```

Переваги GraphQL:

- ✅ Точні дані: клієнт отримує тільки потрібні поля;
- ✅ Один endpoint для всіх запитів;
- ✅ Сильна типізація та інтроспекція;
- ✅ Підтримка real-time через subscriptions;
- ✅ Інструменти розробника (GraphiQL, Apollo DevTools).

Недоліки GraphQL:

- ❌ Складність кешування;
- ❌ Крива навчання вища за REST;
- ❌ Потенційні проблеми з продуктивністю (N+1 queries);
- ❌ Складність авторизації на рівні полів.

### gRPC (Google Remote Procedure Call)

**gRPC** — це високопродуктивний RPC фреймворк з відкритим кодом, розроблений Google.

#### Особливості gRPC:

- **Protocol Buffers (protobuf)**: бінарний формат серіалізації;
- **HTTP/2**: мультиплексування, стискання заголовків;
- **Багатомовність**: підтримка 10+ мов програмування;
- **Чотири типи комунікації**: Unary, Server Streaming, Client Streaming, Bidirectional Streaming.

#### Визначення сервісу в gRPC:

```protobuf
// user_service.proto
syntax = "proto3";

package user;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc GetUsers(GetUsersRequest) returns (stream User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (Empty);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}

message GetUserRequest {
  int32 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}
```

#### Типи викликів gRPC:

```mermaid
graph TB
    subgraph "gRPC Communication Types"
        A[🔄 Unary RPC<br/>Request → Response]
        B[📤 Server Streaming<br/>Request → Stream Response]
        C[📥 Client Streaming<br/>Stream Request → Response]
        D[🔄 Bidirectional Streaming<br/>Stream Request ↔ Stream Response]
    end
```

**Приклад клієнтського коду:**

```javascript
// Node.js gRPC client
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('user_service.proto');
const userService = grpc.loadPackageDefinition(packageDefinition).user;

const client = new userService.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Unary call
client.getUser({ id: 1 }, (error, response) => {
  if (!error) {
    console.log('User:', response);
  } else {
    console.error('Error:', error);
  }
});

// Server streaming
const call = client.getUsers({});
call.on('data', (user) => {
  console.log('Received user:', user);
});
call.on('end', () => {
  console.log('Stream ended');
});
```

### Порівняння REST, GraphQL та gRPC

| Критерій | REST | GraphQL | gRPC |
|----------|------|---------|------|
| **Протокол** | HTTP/1.1, HTTP/2 | HTTP/1.1, HTTP/2 | HTTP/2 |
| **Формат даних** | JSON, XML | JSON | Protocol Buffers (binary) |
| **Продуктивність** | Середня | Середня-Висока | Висока |
| **Кешування** | Відмінне | Складне | Обмежене |
| **Інструменти** | Широка підтримка | Растуча екосистема | Специфічні інструменти |
| **Крива навчання** | Низька | Середня | Висока |
| **Типізація** | Слабка | Сильна | Сильна |
| **Streaming** | Обмежено | Subscriptions | Нативно |

### Вибір підходу для різних сценаріїв

```mermaid
flowchart TD
    A[Вибір API стилю] --> B{Тип додатку?}

    B -->|Публічний API| C[REST]
    B -->|Мобільний додаток| D[GraphQL]
    B -->|Мікросервіси| E[gRPC]
    B -->|Web додаток| F{Складність даних?}

    F -->|Проста| C
    F -->|Складна| D

    C --> G[✅ Простота<br/>✅ Кешування<br/>✅ Стандарти]
    D --> H[✅ Гнучкість<br/>✅ Ефективність<br/>✅ Типізація]
    E --> I[✅ Продуктивність<br/>✅ Streaming<br/>✅ Типізація]
```

## SPA vs MPA vs SSR

### Single Page Application (SPA)

**SPA** — це вебдодаток, який завантажується один раз і динамічно оновлює контент без перезавантаження сторінки.

#### Архітектура SPA:

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant B as 🌐 Browser
    participant S as 🖥️ Server
    participant A as ⚛️ API

    U->>B: Перший візит
    B->>S: GET /
    S->>B: HTML + JS Bundle (React/Vue/Angular)
    B-->>B: Ініціалізація додатку

    U->>B: Навігація (/profile)
    B-->>B: Client-side routing
    B->>A: GET /api/user
    A->>B: JSON data
    B-->>B: Рендер компоненту
```

#### Приклад SPA структури (React):

```
spa-app/
├── public/
│   └── index.html                 # Єдина HTML сторінка
├── src/
│   ├── components/
│   │   ├── Header.jsx
│   │   ├── UserProfile.jsx
│   │   └── ProductList.jsx
│   ├── pages/
│   │   ├── HomePage.jsx
│   │   ├── ProfilePage.jsx
│   │   └── ProductsPage.jsx
│   ├── services/
│   │   └── api.js                 # API виклики
│   ├── App.jsx                    # Головний компонент
│   └── index.js                   # Точка входу
└── package.json
```

Переваги SPA:

- ✅ Швидка навігація після завантаження;
- ✅ Багата інтерактивність;
- ✅ Гарний користувацький досвід;
- ✅ Менше навантаження на сервер після початкового завантаження.

Недоліки SPA:

- ❌ Повільне початкове завантаження;
- ❌ Проблеми з SEO;
- ❌ Складність кешування;
- ❌ JavaScript необхідний для роботи.

### Multi-Page Application (MPA)

**MPA** — це традиційний вебдодаток, де кожна сторінка завантажується окремо з сервера.

#### Архітектура MPA:

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant B as 🌐 Browser
    participant S as 🖥️ Server

    U->>B: Візит домашньої сторінки
    B->>S: GET /
    S->>B: HTML сторінка (index.html)

    U->>B: Клік на "Про нас"
    B->>S: GET /about
    S->>B: HTML сторінка (about.html)

    U->>B: Клік на "Продукти"
    B->>S: GET /products
    S->>B: HTML сторінка з даними продуктів
```

#### Приклад MPA структури:

```
mpa-app/
├── pages/
│   ├── index.html                 # Домашня сторінка
│   ├── about.html                 # Про нас
│   ├── products.html              # Продукти
│   └── contact.html               # Контакти
├── assets/
│   ├── css/
│   │   └── styles.css
│   ├── js/
│   │   └── main.js
│   └── images/
├── server/
│   ├── routes/
│   │   ├── home.js
│   │   ├── products.js
│   │   └── contact.js
│   └── app.js
└── package.json
```

Переваги MPA:

- ✅ Відмінне SEO;
- ✅ Швидше початкове завантаження сторінки;
- ✅ Проста архітектура;
- ✅ Працює без JavaScript.

Недоліки MPA:

- ❌ Повільна навігація (перезавантаження сторінок);
- ❌ Дублювання коду між сторінками;
- ❌ Більше навантаження на сервер;
- ❌ Менш інтерактивний досвід.

### Server-Side Rendering (SSR)

**SSR** — це підхід, при якому HTML генерується на сервері для кожного запиту, що поєднує переваги SPA та MPA.

#### Архітектура SSR:

```mermaid
sequenceDiagram
    participant U as 👤 User
    participant B as 🌐 Browser
    participant S as 🖥️ SSR Server
    participant A as ⚛️ API

    U->>B: Перший візит
    B->>S: GET /products
    S->>A: Fetch data
    A->>S: Product data
    S-->>S: Render React/Vue component
    S->>B: Pre-rendered HTML + JS
    B-->>B: Hydration (активація JS)

    U->>B: Навігація (/profile)
    Note over B: Client-side navigation
    B->>A: GET /api/user
    A->>B: JSON data
    B-->>B: Client-side rendering
```

#### Приклад SSR з Next.js:

```javascript
// pages/products/[id].js - Next.js SSR
import { GetServerSideProps } from 'next';

export default function ProductPage({ product }) {
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p>Ціна: {product.price} грн</p>
    </div>
  );
}

// Ця функція виконується на сервері для кожного запиту
export const getServerSideProps = async (context) => {
  const { id } = context.params;

  try {
    const res = await fetch(`http://api.example.com/products/${id}`);
    const product = await res.json();

    return {
      props: {
        product
      }
    };
  } catch (error) {
    return {
      notFound: true
    };
  }
};
```

#### Static Site Generation (SSG)

**SSG** — це підвид SSR, де сторінки генеруються під час збірки проекту:

```javascript
// Next.js SSG приклад
export const getStaticProps = async () => {
  const res = await fetch('http://api.example.com/products');
  const products = await res.json();

  return {
    props: {
      products
    },
    revalidate: 3600 // Регенерація кожну годину
  };
};

export const getStaticPaths = async () => {
  const res = await fetch('http://api.example.com/products');
  const products = await res.json();

  const paths = products.map((product) => ({
    params: { id: product.id.toString() }
  }));

  return {
    paths,
    fallback: 'blocking' // Генерувати нові сторінки на вимогу
  };
};
```

### Порівняльна таблиця підходів

| Критерій | SPA | MPA | SSR | SSG |
|----------|-----|-----|-----|-----|
| **Початкове завантаження** | Повільне | Швидке | Швидке | Найшвидше |
| **Навігація** | Миттєва | Повільна | Миттєва | Миттєва |
| **SEO** | Погане | Відмінне | Відмінне | Відмінне |
| **Інтерактивність** | Висока | Низька | Висока | Висока |
| **Складність розробки** | Середня | Низька | Висока | Висока |
| **Навантаження на сервер** | Низьке | Високе | Високе | Низьке |
| **Кешування** | Складне | Просте | Середнє | Відмінне |

### Вибір підходу для різних проектів

```mermaid
flowchart TD
    A[Вибір архітектури] --> B{SEO критично важливо?}

    B -->|Ні| C[SPA]
    B -->|Так| D{Контент динамічний?}

    C --> G[✅ Dashboard<br/>✅ Admin панель<br/>✅ CRM системи]

    D -->|Так| E[SSR]
    D -->|Ні| F[SSG]

    E --> H[✅ E-commerce<br/>✅ Новинні сайти<br/>✅ Блоги]
    F --> I[✅ Документація<br/>✅ Landing pages<br/>✅ Портфоліо]
```

## Огляд технологічних стеків

### MEAN Stack

**MEAN** — це повний JavaScript стек для веброзробки:

- **M**ongoDB - NoSQL база даних;
- **E**xpress.js - вебфреймворк для Node.js;
- **A**ngular - frontend фреймворк;
- **N**ode.js - серверне середовище JavaScript.

```mermaid
graph TB
    subgraph "MEAN Stack Architecture"
        A[🅰️ Angular Frontend] --> B[🚀 Express.js API]
        B --> C[🟢 Node.js Runtime]
        C --> D[🍃 MongoDB Database]
    end

    subgraph "Data Flow"
        E[👤 User Interaction] --> A
        A --> F[📡 HTTP Requests]
        F --> B
        B --> G[📊 Database Queries]
        G --> D
    end
```

#### Приклад MEAN додатку:

Backend (Express.js + Node.js):

```javascript
// server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');

const app = express();

// Підключення до MongoDB
mongoose.connect('mongodb://localhost:27017/meanapp');

// Middleware
app.use(cors());
app.use(express.json());

// Модель користувача
const userSchema = new mongoose.Schema({
  name: String,
  email: String,
  createdAt: { type: Date, default: Date.now }
});

const User = mongoose.model('User', userSchema);

// API маршрути
app.get('/api/users', async (req, res) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/api/users', async (req, res) => {
  try {
    const user = new User(req.body);
    await user.save();
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

app.listen(3000, () => {
  console.log('Сервер запущено на порті 3000');
});
```

Frontend (Angular):

```typescript
// user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface User {
  _id?: string;
  name: string;
  email: string;
  createdAt?: Date;
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private apiUrl = 'http://localhost:3000/api';

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(`${this.apiUrl}/users`);
  }

  createUser(user: User): Observable<User> {
    return this.http.post<User>(`${this.apiUrl}/users`, user);
  }
}

// user-list.component.ts
import { Component, OnInit } from '@angular/core';
import { UserService, User } from './user.service';

@Component({
  selector: 'app-user-list',
  template: `
    <div>
      <h2>Користувачі</h2>
      <ul>
        <li *ngFor="let user of users">
          {{ user.name }} - {{ user.email }}
        </li>
      </ul>
    </div>
  `
})
export class UserListComponent implements OnInit {
  users: User[] = [];

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.userService.getUsers().subscribe(
      users => this.users = users,
      error => console.error('Помилка завантаження користувачів:', error)
    );
  }
}
```

### MERN Stack

**MERN** замінює Angular на React:

- **M**ongoDB - NoSQL база даних;
- **E**xpress.js - вебфреймворк для Node.js;
- **R**eact - frontend бібліотека;
- **N**ode.js - серверне середовище JavaScript.

```mermaid
graph TB
    subgraph "MERN Stack Architecture"
        A[⚛️ React Frontend] --> B[🚀 Express.js API]
        B --> C[🟢 Node.js Runtime]
        C --> D[🍃 MongoDB Database]
    end

    subgraph "Development Tools"
        E[📦 Webpack/Vite] --> A
        F[🔧 Nodemon] --> C
        G[🧪 Jest/Testing Library] --> A
        H[📊 MongoDB Compass] --> D
    end
```

#### Приклад MERN компоненту:

```jsx
// UserList.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserList = () => {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUsers = async () => {
      try {
        const response = await axios.get('/api/users');
        setUsers(response.data);
      } catch (err) {
        setError('Не вдалося завантажити користувачів');
      } finally {
        setLoading(false);
      }
    };

    fetchUsers();
  }, []);

  if (loading) return <div>Завантаження...</div>;
  if (error) return <div>Помилка: {error}</div>;

  return (
    <div className="user-list">
      <h2>Список користувачів</h2>
      {users.map(user => (
        <div key={user._id} className="user-card">
          <h3>{user.name}</h3>
          <p>{user.email}</p>
          <small>{new Date(user.createdAt).toLocaleDateString()}</small>
        </div>
      ))}
    </div>
  );
};

export default UserList;
```

### Next.js Full-Stack

**Next.js** — це React фреймворк з вбудованими можливостями full-stack розробки:

```mermaid
graph TB
    subgraph "Next.js Architecture"
        A[🌐 Browser] --> B[⚡ Next.js App]
        B --> C[📄 Pages/App Router]
        B --> D[🔧 API Routes]
        B --> E[🎨 Components]
        D --> F[💾 Database]
        B --> G[📦 Static Assets]
    end

    subgraph "Rendering Strategies"
        H[SSG - Static Site Generation]
        I[SSR - Server-Side Rendering]
        J[ISR - Incremental Static Regeneration]
        K[CSR - Client-Side Rendering]
    end
```

#### Приклад Next.js додатку:

Файлова структура:

```
nextjs-app/
├── pages/                         # Сторінки (до Next.js 13)
│   ├── api/                       # API маршрути
│   │   └── users.js
│   ├── index.js                   # Домашня сторінка
│   └── users/
│       └── [id].js               # Динамічний маршрут
├── app/                          # App Router (Next.js 13+)
│   ├── layout.js                 # Корневий layout
│   ├── page.js                   # Домашня сторінка
│   ├── users/
│   │   ├── page.js              # /users сторінка
│   │   └── [id]/
│   │       └── page.js          # /users/[id] сторінка
│   └── api/
│       └── users/
│           └── route.js         # API endpoint
├── components/
│   ├── UserCard.jsx
│   └── Navigation.jsx
├── lib/
│   └── database.js              # Утиліти для БД
└── public/
    └── images/
```

API маршрут:

```javascript
// app/api/users/route.js (Next.js 13+)
import { NextResponse } from 'next/server';
import { connectToDatabase } from '@/lib/database';

export async function GET() {
  try {
    const db = await connectToDatabase();
    const users = await db.collection('users').find({}).toArray();

    return NextResponse.json(users);
  } catch (error) {
    return NextResponse.json(
      { error: 'Не вдалося завантажити користувачів' },
      { status: 500 }
    );
  }
}

export async function POST(request) {
  try {
    const userData = await request.json();
    const db = await connectToDatabase();

    const result = await db.collection('users').insertOne({
      ...userData,
      createdAt: new Date()
    });

    return NextResponse.json(
      { id: result.insertedId, ...userData },
      { status: 201 }
    );
  } catch (error) {
    return NextResponse.json(
      { error: 'Не вдалося створити користувача' },
      { status: 400 }
    );
  }
}
```

SSR сторінка:

```jsx
// app/users/[id]/page.js
async function getUser(id) {
  const res = await fetch(`${process.env.API_URL}/api/users/${id}`, {
    next: { revalidate: 3600 } // Кешування на годину
  });

  if (!res.ok) {
    throw new Error('Не вдалося завантажити користувача');
  }

  return res.json();
}

export default async function UserPage({ params }) {
  const user = await getUser(params.id);

  return (
    <div className="user-profile">
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <p>Реєстрація: {new Date(user.createdAt).toLocaleDateString()}</p>
    </div>
  );
}

// Генерація статичних параметрів
export async function generateStaticParams() {
  const res = await fetch(`${process.env.API_URL}/api/users`);
  const users = await res.json();

  return users.map((user) => ({
    id: user.id.toString()
  }));
}
```

### Порівняння технологічних стеків

| Критерій | MEAN | MERN | Next.js |
|----------|------|------|---------|
| **Frontend** | Angular | React | React |
| **Крива навчання** | Висока | Середня | Середня-Висока |
| **TypeScript** | Нативно | Додатково | Нативно |
| **SEO** | Потребує Angular Universal | Потребує SSR | Вбудовано |
| **Розмір додатку** | Великий | Середній | Оптимізований |
| **Екосистема** | Менша | Велика | Зростаюча |
| **Продуктивність** | Середня | Висока | Висока |
| **Гнучкість** | Середня | Висока | Середня |

## Висновки

Сучасна веброзробка — це складна екосистема технологій, інструментів та підходів:

### Ключові аспекти:

1. **Архітектура**: Розуміння client-server взаємодії та різних архітектурних підходів є основою якісної розробки.
2. **API Design**: Вибір між REST, GraphQL та gRPC залежить від специфіки проекту та вимог до продуктивності.
3. **Рендеринг**: SPA, MPA, SSR та SSG мають свої переваги та недоліки для різних типів додатків.
4. **Технологічні стеки**: MEAN, MERN та Next.js надають готові рішення для швидкого старту проектів.
5. **Інструменти**: Сучасні IDE, системи збірки та девелоперські інструменти значно підвищують продуктивність розробки.

### Тенденції розвитку:

- **Мікрофронтенди**: Декомпозиція frontend додатків на незалежні частини;
- **Serverless**: Функції як сервіс для backend логіки;
- **Edge Computing**: Виконання коду близько до користувачів;
- **Web Assembly**: Високопродуктивні додатки в браузері;
- **Progressive Web Apps**: Вебдодатки з можливостями нативних.

Успішна веброзробка вимагає постійного навчання та адаптації до нових технологій, але розуміння фундаментальних принципів залишається незмінним.
