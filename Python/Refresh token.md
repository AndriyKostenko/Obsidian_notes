## 📌 Для чего нужен refresh token?

`Access token` обычно выдаётся на короткий срок (5–30 минут), чтобы минимизировать риски при его утечке. `Refresh token` живёт гораздо дольше (дни, недели, месяцы) и **используется исключительно для получения нового access token** без повторного ввода логина и пароля.

**Преимущества:**

- 🔒 Безопасность: короткий lifetime access token снижает окно атаки
- 🔄 Удобство: пользователь не видит формы логина при каждом истечении сессии
- 🚫 Контроль: refresh token можно отозвать на сервере (logout, смена пароля, подозрительная активность)

**Access Token** (токен доступа) обычно живёт от 5 до 30 минут. Если он скомпрометирован, злоумышленник получит доступ к API только на это короткое время.  
**Refresh Token** (токен обновления) живёт значительно дольше (дни/недели) и используется **только** для получения новой пары `access + refresh` без повторного ввода логина/пароля.

**Основные цели:**

1. ✅ Улучшение безопасности: короткий life-time access token минимизирует ущерб при утечке.
2. 🔄 Удобство UX: пользователь не выходит из системы каждые 15 минут.
3. 🚫 Контроль сессий: сервер может отозвать refresh token в любой момент (выход из системы, смена пароля, подозрительная активность).
4. 📱 Поддержка мобильных/SPA приложений, где безопасное хранение токенов ограничено.

### 🛡 Ключевые практики безопасности

| Практика                                   | Зачем                                                                                                               |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `cookie httponly=True`                     | JS не может прочитать cookie → защита от XSS                                                                        |
| `secure=True`                              | Передача только по HTTPS → защита от MitM                                                                           |
| `samesite="lax"` или `"strict"`            | Защита от CSRF                                                                                                      |
| Token Rotation                             | При каждом `/refresh` выдаётся новый refresh, старый отзывается. Утечка одного токена не даёт бесконечного доступа. |
| Ограничение по IP/User-Agent (опционально) | Привязка refresh token к клиенту усложняет replay-атаки                                                             |
| Хранение `jti` в Redis/DB                  | Позволяет мгновенно отзывать сессии без ожидания истечения JWT                                                      |
| Короткий `access` (5-15 мин)               | Минимизирует окно уязвимости                                                                                        |

---

### 🚀 Что изменить для продакшена?

1. **Хранилище:** заменить `dict` на Redis (`redis-py`) или PostgreSQL с TTL-индексами.
2. **Секреты:** `SECRET_KEY`, `ALGORITHM`, таймауты → из переменных окружения (`pydantic-settings`).
3. **Валидация паролей:** `passlib[bcrypt]` или `argon2-cffi`.
4. **Rate Limiting:** на `/auth/login` и `/auth/refresh` (например, `slowapi` или NGINX).
5. **Логирование/Аудит:** фиксация выдач/отзывов токенов, IP, User-Agent.
6. **OAuth 2.1 совместимость:** рассмотрите PKCE для публичных клиентов (SPA/mobile).

## 🛡️ Что обязательно учесть в production

| Аспект                  | Рекомендация                                                                                                                           |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Хранение**            | Сохраняйте `refresh_token` в БД/Redis с привязкой к пользователю, device, IP, времени создания                                         |
| **Отзыв**               | При logout, смене пароля или подозрении на компрометацию удаляйте/блокируйте токен на сервере                                          |
| **Ротация**             | При каждом `/refresh` выдавайте **новый** refresh token, а старый помечайте как использованный. Это ограничивает окно атаки при утечке |
| **Передача**            | Отправляйте `refresh_token` через `HttpOnly; Secure; SameSite=Strict` cookie, а не в JSON body. Это защищает от XSS                    |
| **Разделение секретов** | Используйте разные `SECRET_KEY` или алгоритмы для access и refresh токенов                                                             |
| **Rate Limiting**       | Ограничьте частоту запросов к `/refresh` (например, 10 запросов/мин с одного IP)                                                       |
| **Валидация claims**    | Проверяйте `sub`, `type`, `exp`, `iat`, `jti` (уникальный ID токена)                                                                   |


