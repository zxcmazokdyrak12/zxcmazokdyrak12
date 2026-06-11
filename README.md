# Chainsaw Chat — полное описание проекта

---

## Архитектура

```

chainsaw-chat/

├── client/          ← React фронтенд (Vite)

└── server/          ← Node.js бэкенд

```

Три платформы:
- **Vercel** — хостинг фронта
- **Render** — хостинг сервера
- **Neon** — PostgreSQL база данных

---

## Бэкенд (`server/index.js`)

**Стек:** Node.js + Express + Socket.io + PostgreSQL

### HTTP сервер (Express)
Обрабатывает обычные запросы:
- `GET /ping` — проверка что сервер жив
- `POST /auth/register` — регистрация (bcrypt хеширует пароль, возвращает JWT)
- `POST /auth/login` — логин (сравнивает пароль с хешем, возвращает JWT)
- `GET /auth/me` — проверка токена
- `GET /auth/github` → `GET /auth/github/callback` — OAuth через GitHub
- `GET /auth/google` → `GET /auth/google/callback` — OAuth через Google

### WebSocket сервер (Socket.io)
Обрабатывает реалтайм события:

| Событие | Направление | Что делает |
|---|---|---|
| `get\\\_rooms` | клиент → сервер | получить комнаты пользователя |
| `rooms\\\_list` | сервер → клиент | отдать список комнат |
| `create\\\_room` | клиент → сервер | создать комнату, добавить в room_members |
| `join\\\_by\\\_code` | клиент → сервер | найти комнату по коду, добавить участника |
| `join\\\_room` | клиент → сервер | подключиться к Socket.io комнате |
| `room\\\_history` | сервер → клиент | история сообщений из БД |
| `send\\\_message` | клиент → сервер | сохранить в БД, разослать всем в комнате |
| `receive\\\_message` | сервер → клиент | новое сообщение |
| `typing\\\_start/stop` | клиент → сервер | статус печатает |
| `typing\\\_update` | сервер → клиент | кто печатает |
| `edit\\\_message` | клиент → сервер | обновить текст в БД |
| `delete\\\_message` | клиент → сервер | удалить из БД |
| `pin\\\_message` | клиент → сервер | закрепить/открепить |
| `user\\\_joined` | сервер → клиент | системное сообщение о входе |

### Защита
- **Helmet** — защитные HTTP заголовки
- **Rate limiting** — 100 HTTP запросов в минуту с одного IP
- **Socket rate limit** — 30 сообщений в минуту на сокет
- **bcryptjs** — хеширование паролей (соль 10 раундов)
- **JWT** — токены на 30 дней, хранятся в localStorage

---

## База данных (PostgreSQL / Neon)

### Таблицы:

**`local\\\_users`** — пользователи с логином/паролем
```sql

id, username (уникальный), password (хеш), avatar, created\\\_at

```

**`chat\\\_users`** — пользователи OAuth (GitHub/Google)
```sql

id, provider, provider\\\_id, username, avatar, email, created\\\_at

```

**`rooms`** — комнаты
```sql

id, name, code (уникальный 8 символов), created\\\_by, created\\\_at

```

**`room\\\_members`** — кто состоит в каких комнатах
```sql

room\\\_id, username, joined\\\_at

PRIMARY KEY (room\\\_id, username)

```

**`messages`** — сообщения
```sql

id, room\\\_id, username, content, type, audio\\\_data, avatar,

time, rotate, system, is\\\_edited, is\\\_pinned, created\\\_at

```

### Логика комнат:
Комната не отображается у пользователя пока он не создал её или не вошёл по коду. Это реализовано через JOIN с `room\\\_members`.

---

## Фронтенд (`client/src/App.jsx`)

**Стек:** React + Vite + Framer Motion + Socket.io-client

### Фазы приложения (`phase`):
```

'intro' → 'slashing' → 'username' → 'chat'

```
- `intro` — экран с кнопкой START ENGINE
- `slashing` — анимация разреза (750мс)
- `username` — экран регистрации/логина + выбор аватарки
- `chat` — основной чат

