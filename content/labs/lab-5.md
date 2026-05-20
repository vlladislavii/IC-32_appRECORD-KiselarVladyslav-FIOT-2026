# Лабораторна робота №5

## Тема, Мета, Місце розташування

**Тема:** БЕЗПЕКА ТА ПРОДУКТИВНІСТЬ СЕРВЕРНИХ ДОДАТКІВ: БЕЗПЕКА NODE.JS-ДОДАТКІВ, ОПТИМІЗАЦІЯ ЗАПИТІВ І КЕШУВАННЯ, ТЕСТУВАННЯ API

**Мета:**
1. Забезпечувати базову безпеку Node.js-додатків.
2. Оптимізувати продуктивність REST API.
3. Використовувати кешування для зменшення навантаження на сервер.
4. Тестувати API за допомогою сучасних інструментів.
5. Аналізувати продуктивність backend-застосунків.

**Завдання роботи:**
* Створити REST API на Node.js та Express.
* Реалізувати захист API: Helmet, rate-limit, валідацію даних.
* Реалізувати кешування відповідей.
* Оптимізувати один із маршрутів API.
* Провести тестування API.
* Проаналізувати продуктивність до та після оптимізації.
* Додатково реалізувати: JWT-автентифікацію, Redis-кешування, Swagger-документацію, Docker-контейнеризацію, навантажувальне тестування через Artillery.

**Місце розташування:**
- **Локальний шлях:** `C:\University\KPI\Sem6\Web2\lab5`

---

## Структура проєкту

![Project Structure](/assets/labs/lab-5/structure.png)

```
lab5/
├── src/
│   ├── app.js                 # Express-застосунок (middleware, маршрути, обробка помилок)
│   ├── server.js              # Запуск сервера + graceful shutdown
│   ├── config/
│   │   ├── env.js             # Завантаження конфігурації з .env
│   │   └── swagger.js         # Конфігурація Swagger/OpenAPI
│   ├── cache/
│   │   └── index.js           # Абстракція кешу (node-cache або Redis)
│   ├── middleware/
│   │   ├── auth.js            # JWT: підпис та перевірка токена
│   │   └── validate.js        # Обробка результатів express-validator
│   ├── routes/
│   │   ├── auth.routes.js     # POST /auth/login
│   │   └── products.routes.js # GET / POST /products
│   └── data/
│       ├── products.js        # «БД» товарів + імітація затримки запиту
│       └── users.js           # Користувачі (паролі — bcrypt-хеші)
├── tests/
│   └── products.test.js       # Jest + Supertest (15 тестів)
├── scripts/
│   └── benchmark.js           # Дослідження продуктивності (до/після кешу)
├── load-test/
│   └── artillery.yml          # Навантажувальне тестування
├── Dockerfile                 # Контейнеризація API
├── docker-compose.yml         # API + Redis
├── .env                       # Змінні середовища
└── package.json
```

### Опис основних файлів:

* **`src/app.js`** — головний файл застосунку: підключення middleware безпеки (Helmet, rate-limit, CORS, compression), маршрутів, Swagger UI, обробників 404 та помилок. Винесений окремо від запуску сервера, щоб його можна було імпортувати у тестах.
* **`src/server.js`** — піднімає HTTP-сервер на заданому порту.
* **`src/cache/index.js`** — універсальний інтерфейс кешу з двома драйверами: in-memory (`node-cache`) та `Redis` (`ioredis`), що перемикаються змінною `CACHE_DRIVER`.
* **`src/routes/products.routes.js`** — оптимізований маршрут `GET /products` (кешування + пагінація) та захищений `POST /products` (JWT + валідація).
* **`scripts/benchmark.js`** — порівнює час відповіді та кількість звернень до БД з кешем і без нього.

---

## Встановлені залежності

