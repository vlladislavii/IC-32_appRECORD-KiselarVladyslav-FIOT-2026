# Лабораторна робота №2

## Тема, Мета, Місце розташування

**Тема:** СТВОРЕННЯ БАЗИ ДАНИХ У MYSQL. ПІДКЛЮЧЕННЯ NODE.JS ДО MYSQL. РОБОТА З ORM SEQUELIZE.

**Мета:**
1. Навчитися створювати базу даних у MySQL.
2. Освоїти виконання SQL-запитів (SELECT, INSERT, UPDATE, DELETE).
3. Підключати серверну програму на Node.js до бази даних.
4. Використовувати ORM Sequelize для роботи з БД.
5. Реалізувати зв’язок One-to-Many між таблицями.

**Завдання роботи:**
* Створити базу даних `e_hotel` для власного вебдодатку.
* Створити таблиці (наприклад, `resorts` та `hotels`).
* Виконати SQL-запити: SELECT, INSERT, UPDATE, DELETE.
* Підключити Node.js до MySQL через пакет `mysql2`.
* Використати ORM Sequelize для створення моделей та реалізації зв’язку One-to-Many.

**Місце розташування:**
- **GitHub:** [https://github.com/vlladislavii/E-Hotel](https://github.com/vlladislavii/E-Hotel)
- **Live demo:** [https://vlladislavii.github.io/E-Hotel/](https://vlladislavii.github.io/E-Hotel/)

---

## Теоретичні відомості

* **MySQL** — реляційна система керування базами даних (RDBMS), що використовує SQL.
* **Node.js** — середовище виконання JavaScript для серверних застосунків.
* **Sequelize** — ORM бібліотека, яка відображає структуру БД у вигляді JavaScript-об’єктів (Моделей).
* **mysql2** — драйвер, що забезпечує мережеве з’єднання між Node.js та MySQL.
* **Зв'язок One-to-Many** — тип зв'язку, де один запис першої таблиці пов'язаний з багатьма записами другої (наприклад, один Курорт має багато Готелів).

---

## Реалізація таблиці БД за доменною моделлю
Створення таблиці відбувалось за раніше побудованною доменною моделлю. 
    ![Domain model](/assets/labs/lab-2/domain-model.png)

---

## Реалізація серверної частини та роботи з БД

### Структура проєкту (Backend)
Проєкт організований за принципом:
* **`config/db.js`** — конфігурація підключення до MySQL.
    ![DB configuration](/assets/labs/lab-2/code1.png)
* **`models/`** — опис моделей `Resort.js`, `Hotel.js`, і т.д.
    ![Module dir structure](/assets/labs/lab-2/models.png)
* **`models/index.js`** — опис зв'язків моделей.
    ![index.js code](/assets/labs/lab-2/code4.png)
* **`server.js`** — основний файл сервера та синхронізація моделей.
    ![Server db code](/assets/labs/lab-2/server.png)
* **`test-db.js`** — скрипт для покрокової демонстрації CRUD та SQL запитів (тестування підключення).
    ```javascript
    const readline = require('readline');

    function pause(message) {
        const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
        return new Promise(resolve => rl.question(`\n--- ${message}. Натисніть Enter для продовження ---`, () => {
            rl.close();
            resolve();
        }));
    }

    const { 
    sequelize, Resort, Hotel,
    } = require('../src/models/index'); 
    const mysql = require('mysql2/promise');
    require('dotenv').config({ path: '../.env' });

    // Raw SQL (mysql2)

    async function insertRawSQL(connection) {
        console.log('Виконую SQL INSERT...');
        await connection.execute('INSERT INTO resort (name) VALUES (?)', ['TEST RESORT 1']);
        console.log('Дані вставлено успішно.');
    }

    async function selectRawSQL(connection) {
        console.log('Виконую SQL SELECT...');
        const [resorts] = await connection.execute('SELECT * FROM resort WHERE name = ?', ['TEST RESORT 1']);
        console.log('Результат SELECT:', resorts[0]);
        return resorts[0]?.id;
    }

    async function updateRawSQL(connection, id) {
        console.log('Виконую SQL UPDATE...');
        await connection.execute('UPDATE resort SET name = ? WHERE id = ?', ['TEST RESORT 1 UPDATED', id]);
        console.log('Запис оновлено.');
    }

    async function deleteRawSQL(connection, id) {
        console.log('Виконую SQL DELETE...');
        await connection.execute('DELETE FROM resort WHERE id = ?', [id]);
        console.log('Запис видалено.');
    }

    // Sequelize (ORM)

    async function syncDatabase() {
        await sequelize.sync({ alter: true });
        console.log('ORM: Таблиці синхронізовано.');
    }

    // INSERT
    async function insertHotelSequelize(rawResortId) {
        const hotel = await Hotel.create({
            name: 'TEST HOTEL 1',
            numberOfStars: 5,
            address: 'TEST ADDRESS 1',
            resortId: rawResortId
        });
        console.log('ORM: Готель створено через Hotel.create().');
        return hotel;
    }

    // SELECT
    async function selectHotelSequelize(hotelId) {
        const hotelData = await Hotel.findByPk(hotelId, { 
            include: Resort // One-to-Many
        });
        if (hotelData) {
            console.log(`ORM: Знайдено готель: ${hotelData.name}, Курорт: ${hotelData.Resort.name}`);
        }
        return hotelData;
    }

    // UPDATE
    async function updateHotelSequelize(hotelId) {
        await Hotel.update(
            { numberOfStars: 4 }, 
            { where: { id: hotelId } }
        );
        console.log('ORM: Дані готелю оновлено через Hotel.update().');
    }

    // DELETE
    async function deleteHotelSequelize(hotelId) {
        await Hotel.destroy({ 
            where: { id: hotelId } 
        });
        console.log('ORM: Готель видалено через Hotel.destroy().');
    }

    async function runLab() {
        let connection;
        try {
            // Connect to mysql2
            connection = await mysql.createConnection({
                host: process.env.DB_HOST,
                user: process.env.DB_USER,
                password: process.env.DB_PASSWORD,
                database: process.env.DB_NAME,
                port: process.env.DB_PORT || 3305
            });
            console.log('З\'єднання через mysql2 встановлено.');

            // RAW SQL
            await pause('Зараз ми виконаємо INSERT через сирий SQL');
            await insertRawSQL(connection);
            
            await pause('Тепер виконаємо SELECT, щоб отримати ID створеного курорту');
            const resortId = await selectRawSQL(connection);

            await pause('Демонструємо UPDATE: змінимо назву курорту в базі');
            await updateRawSQL(connection, resortId);
            
            // Sequelize 
            await pause('Синхронізуємо моделі (створюємо/оновлюємо таблиці)');
            await syncDatabase();

            await pause('Створимо готель через ORM Sequelize (метод .create())');
            const hotel = await insertHotelSequelize(resortId);

            await pause('Знайдемо цей готель через .findByPk з використанням JOIN (include)');
            await selectHotelSequelize(hotel.id);

            await pause('Оновимо дані готелю через .update()');
            await updateHotelSequelize(hotel.id);
            
            // DELETING
            await pause('Видалимо готель через Sequelize');
            await deleteHotelSequelize(hotel.id);

            await pause('Видалимо курорт через Raw SQL');
            await deleteRawSQL(connection, resortId);

        } catch (error) {
            console.error(error);
        } finally {
            if (connection) await connection.end();
            process.exit(0);
        }
    }

    runLab();
    ```

---

### Створені таблиці моделей `Resort.js`, `Hotel.js` (виведення через NySQL Workbench)
1. Resort model
    ![Resort model code](/assets/labs/lab-2/code2.png)
    ![Resort table](/assets/labs/lab-2/resort-table-1.png)
    ![Resort table](/assets/labs/lab-2/resort-table-2.png)
2. Hotel model
    ![Hotel model code](/assets/labs/lab-2/code3.png)
    ![Hotel table](/assets/labs/lab-2/hotel-table-1.png)
    ![Hotel table](/assets/labs/lab-2/hotel-table-2.png)
---

### Реалізація **Зв'язоку One-to-Many** між `Resort.js`, `Hotel.js`
    ![One-to-many relation code](/assets/labs/lab-2/code5.png)

---

### Реалізація **Зв'язоку Many-to-Many** між `Booking.js`, `Service.js`
    ![Many-to-many relation code](/assets/labs/lab-2/code6.png)

---

### Програмний код тестового сценарію (`test-db.js`)

Нижче наведено фрагменти коду, що реалізують демонстрацію операцій згідно з завданням.

#### 1) Робота з Raw SQL (Драйвер `mysql2`)
Демонстрація виконання прямих запитів без використання ORM. 

```javascript
// Raw SQL (mysql2)

async function insertRawSQL(connection) {
    console.log('Виконую SQL INSERT...');
    await connection.execute('INSERT INTO resort (name) VALUES (?)', ['TEST RESORT 1']);
    console.log('Дані вставлено успішно.');
}

async function selectRawSQL(connection) {
    console.log('Виконую SQL SELECT...');
    const [resorts] = await connection.execute('SELECT * FROM resort WHERE name = ?', ['TEST RESORT 1']);
    console.log('Результат SELECT:', resorts[0]);
    return resorts[0]?.id;
}

async function updateRawSQL(connection, id) {
    console.log('Виконую SQL UPDATE...');
    await connection.execute('UPDATE resort SET name = ? WHERE id = ?', ['TEST RESORT 1 UPDATED', id]);
    console.log('Запис оновлено.');
}

async function deleteRawSQL(connection, id) {
    console.log('Виконую SQL DELETE...');
    await connection.execute('DELETE FROM resort WHERE id = ?', [id]);
    console.log('Запис видалено.');
}
```
#### 2) Робота з Sequelize
Демонстрація виконання прямих запитів з ORM. 

```javascript
// Sequelize (ORM)

async function syncDatabase() {
    await sequelize.sync({ alter: true });
    console.log('ORM: Таблиці синхронізовано.');
}

// INSERT
async function insertHotelSequelize(rawResortId) {
    const hotel = await Hotel.create({
        name: 'TEST HOTEL 1',
        numberOfStars: 5,
        address: 'TEST ADDRESS 1',
        resortId: rawResortId
    });
    console.log('ORM: Готель створено через Hotel.create().');
    return hotel;
}

// SELECT
async function selectHotelSequelize(hotelId) {
    const hotelData = await Hotel.findByPk(hotelId, { 
        include: Resort // One-to-Many
    });
    if (hotelData) {
        console.log(`ORM: Знайдено готель: ${hotelData.name}, Курорт: ${hotelData.Resort.name}`);
    }
    return hotelData;
}

// UPDATE
async function updateHotelSequelize(hotelId) {
    await Hotel.update(
        { numberOfStars: 4 }, 
        { where: { id: hotelId } }
    );
    console.log('ORM: Дані готелю оновлено через Hotel.update().');
}

// DELETE
async function deleteHotelSequelize(hotelId) {
    await Hotel.destroy({ 
        where: { id: hotelId } 
    });
    console.log('ORM: Готель видалено через Hotel.destroy().');
}
```
---