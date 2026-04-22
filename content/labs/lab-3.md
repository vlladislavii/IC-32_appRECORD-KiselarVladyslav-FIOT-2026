# Лабораторна робота №3

## Тема, Мета, Місце розташування

**Тема:** РОЗРОБКА ФУНКЦІОНАЛЬНОГО REST API. РЕЄСТРАЦІЯ ТА АВТОРИЗАЦІЯ КОРИСТУВАЧІВ. ВАЛІДАЦІЯ ДАНИХ І ОБРОБКА ПОМИЛОК.

**Мета:**
1. Реалізувати безпечну систему реєстрації та авторизації касирів готелю.
2. Впровадити механізми захисту API за допомогою JWT (JSON Web Tokens).
3. Навчитися обробляти помилки та валідувати вхідні дані на сервері.
4. Забезпечити захист від brute-force атак через обмеження спроб входу (Rate Limiting).
5. Задокументувати функціонал за допомогою Swagger.

**Завдання роботи:**
* Створити систему реєстрації та логіну з хешуванням паролів за допомогою bcrypt.
* Реалізувати Middleware для перевірки JWT токенів та захисту приватних маршрутів.
* Додати функціонал Logout та видалення облікових записів касирів.
* Налаштувати обмеження кількості спроб входу для запобігання brute-force атак.
* Реалізувати Dashboard API для отримання статистики готелю.
* Оновити документацію Swagger, додавши всі нові ендпоінти.

