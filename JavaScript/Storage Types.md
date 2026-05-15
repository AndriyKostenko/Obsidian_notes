### USER
для чего и когда используется localstorage, sessionstorage, cookies в javascript?

### ASSISTANT
# Сравнение: localStorage, sessionStorage и cookies

## 📊 Таблица сравнения

| Характеристика         | localStorage              | sessionStorage            | Cookies                   |
| ---------------------- | ------------------------- | ------------------------- | ------------------------- |
| **Срок жизни**         | Постоянно (до удаления)   | До закрытия вкладки       | Настраивается (expires)   |
| **Объем**              | ~5-10 МБ                  | ~5-10 МБ                  | ~4 КБ                     |
| **Отправка на сервер** | Нет                       | Нет                       | Да (автоматически)        |
| **Доступность**        | Все вкладки одного домена | Только текущая вкладка    | Все вкладки одного домена |
| **API**                | Простой (setItem/getItem) | Простой (setItem/getItem) | Сложнее (document.cookie) |

---

## 1. 🟢 localStorage

**Когда использовать:**
- Долгосрочное хранение данных пользователя
- Настройки приложения (тема, язык)
- Кэширование данных
- Корзина покупок (если не критично)
- Черновики форм

**Пример:**
```javascript
// Сохранение
localStorage.setItem('theme', 'dark');
localStorage.setItem('user', JSON.stringify({id: 1, name: 'Alex'}));

// Чтение
const theme = localStorage.getItem('theme');
const user = JSON.parse(localStorage.getItem('user'));

// Удаление
localStorage.removeItem('theme');
localStorage.clear();
```

---

## 2. 🟡 sessionStorage

**Когда использовать:**
- Данные сессии (одна вкладка)
- Временные данные формы
- Данные для текущего визита
- Фильтры/сортировка в таблице

**Пример:**
```javascript
// Сохранение данных формы
sessionStorage.setItem('formData', JSON.stringify(formData));

// Чтение при возврате на страницу
const formData = JSON.parse(sessionStorage.getItem('formData'));

// Автоматически очистится при закрытии вкладки
```

---

## 3. 🔵 Cookies

**Когда использовать:**
- Аутентификация (токены, сессии)
- Данные, нужные на сервере
- Трекинг и аналитика
- Персонализация на сервере

**Пример:**
```javascript
// Установка cookie
document.cookie = "token=abc123; max-age=3600; path=/; secure; samesite=strict";

// Чтение всех cookie
const cookies = document.cookie.split('; ').reduce((acc, cookie) => {
  const [key, value] = cookie.split('=');
  acc[key] = value;
  return acc;
}, {});

// Удаление
document.cookie = "token=; max-age=0; path=/";
```

---

## 🎯 Практические сценарии

### ✅ localStorage
```javascript
// Тема приложения
localStorage.setItem('theme', 'dark');

// Язык интерфейса
localStorage.setItem('lang', 'ru');

// Корзина (некритичные данные)
localStorage.setItem('cart', JSON.stringify(cartItems));
```

### ✅ sessionStorage
```javascript
// Данные текущей сессии
sessionStorage.setItem('searchQuery', 'javascript');

// Временные данные формы
sessionStorage.setItem('draft', JSON.stringify(formDraft));

// ID текущей сессии
sessionStorage.setItem('sessionId', generateId());
```

### ✅ Cookies
```javascript
// JWT токен (с флагом HttpOnly на сервере)
document.cookie = "auth_token=xyz; max-age=86400; secure; samesite=strict";

// Предпочтения для сервера
document.cookie = "currency=USD; max-age=31536000; path=/";

// Аналитика
document.cookie = "visitor_id=123; max-age=63072000";
```

---

## ⚠️ Важные предупреждения

### Безопасность
```javascript
// ❌ НЕ храните в localStorage/sessionStorage:
localStorage.setItem('password', 'secret123');
localStorage.setItem('token', jwtToken); // Уязвимо для XSS

// ✅ Используйте HttpOnly cookies для токенов:
// Устанавливается сервером с флагом HttpOnly
```

### Ограничения
```javascript
// localStorage/sessionStorage - только строки
localStorage.setItem('number', 123); // "123"
const num = parseInt(localStorage.getItem('number'));

// Cookies - ограничение 4KB
// localStorage - 5-10MB
// sessionStorage - 5-10MB
```

---

## 📋 Чеклист выбора

```
Нужно хранить данные?
├─ Нужны на сервере? → 🍪 Cookies
├─ Только для текущей вкладки? → 🟡 sessionStorage
├─ Долгосрочное хранение? → 🟢 localStorage
│
├─ Чувствительные данные (токены)? → 🍪 HttpOnly Cookies
├─ Настройки пользователя? → 🟢 localStorage
└─ Временные данные формы? → 🟡 sessionStorage
```

---

## 🔒 Best Practices

```javascript
// 1. Всегда проверяйте наличие данных
const data = localStorage.getItem('key');
if (data) {
  // обработка
}

// 2. Обрабатывайте ошибки (quota exceeded)
try {
  localStorage.setItem('key', largeData);
} catch (e) {
  console.log('LocalStorage full');
}

// 3. Очищайте старые данные
const expiry = localStorage.getItem('data_expiry');
if (expiry && Date.now() > expiry) {
  localStorage.removeItem('data');
}

// 4. Используйте префиксы для ключей
localStorage.setItem('app_user_settings', value);
```

**Выбор зависит от:** срока жизни данных, необходимости доступа на сервере, требований безопасности и объема данных.