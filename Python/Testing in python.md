Тестирование в Python вращается вокруг двух основных фреймворков: встроенного `unittest` и более гибкого `pytest`. Оба поддерживают мокирование, подготовку состояния и изоляцию зависимостей, но реализуют это разными механизмами. Разберём запрошенные концепции и дополним их часто используемыми инструментами.

---
## 🔹 1. `fixture` в pytest
**Что это:** Функция, которая подготавливает и/или очищает состояние для тестов. Заменяет `setUp`/`tearDown` из `unittest`, но гораздо мощнее за счёт системы зависимостей (dependency injection).

**Как работает:**
```python
import pytest

@pytest.fixture
def db_connection():
    conn = create_database()
    yield conn          # выполнение теста происходит здесь
    conn.close()        # teardown

def test_user_creation(db_connection):
    db_connection.insert({"name": "Alice"})
    assert db_connection.count() == 1
```

**Ключевые особенности:**
- `scope`: `function` (по умолчанию), `class`, `module`, `session` → контролирует, как часто фикстура пересоздаётся.
- `@pytest.fixture(params=[...])` → параметризация фикстур.
- `request` → встроенный объект, даёт доступ к имени теста, параметрам, финализаторам.
- `conftest.py` → файл, в котором pytest автоматически ищет фикстуры для всей директории/пакета.
- `@pytest.mark.usefixtures("name")` → применить фикстуру, не передавая её в аргументы теста.

---
## 🔹 2. `MagicMock` в unittest
**Что это:** Класс из `unittest.mock`, который создаёт объект-заглушку. При обращении к любому атрибуту или вызове метода автоматически создаёт новый `MagicMock` и записывает все взаимодействия.

**Зачем нужен:** Изоляция кода от внешних зависимостей (HTTP, БД, файлы, сторонние SDK) без запуска реальных систем.

**Основные возможности:**
```python
from unittest.mock import MagicMock, patch

def test_api_call():
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"id": 1, "status": "ok"}

    with patch("requests.get", return_value=mock_response) as mock_get:
        result = fetch_user(1)
        mock_get.assert_called_once_with("https://api.example.com/users/1")
        assert result == {"id": 1, "status": "ok"}
```

**Полезные методы/атрибуты:**
- `return_value`, `side_effect` (функция/исключение/итератор)
- `assert_called_once()`, `assert_called_with()`, `assert_any_call()`, `assert_has_calls()`
- `mock_calls`, `call_args_list` → история вызовов
- `patch`, `patch.object`, `patch.dict` → замена реальных объектов на моки

> 💡 `Mock` vs `MagicMock`: `MagicMock` автоматически имитирует магические методы (`__str__`, `__enter__`, `__getitem__` и т.д.), поэтому используется почти всегда.

---
## 🔹 3. `AsyncMock` для асинхронного кода
**Проблема:** Обычный `MagicMock` не является корутиной. `await mock_func()` выбросит `TypeError`.

**Решение:** `AsyncMock` (добавлен в Python 3.8) наследуется от `MagicMock`, но все методы возвращают `coroutine`, а проверки используют `assert_awaited_*`.

### В `unittest`:
```python
from unittest.mock import AsyncMock, patch
import asyncio

async def test_async_client():
    mock_client = AsyncMock()
    mock_client.fetch.return_value = {"data": 42}

    await my_async_service(mock_client)
    mock_client.fetch.assert_awaited_once()
```

### В `pytest`:
Нативно работает с Python 3.8+, но для запуска асинхронных тестов нужен плагин:
```bash
pip install pytest-asyncio
```
```python
import pytest
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_async_flow():
    mock_db = AsyncMock()
    mock_db.query.return_value = [{"id": 1}]

    result = await get_users(mock_db)
    mock_db.query.assert_awaited_once_with("SELECT * FROM users")
    assert len(result) == 1
```

> 💡 В `pytest` часто используют плагин `pytest-mock`, который даёт фикстуру `mocker`. Она оборачивает `unittest.mock` и упрощает работу:
> ```python
> def test_with_mocker(mocker):
>     mock = mocker.patch("module.func", new_callable=AsyncMock)
>     mock.return_value = "ok"
> ```