```json
{
  "dependencies": {
    "express": "^4.19.2",
    "helmet": "^7.1.0",
    "express-rate-limit": "^7.4.0",
    "express-validator": "^7.2.0",
    "node-cache": "^5.1.2",
    "ioredis": "^5.4.1",
    "jsonwebtoken": "^9.0.2",
    "bcryptjs": "^2.4.3",
    "swagger-jsdoc": "^6.2.8",
    "swagger-ui-express": "^5.0.1",
    "compression": "^1.7.4",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^7.0.0",
    "artillery": "^2.0.20",
    "cross-env": "^7.0.3"
  }
}
```

| Пакет | Призначення |
|-------|-------------|
| **express** | Веб-фреймворк для побудови REST API |
| **helmet** | Захист HTTP-заголовків |
| **express-rate-limit** | Обмеження кількості запитів (rate limiting) |
| **express-validator** | Валідація вхідних даних |
| **node-cache** | In-memory кешування |
| **ioredis** | Клієнт Redis для кешування |
| **jsonwebtoken** | Генерація та перевірка JWT |
| **bcryptjs** | Хешування паролів |
| **swagger-jsdoc / swagger-ui-express** | Документація API (OpenAPI) |
| **compression** | Стиснення відповідей (gzip) |
| **jest / supertest** | Автоматизоване тестування API |
| **artillery** | Навантажувальне тестування |

**Команди для ініціалізації:**
```bash
npm init -y
npm install express helmet express-rate-limit cors dotenv express-validator
npm install node-cache ioredis jsonwebtoken bcryptjs
npm install swagger-jsdoc swagger-ui-express compression
npm install jest supertest artillery cross-env --save-dev
```
---

## Запуск застосунку

**1. Встановити залежності:**
```bash
npm install
```

**2. Створити файл `.env`** (зразок у `.env.example`):
```ini
PORT=3000
NODE_ENV=development
JWT_SECRET=super_secret_change_me
JWT_EXPIRES_IN=1h
CACHE_DRIVER=memory
CACHE_TTL=60
REDIS_URL=redis://localhost:6379
DB_LATENCY_MS=120
```

**Опис змінних середовища:**

| Змінна | Значення | Призначення |
|--------|----------|-------------|
| **`PORT`** | `3000` | Порт, на якому запускається HTTP-сервер. Сервер буде доступний за `http://localhost:3000`. |
| **`NODE_ENV`** | `development` | Режим роботи застосунку (`development`, `production` або `test`). У режимі `test` ліміт запитів (rate-limit) послаблюється, щоб не заважати автотестам. |
| **`JWT_SECRET`** | `super_secret_change_me` | Секретний ключ для підпису та перевірки JWT-токенів. Має бути довгим і випадковим; у production його обов'язково треба замінити на власне значення. |
| **`JWT_EXPIRES_IN`** | `1h` | Термін дії виданого JWT-токена (`1h` — одна година). Після завершення цього часу токен стає недійсним і потрібна повторна автентифікація. |
| **`CACHE_DRIVER`** | `memory` | Драйвер кешу: `memory` — внутрішній кеш у пам'яті процесу (`node-cache`), `redis` — зовнішній кеш Redis. |
| **`CACHE_TTL`** | `60` | Час життя записів у кеші (Time To Live) у секундах. Через 60 секунд закешовані дані вважаються застарілими й перечитуються з БД. |
| **`REDIS_URL`** | `redis://localhost:6379` | Адреса підключення до Redis. Використовується лише коли `CACHE_DRIVER=redis`. |
| **`DB_LATENCY_MS`** | `120` | Штучна затримка (у мілісекундах) при зверненні до «бази даних». Імітує реальний запит до БД, щоб ефект кешування та оптимізації був помітним у вимірюваннях продуктивності. |

**3. Запустити сервер:**
```bash
npm start
```

![Server Start](/assets/labs/lab-5/server-start.png)

Після цього сервер доступний за адресою `http://localhost:3000`. Перевірити, що він живий, можна через endpoint `/health`:
```bash
curl http://localhost:3000/health
```
```json
{ "status": "ok", "cache": { "driver": "memory", "hits": 0, "misses": 0 } }
```

