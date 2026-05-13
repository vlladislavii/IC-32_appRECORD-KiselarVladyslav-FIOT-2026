# Лабораторна робота №4

## Тема, Мета, Місце розташування

**Тема:** РОЗШИРЕНІ МОЖЛИВОСТІ NODE.JS-ДОДАТКІВ: ЛОГУВАННЯ, ЗАВАНТАЖЕННЯ ФАЙЛІВ, МОНІТОРИНГ ПРОДУКТИВНОСТІ

**Мета:**
1. Ознайомитися з розширеними можливостями серверних застосунків на базі Node.js.
2. Реалізувати логування HTTP-запитів за допомогою Morgan.
3. Впровадити професійне логування подій у файл за допомогою Winston.
4. Навчитися обробляти завантаження файлів за допомогою Multer.
5. Реалізувати моніторинг продуктивності сервера.
6. Навчитися працювати з менеджером процесів PM2.

**Завдання роботи:**
* Створити базовий Express-сервер з підтримкою логування.
* Реалізувати завантаження одного та кількох файлів.
* Додати валідацію файлів (тип та розмір).
* Створити endpoint для моніторингу стану сервера.
* Реалізувати вимірювання часу відповіді сервера.
* Запустити застосунок через PM2.

**Місце розташування:**
- **Локальний шлях:** `C:\University\KPI\Sem6\Web2\lab4`

---

## Структура проєкту

![Project Structure](/assets/labs/lab-4/structure.png)

### Опис основних файлів та директорій:

* **`app.js`** — головний файл сервера, містить всю логіку застосунку:
    * Налаштування Express та middleware
    * Конфігурація Morgan та Winston для логування
    * Налаштування Multer для завантаження файлів
    * Endpoints для upload, status та інших функцій
    * Error handling middleware

* **`uploads/`** — директорія для збереження завантажених файлів.
    * Файли зберігаються з унікальними іменами (timestamp + original name)

* **`app.log`** — файл логів Winston.
    * Містить JSON-записи з timestamp, level, message та duration

* **`package.json`** — конфігурація проєкту та залежності.

* **`node_modules/`** — встановлені npm-пакети.

---

## Встановлені залежності

```json
{
  "dependencies": {
    "express": "^4.x.x",
    "morgan": "^1.x.x",
    "winston": "^3.x.x",
    "multer": "^1.x.x"
  }
}
```

| Пакет | Призначення |
|-------|-------------|
| **express** | Веб-фреймворк для Node.js |
| **morgan** | HTTP request logger middleware |
| **winston** | Професійна бібліотека логування |
| **multer** | Middleware для обробки multipart/form-data (файли) |

---

## Реалізація завдань

### Завдання 1: Ініціалізація проєкту та створення базового сервера

Створено базовий Express-сервер, який відповідає на GET-запит `/` та повертає HTML-форму для завантаження файлів.

**Команди для ініціалізації:**
```bash
npm init -y
npm install express morgan winston multer
```

**Код сервера:**
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send(`
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>File Upload</title>
</head>
<body>
    <h1>File Upload</h1>
    <!-- Forms here -->
</body>
</html>
    `);
});

app.listen(3000, () => {
    console.log('Server started on port 3000');
});
```

**Перевірка:**
```bash
node app.js
```
Відкрити браузер: `http://localhost:3000`

![Main Page](/assets/labs/lab-4/main-page.png)

---

### Завдання 2: Логування HTTP-запитів (Morgan)

Morgan — це middleware, який автоматично логує всі HTTP-запити до консолі. Формат `combined` показує: IP-адресу, дату, метод, URL, статус, user-agent.

**Код:**
```javascript
const morgan = require('morgan');

// Format 'combined' shows: IP, date, method, URL, status, response time
app.use(morgan('combined'));
```

**Перевірка:**
```bash
node app.js
```
Зробіть кілька запитів до сервера і подивіться консоль:

![Morgan Console Logs](/assets/labs/lab-4/morgan-logs.png)

**Приклад виводу Morgan:**
```
::1 - - [13/May/2026:18:43:20 +0000] "GET / HTTP/1.1" 200 1573 "-" "Mozilla/5.0..."
::1 - - [13/May/2026:18:43:25 +0000] "GET /status HTTP/1.1" 200 106 "-" "Mozilla/5.0..."
::1 - - [13/May/2026:18:44:04 +0000] "POST /upload HTTP/1.1" 200 482 "-" "Mozilla/5.0..."
```

