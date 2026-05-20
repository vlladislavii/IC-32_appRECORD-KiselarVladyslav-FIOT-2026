# Лабораторна робота №6

## Тема, Мета, Місце розташування

**Тема:** РОЗРОБКА REST API З ВИКОРИСТАННЯМ MYSQL ТА ДОКУМЕНТУВАННЯ API ЧЕРЕЗ SWAGGER

**Мета:**
1. Створити REST API на Node.js + Express.
2. Підключити базу даних MySQL та реалізувати CRUD-операції.
3. Інтегрувати Swagger UI у проєкт.
4. Задокументувати всі endpoint-и.
5. Виконати тестування API через Swagger UI.
6. Продемонструвати підсумковий проєкт: REST API з MySQL та Swagger-документацією.

**Завдання роботи:**
* Створити REST API на Node.js + Express.
* Підключити MySQL та реалізувати CRUD-операції (через ORM Sequelize).
* Інтегрувати Swagger UI у проєкт.
* Задокументувати всі endpoint-и в OpenAPI-специфікації.
* Виконати тестування API через Swagger UI.
* Продемонструвати підсумковий проєкт.

> **Примітка:** Згідно з умовою, дозволено застосувати REST API, підключену БД та CRUD-операції з попередніх лабораторних / курсових проєктів. У цій роботі використано backend готового проєкту **E-Hotel** (система керування бронюваннями готелів), який уже містить REST API на Express, MySQL через Sequelize та налаштований Swagger.

**Місце розташування:**
- **Git-hub репозиторій проекту**: https://github.com/vlladislavii/E-Hotel

---

## Опис проєкту

**E-Hotel** — це backend-система для мережі курортних готелів (resort), яка дозволяє касирам (Cashier) керувати готелями, номерами, бронюваннями, послугами та купонами. API працює з реляційною базою даних MySQL через ORM **Sequelize** і повністю задокументоване у Swagger.

Основні сутності предметної області:

| Сутність | Опис |
|----------|------|
| **Resort / Hotel** | Курорт та готелі, що йому належать |
| **Room** | Номери готелю (типи `single` / `double`, ціна) |
| **Service** | Додаткові послуги готелю |
| **Tourist / CreditCard** | Турист та його платіжні картки |
| **Booking** | Бронювання номера (статуси: confirmed, checked-in, completed, canceled, invalid) |
| **Cashier** | Працівник, що автентифікується через JWT |
| **Coupon** | Промокоди зі знижками |
| **Bill** | Рахунок, що формується при виїзді |
| **BookingStatusLog** | Журнал змін статусів бронювання |

---

## Структура проєкту

![Project Structure](/assets/labs/lab-6/structure.png)

```
backend/
├── src/
│   ├── server.js                  # Точка входу: Express, підключення БД, Swagger, маршрути
│   ├── config/
│   │   └── db.js                  # Налаштування Sequelize (підключення до MySQL)
│   ├── models/                    # Sequelize-моделі (таблиці) та зв'язки
│   │   ├── index.js               # Реєстрація моделей та асоціацій між ними
│   │   ├── Hotel.js, Room.js, Booking.js, Cashier.js, Coupon.js, ...
│   ├── controllers/               # Логіка обробки запитів (CRUD)
│   │   ├── authController.js       # register / login / deleteCashier
│   │   ├── hotelController.js      # готелі та послуги
│   │   ├── roomController.js       # пошук та деталі номерів
│   │   ├── bookingController.js    # бронювання + зміни статусів
│   │   ├── couponController.js     # перевірка купонів
│   │   └── dashboardController.js  # статистика
│   ├── routes/                    # Опис маршрутів (endpoint-ів) Express
│   │   ├── authRoutes.js, hotelRoutes.js, roomRoutes.js,
│   │   └── bookingRoutes.js, couponRoutes.js, dashboardRoutes.js
│   └── middleware/
│       ├── authenticateToken.js   # перевірка JWT
│       └── loginLimiter.js        # rate-limit на /login
├── swagger/
│   └── e-hotel.yaml               # OpenAPI-специфікація (документація API)
├── .env                           # Параметри підключення до БД та порт
└── package.json
```