---

## Реалізація завдань

### Завдання 1: Створення REST API на Node.js та Express

Створено REST API з розділенням застосунку (`app.js`) та запуску сервера (`server.js`). Таке розділення дозволяє імпортувати застосунок у тестах без підняття реального порту.

**Код `src/app.js` (фрагмент):**
```javascript
const express = require('express');
const app = express();

app.use(express.json({ limit: '10kb' })); // парсинг JSON з обмеженням розміру

// Маршрути
app.get('/health', (req, res) => {
  res.json({ status: 'ok', cache: cache.getStats() });
});
app.use('/auth', authRoutes);
app.use('/products', productRoutes);

// Обробка неіснуючих маршрутів
app.use((req, res) => res.status(404).json({ error: 'Not found' }));

module.exports = app;
```

**Код `src/server.js`:**
```javascript
const app = require('./app');
const config = require('./config/env');

const server = app.listen(config.port, () => {
  console.log(`Server started on port ${config.port} (${config.env})`);
});
```

**Перевірка:**
```bash
npm start
curl http://localhost:3000/products
```

![GET Products](/assets/labs/lab-5/get-products.png)

---

### Завдання 2: Захист API (Helmet, Rate Limiting, Валідація)

#### 2.1 Helmet — захист HTTP-заголовків

**HTTP-заголовки** — це службові поля, які сервер додає до кожної відповіді. Деякі з них керують поведінкою браузера у питаннях безпеки: дозволяють чи забороняють вбудовувати сторінку у `<iframe>`, виконувати сторонні скрипти, працювати через незахищений HTTP тощо. За замовчуванням Express ці заголовки **не встановлює**, тому застосунок залишається вразливим до низки атак (XSS, clickjacking, MIME-sniffing, перехоплення трафіку).

**Helmet** — це middleware, який одним рядком (`app.use(helmet())`) додає до всіх відповідей набір безпечних заголовків і прибирає потенційно небезпечні. Технічно це колекція з кількох менших middleware, кожен з яких відповідає за один заголовок.

```javascript
const helmet = require('helmet');
app.use(helmet()); // підключати якомога раніше, до маршрутів
```

**Заголовки, які встановлює Helmet за замовчуванням:**

| Заголовок | Призначення / від чого захищає |
|-----------|--------------------------------|
| **`Content-Security-Policy`** | Визначає, з яких джерел дозволено завантажувати ресурси (скрипти, стилі, зображення). Головний захист від **XSS** — браузер блокує сторонні/інжектовані скрипти. |
| **`X-Frame-Options: SAMEORIGIN`** | Забороняє вбудовувати сторінку у `<iframe>` на чужих сайтах. Захист від **clickjacking** (коли зловмисник «накладає» вашу сторінку під свою). |
| **`Strict-Transport-Security` (HSTS)** | Змушує браузер звертатися до сайту лише через **HTTPS**, навіть якщо користувач увів `http://`. Захист від перехоплення трафіку (man-in-the-middle). |
| **`X-Content-Type-Options: nosniff`** | Забороняє браузеру «вгадувати» тип файлу (MIME-sniffing). Захист від виконання, наприклад, картинки як скрипта. |
| **`X-DNS-Prefetch-Control: off`** | Вимикає попереднє розв'язання DNS — менший витік інформації про відвідані ресурси. |
| **`Cross-Origin-*` (Resource/Opener/Embedder Policy)** | Ізолюють сторінку від сторонніх ресурсів та вікон, обмежуючи крос-доменні атаки. |
| **`Origin-Agent-Cluster`** | Просить браузер ізолювати застосунок в окремий процес (захист пам'яті). |

**Що прибирає Helmet:**

| Заголовок | Чому прибирається |
|-----------|-------------------|
| **`X-Powered-By: Express`** | За замовчуванням Express повідомляє, що сайт працює на Express. Це підказка для зловмисника (які вразливості шукати). Helmet прибирає цей заголовок, щоб **не розкривати технологію сервера**. |

**Перевірка:** надіслати запит та переглянути заголовки відповіді:
```bash
http://localhost:3000/products
```
У відповіді з'являться заголовки безпеки:
```
Content-Security-Policy: default-src 'self';base-uri 'self';...
Strict-Transport-Security: max-age=15552000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
```

![Helmet Headers](/assets/labs/lab-5/helmet-headers.png)

#### 2.2 Rate Limiting — обмеження кількості запитів

Захист від брутфорсу та DDoS: не більше 100 запитів за хвилину з однієї IP-адреси.

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 60 * 1000, // 1 хвилина
  max: 100,            // максимум запитів
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: 'Too many requests, please try again later.' },
});
app.use(limiter);
```

**Перевірка:** швидко надіслати понад 100 запитів — сервер поверне `429 Too Many Requests`:

#### 2.3 Валідація даних

Вхідні дані перевіряються через `express-validator`. У разі помилки повертається `400` з переліком проблем.

```javascript
const { body } = require('express-validator');
const { validate } = require('../middleware/validate');