---
## 🔹 4. Другие часто используемые инструменты

### ✅ В `pytest`
| Инструмент | Назначение |
|------------|------------|
| `@pytest.mark.parametrize` | Один тест с разными входными/выходными данными |
| `tmp_path` / `tmpdir` | Временные директории/файлы (автоматическая очистка) |
| `capsys`, `caplog` | Захват `stdout`/`stderr` и логов для ассертов |
| `monkeypatch` | Безопасная замена атрибутов, переменных окружения, импортов |
| `pytest-cov` | Измерение покрытия кода (`pytest --cov=myapp`) |
| `pytest-xdist` | Параллельный запуск тестов (`pytest -n auto`) |
| `pytest-asyncio` / `anyio` | Запуск `async def` тестов |
| `pytest-mock` | Фикстура `mocker` для удобного `patch` без `with`/декораторов |

### ✅ В `unittest`
| Инструмент | Назначение |
|------------|------------|
| `setUp` / `tearDown` | Подготовка/очистка перед каждым тестом |
| `setUpClass` / `tearDownClass` | Однократная подготовка для всего класса |
| `subTest` | Группировка проверок внутри одного теста (не прерывает тест при падении одной проверки) |
| `assertRaises`, `assertLogs`, `assertWarns` | Контекстные менеджеры для исключений/логов |
| `python -m unittest discover` | Автоматический поиск тестов по маске |
| `unittest.mock` | Встроенное мокирование (`Mock`, `MagicMock`, `AsyncMock`, `patch`) |

### 🌍 Общие/Экосистемные
- `hypothesis` → Property-based testing (генерация входных данных)
- `responses` / `aioresponses` / `httpx.MockTransport` → Мокирование HTTP без `patch`
- `syrupy` / `pytest-snapshot` → Snapshot-тесты (сравнение с сохранённым эталоном)
- `factory_boy` / `polyfactory` → Генерация тестовых данных/объектов
- `coverage.py` → Отчёты по покрытию (интегрируется с обоими фреймворками)

---
## 🔹 5. Рекомендации и best practices
1. **Не мокать всё подряд.** Мокайте только внешние границы (API, БД, файловая система, сеть). Бизнес-логику тестируйте на реальных объектах или лёгких двойниках.
2. **Фикстуры vs `setUp`.** В `pytest` фикстуры переиспользуются между тестами, имеют гибкий scope и поддерживают dependency injection. В `unittest` `setUp` проще, но менее гибок.
3. **Асинхронность.** Используйте `pytest-asyncio` + `AsyncMock`. Избегайте `asyncio.run()` внутри тестов → ломают event-loop.
4. `pytest-mock` предпочтительнее сырого `unittest.mock` в pytest-проектах: фикстура `mocker` автоматически откатывает патчи, даже если тест упал.
5. **Scope фикстур.** `session`/`module` ускоряют запуск, но могут приводить к "грязному" состоянию. Используйте их осознанно.
6. **Ассерты.** В `pytest` пишите обычные `assert a == b` → фреймворк переписывает их и показывает разницу значений. В `unittest` требуются `self.assertEqual`, `self.assertTrue` и т.д.

---
## 📦 Краткая сравнительная таблица
| Задача | `pytest` | `unittest` |
|--------|----------|------------|
| Подготовка состояния | `@pytest.fixture` | `setUp` / `setUpClass` |
| Мокирование | `unittest.mock` + `pytest-mock` | `unittest.mock` (встроен) |
| Асинхронные тесты | `@pytest.mark.asyncio` + `pytest-asyncio` | `asyncio` вручную или `IsolatedAsyncioTestCase` (Python 3.8+) |
| Параметризация | `@pytest.mark.parametrize` | `subTest` или циклы внутри теста |
| Запуск | `pytest` / `pytest -xvs` | `python -m unittest` |

---
Если уточните стек (веб-фреймворк, тип зависимостей, уровень тестов: unit/integration/e2e), смогу привести конкретные шаблоны под ваш проект.