### Опис основних файлів:

* **`src/server.js`** — головний файл: створює Express-додаток, підключає CORS та JSON-парсер, монтує Swagger UI на `/api-docs`, підключає всі маршрути та запускає сервер **лише після** успішного з'єднання з MySQL.
* **`src/config/db.js`** — створює екземпляр Sequelize з параметрами з `.env` (діалект `mysql`).
* **`src/models/`** — кожна модель описує таблицю БД; `index.js` визначає зв'язки (one-to-many, many-to-many, one-to-one).
* **`src/controllers/`** — реалізують CRUD-логіку, звертаючись до моделей Sequelize.
* **`swagger/e-hotel.yaml`** — повна OpenAPI-документація всіх endpoint-ів.

---

## Встановлені залежності

```json
{
  "dependencies": {
    "express": "^5.2.1",
    "mysql2": "^3.18.2",
    "sequelize": "^6.37.7",
    "swagger-ui-express": "^5.0.1",
    "yamljs": "^0.3.0",
    "cors": "^2.8.6",
    "dotenv": "^17.3.1",
    "express-rate-limit": "^8.3.2"
  },
  "devDependencies": {
    "nodemon": "^3.1.14"
  }
}
```

| Пакет | Призначення |
|-------|-------------|
| **express** | Веб-фреймворк для побудови REST API |
| **mysql2** | Драйвер для підключення до бази даних MySQL |
| **sequelize** | ORM — робота з MySQL через JavaScript-моделі замість «сирого» SQL |
| **swagger-ui-express** | Інтеграція інтерфейсу Swagger UI у застосунок |
| **yamljs** | Завантаження OpenAPI-специфікації з YAML-файлу |
| **cors** | Дозвіл крос-доменних запитів (для React-фронтенду) |
| **dotenv** | Завантаження конфігурації з `.env` |
| **express-rate-limit** | Обмеження кількості спроб входу (захист `/login`) |
| **jsonwebtoken / bcrypt** | JWT-автентифікація та хешування паролів касирів |
| **nodemon** | Автоперезапуск сервера під час розробки |

**Команди для встановлення:**
```bash
npm install express mysql2 sequelize swagger-ui-express yamljs cors dotenv express-rate-limit
npm install nodemon --save-dev
```

---

## Запуск застосунку

**Встановлені залежності:**
```bash
cd backend
npm install
```

**2. Налаштувати файл `.env`** (параметри підключення до MySQL):
```ini
DB_HOST=127.0.0.1
DB_USER=root
DB_PASSWORD=root
DB_NAME=e_hotel
DB_PORT=3305

PORT=5000
```

**Опис змінних:**

| Змінна | Призначення |
|--------|-------------|
| `DB_HOST` | Адреса сервера MySQL (локально — `127.0.0.1`) |
| `DB_USER` | Користувач БД |
| `DB_PASSWORD` | Пароль користувача БД |
| `DB_NAME` | Назва бази даних (`e_hotel`) |
| `DB_PORT` | Порт MySQL (тут `3305`) |
| `PORT` | Порт, на якому працює backend (`5000`) |

**MySQL сервер** - база `e_hotel` існує (Sequelize створить/оновить таблиці автоматично через `sequelize.sync()`).

![MySQL](/assets/labs/lab-6/mySQL.png)

**Запуск сервера:**
```bash
npm run dev      # режим розробки (nodemon)
# або
npm start        # звичайний запуск
```

```
MySQL connected on port 3305
Database synced: All tables from Domain Model created/updated.
Backend server is running on port 5000
```

![Server Start](/assets/labs/lab-6/server-start.png)

Після запуску:
- API доступне за `http://localhost:5000/api/...`
- Swagger UI — за `http://localhost:5000/api-docs`

---

## Реалізація завдань

### Завдання 1: REST API на Node.js + Express

API побудовано на Express із чітким поділом на шари: **маршрути → контролери → моделі**. Точка входу `server.js` створює застосунок, підключає middleware, монтує маршрути та запускає сервер тільки після успішного з'єднання з БД.