router.post(
  '/',
  authenticate,
  body('name').isString().trim().isLength({ min: 3 }).withMessage('name must be at least 3 chars'),
  body('price').isNumeric().withMessage('price must be a number'),
  validate, // повертає 400, якщо є помилки
  async (req, res) => { /* ... */ }
);
```

**Перевірка:** надіслати некоректні дані:
```json
{
  "errors": [
    { "msg": "name must be at least 3 chars", "path": "name" },
    { "msg": "price must be a number", "path": "price" }
  ]
}
```
![Validation Error](/assets/labs/lab-5/error.png)
![Validation Error](/assets/labs/lab-5/validation-error.png)

---

### Завдання 3: Кешування відповідей

Реалізовано універсальний кеш (`src/cache/index.js`) з двома драйверами — `node-cache` (in-memory, за замовчуванням) та `Redis`. Маршрут `GET /products` спочатку перевіряє кеш; у відповіді є поле `source` (`cache` або `database`) та заголовок `X-Cache` (`HIT`/`MISS`).

```javascript
router.get('/', /* ...валідація... */ async (req, res) => {
  const cacheKey = `products:page=${page}:limit=${limit}`;

  const cached = await cache.get(cacheKey);
  if (cached) {
    res.set('X-Cache', 'HIT');
    return res.json({ source: 'cache', ...cached });
  }

  const result = await productsData.findProducts({ page, limit });
  await cache.set(cacheKey, result);

  res.set('X-Cache', 'MISS');
  return res.json({ source: 'database', ...result });
});
```

При створенні товару кеш інвалідовується, щоб новий запис одразу був видимий:
```javascript
await cache.del('products:*');
```
Для наочності у шарі даних (`src/data/products.js`) імітується затримка запиту до БД (`DB_LATENCY_MS=120`) та лічильник звернень до БД — це дозволяє виміряти ефект оптимізації.

**Перевірка:** надіслати два однакові GET-запити поспіль.

Перший запит — дані з бази (`source: database`, `X-Cache: MISS`):
```bash
curl -D - http://localhost:3000/products
```
![GET from Database](/assets/labs/lab-5/get-products-db.png)

Другий запит — дані з кешу (`source: cache`, `X-Cache: HIT`):
```bash
curl -D - http://localhost:3000/products
```
![GET from Cache](/assets/labs/lab-5/get-products-cache.png)

---

### Завдання 4: Оптимізація маршруту API

Маршрут `GET /products` оптимізовано кількома способами:

1. **Кешування** — повторні запити не звертаються до БД.
2. **Пагінація** (`?page`, `?limit`) — передається лише потрібна порція даних, а не весь масив.
3. **Стиснення відповідей** (gzip) через middleware `compression`.

```javascript
const compression = require('compression');
app.use(compression()); // зменшує розмір тіла відповіді
```

**Перевірка пагінації:**
```bash
curl "http://localhost:3000/products?page=2&limit=3"
```
```json
{
  "source": "database",
  "items": [ /* 3 товари */ ],
  "pagination": { "page": 2, "limit": 3, "total": 10, "totalPages": 4 }
}
```

![Pagination](/assets/labs/lab-5/pagination.png)

---

### Завдання 5: Тестування API

#### 5.1 Ручне тестування через Swagger UI

Для ручного тестування використовується вбудована документація **Swagger UI**, доступна за адресою `http://localhost:3000/api-docs`. Вона дозволяє надсилати реальні запити до API прямо з браузера, без сторонніх інструментів.