### Стейты:
```javascript

phase, username, usernameInput        // авторизация

authMode, authForm, authError         // форма логин/регистрация

messages, input                       // чат

theme ('light'|'dark')                // тема

activeTab                             // chats/rooms/settings/profile

currentRoom, rooms                    // комнаты

typingUsers                           // кто печатает

pochitaVisible, hearts                // пасхалка

avatar                                // выбранный персонаж

sidebarOpen                           // сайдбар

showCreateRoom, showJoinByCode        // модалки комнат

isRecording                           // запись аудио

```

### Основные компоненты UI:

**Экран входа** — анимация разреза экрана, рука бензопилы летит по диагонали, звук chainsaw.mp3

**Экран авторизации** — переключатель LOGIN/REGISTER, сетка из 21 аватарки персонажей, форма, кнопки OAuth

**Шапка** — логотип + текущая комната + имя пользователя + кнопка темы

**Навигация** — 4 вкладки с чиби персонажами на иконках (Денджи/Аки/Макима/Резе). При клике персонаж подпрыгивает

**Сайдбар** — список комнат пользователя, скрывается кнопкой

**Область сообщений** — манга-баблы с хвостиками, наклон каждого бабла случайный, SFX слова (VROOM/SLASH/BANG...)

**Инпут** — кнопка микрофона (зажать = запись), поле ввода, кнопка отправки

**Вкладка ROOMS** — создать комнату, войти по коду, список комнат с кодами

**Вкладка PROFILE** — аватарка персонажа, имя, текущая комната

### Пасхалки:
- **Почита + сердечки** — при словах "love", "cute", "мило" и т.д. с вероятностью 40%
- **Звук бензопилы** — при отправке сообщения ЗАГЛАВНЫМИ с вероятностью 10%
- **Стиль крика** — CAPS текст рендерится другим шрифтом и размером

### Темы:
- **Светлая** — фон цвета старой бумаги `#f0ebe0`, чёрные обводки, красные акценты
- **Тёмная** — `#111`, белые обводки, минимум цвета

### Шрифты (из архива Chainsaw Man):
- `BlambotClassic` — основной текст в баблах
- `CCDoohickey` — SFX слова
- `DeathRattle` — крик (капс)
- `AnimeAce` — имена, время, системные сообщения
- `Broadband` — заголовок

### Аватарки:
21 персонаж: Denji, Power, Makima, Aki, Reze, Himeno, Kishibe, Kobeni, Asa Mitaka, Quanxi, Nayuta, Yoru, Cosmo, Beam, Yoshida, Tenshi, Pingtsi, Sawatari, Kiga, Tendou, Pochita

---

## Медиа файлы (`client/public/`)

```

hand.png          ← бензопила на экране входа

chainsaw.mp3      ← звук бензопилы

pochita.png       ← Почита для пасхалки

bubble.png        ← бабл для чужих сообщений

bubble\\\_me.png     ← бабл для своих

heart\\\_0..8.png    ← 9 нарисованных сердечек

avatars/          ← 20 аватарок персонажей

icon\\\_chat.png     ← иконка вкладки чатов

icon\\\_rooms.png    ← иконка вкладки комнат

icon\\\_settings.png ← иконка настроек

icon\\\_profile.png  ← иконка профиля

denji/aki/makima/reze.png ← чиби для навигации

fonts/            ← шрифты Chainsaw Man

```

---

## Что умеет проект

- ✅ Реалтайм мультиплеер через WebSocket
- ✅ Регистрация/логин с паролем
- ✅ OAuth через GitHub и Google
- ✅ Приватные комнаты — видны только участникам
- ✅ Создание комнат с уникальным кодом
- ✅ Вход по коду
- ✅ История сообщений из PostgreSQL
- ✅ Typing indicator
- ✅ Аудио сообщения
- ✅ Выбор аватарки из 21 персонажа
- ✅ Две темы
- ✅ Helmet + Rate limiting
- ✅ Задеплоен на Vercel + Render + Neon
