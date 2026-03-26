
### USER
Popular design patterns in Python

### ASSISTANT
Here are the most popular and widely-used design patterns in Python:

## 1. Singleton Pattern
Ensures only one instance of a class exists.

```python
class Singleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Using decorator approach
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class Database:
    pass
```

## 2. Factory Pattern
Creates objects without specifying the exact class.

```python
class Animal:
    def speak(self):
        pass

class Dog(Animal):
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

class AnimalFactory:
    @staticmethod
    def create_animal(animal_type):
        if animal_type == "dog":
            return Dog()
        elif animal_type == "cat":
            return Cat()
        else:
            raise ValueError("Unknown animal type")

# Usage
animal = AnimalFactory.create_animal("dog")
```

## 3. Observer Pattern
Defines a one-to-many dependency between objects.

```python
class Subject:
    def __init__(self):
        self._observers = []
    
    def attach(self, observer):
        self._observers.append(observer)
    
    def notify(self, message):
        for observer in self._observers:
            observer.update(message)

class Observer:
    def update(self, message):
        pass

class EmailObserver(Observer):
    def update(self, message):
        print(f"Email: {message}")

class SMSObserver(Observer):
    def update(self, message):
        print(f"SMS: {message}")

# Usage
subject = Subject()
subject.attach(EmailObserver())
subject.attach(SMSObserver())
subject.notify("Hello!")
```

## 4. Decorator Pattern
Adds behavior to objects dynamically.

```python
class Coffee:
    def cost(self):
        return 5

class MilkDecorator:
    def __init__(self, coffee):
        self._coffee = coffee
    
    def cost(self):
        return self._coffee.cost() + 2

class SugarDecorator:
    def __init__(self, coffee):
        self._coffee = coffee
    
    def cost(self):
        return self._coffee.cost() + 1

# Usage
coffee = Coffee()
coffee_with_milk = MilkDecorator(coffee)
coffee_with_milk_and_sugar = SugarDecorator(coffee_with_milk)
print(coffee_with_milk_and_sugar.cost())  # Output: 8
```

## 5. Strategy Pattern
Defines a family of algorithms and makes them interchangeable.

```python
from abc import ABC, abstractmethod

class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount):
        pass

class CreditCardPayment(PaymentStrategy):
    def pay(self, amount):
        return f"Paid ${amount} using Credit Card"

class PayPalPayment(PaymentStrategy):
    def pay(self, amount):
        return f"Paid ${amount} using PayPal"

class ShoppingCart:
    def __init__(self, payment_strategy):
        self._payment_strategy = payment_strategy
    
    def checkout(self, amount):
        return self._payment_strategy.pay(amount)

# Usage
cart = ShoppingCart(CreditCardPayment())
print(cart.checkout(100))
```

## 6. Command Pattern
Encapsulates a request as an object.

```python
class Command:
    def execute(self):
        pass

class Light:
    def on(self):
        print("Light is ON")
    
    def off(self):
        print("Light is OFF")

class LightOnCommand(Command):
    def __init__(self, light):
        self._light = light
    
    def execute(self):
        self._light.on()

class LightOffCommand(Command):
    def __init__(self, light):
        self._light = light
    
    def execute(self):
        self._light.off()

class RemoteControl:
    def __init__(self):
        self._command = None
    
    def set_command(self, command):
        self._command = command
    
    def press_button(self):
        self._command.execute()

# Usage
light = Light()
light_on = LightOnCommand(light)
remote = RemoteControl()
remote.set_command(light_on)
remote.press_button()
```

## 7. Adapter Pattern
Allows incompatible interfaces to work together.

```python
class EuropeanSocket:
    def voltage(self):
        return 230

class USASocket:
    def voltage(self):
        return 120

class SocketAdapter:
    def __init__(self, socket):
        self._socket = socket
    
    def voltage(self):
        # Convert voltage to desired format
        return self._socket.voltage() * 0.52

# Usage
european_socket = EuropeanSocket()
adapter = SocketAdapter(european_socket)
print(adapter.voltage())  # Converted voltage
```

## 8. Template Method Pattern
Defines the skeleton of an algorithm.

```python
from abc import ABC, abstractmethod

class DataProcessor(ABC):
    def process(self):
        self.read_data()
        self.process_data()
        self.save_data()
    
    @abstractmethod
    def read_data(self):
        pass
    
    @abstractmethod
    def process_data(self):
        pass
    
    def save_data(self):
        print("Saving data to database")

class CSVProcessor(DataProcessor):
    def read_data(self):
        print("Reading CSV file")
    
    def process_data(self):
        print("Processing CSV data")

class JSONProcessor(DataProcessor):
    def read_data(self):
        print("Reading JSON file")
    
    def process_data(self):
        print("Processing JSON data")

# Usage
processor = CSVProcessor()
processor.process()
```

## 9. Builder Pattern
Constructs complex objects step by step.

```python
class Computer:
    def __init__(self):
        self.cpu = None
        self.memory = None
        self.storage = None
    
    def __str__(self):
        return f"Computer(CPU: {self.cpu}, Memory: {self.memory}, Storage: {self.storage})"

class ComputerBuilder:
    def __init__(self):
        self._computer = Computer()
    
    def set_cpu(self, cpu):
        self._computer.cpu = cpu
        return self
    
    def set_memory(self, memory):
        self._computer.memory = memory
        return self
    
    def set_storage(self, storage):
        self._computer.storage = storage
        return self
    
    def build(self):
        return self._computer

# Usage
builder = ComputerBuilder()
computer = (builder
            .set_cpu("Intel i7")
            .set_memory("16GB")
            .set_storage("512GB SSD")
            .build())
print(computer)
```

## 10. Context Manager Pattern
Manages resources with `with` statement.

```python
class DatabaseConnection:
    def __enter__(self):
        print("Connecting to database")
        return self
    
    def __exit__(self, exc_type, exc_value, traceback):
        print("Closing database connection")
    
    def query(self, sql):
        return f"Executing: {sql}"

# Usage
with DatabaseConnection() as db:
    result = db.query("SELECT * FROM users")
    print(result)
```

## Python-Specific Patterns

### Using `__init_subclass__` for Plugin Systems
```python
class Plugin:
    plugins = []
    
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.plugins.append(cls)

class AudioPlugin(Plugin):
    def process(self):
        return "Processing audio"

class VideoPlugin(Plugin):
    def process(self):
        return "Processing video"

# All subclasses automatically registered
print(Plugin.plugins)  # [<class 'AudioPlugin'>, <class 'VideoPlugin'>]
```

### Descriptor Pattern
```python
class ValidatedAttribute:
    def __init__(self, validator=None):
        self.validator = validator
        self.value = None
    
    def __get__(self, obj, objtype=None):
        return self.value
    
    def __set__(self, obj, value):
        if self.validator and not self.validator(value):
            raise ValueError("Invalid value")
        self.value = value

class Person:
    age = ValidatedAttribute(lambda x: 0 <= x <= 150)
    
    def __init__(self, name, age):
        self.name = name
        self.age = age

# Usage
person = Person("Alice", 25)
# person.age = -5  # Raises ValueError
```

These patterns help solve common design problems and make code more maintainable, flexible, and reusable. Choose patterns based on your specific use cases rather than applying them unnecessarily.