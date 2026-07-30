 ## 🔍 Что такое уровни изоляции транзакций?

Уровни изоляции — часть принципа **ACID** (Isolation). Они определяют, **как concurrent-транзакции видят изменения друг друга**. Без изоляции параллельные запросы могут приводить к аномалиям: «грязным» чтениям, неповторяемым чтениям, фантомным строкам и нарушениям сериализуемости.

PostgreSQL использует **MVCC** (Multi-Version Concurrency Control): каждая транзакция работает со своим «снэпшотом» данных. Уровень изоляции управляет тем, насколько свежий этот снэпшот и как строго PostgreSQL контролирует конфликты.

---

## 📊 Уровни изоляции в PostgreSQL vs SQL-стандарт

SQL-стандарт определяет 4 уровня. PostgreSQL реализует их с особенностями:

| Уровень | Поддержка в PG | Что предотвращает |
|--------|----------------|-------------------|
| `READ UNCOMMITTED` | ❌ Не поддерживается. Автоматически маппится на `READ COMMITTED` | — |
| `READ COMMITTED` | ✅ Default | Dirty reads |
| `REPEATABLE READ` | ✅ | Dirty reads, Non-repeatable reads, **Phantom reads** (в PG строже стандарта) |
| `SERIALIZABLE` | ✅ (с 9.1 через SSI) | Все выше + Serialization anomalies |

📌 **Важно:** В PostgreSQL фактически два разных механизма изоляции:
- `READ COMMITTED` → каждый запрос видит актуальный коммит на момент своего старта
- `REPEATABLE READ` / `SERIALIZABLE` → транзакция работает с единым снэпшотом. `SERIALIZABLE` добавляет детектор аномалий (SSI) и может откатить транзакцию при конфликте сериализации.

---

## ⚙️ Как настраивать уровни изоляции в PostgreSQL

### 1. В рамках одной транзакции
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- или сразу: BEGIN ISOLATION LEVEL REPEATABLE READ;
-- ... ваши запросы ...
COMMIT;
```
⚠️ `SET TRANSACTION` должен быть **первой командой** в транзакции. Менять уровень после выполнения запросов нельзя.

### 2. Для всей сессии (до конца соединения)
```sql
SET default_transaction_isolation TO 'serializable';
-- или
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### 3. На уровне базы данных или роли
```sql
ALTER DATABASE mydb SET default_transaction_isolation TO 'repeatable read';
ALTER ROLE myuser SET default_transaction_isolation TO 'serializable';
```

### 4. Глобально (в `postgresql.conf`)
```ini
default_transaction_isolation = 'read committed'
```
После изменения: `SELECT pg_reload_conf();` или рестарт сервера.

---

## 💻 Как применять в коде (примеры)

### 🐍 Python (psycopg2)
```python
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_SERIALIZABLE

conn = psycopg2.connect("dbname=mydb user=myuser")
conn.set_isolation_level(ISOLATION_LEVEL_SERIALIZABLE)

with conn.cursor() as cur:
    cur.execute("BEGIN")  # psycopg2 уже управляет транзакцией
    cur.execute("SELECT balance FROM accounts WHERE id = 1")
    # ... логика ...
    conn.commit()
```
В SQLAlchemy:
```python
session.execute(
    text("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE"),
    execution_options={"isolation_level": "SERIALIZABLE"}
)
```

### 🟩 Node.js (`pg`)
```js
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('SET TRANSACTION ISOLATION LEVEL REPEATABLE READ');
  // ... запросы ...
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release();
}
```

### ☕ Java (JDBC)
```java
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false);
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
// ... выполнение запросов ...
conn.commit();
```

### 🗄️ ORM (Prisma, Django, Hibernate)
- Большинство ORM позволяют задать уровень изоляции при начале транзакции:
  - Prisma: `prisma.$transaction([...], { isolationLevel: 'Serializable' })`
  - Django: `with transaction.atomic():` + настройка через `ATOMIC_REQUESTS` или сырой SQL
  - Hibernate/JPA: `session.doWork(conn -> conn.setTransactionIsolation(...))` или аннотации `@Transactional(isolation = Isolation.SERIALIZABLE)`

---

## 🛡️ Практические рекомендации и подводные камни

| Уровень | Когда использовать | Нюансы |
|--------|-------------------|--------|
| `READ COMMITTED` | 90% веб-приложений, CRUD, отчёты, очереди | Default. Быстрый, но возможны «скачки» данных между запросами в одной транзакции |
| `REPEATABLE READ` | Финансовые вычёты, согласованные выборки, миграции данных | Стабильный снэпшот. В PG защищает и от фантомов. Не блокирует запись другими транзакциями |
| `SERIALIZABLE` | Платежи, инвентаризация, критичная консистентность | Может возвращать ошибку `40001 (serialization_failure)`. **Обязательно нужен retry-механизм** |

### 🔁 Как обрабатывать `40001` (SERIALIZABLE)
```python
import psycopg2
import time

def run_serializable(fn, max_retries=3):
    for attempt in range(max_retries):
        try:
            with conn:
                conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_SERIALIZABLE)
                return fn()
        except psycopg2.errors.SerializationFailure:
            conn.rollback()
            time.sleep(0.1 * (2 ** attempt))  # экспоненциальная задержка
    raise Exception("Max retries exceeded")
```

### 📌 Важные правила
1. **Изоляция применяется к транзакции**, а не к отдельному запросу.
2. Нельзя изменить уровень изоляции после выполнения первого `SELECT`/`INSERT`/`UPDATE` в транзакции.
3. `REPEATABLE READ` и `SERIALIZABLE` не блокируют чтение. Блокировки возникают только при конфликтах модификаций или при детекции аномалий SSI.
4. Длинные транзакции с высоким уровнем изоляции увеличивают нагрузку на undo-систему (хранение старых версий строк). Используйте `VACUUM` и мониторьте `pg_stat_user_tables.dead_tuples`.

---

## ✅ Итог
- По умолчанию в PostgreSQL стоит `READ COMMITTED` → подходит для большинства задач.
- Используйте `REPEATABLE READ`, когда нужно гарантировать, что данные не изменятся между запросами в одной транзакции.
- `SERIALIZABLE` → для строгих требований к консистентности, но готовьте retry-логику и измеряйте влияние на throughput.
- Настройка возможна на уровне транзакции, сессии, роли, БД или глобально в конфиге.
- В коде уровень задаётся либо через драйвер/ORM, либо явным `SET TRANSACTION ...` сразу после `BEGIN`.

Если скажете, какой язык/ORM используете, могу дать готовый шаблон транзакции с нужным уровнем изоляции и обработкой конфликтов.