---

### Завдання 3: Файлове логування подій (Winston)

Winston — професійна бібліотека для логування з підтримкою різних рівнів (info, error, warn, debug) та транспортів (console, file).

**Код:**
```javascript
const winston = require('winston');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'app.log' })
    ]
});

// Використання:
logger.info('Server started on port 3000');
logger.error('Something went wrong');
```

**Перевірка:**
```bash
type app.log
```

![Winston Log File](/assets/labs/lab-4/winston-logs.png)

**Приклад вмісту app.log:**
```json
{"level":"info","message":"Server started on port 3000","timestamp":"2026-05-13T21:38:25.123Z"}
{"level":"info","message":"File uploaded: photo.jpg","timestamp":"2026-05-13T21:40:10.456Z"}
{"method":"GET","url":"/status","status":200,"duration":"2ms","timestamp":"2026-05-13T21:41:00.789Z"}
```

---

### Завдання 4: Обробка помилок (Error Handling Middleware)

Реалізовано middleware для централізованої обробки помилок. Усі помилки логуються через Winston та повертаються клієнту у форматі JSON.

**Код:**
```javascript
// Error handling middleware (must be last!)
app.use((err, req, res, next) => {
    logger.error({
        message: err.message,
        stack: err.stack,
        url: req.url,
        method: req.method
    });

    res.status(err.status || 500).json({
        error: {
            message: err.message || 'Internal Server Error',
            status: err.status || 500
        }
    });
});
```

**Перевірка:**
Спробуйте завантажити файл з недозволеним типом (наприклад, `.txt`):

![Error Handling](/assets/labs/lab-4/error-handling.png)

**Приклад відповіді сервера:**
```json
{
    "error": {
        "message": "Invalid file type. Only JPG, PNG and PDF allowed.",
        "status": 500
    }
}
```

**Перевірка:**
```bash
type app.log
```
Помилка буде записана у файл з повним stack trace.

---

### Завдання 5: Завантаження одного файлу (Multer)

Multer — middleware для обробки `multipart/form-data`, який використовується для завантаження файлів.

**Код конфігурації:**
```javascript
const multer = require('multer');
const path = require('path');

// Custom storage configuration
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => {
        // Filename: timestamp + original name
        cb(null, Date.now() + '-' + file.originalname);
    }
});

const upload = multer({ storage });
```

**Endpoint для завантаження:**
```javascript
app.post('/upload', upload.single('file'), (req, res) => {
    if (!req.file) {
        return res.status(400).json({ error: 'No file uploaded' });
    }
    logger.info(`File uploaded: ${req.file.originalname}`);
    res.json({
        message: 'File uploaded successfully',
        file: req.file
    });
});
```

**Перевірка:**
1. Відкрийте `http://localhost:3000`
2. Виберіть файл у формі "Single File Upload"
3. Натисніть "Upload Single File"

![Single File Upload](/assets/labs/lab-4/single-upload.png)

**Приклад відповіді сервера:**
```json
{
    "message": "File uploaded successfully",
    "file": {
        "fieldname": "file",
        "originalname": "photo.jpg",
        "encoding": "7bit",
        "mimetype": "image/jpeg",
        "destination": "uploads/",
        "filename": "1715636400000-photo.jpg",
        "path": "uploads/1715636400000-photo.jpg",
        "size": 12345
    }
}
```

---

### Завдання 6: Завантаження кількох файлів

Розширено функціонал для підтримки завантаження до 5 файлів одночасно.

**Код:**
```javascript
// upload.array('files', 5) - accepts up to 5 files
app.post('/upload-multiple', upload.array('files', 5), (req, res) => {
    if (!req.files || req.files.length === 0) {
        return res.status(400).json({ error: 'No files uploaded' });
    }
    logger.info(`Multiple files uploaded: ${req.files.length} files`);
    res.json({
        message: 'Files uploaded successfully',
        count: req.files.length,
        files: req.files
    });
});
```

**Як перевірити:**
1. Відкрийте `http://localhost:3000`
2. У формі "Multiple Files Upload" виберіть кілька файлів (Ctrl+Click)
3. Натисніть "Upload Multiple Files"