**Місце розташування:**
- **GitHub:** [https://github.com/vlladislavii/E-Hotel](https://github.com/vlladislavii/E-Hotel)
- **Live demo:** [https://vlladislavii.github.io/E-Hotel/](https://vlladislavii.github.io/E-Hotel/)

---

## Структура backend

Архітектура організована за принципом розділення відповідальності (**Separation of Concerns**), що забезпечує чистоту коду та легкість масштабування.

![Backend Structure](/assets/labs/lab-3/structure.png)

### Опис основних директорій та файлів:

* **`config/`** — містить конфігураційні файли проєкту.
    * **`db.js`**: Налаштування підключення до бази даних MySQL за допомогою ORM Sequelize. Тут зберігаються параметри з'єднання (host, user, password), які зчитуються з файлу оточення `.env`.
* **`controllers/`** — містить логіку обробки запитів.
    * Це "мозок" програми, де функції отримують дані з запитів, взаємодіють із моделями бази даних та формують JSON-відповіді для фронтенду. Наприклад, тут реалізована логіка входу користувача та розрахунку статистики.
* **`middleware/`** — проміжне програмне забезпечення.
    * Функції, що виконуються перед основним контролером. Тут реалізовано:
        * `authenticateToken.js`: перевірка JWT-токена для захисту маршрутів.
        * `rate-limit`: обмеження кількості запитів для захисту від Brute-force атак.
* **`models/`** — опис структури даних.
    * Файли моделей (наприклад, `Cashier.js`, `Booking.js`) визначають схеми таблиць у БД. 
    * **`index.js`**: центральний вузол, де ініціалізується Sequelize та описуються зв'язки між таблицями (One-to-Many, Many-to-Many).
* **`routes/`** — маршрутизація API.
    * Тут визначаються ендпоінти (URL-адреси). Маршрути пов'язують зовнішні HTTP-запити з конкретними функціями в контролерах.
* **`swagger/`** — документація API.
    * **`e-hotel.yaml`**: Опис усіх доступних методів API згідно зі стандартом OpenAPI 3.0. Використовується для візуалізації документації через Swagger UI.
* **`server.js`** — точка входу (Entry Point).
    * Головний файл, що запускає сервер. Він підключає всі маршрути, налаштовує CORS, парсинг JSON та ініціює синхронізацію з базою даних.

---

## Конфігурація маршрутизації API (Routing)

Для забезпечення модульності та зручності підтримки коду, у головному файлі сервера (`server.js`) реалізовано централізоване підключення маршрутів. Кожен функціональний блок системи винесений в окремий модуль, що дозволяє чітко розділити API на логічні сегменти за принципом **Separation of Concerns**.

#### Основні сегменти API:

* **`/api/auth`** — автентифікація та реєстрація касирів (Login, Register, Delete).
* **`/api/hotels`** — доступ до інформації про готелі та додаткові послуги.
* **`/api/rooms`** — перегляд номерного фонду та перевірка доступності кімнат.
* **`/api/bookings`** — повний цикл управління бронюваннями (створення, перегляд, логування статусів).
* **`/api/coupons`** — перевірка валідності промокодів та розрахунок знижок.
* **`/api/dashboard`** — агрегація статистичних даних для панелі управління.

#### Технічна реалізація:

Використання методу `app.use()` у поєднанні з `express.Router()` дозволяє делегувати обробку запитів відповідним файлам у папці `routes/`. Це також спрощує впровадження Middleware (наприклад, перевірки JWT-токена) для цілих груп маршрутів одночасно.

```javascript
// Фрагмент підключення маршрутів у server.js
const authRoutes = require('./routes/authRoutes');
const hotelRoutes = require('./routes/hotelRoutes');
const bookingRoutes = require('./routes/bookingRoutes');

app.use('/api/auth', authRoutes);
app.use('/api/hotels', hotelRoutes);
app.use('/api/bookings', bookingRoutes);
```
---

## Модель Касира - користувача (Cashier)
Модель містить поля `CNP` - id (як Primary Key), `name`, `username` та `password`.

![Cashier Model Structure](/assets/labs/lab-3/cashier-fields.png)

---

## Реалізація завдань

### Реєстрація, авторизація, видалення користувача
Перед збереженням у БД пароль хешується.
```javascript
const { Cashier } = require('../models');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

exports.login = async (req, res) => {
    try {
        const { username, password } = req.body;

        const cashier = await Cashier.findOne({ where: { username } });
        if (!cashier) {
            return res.status(401).json({ message: "Invalid username or password" });
        }

        const isMatch = await bcrypt.compare(password, cashier.password);
        if (!isMatch) {
            return res.status(401).json({ message: "Invalid username or password" });
        }

        const token = jwt.sign(
            { CNP: cashier.CNP, username: cashier.username },
            process.env.JWT_SECRET || 'supersecretkey',
            { expiresIn: '1h' }
        );

        res.status(200).json({
            message: "Login successful",
            token,
            cashier: {
                name: cashier.name,
                username: cashier.username,
                CNP: cashier.CNP
            }
        });
    } catch (error) {
        res.status(500).json({ message: "Internal server error", error: error.message });
    }
};

exports.register = async (req, res) => {
    try {
        const { CNP, name, username, password } = req.body;

        const existingCashier = await Cashier.findOne({ where: { username } });
        if (existingCashier) {
            return res.status(400).json({ message: "Username already exists" });
        }

        const saltRounds = 10;
        const hashedPassword = await bcrypt.hash(password, saltRounds);

        const newCashier = await Cashier.create({
            CNP,
            name,
            username,
            password: hashedPassword
        });

        res.status(201).json({
            message: "Cashier registered successfully",
            cashier: {
                username: newCashier.username,
                name: newCashier.name
            }
        });

    } catch (error) {
        console.error("Registration error:", error);
        res.status(500).json({ message: "Error registering cashier", error: error.message });
    }
};

exports.deleteCashier = async (req, res) => {
    try {
        const { cnp } = req.params;
        
        const cashier = await Cashier.findOne({ where: { CNP: cnp } });

        if (!cashier) {
            return res.status(404).json({ message: "Cashier was not found." });
        }

        await cashier.destroy();

        res.status(200).json({ message: `Cashier ${cashier.name} was deleted.` });
    } catch (error) {
        res.status(500).json({ message: "Error", error: error.message });
    }
};
```
![Login page](/assets/labs/lab-3/login-page.png)
![Login page swagger](/assets/labs/lab-3/login-page-swagger.png)

### Реалізація logout
```javascript
const logoutBtn = container.querySelector('#logout-btn');
        if (logoutBtn) {
            logoutBtn.onclick = function() {
                sessionStorage.removeItem('hotel_token');
                sessionStorage.removeItem('cashier_info');
                window.location.href = baseUrl;
            };
        }
```

### Валідація даних, обробка помилок (на прикладі процесу Booking)

Процес створення бронювання є критичним вузлом системи, тому він потребує багаторівневої перевірки даних та надійної обробки виключних ситуацій.

#### 1. Валідація вхідних даних
Перш ніж дані потраплять у базу даних, вони проходять перевірку на рівні контролера (`bookingController.js`). Це запобігає збереженню некоректної інформації:

* **Перевірка дат:** Система контролює, щоб дата заїзду (`checkInDate`) була не раніше поточної дати, а дата виїзду була пізнішою за дату заїзду.
* **Наявність обов'язкових полів:** Перевірка наявності даних туриста (ім'я, CNP - id) та платіжних реквізитів.
* **Валідація логіки:** Перевірка, чи не зайнятий номер на обрані дати (захист від Double Booking).

```javascript
exports.createBooking = async (req, res) => {
    const t = await sequelize.transaction();
    try {
        const { roomId, checkInDate, checkOutDate, tourist, payment, couponCode, services } = req.body;

        const room = await Room.findByPk(roomId, { 
            include: [{ model: Hotel }],
            transaction: t 
        });
        if (!room) {
            await t.rollback();
            return res.status(404).json({ message: "Room not found" });
        }

        let validServices = [];
        let servicesTotalPerDay = 0;

        if (services && services.length > 0) {
            validServices = await Service.findAll({
                where: { 
                    id: services,
                    hotelId: room.hotelId
                },
                transaction: t
            });

            if (validServices.length !== services.length) {
                await t.rollback();
                return res.status(400).json({ message: "One or more services do not belong to this hotel." });
            }

            servicesTotalPerDay = validServices.reduce((sum, s) => sum + parseFloat(s.price), 0);
        }

        let existingTourist = await Tourist.findByPk(tourist.CNP);
        if (existingTourist) {
            if (existingTourist.name !== tourist.name) {
                await t.rollback();
                return res.status(400).json({ message: "CNP already exists with a different name." });
            }
        } else {
            existingTourist = await Tourist.create({ CNP: tourist.CNP, name: tourist.name }, { transaction: t });
        }

        const newToken = generateCardToken(payment.cardNumber, payment.expiryDate, payment.cvv);
        let card = await CreditCard.findByPk(payment.cardNumber);
        if (card) {
            if (card.token !== newToken) {
                await t.rollback();
                return res.status(400).json({ message: "Card details mismatch." });
            }
        } else {
            card = await CreditCard.create({
                cardNumber: payment.cardNumber,
                token: newToken,
                touristId: existingTourist.CNP
            }, { transaction: t });
        }

        let appliedCouponId = null;
        let discount = 0;
        if (couponCode) {
            const coupon = await Coupon.findOne({ where: { number: couponCode }, transaction: t });
            if (!coupon) {
                await t.rollback();
                return res.status(400).json({ message: "Invalid coupon." });
            }
            appliedCouponId = coupon.id;
            discount = coupon.percentage;
        }

        const overlappingBooking = await Booking.findOne({
            where: {
                roomId,
                status: { [Op.notIn]: ['canceled', 'invalid'] },
                [Op.and]: [
                    { checkInDate: { [Op.lt]: checkOutDate } },
                    { checkOutDate: { [Op.gt]: checkInDate } }
                ]
            },
            transaction: t
        });
        if (overlappingBooking) {
            await t.rollback();
            return res.status(409).json({ message: "Room is already booked." });
        }

        const start = new Date(checkInDate);
        const end = new Date(checkOutDate);
        const nights = Math.ceil(Math.abs(end - start) / (1000 * 60 * 60 * 24)) || 1;

        let totalCost = (parseFloat(room.price) + servicesTotalPerDay) * nights;
        if (discount > 0) totalCost -= (totalCost * discount) / 100;

        const creationDate = new Date();
        const checkInDateObj = new Date(checkInDate);
        const limit7Days = new Date(checkInDateObj.getTime() - (7 * 24 * 60 * 60 * 1000));
        limit7Days.setHours(23, 59, 59, 999);
        let gracePeriodDate = creationDate < limit7Days ? limit7Days : (new Date(creationDate.getTime() + 86400000) < checkInDateObj ? new Date(creationDate.getTime() + 86400000) : checkInDateObj);

        const bookingNumber = `BK-${room.hotelId}-${Date.now().toString().slice(-6)}`;
        const newBooking = await Booking.create({
            number: bookingNumber,
            checkInDate,
            checkOutDate,
            status: 'confirmed',
            totalCost: totalCost.toFixed(2),
            gracePeriodEndTimeStamp: gracePeriodDate,
            creationTimeStamp: creationDate,
            roomId,
            cardId: card.cardNumber,
            appliedCouponId
        }, { transaction: t });

        if (validServices.length > 0) {
            const bookingServiceEntries = validServices.map(s => ({
                bookingNumber: newBooking.number,
                serviceId: s.id
            }));
            await Booking_Service.bulkCreate(bookingServiceEntries, { transaction: t });
        }

        await BookingStatusLog.create({
            toStatus: 'confirmed',
            bookingNumber: newBooking.number,
            changedByCashierCNP: req.user ? req.user.CNP : null
        }, { transaction: t });

        await t.commit();
        res.status(201).json({ 
            message: "Booking created successfully", 
            bookingNumber: newBooking.number,
            totalCost: newBooking.totalCost
        });

    } catch (error) {
        if (t) await t.rollback();
        console.error(error);
        res.status(500).json({ message: "Booking failed", error: error.message });
    }
};
```
#### 2. Обробка помилок (Error Handling)
Для забезпечення стабільності сервера весь код обгорнутий у блоки try...catch.

У разі виникнення помилки (наприклад, збій з'єднання з БД), клієнт отримує статус 500 Internal Server Error та зрозуміле повідомлення замість "падіння" сервера.

![Booking page](/assets/labs/lab-3/booking-page.png)
![Booking page error](/assets/labs/lab-3/booking-page-error.png)
![Booking page swagger](/assets/labs/lab-3/booking-page-swagger.png)

### Обмеження кількості спроб входу (Rate Limiting)

Для захисту системи від атак типу Brute-force (автоматизованого підбору паролів) було впроваджено механізм обмеження частоти запитів до ендпоінтів автентифікації.

Використано пакет `express-rate-limit`, який дозволяє відстежувати кількість запитів з однієї IP-адреси за певний проміжок часу. Якщо ліміт перевищено, сервер автоматично повертає помилку `429 Too Many Requests`.

Конфігурація для маршруту `/api/auth/login`:
* **windowMs:** 15 хвилин (часове вікно).
* **max:** 5 спроб (максимальна кількість запитів).
* **message:** Інформативне повідомлення про блокування.

```javascript
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 хвилин
    max: 5, // ліміт 5 спроб
    message: {
        message: "Занадто багато спроб входу. Будь ласка, спробуйте пізніше через 15 хвилин."
    },
    standardHeaders: true,
    legacyHeaders: false, 
});

module.exports = loginLimiter;
```
Даний обмежувач підключений безпосередньо до маршруту авторизації у файлі `authRoutes.js`, що гарантує перевірку до того, як запит потрапить до бази даних.

```javascript
const express = require('express');
const router = express.Router();
const authController = require('../controllers/authController');
const loginLimiter = require('../middleware/loginLimiter');

// POST /api/auth/login
router.post('/login', loginLimiter, authController.login);

// POST /api/auth/register
router.post('/register', authController.register);

// DELETE /api/auth/cashiers/:cnp
router.delete('/cashiers/:cnp', authController.deleteCashier);

module.exports = router;
```

![Login error](/assets/labs/lab-3/login-error.png)

### Middleware для перевірки JWT-токена

Для захисту приватних маршрутів (наприклад, перегляд статистики або управління бронюваннями) було реалізовано механізм перевірки автентичності користувача за допомогою **JSON Web Tokens (JWT)**.

#### 1. Роль Middleware
Middleware `authenticateToken.js` виступає захисним шаром між клієнтом і контролером. Він перехоплює вхідний HTTP-запит, витягує токен із заголовка `Authorization` і перевіряє його валідність.

```javascript
const getAuthHeaders = () => ({
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${sessionStorage.getItem('hotel_token')}`
});
```

#### 2. Процес автентифікації
1. Клієнт надсилає токен у форматі: `Authorization: Bearer <token>`.
2. Middleware перевіряє наявність токена. Якщо токен відсутній — повертається помилка `401 Unauthorized`.
3. За допомогою секретного ключа (`JWT_SECRET`) відбувається перевірка підпису токена. Якщо токен підроблений або термін його дії закінчився — повертається `403 Forbidden`.
4. У разі успіху дані про касира (наприклад, його ID/CNP) додаються до об'єкта запиту `req.user`, і виконання передається наступній функції.

#### 3. Програмна реалізація токена

```javascript
const jwt = require('jsonwebtoken');

const authenticateToken = (req, res, next) => {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];

    if (!token) {
        return res.status(401).json({ message: "Access denied. No token provided." });
    }

    jwt.verify(token, process.env.JWT_SECRET || 'supersecretkey', (err, user) => {
        if (err) {
            return res.status(403).json({ message: "Invalid or expired token." });
        }
        
        req.user = user; 
        
        next();
    });
};

module.exports = authenticateToken;
```
---

## Повна документація API в Swagger

![Full Swagger](/assets/labs/lab-3/swagger.png)

---