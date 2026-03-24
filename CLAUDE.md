# CRM "Yura Transportation" — Система управління перевезеннями та посилками

## Огляд проєкту

Це повноцінна CRM-система для транспортної компанії "Yura Transportation", яка займається міжнародними перевезеннями пасажирів та доставкою посилок між Україною та Європою. Система складається з 4 інтерфейсів (логін, CRM посилок, CRM пасажирів, додаток водія) та 5 серверних скриптів (Google Apps Script).

## Технічний стек

- **Backend:** Google Apps Script (GAS) — серверні скрипти, деплояться як веб-додатки з `doPost()`/`doGet()` обробниками
- **Frontend:** Vanilla HTML/CSS/JavaScript — SPA-подібні сторінки без фреймворків, все вбудовано в один HTML-файл на кожен модуль
- **База даних:** Google Sheets — кілька таблиць як сховище даних
- **Карти:** Google Maps API — маршрутизація, геокодинг, оптимізація маршрутів
- **Інтеграції:** SmartSender (CRM контактів/розсилки)

## Структура файлів

```
Login.html                       — Сторінка входу + адмін-панель (946 рядків)
Cargo.html                      — CRM для посилок (7658 рядків)
Passengers.html                       — CRM для пасажирів (6861 рядків)
Drivers.html                  — Додаток для водіїв (1813 рядків)
Script-Cargo.gs            — API посилок (1192 рядків)
Script-Passengers.gs          — API пасажирів (1830 рядків)
Script-cargo-marshrut.gs     — API маршрутів посилок (1512 рядків)
Script-passengers-marshrut.gs   — API маршрутів пасажирів (1637 рядків)
Script-Archiv.gs                          — API архіву (1343 рядків)
```

## Архітектура бази даних (Google Sheets)

| ID таблиці | Призначення |
|---|---|
| `1RyWJ-ZQ-OQbeD65fZXR-WEwP_kwuNllikiA3Q1rjtlo` | Бот Посилки — основна таблиця посилок |
| `1U1deQJvMPZ9fctIEoHCXr8cFQmgWLVe2VRhlzb5IpjI` | Бот Пасажири — основна таблиця пасажирів |
| `1Pd3nv3fbwZ_0YSzdG4cda-q52BQT57E0hDe7eQej6z8` | Маршрутні листи посилок |
| `1iKlD0Bj-5qB3Gc1d5ZBHscbRipcSe5xU7svqBfpB77Y` | Маршрутні листи пасажирів |
| `1Kmf6NF1sJUi-j3SamrhUqz337pcZSvZCUkGxBzari6U` | Архів |

### Листи (sheets) в таблицях

**Посилки:** `Реєстрація ТТН` (UA→EU), `Виклик курєра` (EU→UA)
**Пасажири:** `Україна-єв` (UA→EU), `Європа-ук` (EU→UA)
**Маршрути:** `Пас. Маршрут 1`, `Пас. Маршрут 2`, `Пас. Маршрут 3` (пасажири); `Братислава марш.` та інші (посилки)

## Сутності (Entities)

### Посилка (Parcel)
24 колонки: ВО, Номер, ТТН, Вага, Адреса, Напрямок, Телефон, Сума, Статус оплати, Оплата, Телефон реєстрації, Примітка, Статус посилки (Невідомий/Зареєстровано/Оформлено/Кордон/Доставка), ID, Ім'я, Дата реєстрації, Тайминг, СМС примітка, Дата отримання, Фото, Статус, Дата архіву, Архівовано ким, Причина архіву, Лист-джерело, ID архіву.

### Пасажир (Passenger)
20 колонок: Дата, Звідки, Куди, Місця, Ім'я, Телефон, Мітка, Оплата, Відсоток, Диспетчер, ID, Телефон реєстрації, Вага, Авто, Тайминг, Дата реєстрації, Примітка, Статус, Дата архіву, ID архіву.

### Користувач (User)
Поля: Ім'я, Роль (Менеджер/Власник), Пароль, Статус (Активний/Заблокований), Час останнього входу, Пристрій, Дата створення.