**Приклад відповіді:**
```json
{
    "message": "Files uploaded successfully",
    "count": 3,
    "files": [
        { "originalname": "photo1.jpg", "filename": "1715636400001-photo1.jpg", ... },
        { "originalname": "photo2.png", "filename": "1715636400002-photo2.png", ... },
        { "originalname": "doc.pdf", "filename": "1715636400003-doc.pdf", ... }
    ]
}
```

---

### Завдання 7: Валідація файлів

Додано перевірки безпеки:
1. **Тип файлу** — дозволені тільки JPG, PNG, PDF
2. **Розмір файлу** — максимум 5MB

**Код fileFilter:**
```javascript
const fileFilter = (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (allowedTypes.includes(file.mimetype)) {
        cb(null, true); // Accept file
    } else {
        cb(new Error('Invalid file type. Only JPG, PNG and PDF allowed.'), false);
    }
};

// Multer with validation
const upload = multer({
    storage,
    fileFilter,
    limits: {
        fileSize: 5 * 1024 * 1024 // 5MB limit
    }
});
```

---

### Завдання 8: Моніторинг стану сервера

Створено endpoint `/status`, який повертає інформацію про здоров'я сервера:
- **uptime** — час роботи сервера
- **memoryUsage** — використання оперативної пам'яті

**Код:**
```javascript
app.get('/status', (req, res) => {
    const memoryUsage = process.memoryUsage();
    const uptime = process.uptime();

    res.json({
        status: 'OK',
        uptime: `${Math.floor(uptime)} seconds`,
        memoryUsage: {
            rss: `${Math.round(memoryUsage.rss / 1024 / 1024)} MB`,
            heapTotal: `${Math.round(memoryUsage.heapTotal / 1024 / 1024)} MB`,
            heapUsed: `${Math.round(memoryUsage.heapUsed / 1024 / 1024)} MB`
        }
    });
});
```

**Перевірка:**
```bash
curl http://localhost:3000/status
```

![Server Status](/assets/labs/lab-4/server-status.png)

**Приклад відповіді:**
```json
{
    "status": "OK",
    "uptime": "125 seconds",
    "memoryUsage": {
        "rss": "48 MB",
        "heapTotal": "18 MB",
        "heapUsed": "10 MB"
    }
}
```

---

### Завдання 9: Вимірювання часу відповіді

Реалізовано middleware, який вимірює час обробки кожного запиту та записує його у лог.

**Код:**
```javascript
app.use((req, res, next) => {
    const start = Date.now();

    res.on('finish', () => {
        const duration = Date.now() - start;
        logger.info({
            method: req.method,
            url: req.url,
            status: res.statusCode,
            duration: `${duration}ms`
        });
    });

    next();
});
```

**Перевірка:**
1. Зробіть кілька запитів до сервера
2. Перегляньте лог-файл:
```bash
type app.log
```

**Приклад записів у app.log:**
```json
{"level":"info","method":"GET","url":"/","status":200,"duration":"3ms","timestamp":"..."}
{"level":"info","method":"POST","url":"/upload","status":200,"duration":"45ms","timestamp":"..."}
{"level":"info","method":"GET","url":"/status","status":200,"duration":"1ms","timestamp":"..."}
```

---

### Завдання 10: Інтеграція менеджера процесів PM2

PM2 — це менеджер процесів для Node.js, який забезпечує:
- Автоматичний перезапуск при падінні
- Моніторинг у реальному часі
- Управління логами

**Встановлення PM2:**
```bash
npm install -g pm2
```

**Запуск застосунку:**
```bash
pm2 start app.js --name "lab4-server"
```

![PM2 Start](/assets/labs/lab-4/pm2-start.png)

**Перегляд списку процесів:**
```bash
pm2 list
```

**Перегляд логів:**
```bash
pm2 logs
```

![PM2 Logs](/assets/labs/lab-4/pm2-logs.png)

**Інтерактивний моніторинг:**
```bash
pm2 monit
```

![PM2 Monit](/assets/labs/lab-4/pm2-monit.png)

**Корисні команди PM2:**
| Команда | Опис |
|---------|------|
| `pm2 list` | Показати список процесів |
| `pm2 logs` | Переглянути логи |
| `pm2 monit` | Інтерактивна панель моніторингу |
| `pm2 restart lab4-server` | Перезапустити застосунок |
| `pm2 stop lab4-server` | Зупинити застосунок |
| `pm2 delete lab4-server` | Видалити з PM2 |