### 💻 Пример на FastAPI

> ⚠️ Пример упрощён для наглядности. В продакшене вместо `dict` используйте Redis/PostgreSQL, а секреты выносите в `.env`.

```python
from fastapi import FastAPI, HTTPException, Response, Cookie, Depends
from pydantic import BaseModel
from jose import JWTError, jwt
from datetime import datetime, timedelta, timezone
import secrets
from typing import Optional

app = FastAPI(title="FastAPI Refresh Token Example")

# 🔑 Настройки (в проде брать из env)
SECRET_KEY = "super-secret-key-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15
REFRESH_TOKEN_EXPIRE_DAYS = 7

# 📦 Имитация БД для отслеживания refresh-токенов (в проде → Redis/PostgreSQL)
refresh_token_store: dict[str, dict] = {}

# 🔧 Утилиты создания токенов
def create_access_token(user_id: str) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    payload = {"sub": user_id, "exp": expire, "iat": datetime.now(timezone.utc), "type": "access"}
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def create_refresh_token(user_id: str) -> str:
    jti = secrets.token_hex(16)  # уникальный ID токена
    expire = datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    payload = {"sub": user_id, "jti": jti, "exp": expire, "iat": datetime.now(timezone.utc), "type": "refresh"}
    token = jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    
    # Сохраняем метаданные для возможности отзыва
    refresh_token_store[jti] = {"user_id": user_id, "exp": expire, "revoked": False}
    return token

# 📥 Модели запросов
class LoginRequest(BaseModel):
    username: str
    password: str

# 🔑 1. Авторизация
@app.post("/auth/login")
def login(req: LoginRequest, response: Response):
    # Имитация проверки учётных данных
    if req.username != "admin" or req.password != "secret":
        raise HTTPException(status_code=401, detail="Invalid credentials")

    access_token = create_access_token(req.username)
    refresh_token = create_refresh_token(req.username)

    # 🔒 Отправляем refresh только в httpOnly cookie (недоступен JS)
    response.set_cookie(
        key="refresh_token",
        value=refresh_token,
        httponly=True,
        secure=True,          # требует HTTPS в проде
        samesite="lax",
        max_age=REFRESH_TOKEN_EXPIRE_DAYS * 24 * 3600,
        path="/auth",
    )
    return {"access_token": access_token, "token_type": "bearer"}

# 🔄 2. Обновление access-токена
@app.post("/auth/refresh")
def refresh_token(refresh_token: Optional[str] = Cookie(None), response: Response = None):
    if not refresh_token:
        raise HTTPException(status_code=401, detail="Missing refresh token")

    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        
        if payload.get("type") != "refresh":
            raise HTTPException(status_code=401, detail="Invalid token type")

        jti = payload.get("jti")
        if jti not in refresh_token_store or refresh_token_store[jti]["revoked"]:
            raise HTTPException(status_code=401, detail="Token revoked or expired")

        # 🔄 Token Rotation: выдаём новый refresh, старый отзываем
        new_access = create_access_token(payload["sub"])
        new_refresh = create_refresh_token(payload["sub"])
        refresh_token_store[jti]["revoked"] = True

        response.set_cookie(
            key="refresh_token",
            value=new_refresh,
            httponly=True,
            secure=True,
            samesite="lax",
            max_age=REFRESH_TOKEN_EXPIRE_DAYS * 24 * 3600,
            path="/auth",
        )
        return {"access_token": new_access, "token_type": "bearer"}

    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid refresh token")

# 🚪 3. Выход (отзыв токена)
@app.post("/auth/logout")
def logout(response: Response, refresh_token: Optional[str] = Cookie(None)):
    if refresh_token:
        try:
            payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
            jti = payload.get("jti")
            if jti in refresh_token_store:
                refresh_token_store[jti]["revoked"] = True
        except JWTError:
            pass  # Токен уже просрочен или неверен

    response.delete_cookie(key="refresh_token", path="/auth")
    return {"detail": "Logged out successfully"}
```

---