## Бізнес-логіка

### Напрямки перевезень
- **UA→EU:** Реєстрація в листі `Реєстрація ТТН` (посилки) або `Україна-єв` (пасажири)
- **EU→UA:** Реєстрація в листі `Виклик курєра` (посилки) або `Європа-ук` (пасажири)

### Життєвий цикл статусів
```
new (Новий) → work (В роботі) → route (На маршруті) → archived/refused/transferred/deleted/completed
```
Статуси посилки окремо: `Невідомий → Зареєстровано → Оформлено → Кордон → Доставка`

### Детекція дублікатів
- Повний збіг: Телефон + Ім'я + Дата
- М'який збіг: Телефон + Ім'я (без дати)
- Блокує створення, якщо не передано `force=true`

### Управління потужністю транспорту
- Ліміт ваги на авто: 2000 кг (за замовчуванням)
- Максимум точок доставки: 60 (за замовчуванням)
- Попередження при перевищенні
- Перевірка кожні 5 хвилин

### Синхронізація даних
- Тиха синхронізація кожні 60 секунд (без лоадера)
- Злиття серверних даних з локальними змінами
- Збереження CRM-only полів при синхронізації
- Захист від втрати даних при конфліктах

## API ендпоінти

### API Посилок (`Script-Cargo.gs`)
POST: `getAll`, `getStructure`, `addPackage`, `updatePackage`, `updateField`, `updateStatus`, `bulkUpdateStatus`, `bulkAssignVehicle`, `deletePackage`, `archivePackage`, `bulkArchive`, `checkDuplicates`
GET: `health`

### API Пасажирів (`Script-Passengers.gs`)
POST: `getAll`, `getUaEu`, `getEuUa`, `getStructure`, `addPassenger`, `updatePassenger`, `updateMultiple`, `updateField`, `updateStatus`, `archivePassengers`, `restorePassengers`, `refusePassengers`, `transferPassengers`, `deletePassengers`, `deletePassengersPermanently`, `archivePassenger`, `bulkArchive`, `checkDuplicates`, `optimize`, `copyToRoute`, `checkRouteSheets`, `createRouteSheet`, `getRoutePassengers`, `getAvailableRoutes`, `deleteRouteSheet`
GET: `health`, `getPassengers`

### API Маршрутів Посилок (`Script-cargo-marshrut.gs`)
`getDeliveries`, `getAvailableRoutes` — для водіїв

### API Маршрутів Пасажирів (`Script-passengers-marshrut.gss`)
`getPassengers`, `getAvailableRoutes` — для водіїв

### API Архіву (`Script-Archiv.gs `)
POST: `archiveByStatus`, `archiveByIds`, `archiveByAge`, `archiveAll`, `restoreArchived`, `getArchived`, `deleteArchived`, `syncArchive`

## Аутентифікація

### Потік входу
1. Користувач обирає роль (Менеджер/Власник) на сторінці логіну
2. Вводить логін/пароль → POST на `AUTH_API_URL`
3. Бекенд перевіряє креденшали в Google Sheets
4. Успіх → `yura_auth` в `sessionStorage` + редірект на відповідну CRM
5. `localStorage` зберігає останнє ім'я для швидкого повторного входу

### Ролі
- **Менеджер (Manager):** Доступ до основних CRUD-операцій з лідами
- **Власник (Owner):** Повний доступ + дашборд зі статистикою + адмін-панель (управління користувачами, зміна паролів, блокування)

### Адмін-панель (тільки Власник)
- Перегляд списку користувачів
- Додавання нового користувача
- Зміна паролів
- Блокування/розблокування користувачів
- Трекінг пристроїв та історії входів

## Інтерфейси (Frontend)

### Сторінка входу (`Login.html`)
- Вибір ролі, форми логіну, адмін-панель

### CRM Посилок (`Cargo.html`)
- Дашборд (тільки Власник) — статистика, попередження потужності
- Нові ліди (основний вид)
- Всі посилки, фільтрація за напрямком (UA→EU, EU→UA)
- Архів, детекція дублікатів
- Бокова панель: фільтри за датою, авто, пошук, статус, напрямок
- Календар, inline-редагування, масові дії (статус, авто, архів, видалення)
- Модал карти для візуалізації адрес
- Збереження налаштувань в localStorage