---

## Повний код app.js

```javascript
const express = require('express');
const morgan = require('morgan');
const winston = require('winston');
const multer = require('multer');
const path = require('path');

const app = express();

// Multer configuration
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, 'uploads/');
    },
    filename: (req, file, cb) => {
        cb(null, Date.now() + '-' + file.originalname);
    }
});

// File validation
const fileFilter = (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (allowedTypes.includes(file.mimetype)) {
        cb(null, true);
    } else {
        cb(new Error('Invalid file type. Only JPG, PNG and PDF allowed.'), false);
    }
};

const upload = multer({
    storage,
    fileFilter,
    limits: { fileSize: 5 * 1024 * 1024 }
});

// Winston logger
const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'app.log' })
    ]
});

// Morgan - HTTP request logging
app.use(morgan('combined'));

// Response time middleware
app.use((req, res, next) => {
    const start = Date.now();
    res.on('finish', () => {
        const duration = Date.now() - start;
        logger.info({
            method: req.method,
            url: req.url,
            status: res.statusCode,
            duration: `${duration}ms`
        });
    });
    next();
});

// Main page with upload form
app.get('/', (req, res) => {
    res.send(`<!-- HTML Form -->`);
});

// Single file upload
app.post('/upload', upload.single('file'), (req, res) => {
    if (!req.file) {
        return res.status(400).json({ error: 'No file uploaded' });
    }
    logger.info(`File uploaded: ${req.file.originalname}`);
    res.json({ message: 'File uploaded successfully', file: req.file });
});

// Multiple files upload
app.post('/upload-multiple', upload.array('files', 5), (req, res) => {
    if (!req.files || req.files.length === 0) {
        return res.status(400).json({ error: 'No files uploaded' });
    }
    logger.info(`Multiple files uploaded: ${req.files.length} files`);
    res.json({ message: 'Files uploaded successfully', count: req.files.length, files: req.files });
});

// Server status
app.get('/status', (req, res) => {
    const memoryUsage = process.memoryUsage();
    const uptime = process.uptime();
    res.json({
        status: 'OK',
        uptime: `${Math.floor(uptime)} seconds`,
        memoryUsage: {
            rss: `${Math.round(memoryUsage.rss / 1024 / 1024)} MB`,
            heapTotal: `${Math.round(memoryUsage.heapTotal / 1024 / 1024)} MB`,
            heapUsed: `${Math.round(memoryUsage.heapUsed / 1024 / 1024)} MB`
        }
    });
});

// Error handling middleware
app.use((err, req, res, next) => {
    logger.error({ message: err.message, stack: err.stack, url: req.url, method: req.method });
    res.status(err.status || 500).json({
        error: { message: err.message || 'Internal Server Error', status: err.status || 500 }
    });
});

app.listen(3000, () => {
    console.log('Server started on port 3000');
    logger.info('Server started on port 3000');
});
```
---

## Список endpoints

| Endpoint | Метод | Опис |
|----------|-------|------|
| `/` | GET | Головна сторінка з формою завантаження |
| `/upload` | POST | Завантаження одного файлу (field: `file`) |
| `/upload-multiple` | POST | Завантаження кількох файлів (field: `files`) |
| `/status` | GET | Моніторинг стану сервера |

---

## Команди

```bash
# 1. Запуск сервера
node app.js

# 2. Або через PM2
pm2 start app.js --name "lab4-server"

# 3. Перевірка статусу
curl http://localhost:3000/status

# 4. Перегляд логів Winston
type app.log

# 5. Перегляд завантажених файлів
dir uploads

# 6. PM2 команди
pm2 list
pm2 logs
pm2 monit
```

---

## Висновки

У ході виконання лабораторної роботи було:

1. **Вивчено логування** — Morgan для HTTP-запитів, Winston для файлового логування з рівнями та timestamps.

2. **Реалізовано завантаження файлів** — Multer для обробки multipart/form-data з валідацією типу та розміру.

3. **Створено систему моніторингу** — endpoint `/status` для відстеження uptime та використання пам'яті.

4. **Впроваджено PM2** — менеджер процесів для production-середовища з автоматичним перезапуском.
