# Domain Factory

## Зміст

- [Вступ](#вступ)
- [Що таке Domain Factory і яку проблему вона вирішує](#що-таке-domain-factory-і-яку-проблему-вона-вирішує)
- [Factory vs Constructor](#factory-vs-constructor)
- [Factory та зовнішні залежності](#factory-та-зовнішні-залежності)
- [Factory та доменні помилки](#factory-та-доменні-помилки)
- [Де живе Factory](#де-живе-factory)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

Створення доменного об'єкта — не завжди тривіальна операція. Інколи потрібно перевірити, що email унікальний. Що часовий слот не перетинається з існуючими бронюваннями. Що ресурс взагалі існує. Ці перевірки потребують звернення до бази даних — але [доменна модель](anemic-vs-rich-model.md) не може залежати від інфраструктури.

Конструктор моделі не підходить — він не може приймати репозиторій (це порушить DIP). Application Service може це зробити, але тоді логіка валідації при створенні «витікає» з домену в оркестрацію. Domain Factory вирішує цю проблему.

---

## Що таке Domain Factory і яку проблему вона вирішує

### Проблема

Ви створюєте бронювання. Перед створенням потрібно перевірити:

1. Час початку < часу завершення (простий інваріант)
2. Слот не в минулому (простий інваріант)
3. Ресурс існує (потрібна БД)
4. Часовий слот не зайнятий іншим бронюванням (потрібна БД)

Де це все перевіряти?

**В конструкторі?** Тоді конструктор потребує репозиторій — доменна модель залежить від інфраструктури:

```python
class Booking:
    def __init__(self, user_id, resource_id, start, end, repo):
        # домен імпортує інфраструктуру — порушення DIP
        if repo.find_overlapping(resource_id, start, end):
            raise DomainError("Slot taken")
        ...
```

**В Application Service?** Тоді бізнес-правила створення розмазані між доменом і оркестрацією:

```python
class CreateBookingHandler:
    def handle(self, command):
        # перевірка #3 і #4 — тут
        if not self.resource_repo.exists(command.resource_id):
            raise ...
        if self.booking_repo.find_overlapping(...):
            raise ...
        # перевірка #1 і #2 — в конструкторі Booking
        booking = Booking(...)
```

Частина правил — в handler'і, частина — в конструкторі. Знайти «всі правила створення бронювання» — потрібно дивитися в два місця.

### Рішення: Domain Factory

Factory — доменний об'єкт, який **інкапсулює всю логіку створення** [агрегатів та сутностей](entities-and-aggregates.md), коли це створення потребує складних перевірок або зовнішніх залежностей. Вона приймає залежності через інтерфейси (DIP збережений) і перевіряє всі інваріанти в одному місці:

```python
class BookingFactory:
    def __init__(self, booking_repo: BookingRepository,
                 resource_repo: ResourceRepository):
        self._booking_repo = booking_repo
        self._resource_repo = resource_repo

    def create(self, user_id: str, resource_id: str,
               start_time: datetime, end_time: datetime) -> Booking:
        # всі перевірки — тут, в одному місці
        time_slot = TimeSlot(start_time, end_time)  # перевірка #1: start < end

        if time_slot.is_in_past():                   # перевірка #2
            raise DomainError("Не можна бронювати час у минулому")

        resource = self._resource_repo.find_by_id(resource_id)
        if resource is None:                          # перевірка #3
            raise DomainError("Ресурс не знайдено")

        overlapping = self._booking_repo.find_overlapping(resource_id, time_slot)
        if overlapping:                               # перевірка #4
            raise DomainError("Часовий слот вже зайнятий")

        return Booking(
            id=generate_id(),
            user_id=user_id,
            resource_id=resource_id,
            time_slot=time_slot,
            status=BookingStatus.CREATED,
        )
```

Тепер Application Service делегує створення фабриці:

```python
class CreateBookingHandler:
    def __init__(self, factory: BookingFactory, repo: BookingRepository):
        self._factory = factory
        self._repo = repo

    def handle(self, command):
        booking = self._factory.create(
            command.user_id, command.resource_id,
            command.start_time, command.end_time,
        )
        self._repo.save(booking)
        return booking.id
```

Handler — чистий, без бізнес-логіки. Усі правила створення — в одному місці.

---

## Factory vs Constructor

Конструктор і Factory мають різні ролі:

| Аспект | Конструктор | Domain Factory |
|--------|-------------|----------------|
| Коли використовується | Відновлення об'єкта з існуючих (валідних) даних | Створення нового об'єкта з перевіркою інваріантів |
| Прості інваріанти | Перевіряє (формат, типи) | Перевіряє |
| Складні інваріанти (з БД) | Не може (немає залежностей) | Може (приймає інтерфейси) |
| Залежності | Немає | Репозиторії, сервіси — через інтерфейси |
| Хто викликає | Infrastructure (маппер, при завантаженні з БД) | Application Layer (handler) |

Конструктор потрібен для відновлення об'єкта з БД — дані вже були валідовані при створенні, маппер просто будує об'єкт з цих даних.
Factory потрібна для створення нового об'єкта — коли потрібно перевірити бізнес-правила.

---

## Factory та зовнішні залежності

Ключова особливість Domain Factory — вона може приймати залежності через інтерфейси, не порушуючи DIP:

```
Domain Layer:          BookingFactory (використовує BookingRepository інтерфейс)
                                                ↑
Infrastructure Layer:                  PostgresBookingRepository (реалізація)
```

Factory залежить від **інтерфейсу** `BookingRepository`, який визначений в домені. Конкретна реалізація впроваджується ззовні (через конструктор Factory). Домен залишається чистим.

### Які залежності допустимі?

- **Інтерфейси репозиторіїв** — для перевірки унікальності, перетинів, існування
- **Domain Services** — для складних обчислень, що залучають кілька агрегатів
- **Генератори ID** — якщо генерація ID — бізнес-вимога (наприклад, формат номера бронювання)

### Які залежності неприпустимі?

- HTTP-клієнти, ORM-сесії, файлові системи — це інфраструктура, вона ховається за інтерфейсами

---

## Factory та доменні помилки

Factory кидає **[доменні помилки](domain-errors.md)** — не HTTP-винятки, не помилки фреймворку. Домен не знає про HTTP:

```python
class SlotConflictError(DomainError):
    def __init__(self, resource_id: str, time_slot: TimeSlot):
        self.resource_id = resource_id
        self.time_slot = time_slot
        super().__init__(f"Часовий слот {time_slot} вже зайнятий для ресурсу {resource_id}")

class ResourceNotFoundError(DomainError):
    def __init__(self, resource_id: str):
        self.resource_id = resource_id
        super().__init__(f"Ресурс {resource_id} не знайдено")
```

Presentation Layer перехоплює доменні помилки і маппить їх у HTTP-статуси. Домен каже: «слот зайнятий». Presentation вирішує, що це HTTP 409. Кожен відповідає за своє. Детальніше — в [доменних помилках](domain-errors.md).

---

## Де живе Factory

Factory — живе в Domain Layer:

```
src/
├── domain/
│   ├── models/
│   │   └── booking.py
│   ├── factories/
│   │   └── booking_factory.py    # Factory — тут
│   └── repositories/
│       └── booking_repository.py # інтерфейс
├── application/
│   └── commands/
│       └── create_booking.py     # handler викликає Factory
└── infrastructure/
    └── repositories/
        └── postgres_booking_repo.py
```

Factory використовується в Application Layer (handler викликає `factory.create(...)`), але сама належить домену — бо вона інкапсулює **доменні правила** створення.

---

## Поширені міфи

### «Factory — це патерн GoF (Abstract Factory / Factory Method)»

Domain Factory з DDD — це інша концепція. GoF Factory вирішує проблему вибору типу об'єкта (поліморфізм). DDD Factory вирішує проблему **складного створення з перевіркою інваріантів**. Вони можуть перетинатися, але мотивація різна.

### «Factory потрібна для кожної сутності»

Ні. Якщо створення об'єкта просте (немає складних інваріантів, не потрібна БД) — конструктора достатньо. Factory виправдана, коли створення потребує зовнішніх перевірок або складної логіки.

### «Factory має повертати різні типи»

Не обов'язково. Domain Factory може завжди повертати один тип (`Booking`). Її мета — не поліморфізм, а інкапсуляція логіки створення.

### «Валідація в Factory дублює валідацію в моделі»

Ні. Вони перевіряють різне. Конструктор моделі (або [Value Object](value-objects.md)) перевіряє прості інваріанти (формат, типи). Factory перевіряє інваріанти, що потребують зовнішніх даних (унікальність, доступність). Вони доповнюють одне одного.

---

## Джерела

- **Eric Evans** — *Domain-Driven Design* (2003) — Factory як тактичний патерн DDD (Chapter 6: «The Life Cycle of a Domain Object — Factories»)
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (2013) — практичні приклади Factory з залежностями через інтерфейси
