# Принципи SOLID

## Зміст

- [Вступ](#вступ)
- [Single Responsibility Principle (SRP)](#single-responsibility-principle-srp)
- [Open/Closed Principle (OCP)](#openclosed-principle-ocp)
- [Liskov Substitution Principle (LSP)](#liskov-substitution-principle-lsp)
- [Interface Segregation Principle (ISP)](#interface-segregation-principle-isp)
- [Dependency Inversion Principle (DIP)](#dependency-inversion-principle-dip)
- [Як принципи працюють разом](#як-принципи-працюють-разом)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

Уявіть: ви працюєте над системою бронювання. Перша версія — простий CRUD, кілька контролерів, усе працює. Через місяць приходять нові вимоги: додати нотифікації, змінити формат звітів, підтримати нову БД. І кожна зміна ламає щось в іншому місці. Контролер на 500 рядків, який відповідає за все — від валідації до відправки email. Тести, які падають через будь-яку зміну в БД. Знайоме?

SOLID — це п'ять принципів проєктування, які допомагають писати код, стійкий до змін. Їх сформулював Роберт Мартін (Robert C. Martin) на початку 2000-х. Це не жорсткі правила, а орієнтири — вони підказують, як розподілити відповідальності, щоб зміни в одній частині системи не каскадно ламали решту.

---

## Single Responsibility Principle (SRP)

> Клас має мати лише одну причину для зміни.

### Проблема

Ви написали `BookingService`, який створює бронювання, валідує дані, перевіряє доступність слотів, зберігає в БД, відправляє email-підтвердження і логує операцію:

```python
class BookingService:
    def create_booking(self, data: dict):
        # валідація
        if not data.get("email") or "@" not in data["email"]:
            raise ValueError("Invalid email")
        if data["start_time"] >= data["end_time"]:
            raise ValueError("Invalid time range")

        # перевірка доступності
        existing = db.query("SELECT * FROM bookings WHERE ...")
        if existing:
            raise ConflictError("Slot taken")

        # збереження
        db.execute("INSERT INTO bookings ...")

        # нотифікація
        smtp.send_email(data["email"], "Booking confirmed", "...")

        # логування
        logger.info(f"Booking created for {data['email']}")
```

Цей клас зміниться, якщо:
- зміняться правила валідації
- зміниться спосіб перевірки доступності
- зміниться схема БД
- зміниться поштовий провайдер
- зміниться формат логів

П'ять причин для зміни в одному класі. Кожна зміна ризикує зламати все інше.

### Рішення

Кожна відповідальність — в окремому класі:

```python
class BookingValidator:
    def validate(self, data: dict) -> None: ...

class SlotAvailabilityChecker:
    def check(self, resource_id: str, time_slot: TimeSlot) -> None: ...

class BookingRepository:
    def save(self, booking: Booking) -> None: ...

class BookingNotifier:
    def send_confirmation(self, booking: Booking) -> None: ...
```

Тепер зміна поштового провайдера торкнеться лише `BookingNotifier`. Зміна схеми БД — лише `BookingRepository`. Кожен клас має одну причину для зміни.

### Суть

SRP — не про те, що клас має робити «одну річ». Це про те, що клас має відповідати **одному актору** (одній групі стейкхолдерів, одній причині змін). Якщо різні люди з різних причин можуть попросити змінити один і той самий клас — він робить забагато.

---

## Open/Closed Principle (OCP)

> Програмні сутності мають бути відкриті для розширення, але закриті для модифікації.

### Проблема

Система підтримує нотифікації через email. Потім додається SMS. Потім — push-нотифікації. Кожного разу ви лізете в один і той самий метод:

```python
class NotificationService:
    def notify(self, user: User, message: str, channel: str):
        if channel == "email":
            self._send_email(user.email, message)
        elif channel == "sms":
            self._send_sms(user.phone, message)
        elif channel == "push":
            self._send_push(user.device_token, message)
        # кожен новий канал = зміна цього методу
```

Кожне розширення вимагає модифікації існуючого коду. А кожна модифікація — це ризик зламати те, що вже працює.

### Рішення

Визначити абстракцію, яку можна розширювати без зміни існуючого коду:

```python
class NotificationChannel(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> None: ...

class EmailChannel(NotificationChannel):
    def send(self, recipient: str, message: str) -> None:
        # логіка відправки email
        ...

class SmsChannel(NotificationChannel):
    def send(self, recipient: str, message: str) -> None:
        # логіка відправки SMS
        ...

class NotificationService:
    def __init__(self, channels: list[NotificationChannel]):
        self._channels = channels

    def notify(self, message: str) -> None:
        for channel in self._channels:
            channel.send(message)
```

Додати push-нотифікації? Створюємо `PushChannel(NotificationChannel)` — і жодного рядка в існуючому коді не міняємо.

### Суть

OCP не означає, що код ніколи не змінюється. Це означає, що **передбачувані розширення** (нові канали, нові типи, нові стратегії) мають додаватися без модифікації існуючих класів. Ключовий інструмент — поліморфізм через абстракції.

---

## Liskov Substitution Principle (LSP)

> Об'єкти підкласу мають бути взаємозамінними з об'єктами базового класу без порушення коректності програми.

### Проблема

Класичний приклад. У вас є `Rectangle`:

```python
class Rectangle:
    def __init__(self, width: float, height: float):
        self._width = width
        self._height = height

    def set_width(self, w: float): self._width = w
    def set_height(self, h: float): self._height = h
    def area(self) -> float: return self._width * self._height
```

«Квадрат — це прямокутник», тому ви робите `Square(Rectangle)`:

```python
class Square(Rectangle):
    def set_width(self, w: float):
        self._width = w
        self._height = w  # "зберігаємо інваріант квадрата"

    def set_height(self, h: float):
        self._width = h
        self._height = h
```

Тест, який працює з будь-яким прямокутником:

```python
def test_area(rect: Rectangle):
    rect.set_width(5)
    rect.set_height(3)
    assert rect.area() == 15  # для Rectangle — OK, для Square — FAIL (9)
```

`Square` не може замінити `Rectangle` без порушення поведінки. Наслідування за «is-a» ≠ підстановка.

### Як це виглядає в реальному проєкті?

У вас є інтерфейс `BookingRepository`:

```python
class BookingRepository(ABC):
    @abstractmethod
    def save(self, booking: Booking) -> None: ...

    @abstractmethod
    def find_by_id(self, id: str) -> Booking | None: ...
```

Одна реалізація — `PostgresBookingRepository`, інша — `CachedBookingRepository`. Якщо `CachedBookingRepository.save()` мовчки не зберігає дані (бо «це ж кеш»), то він порушує LSP — код, який очікує що `save()` персистить дані, зламається.

### Суть

LSP — це контракт. Якщо базовий клас або інтерфейс обіцяє певну поведінку, кожна реалізація мусить цю обіцянку тримати. Інакше поліморфізм стає пасткою: код виглядає правильним, але ламається з конкретними реалізаціями.

Правила:
- Підклас не посилює передумови (не вимагає більше, ніж базовий)
- Підклас не послаблює постумови (не повертає менше, ніж обіцяно)
- Підклас не кидає нових неочікуваних винятків

---

## Interface Segregation Principle (ISP)

> Клієнт не повинен залежати від методів, які він не використовує.

### Проблема

Ви створили інтерфейс репозиторію «на всі випадки»:

```python
class Repository(ABC):
    @abstractmethod
    def find_by_id(self, id: str) -> Entity: ...

    @abstractmethod
    def find_all(self) -> list[Entity]: ...

    @abstractmethod
    def save(self, entity: Entity) -> None: ...

    @abstractmethod
    def delete(self, id: str) -> None: ...

    @abstractmethod
    def find_by_criteria(self, criteria: dict) -> list[Entity]: ...

    @abstractmethod
    def count(self) -> int: ...

    @abstractmethod
    def exists(self, id: str) -> bool: ...
```

Тепер `CreateBookingCommandHandler` залежить від усього цього інтерфейсу, хоча використовує лише `save()` та `find_by_id()`. А `GetBookingQueryHandler` потребує лише `find_by_id()` і `find_by_criteria()`.

Наслідки:
- Кожна зміна інтерфейсу торкається всіх клієнтів, навіть тих, яким ця зміна не потрібна
- Тестування ускладнюється — потрібно мокати 7 методів, з яких використовуються 2
- Реалізація нового адаптера вимагає імплементувати все, навіть непотрібне

### Рішення

Розбити на вузькі інтерфейси за потребами клієнтів:

```python
class BookingReader(ABC):
    @abstractmethod
    def find_by_id(self, id: str) -> Booking | None: ...

class BookingWriter(ABC):
    @abstractmethod
    def save(self, booking: Booking) -> None: ...

class BookingSearcher(ABC):
    @abstractmethod
    def find_by_criteria(self, criteria: dict) -> list[Booking]: ...
```

`CreateBookingCommandHandler` залежить від `BookingWriter`. `GetBookingQueryHandler` — від `BookingReader`. Один конкретний клас може реалізувати кілька інтерфейсів:

```python
class PostgresBookingRepository(BookingReader, BookingWriter, BookingSearcher):
    ...
```

### Суть

ISP — це про те, що інтерфейси мають бути **під розмір клієнта**, а не під розмір реалізації. Вузький інтерфейс = менше залежностей = простіше тестувати, змінювати і підтримувати.

---

## Dependency Inversion Principle (DIP)

> Модулі верхнього рівня не повинні залежати від модулів нижнього рівня. Обидва мають залежати від абстракцій.

### Проблема

`CreateBookingHandler` працює з PostgreSQL. Напряму:

```python
from infrastructure.postgres_repo import PostgresBookingRepository

class CreateBookingHandler:
    def __init__(self):
        self.repo = PostgresBookingRepository()

    def handle(self, command):
        booking = self._create_booking(command)
        self.repo.save(booking)
```

Тепер:
- Щоб написати unit-тест — потрібен запущений PostgreSQL
- Щоб замінити БД — потрібно міняти бізнес-логіку
- Application layer імпортує Infrastructure layer — залежність направлена «вниз», до деталей

Бізнес-логіка (високорівневий модуль) залежить від деталей реалізації (низькорівневий модуль). Хвіст крутить собакою.

### Рішення

Бізнес-логіка визначає інтерфейс — **що їй потрібно**. Інфраструктура реалізує цей інтерфейс — **як це зробити**:

```python
# domain/repositories.py — визначає домен
class BookingRepository(ABC):
    @abstractmethod
    def save(self, booking: Booking) -> None: ...
```

```python
# infrastructure/postgres_repo.py — реалізує інфраструктура
class PostgresBookingRepository(BookingRepository):
    def save(self, booking: Booking) -> None:
        # SQL-логіка
        ...
```

```python
# application/create_booking.py — залежить від абстракції
class CreateBookingHandler:
    def __init__(self, repo: BookingRepository):  # інтерфейс, не реалізація
        self.repo = repo
```

Залежності інвертовані: і Handler, і Repository залежать від абстракції `BookingRepository`, яка живе в домені. Інфраструктура підлаштовується під домен, а не навпаки.

### Суть

DIP — це фундамент шарової архітектури. Саме завдяки йому домен не залежить від інфраструктури, тести домену працюють без БД, а заміна PostgreSQL на MongoDB торкається лише Infrastructure Layer.

---

## Як принципи працюють разом

SOLID — це не п'ять ізольованих правил. Вони підсилюють одне одного:

- **SRP** каже: розділи відповідальності
- **OCP** каже: зроби так, щоб нові варіанти додавалися без зміни існуючого
- **LSP** каже: забезпеч, що замінити одну реалізацію на іншу безпечно
- **ISP** каже: не змушуй клієнтів залежати від того, що їм не потрібно
- **DIP** каже: залежи від абстракцій, а не від деталей

Разом вони ведуть до архітектури, де модулі слабко зв'язані (loose coupling), мають чітку відповідальність (high cohesion), і де зміна вимог не перетворюється на лавину правок по всьому проєкту.

---

## Поширені міфи

### «SRP означає, що клас має робити одну річ»

Не зовсім. Martin уточнив формулювання: клас має мати **одну причину для зміни**, тобто відповідати одному актору. Клас `BookingRepository` робить кілька речей (save, find, delete), але всі вони стосуються одного актора — persistence. Це нормально. SRP порушується, коли один клас обслуговує різних стейкхолдерів з різними причинами для змін.

### «Кожен клас потребує інтерфейс — це DIP»

DIP — це не про кількість інтерфейсів. Це про **напрямок залежності**. Якщо у вас є `BookingRepository(Protocol)`, але він живе в пакеті `infrastructure` — DIP порушено. Інтерфейс має належати тому, хто його **потребує** (домену), а не тому, хто його реалізує (інфраструктурі). Без правильного напрямку інтерфейс — просто зайвий рівень абстракції.

### «SOLID — це про ООП і класи»

Принципи SOLID сформульовані в термінах класів, але їхня суть — про **модулі та залежності між ними**. SRP застосовується до функцій і модулів. OCP — до будь-якої точки розширення (стратегії, плагіни, middleware). DIP — до залежностей між шарами. Ви можете писати функціональний код і дотримуватись SOLID.

### «ISP каже розбивати все на інтерфейси з одним методом»

ISP каже: інтерфейс має бути **під розмір клієнта**. Якщо клієнту потрібні три методи — інтерфейс із трьох методів цілком нормальний. Проблема виникає, коли клієнт залежить від інтерфейсу з десятьма методами, з яких використовує два. Розбивати все до одного методу — це інша крайність, яка ускладнює код без користі.

### «LSP — це про наслідування»

LSP працює з будь-яким поліморфізмом, не лише з наслідуванням. Якщо функція приймає `BookingRepository` (Protocol) — кожна реалізація повинна дотримуватись контракту. `InMemoryBookingRepository`, який мовчки ігнорує `save()`, порушує LSP, хоча наслідування немає. LSP — це про **поведінкову сумісність**, а не про ієрархію класів.

### «OCP означає, що код ніколи не можна змінювати»

OCP стосується **передбачуваних точок розширення**. Якщо ви знаєте, що будуть нові канали нотифікацій — спроєктуйте абстракцію `NotificationChannel`. Але якщо змінюється фундаментальна логіка — змінюйте код. OCP — це про дизайн, який мінімізує каскадні зміни, а не про заборону будь-яких модифікацій.

### «Якщо дотримуватись SOLID — код буде простий»

SOLID не спрощує код — він робить його **стійким до змін**. Дотримання принципів часто додає класи, інтерфейси, рівні абстракції. Для простого CRUD з трьома ендпоінтами це може бути overkill. SOLID окуповується, коли система росте і вимоги змінюються — тоді інвестиція в структуру починає зберігати час.

---

## Джерела

- **Robert C. Martin** — *Agile Software Development: Principles, Patterns, and Practices* (2002) — книга, де SOLID-принципи були вперше системно описані
- **Robert C. Martin** — *Clean Architecture: A Craftsman's Guide to Software Structure and Design* (2017) — розвиток ідей SOLID у контексті архітектури застосунків
- **Robert C. Martin** — [The Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod) — оригінальні статті, де принципи формулювалися