**Код `src/server.js` (фрагмент):**
```javascript
const express = require('express');
const cors = require('cors');
const sequelize = require('./config/db');
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');
const path = require('path');

const swaggerDocument = YAML.load(path.join(__dirname, '../swagger/e-hotel.yaml'));
const app = express();

app.use(cors());
app.use(express.json());

// Swagger UI
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument));

// Маршрути
app.use('/api/auth', require('./routes/authRoutes'));
app.use('/api/hotels', require('./routes/hotelRoutes'));
app.use('/api/rooms', require('./routes/roomRoutes'));
app.use('/api/bookings', require('./routes/bookingRoutes'));
app.use('/api/coupons', require('./routes/couponRoutes'));
app.use('/api/dashboard', require('./routes/dashboardRoutes'));

// Запуск сервера ПІСЛЯ підключення до БД
const startServer = async () => {
  await sequelize.authenticate();
  await db.sequelize.sync();
  app.listen(PORT, () => console.log(`Backend server is running on port ${PORT}`));
};
startServer();
```

**Перевірка:** після `npm run dev` відкрити у браузері `http://localhost:5000/api/hotels` — повернеться JSON зі списком готелів.

![REST API Check](/assets/labs/lab-6/api-hotels.png)

---

### Завдання 2: Підключення MySQL та CRUD-операції

#### 2.1 Підключення MySQL через Sequelize

Підключення до MySQL налаштовується у `src/config/db.js` за допомогою ORM Sequelize. Параметри беруться з `.env`.

```javascript
const { Sequelize } = require('sequelize');
require('dotenv').config();

const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASSWORD,
  {
    host: process.env.DB_HOST,
    port: process.env.DB_PORT || 3305,
    dialect: 'mysql',
    logging: false,
    define: { freezeTableName: true }
  }
);

module.exports = sequelize;
```

#### 2.2 Моделі та зв'язки

Кожна таблиця БД описана як Sequelize-модель. Приклад моделі номера `Room`:

```javascript
module.exports = sequelize.define('Room', {
  id:     { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  number: { type: DataTypes.STRING, allowNull: false },
  type:   { type: DataTypes.ENUM('single', 'double'), allowNull: false },
  price:  { type: DataTypes.DECIMAL(10, 2), allowNull: false }
}, { timestamps: false });
```

Зв'язки між моделями описано у `src/models/index.js`:

```javascript
Hotel.hasMany(Room, { foreignKey: 'hotelId' });        // готель -> номери
Room.belongsTo(Hotel, { foreignKey: 'hotelId' });

Room.hasMany(Booking, { foreignKey: 'roomId' });       // номер -> бронювання
Booking.belongsToMany(Service, { through: Booking_Service }); // багато-до-багатьох
```

#### 2.3 CRUD-операції

API реалізує повний набір CRUD-операцій над сутностями БД:

| Операція | HTTP | Endpoint | Приклад |
|----------|------|----------|---------|
| **Create** | POST | `/api/auth/register` | Реєстрація касира |
| **Create** | POST | `/api/bookings` | Створення бронювання |
| **Read** | GET | `/api/hotels` | Список готелів |
| **Read** | GET | `/api/rooms/search` | Пошук вільних номерів |
| **Read** | GET | `/api/bookings` | Список бронювань |
| **Update** | PATCH | `/api/bookings/{number}/check-in` | Зміна статусу на «заселено» |
| **Update** | PATCH | `/api/bookings/{number}/cancel` | Скасування бронювання |
| **Delete** | DELETE | `/api/auth/cashiers/{cnp}` | Видалення касира |

**Приклад READ** — отримання списку готелів із послугами (`hotelController.js`):
```javascript
exports.getAllHotels = async (req, res) => {
  const hotels = await Hotel.findAll({
    include: [{ model: Service, as: 'hotelServices' }]
  });
  res.status(200).json(hotels);
};
```

**Приклад CREATE** — реєстрація касира з хешуванням пароля (`authController.js`):
```javascript
exports.register = async (req, res) => {
  const { CNP, name, username, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);
  const newCashier = await Cashier.create({ CNP, name, username, password: hashedPassword });
  res.status(201).json({ message: "Cashier registered successfully" });
};
```