### CRM Пасажирів (`Passengers.html `)
- Аналогічно до посилок + управління маршрутами + оптимізація маршрутів через Google Maps
- Детекція конфліктів маршрутів
- Копіювання пасажирів на маршрутний лист

### Додаток водія (`Drivers.html`)
- Вхід з автентифікацією маршруту (пароль)
- Список доставок за маршрутом (read-only)
- Трекінг статусів
- Інтеграція з картами

## URL-и API (деплойнуті Google Apps Script)

| Скрипт | URL |
|---|---|
| **Бот Пасажири** (`Script-Passengers.gs`) | `https://script.google.com/macros/s/AKfycbwIHwCKKabHJmODRV_L8v89B1b0GMT0EUuvUGt1QJGe2moACGCI3kOyOZt372InMGQr/exec` |
| **Бот Посилки** (`Script-Cargo.gs`) | `https://script.google.com/macros/s/AKfycbyMWUcDENjdTFwJeMkvv5s3G0v3SeCXaWBljp8l8TLaL1rJjl73lIcQxwXO9RQcewst/exec` |
| **Маршрути пасажирів** (`Script-passengers-marshrut.gs`) | `https://script.google.com/macros/s/AKfycbz2qmOqa_CgBWxUgY7Hc9Ni_pWQ8gfOBjN7iYhC0UwJKK36ha6BNfaz4MWGdCFmjMmy/exec` |
| **Маршрути посилок** (`Script-cargo-marshrut.gs`) | `https://script.google.com/macros/s/AKfycbzmBTSKUMZ8XCvH46whGhiV5eVvG1FRbvkZRIQjyFC2VqzsRd4Lox_sqm09Zqlamm1vVg/exec` |
| **Архів** (`Script-Archiv.gs`) | `https://script.google.com/macros/s/AKfycbwJLGZgYT333VdMW-nM5kPjYs2WIGGjfqkZnDJYjJxUt8nzE8GDGCPm7EzMHhcxNDOn/exec` |
| **Авторизація** (Login) | `https://script.google.com/macros/s/AKfycbzZ6QAMEANL2DI5cbBgSpcg5KaW9xW8643u5SEkbQplljr4fh3BU_bek4WBQH6ikO34ww/exec` |

## Стильові конвенції

- **Мова інтерфейсу:** Українська (uk_UA)
- **Формат дат:** YYYY-MM-DD
- **Часовий пояс:** Europe/Kiev (UTC+2)
- **Палітра:** Синій (#1a3a5e), Червоний (#d90d0d), Зелений (#10b981)
- **Layout:** CSS Grid + Flexbox, адаптивний дизайн (мобільний, планшет, десктоп)
- **Код:** Все в одному HTML-файлі на модуль (стилі, скрипти, розмітка)

## Правила розробки

1. **Кожен HTML-файл — самодостатній модуль** з вбудованими стилями та скриптами (Single File Architecture)
2. **Google Apps Script файли (.gs)** — серверна логіка з `doPost(e)`/`doGet(e)` точками входу, повертають `ContentService.createTextOutput(JSON.stringify(result)).setMimeType(ContentService.MimeType.JSON)`
3. **API комунікація** — завжди через `fetch()` з POST, тіло — JSON з полем `action` для маршрутизації
4. **Дані зберігаються в Google Sheets** — CRUD через `SpreadsheetApp.openById()`, рядки = записи, колонки = поля
5. **Статуси кольоровані:** pending=жовтий, in-progress=синій, completed=зелений, cancelled=червоний
6. **Фронтенд зберігає стан** в `sessionStorage` (авторизація) та `localStorage` (налаштування, фільтри, останній логін)
7. **Кожна зміна синхронізується** — після будь-якої дії виконується `loadData()` або тиха синхронізація
8. **Всі тексти українською** — змінні, коментарі, UI-тексти