**Крок 1. Відкрити Swagger UI**

У браузері перейти на `http://localhost:3000/api-docs` — відобразиться список усіх endpoints (Auth, Products).

![Swagger UI](/assets/labs/lab-5/swagger.png)

**Крок 2. Тестування GET /products**

1. Розгорнути блок `GET /products`.
2. Натиснути **Try it out**.
3. (Опційно) задати параметри `page` та `limit`.
4. Натиснути **Execute**.

У відповіді (Response body) видно товари, поле `source` (`database` або `cache`) та код `200`.

**Крок 3. Отримання JWT-токена (POST /auth/login)**

Оскільки `POST /products` захищений, спершу треба отримати токен:

1. Розгорнути `POST /auth/login` → **Try it out**.
2. У тілі запиту ввести облікові дані:
   ```json
   { "username": "admin", "password": "admin123" }
   ```
3. **Execute** — у відповіді скопіювати значення поля `token`.

**Крок 4. Авторизація у Swagger**

1. Натиснути кнопку **Authorize** (значок замка вгорі праворуч).
2. Вставити токен у поле та підтвердити.

Після цього Swagger автоматично додаватиме заголовок `Authorization: Bearer <TOKEN>` до захищених запитів.

**Крок 5. Тестування POST /products**

1. Розгорнути `POST /products` → **Try it out**.
2. У тілі запиту ввести:
   ```json
   { "name": "Tablet", "price": 15000 }
   ```
3. **Execute**.

За валідних даних та наявного токена сервер поверне `201 Created` зі створеним товаром. Без токена відповідь буде `401`, за некоректних даних — `400`.

![Swagger](/assets/labs/lab-5/swagger.png)

#### 5.2 Автоматичне тестування (Jest + Supertest)

Написано 15 тестів, що покривають: health/Swagger, автентифікацію, кешування (перехід database → cache), пагінацію, валідацію, захист JWT та інвалідацію кешу.

**Фрагмент `tests/products.test.js`:**
```javascript
const request = require('supertest');
const app = require('../src/app');

test('first request hits database, second hits cache', async () => {
  const first = await request(app).get('/products');
  expect(first.body.source).toBe('database');
  expect(first.headers['x-cache']).toBe('MISS');

  const second = await request(app).get('/products');
  expect(second.body.source).toBe('cache');
  expect(second.headers['x-cache']).toBe('HIT');
});

test('rejects request without a token', async () => {
  const res = await request(app)
    .post('/products')
    .send({ name: 'Smartwatch', price: 5000 });
  expect(res.statusCode).toBe(401);
});
```

**Запуск тестів:**
```bash
npm test
```

**Результат:**
```
Test Suites: 1 passed, 1 total
Tests:       15 passed, 15 total
Time:        ~2.7 s
```

![Jest Tests](/assets/labs/lab-5/jest-tests.png)

---

### Завдання 6: Аналіз продуктивності до та після оптимізації

Скрипт `scripts/benchmark.js` виконує 10 GET-запитів **без кешу** (`?nocache=true`) і 10 запитів **із кешем**, після чого порівнює середній час відповіді та кількість звернень до БД.

**Запуск:**
```bash
npm run benchmark
```

**Результат:**
```
=== Performance study: 10 GET /products requests ===

Scenario               | avg ms | min ms | max ms | DB reads
-----------------------|--------|--------|--------|---------
WITHOUT cache (nocache)| 143.36 | 124.54 | 218.64 |      10
WITH cache             |  15.17 |   1.12 | 125.14 |       1

Average speedup with cache: ~9.5x
DB reads reduced from 10 to 1 (cache served 9/10 requests).
```