**Приклад UPDATE** — зміна статусу бронювання (`bookingController.js`):
```javascript
exports.checkIn = async (req, res) => {
  const booking = await Booking.findByPk(req.params.number);
  if (!booking) return res.status(404).json({ message: "Booking not found" });
  booking.status = 'checked-in';
  await booking.save();
  res.status(200).json({ message: "Checked in successfully", booking });
};
```

**Приклад DELETE** — видалення касира (`authController.js`):
```javascript
exports.deleteCashier = async (req, res) => {
  const cashier = await Cashier.findOne({ where: { CNP: req.params.cnp } });
  if (!cashier) return res.status(404).json({ message: "Cashier was not found." });
  await cashier.destroy();
  res.status(200).json({ message: `Cashier ${cashier.name} was deleted.` });
};
```

**Перевірка:** таблиці, створені Sequelize у MySQL (наприклад, у MySQL Workbench):

![MySQL Tables](/assets/labs/lab-6/mysql-table.png)

---

### Завдання 3: Інтеграція Swagger UI

Swagger UI підключено у `server.js` за допомогою `swagger-ui-express`. Документація читається з YAML-файлу `swagger/e-hotel.yaml` (через `yamljs`), що зручніше для редагування, ніж inline-анотації.

```javascript
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');
const path = require('path');

const swaggerDocument = YAML.load(path.join(__dirname, '../swagger/e-hotel.yaml'));
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument));
```

**Перевірка:** відкрити у браузері `http://localhost:5000/api-docs` — відобразиться інтерактивна документація, згрупована за тегами (Authentication, Hotels, Rooms, Bookings, Coupons, Dashboard).

![Swagger UI](/assets/labs/lab-6/swagger-ui.png)

---

### Завдання 4: Документування всіх endpoint-ів

Усі endpoint-и описані в OpenAPI-специфікації `swagger/e-hotel.yaml`: для кожного вказано метод, теги, параметри, тіло запиту, схему відповідей та коди статусів. Захищені маршрути позначено схемою безпеки `bearerAuth` (JWT).

**Приклад документування endpoint-а `POST /api/bookings`:**
```yaml
/api/bookings:
  post:
    summary: Create a new booking
    tags: [Bookings]
    security:
      - bearerAuth: []
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/CreateBookingRequest'
    responses:
      '201': { description: Booking created }
      '401': { description: Unauthorized }
```

**Опис схеми безпеки та компонентів:**
```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    LoginRequest:
      type: object
      required: [username, password]
      properties:
        username: { type: string, example: "vlad" }
        password: { type: string, format: password, example: "password123" }
```

---

### Завдання 5: Тестування API через Swagger UI

Swagger UI дозволяє надсилати реальні запити до API прямо з браузера.

**Крок 1. Тестування публічного GET-запиту (`GET /api/hotels`)**

1. Відкрити `http://localhost:5000/api-docs`.
2. Розгорнути блок `GET /api/hotels` → **Try it out** → **Execute**.
3. У відповіді (Response body) — масив готелів та код `200`.

![Swagger GET Hotels](/assets/labs/lab-6/swagger-get-hotels.png)

**Крок 2. Автентифікація (`POST /api/auth/login`)**

Частина маршрутів (бронювання) захищена JWT. Спершу треба отримати токен:

1. Розгорнути `POST /api/auth/login` → **Try it out**.
2. Ввести облікові дані касира:
   ```json
   { "username": "vlad", "password": "password123" }
   ```
3. **Execute** — скопіювати значення `token` з відповіді.

![Swagger Login](/assets/labs/lab-6/swagger-login.png)

**Крок 3. Тестування захищеного запиту (`POST /api/bookings`)**

1. Розгорнути `POST /api/bookings` → **Try it out**.
2. Ввести тіло запиту:
   ```json
   {
     "roomId": 1,
     "checkInDate": "2026-06-01",
     "checkOutDate": "2026-06-05",
     "tourist": { "CNP": "1234567890123", "name": "John Doe" },
     "payment": { "cardNumber": "4111111111111111", "expiryDate": "12/28", "cvv": "123" }
   }
   ```
