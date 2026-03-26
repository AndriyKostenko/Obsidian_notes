
### USER
как работает event loop в Python?

### ASSISTANT
# Event Loop в Python: как работает

**Event loop** (цикл событий) — это ядро асинхронного программирования в Python. Он позволяет выполнять множество задач конкурентно в одном потоке, переключаясь между ними во время ожидания (например, ввода-вывода), не блокируя выполнение программы.

## 🔁 Что такое Event Loop

Event loop — это бесконечный цикл, который:
- Планирует и выполняет асинхронные задачи (coroutines)
- Отслеживает завершение операций ввода-вывода
- Переключает управление между задачами, когда они ожидают

## ⚙️ Как работает Event Loop

```python
import asyncio

async def task(name, delay):
    print(f"{name} started")
    await asyncio.sleep(delay)  # Неблокирующая задержка
    print(f"{name} completed")

async def main():
    await asyncio.gather(
        task("Task 1", 2),
        task("Task 2", 1),
        task("Task 3", 3)
    )

asyncio.run(main())
```

**Вывод:**
```
Task 1 started
Task 2 started
Task 3 started
Task 2 completed  # через 1 секунду
Task 1 completed  # через 2 секунды
Task 3 completed  # через 3 секунды
```

## 🧠 Ключевые концепции

| Концепция       | Описание                                               |
|----------------|--------------------------------------------------------|
| **Coroutine**   | Асинхронная функция (`async def`)                     |
| **Task**        | Запланированная coroutine в event loop                |
| **Future**      | Объект, представляющий результат асинхронной операции |
| **await**       | Приостанавливает выполнение до завершения задачи      |

## 🛠 Управление Event Loop

### Создание и запуск
```python
# Python 3.7+
asyncio.run(main())

# Ручное управление (для продвинутых сценариев)
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```

### Планирование задач
```python
# Создание задачи
task = asyncio.create_task(task_function())

# Ожидание нескольких задач
await asyncio.gather(task1, task2, task3)

# Ожидание с таймаутом
await asyncio.wait_for(task, timeout=5.0)
```

## 🌐 Практический пример: асинхронные HTTP-запросы

```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    urls = [
        'https://api.github.com',
        'https://httpbin.org/get',
        'https://jsonplaceholder.typicode.com/posts/1'
    ]
    
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        
        for i, result in enumerate(results):
            print(f"Response {i+1}: {len(result)} bytes")

asyncio.run(main())
```

## 🔄 Внутреннее устройство Event Loop

```python
# Упрощённая модель работы
class EventLoop:
    def __init__(self):
        self.ready_queue = []      # Готовые к выполнению задачи
        self.waiting_tasks = {}    # Задачи в ожидании
    
    def run_until_complete(self, coro):
        task = self.create_task(coro)
        while not task.done():
            self._run_once()
        return task.result()
    
    def _run_once(self):
        # Выполняет готовые задачи
        # Проверяет завершение операций I/O
        # Планирует следующие задачи
        pass
```

## ⚠️ Важные моменты

### ❌ Блокирующие операции
```python
# ПЛОХО - блокирует весь event loop
async def bad_example():
    time.sleep(1)  # Блокирующая функция

# ХОРОШО - не блокирует
async def good_example():
    await asyncio.sleep(1)  # Асинхронная задержка
```

### 🔁 Запуск в существующем loop
```python
# Проверка наличия запущенного loop
try:
    loop = asyncio.get_running_loop()
except RuntimeError:
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
```

## 📊 Производительность

```python
import time

# Синхронный подход
def sync_example():
    start = time.time()
    for i in range(5):
        time.sleep(1)
    print(f"Sync: {time.time() - start:.2f}s")  # ~5 секунд

# Асинхронный подход
async def async_example():
    start = time.time()
    tasks = [asyncio.sleep(1) for _ in range(5)]
    await asyncio.gather(*tasks)
    print(f"Async: {time.time() - start:.2f}s")  # ~1 секунда

asyncio.run(async_example())
```

## 🎯 Когда использовать Event Loop

✅ **Подходит для:**
- Сетевого I/O (HTTP, WebSocket, базы данных)
- Работы с файлами (через `aiofiles`)
- Ожидания внешних событий
- Высоконагруженных серверов

❌ **Не подходит для:**
- CPU-интенсивных задач (используйте `multiprocessing`)
- Простых синхронных скриптов
- Когда нужна простота и читаемость кода

## 🔧 Полезные утилиты

```python
# Отладка event loop
asyncio.get_event_loop().set_debug(True)

# Получение текущего времени
asyncio.get_event_loop().time()

# Отмена задачи
task.cancel()

# Проверка статуса
task.done()
task.cancelled()
```

Event loop — мощный инструмент для создания эффективных асинхронных приложений в Python. Понимание его работы поможет писать более производительный и масштабируемый код! 🚀