![Benchmark](/assets/labs/lab-5/benchmark.png)

**Висновок щодо продуктивності:**

| Показник | Без кешу | З кешем |
|----------|----------|---------|
| Середній час відповіді | ~143 мс | ~15 мс |
| Звернень до БД (на 10 запитів) | 10 | 1 |
| Прискорення | — | ~9.5× |

Кешування скоротило середній час відповіді приблизно у 9.5 раза та зменшило навантаження на БД на 90%.

---

### Завдання (додатково): JWT-автентифікація

Маршрут `POST /products` захищено JWT. Токен видається через `POST /auth/login`. Паролі зберігаються лише як bcrypt-хеші.

```javascript
// src/middleware/auth.js
function authenticate(req, res, next) {
  const [scheme, token] = (req.headers.authorization || '').split(' ');
  if (scheme !== 'Bearer' || !token) {
    return res.status(401).json({ error: 'Missing or malformed Authorization header' });
  }
  try {
    req.user = jwt.verify(token, config.jwt.secret);
    return next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
}
```

**Перевірка:**

1. Отримати токен (логін `admin` / пароль `admin123`):
```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```
```json
{ "token": "eyJhbGciOiJIUzI1NiІ...", "tokenType": "Bearer" }
```
![JWT Login](/assets/labs/lab-5/token.png)
---

### Завдання (додатково): Docker-контейнеризація

Створено `Dockerfile` та `docker-compose.yml`, що піднімають API разом із контейнером Redis.

**`docker-compose.yml` (фрагмент):**
```yaml
services:
  api:
    build: .
    ports: ["3000:3000"]
    environment:
      CACHE_DRIVER: redis
      REDIS_URL: redis://redis:6379
    depends_on: [redis]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

**Запуск:**
```bash
docker compose up --build
```
---

## Список endpoints

| Endpoint | Метод | Захист | Опис |
|----------|-------|--------|------|
| `/health` | GET | — | Статус сервера та кешу |
| `/api-docs` | GET | — | Swagger UI документація |
| `/openapi.json` | GET | — | OpenAPI-специфікація |
| `/auth/login` | POST | — | Автентифікація, видача JWT |
| `/products` | GET | — | Список товарів (кеш + пагінація) |
| `/products` | POST | JWT | Створення товару (валідація) |

---

## Команди

```bash
# Встановлення залежностей
npm install

# Запуск сервера
npm start

# Запуск у режимі розробки (watch)
npm run dev

# Автоматичні тести
npm test

# Дослідження продуктивності (до/після кешу)
npm run benchmark

# Навантажувальне тестування
npm run loadtest

# Docker
docker compose up --build
```
---

## Висновки

У ході виконання лабораторної роботи було:

1. **Створено REST API** на Node.js та Express із чистою структурою (розділення застосунку та запуску сервера).

2. **Реалізовано базову безпеку:**
   - **Helmet** — захист HTTP-заголовків;
   - **Rate Limiting** — обмеження кількості запитів (захист від DDoS/брутфорсу);
   - **Валідація** вхідних даних через express-validator;
   - **JWT-автентифікація** з bcrypt-хешуванням паролів.

3. **Впроваджено кешування** з двома драйверами (in-memory та Redis) та інвалідацією кешу при зміні даних.

4. **Оптимізовано маршрут** `GET /products` (кешування, пагінація, gzip-стиснення).

5. **Проведено тестування** — 15 автоматичних тестів (Jest + Supertest) та навантажувальне тестування (Artillery).

6. **Проаналізовано продуктивність:** кешування прискорило відповіді приблизно у **9.5 раза** та зменшило кількість звернень до БД на **90%**.

7. **Додатково** реалізовано Swagger-документацію та Docker-контейнеризацію API разом із Redis.