3. **Execute** — за успіху сервер поверне `201 Created` із номером бронювання. Якщо користувач не авторизувався - помилка 401

![Swagger Create Booking](/assets/labs/lab-6/swagger-create-booking.png)

> Без токена захищені запити повертають `401 Unauthorized`, з недійсним токеном — `403`.

---

### Завдання 6: Демонстрація підсумкового проєкту

Підсумковий проєкт — повноцінне REST API **E-Hotel** з:
- базою даних **MySQL** (через ORM Sequelize, ~13 пов'язаних таблиць);
- повним набором **CRUD-операцій** над сутностями;
- **JWT-автентифікацією** касирів та rate-limit на вході;
- інтерактивною **Swagger-документацією** всіх endpoint-ів за адресою `/api-docs`.

Сервер успішно підключається до MySQL, синхронізує таблиці та обслуговує запити; усі endpoint-и доступні й тестуються через Swagger UI.

![Final Demo](/assets/labs/lab-6/final-demo.png)

---

## Список endpoints

| # | Метод | Endpoint | CRUD | Захист | Опис |
|---|-------|----------|------|--------|------|
| 1 | POST | `/api/auth/login` | — | rate-limit | Вхід касира, видача JWT |
| 2 | POST | `/api/auth/register` | Create | — | Реєстрація касира |
| 3 | DELETE | `/api/auth/cashiers/{cnp}` | Delete | — | Видалення касира |
| 4 | GET | `/api/hotels` | Read | — | Список готелів із послугами |
| 5 | GET | `/api/hotels/{id}/services` | Read | — | Послуги конкретного готелю |
| 6 | GET | `/api/rooms/search` | Read | — | Пошук вільних номерів |
| 7 | GET | `/api/rooms/{id}` | Read | — | Деталі номера |
| 8 | GET | `/api/coupons/validate` | Read | — | Перевірка промокоду |
| 9 | GET | `/api/bookings` | Read | JWT | Список бронювань |
| 10 | POST | `/api/bookings` | Create | JWT | Створення бронювання |
| 11 | PATCH | `/api/bookings/{number}/check-in` | Update | JWT | Заселення |
| 12 | PATCH | `/api/bookings/{number}/check-out` | Update | JWT | Виїзд + формування рахунку |
| 13 | PATCH | `/api/bookings/{number}/extend` | Update | JWT | Продовження проживання |
| 14 | PATCH | `/api/bookings/{number}/cancel` | Update | JWT | Скасування бронювання |
| 15 | PATCH | `/api/bookings/{number}/invalid` | Update | JWT | Позначення як недійсне |
| 16 | GET | `/api/dashboard/stats` | Read | — | Статистика для дашборду |

---

## Команди

```bash
# Встановлення залежностей
cd backend
npm install

# Запуск у режимі розробки (nodemon)
npm run dev

# Звичайний запуск
npm start

# Swagger UI
# http://localhost:5000/api-docs
```

---

## Висновки

У ході виконання лабораторної роботи було:

1. **Створено REST API** на Node.js + Express із чистою архітектурою «маршрути → контролери → моделі».

2. **Підключено базу даних MySQL** через ORM Sequelize: описано ~13 моделей предметної області та зв'язки між ними (one-to-many, many-to-many, one-to-one), реалізовано автоматичну синхронізацію схеми.

3. **Реалізовано CRUD-операції** над сутностями (готелі, номери, бронювання, касири, купони) з прикладами Create / Read / Update / Delete.

4. **Інтегровано Swagger UI** (`swagger-ui-express` + YAML-специфікація) за адресою `/api-docs`.

5. **Задокументовано всі endpoint-и** в OpenAPI-форматі: методи, параметри, тіла запитів, схеми відповідей та схему безпеки JWT.

6. **Виконано тестування API** через Swagger UI, включно з автентифікацією та захищеними маршрутами.

7. **Продемонстровано підсумковий проєкт** — повноцінне REST API E-Hotel з MySQL та Swagger-документацією.
