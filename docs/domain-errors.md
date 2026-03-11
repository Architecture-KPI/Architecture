# Доменні помилки

## Зміст

- [Вступ](#вступ)
- [Проблема: помилки без структури](#проблема-помилки-без-структури)
- [Що таке доменна помилка](#що-таке-доменна-помилка)
- [Ієрархія помилок](#ієрархія-помилок)
- [Хто кидає доменні помилки](#хто-кидає-доменні-помилки)
- [Хто обробляє доменні помилки](#хто-обробляє-доменні-помилки)
- [Помилки як частина Ubiquitous Language](#помилки-як-частина-ubiquitous-language)
- [Доменні помилки vs валідаційні помилки](#доменні-помилки-vs-валідаційні-помилки)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

У кожному проєкті є помилки. Користувач передав невалідний email. Часовий слот вже зайнятий. Бронювання не можна скасувати, бо воно вже скасоване. Ресурс не знайдено.

Що робить типовий розробник? Кидає `ValueError`, `HTTPException(409)`, або повертає `{"error": "slot taken"}` прямо з контролера. Все працює — до моменту, коли потрібно зрозуміти, чому саме операція не вдалася, і як на це реагувати. `ValueError` не відрізняє бізнес-правило від друкарської помилки в коді. `HTTPException` прив'язує домен до HTTP. А рядок `"slot taken"` — це не контракт, а надія, що ніхто не перейменує повідомлення.

Доменні помилки — це спосіб зробити помилки **частиною бізнес-моделі**, а не технічним артефактом.

---

## Проблема: помилки без структури

### Варіант 1: generic exceptions

```python
class BookingFactory:
    def create(self, user_id, resource_id, start, end):
        if start >= end:
            raise ValueError("start must be before end")
        if self._repo.find_overlapping(resource_id, TimeSlot(start, end)):
            raise ValueError("slot is taken")
        if self._resource_repo.find_by_id(resource_id) is None:
            raise ValueError("resource not found")
```

Три різні бізнес-ситуації — одне й те саме `ValueError`. Presentation Layer не може відрізнити конфлікт слотів від невалідного діапазону. Єдиний варіант — парсити текст повідомлення. Це крихко: перейменували рядок — зламали обробку помилок.

### Варіант 2: HTTP-винятки в домені

```python
from fastapi import HTTPException

class BookingFactory:
    def create(self, ...):
        if self._repo.find_overlapping(...):
            raise HTTPException(status_code=409, detail="Slot taken")
```

Домен знає про HTTP-статуси. Зміна фреймворку (FastAPI → Flask → gRPC) ламає бізнес-логіку. Доменні тести потребують HTTP-фреймворк. Порушено [DIP](solid.md) — домен залежить від інфраструктури.

### Варіант 3: помилки як рядки

```python
class BookingFactory:
    def create(self, ...) -> Booking | str:
        if self._repo.find_overlapping(...):
            return "slot_taken"
```

Немає типізації, немає ієрархії. Що робити з помилкою — залежить від того, хто прочитає рядок. Компілятор/лінтер не допоможе — забули обробити новий випадок, і це стане помилкою лише в рантаймі.

---

## Що таке доменна помилка

Доменна помилка — це **виняток, що описує порушення бізнес-правила** мовою домену. Вона не знає про HTTP, фреймворки або UI — вона знає лише про бізнес.

```python
class DomainError(Exception):
    """Базовий клас для всіх доменних помилок."""
    pass
```

Ключові характеристики:

| Характеристика | Опис |
|---------------|------|
| **Мова домену** | Іменується бізнес-термінами, не технічними (`SlotConflictError`, не `Http409Error`) |
| **Живе в Domain Layer** | Не імпортує нічого з інфраструктури, фреймворків, HTTP |
| **Типізована** | Кожна бізнес-ситуація — окремий клас, не рядок і не generic Exception |
| **Несе контекст** | Містить дані, потрібні для розуміння, що сталося |

---

## Ієрархія помилок

### Базовий клас

Усі доменні помилки наслідують від одного базового класу. Це дає можливість перехоплювати всі доменні помилки одним `except DomainError`:

```python
class DomainError(Exception):
    pass
```

### Конкретні помилки

Кожне бізнес-правило, яке може бути порушене, отримує свій клас:

```python
class SlotConflictError(DomainError):
    def __init__(self, resource_id: str, time_slot: 'TimeSlot'):
        self.resource_id = resource_id
        self.time_slot = time_slot
        super().__init__(
            f"Часовий слот {time_slot} вже зайнятий для ресурсу {resource_id}"
        )

class BookingAlreadyCancelledError(DomainError):
    def __init__(self, booking_id: str):
        self.booking_id = booking_id
        super().__init__(f"Бронювання {booking_id} вже скасоване")

class InvalidTimeSlotError(DomainError):
    def __init__(self, start: datetime, end: datetime):
        self.start = start
        self.end = end
        super().__init__(f"Некоректний часовий діапазон: {start} >= {end}")

class ResourceNotFoundError(DomainError):
    def __init__(self, resource_id: str):
        self.resource_id = resource_id
        super().__init__(f"Ресурс {resource_id} не знайдено")
```

Кожна помилка:
- Має зрозумілу назву, що описує **бізнес-ситуацію**
- Містить **контекстні дані** (який ресурс, який слот, яке бронювання)
- Генерує людиночитабельне повідомлення

### Де живе ієрархія

```
src/
├── domain/
│   ├── errors.py           # DomainError, SlotConflictError, ...
│   ├── entities/
│   ├── value_objects/
│   ├── factories/
│   └── repositories/
```

Помилки — частина домену. Вони живуть поруч із моделями, [фабриками](domain-factory.md) та [інтерфейсами репозиторіїв](repository.md).

---

## Хто кидає доменні помилки

### Value Objects

[Value Object](value-objects.md) кидає помилку, якщо його інваріант порушено при створенні:

```python
@dataclass(frozen=True)
class TimeSlot:
    start: datetime
    end: datetime

    def __post_init__(self):
        if self.start >= self.end:
            raise InvalidTimeSlotError(self.start, self.end)
```

Якщо `TimeSlot` існує — він валідний. Це гарантія, яку дає Value Object.

### Entities та Aggregates

[Entity](entities-and-aggregates.md) кидає помилку, коли бізнес-операція неможлива в поточному стані:

```python
class Booking:
    def cancel(self) -> None:
        if self._status == BookingStatus.CANCELLED:
            raise BookingAlreadyCancelledError(self._id)
        self._status = BookingStatus.CANCELLED
```

### Domain Factory

[Factory](domain-factory.md) кидає помилку, коли створення об'єкта порушує складні інваріанти:

```python
class BookingFactory:
    def create(self, user_id, resource_id, start, end) -> Booking:
        time_slot = TimeSlot(start, end)  # може кинути InvalidTimeSlotError

        if not self._resource_repo.find_by_id(resource_id):
            raise ResourceNotFoundError(resource_id)

        if self._booking_repo.find_overlapping(resource_id, time_slot):
            raise SlotConflictError(resource_id, time_slot)

        return Booking(...)
```

### Domain Service

Якщо бізнес-правило залучає кілька агрегатів — його перевіряє Domain Service, і він теж кидає доменну помилку.

---

## Хто обробляє доменні помилки

Домен кидає помилку. Але хто вирішує, що з нею робити? **Presentation Layer**.

### Маппінг у HTTP-статуси

Presentation Layer перехоплює доменні помилки і транслює їх у відповідь, зрозумілу клієнту:

```python
# presentation/error_handler.py
def handle_domain_error(error: DomainError) -> tuple[dict, int]:
    if isinstance(error, SlotConflictError):
        return {"error": str(error)}, 409
    if isinstance(error, ResourceNotFoundError):
        return {"error": str(error)}, 404
    if isinstance(error, BookingAlreadyCancelledError):
        return {"error": str(error)}, 409
    return {"error": str(error)}, 400
```

Або через реєстр:

```python
ERROR_STATUS_MAP: dict[type[DomainError], int] = {
    SlotConflictError: 409,
    ResourceNotFoundError: 404,
    BookingAlreadyCancelledError: 409,
    InvalidTimeSlotError: 422,
}

def handle_domain_error(error: DomainError) -> tuple[dict, int]:
    status = ERROR_STATUS_MAP.get(type(error), 400)
    return {"error": str(error)}, status
```

### Чому саме Presentation Layer?

Домен каже: «слот зайнятий». Це бізнес-факт, і він не знає, через який канал прийшов запит. Що з цим робити — залежить від контексту:

- **REST API** — повертає HTTP 409
- **gRPC** — повертає `ALREADY_EXISTS`
- **CLI** — виводить повідомлення в термінал
- **Тест** — перевіряє, що виняток кинуто

Якщо домен знає про HTTP — він не зможе працювати в інших контекстах. Розділення дає гнучкість: один і той самий домен обслуговує REST, gRPC, CLI і тести без змін.

---

## Помилки як частина Ubiquitous Language

Доменні помилки — це не технічна деталь. Вони описують ситуації, про які знає бізнес:

| Помилка | Бізнес-сценарій |
|---------|----------------|
| `SlotConflictError` | Клієнт намагається забронювати вже зайнятий слот |
| `BookingAlreadyCancelledError` | Адміністратор намагається скасувати вже скасоване бронювання |
| `InvalidTimeSlotError` | Вказано некоректний часовий діапазон |
| `PastBookingModificationError` | Спроба змінити бронювання в минулому |

Назви помилок мають бути зрозумілими не лише розробникам, а й доменним експертам. Якщо бізнес каже «конфлікт слотів» — помилка має називатися `SlotConflictError`, а не `Error409` або `DataIntegrityViolation`.

Це частина Ubiquitous Language з DDD: код говорить мовою домену, включаючи помилки.

---

## Доменні помилки vs валідаційні помилки

Є два рівні «щось пішло не так», і вони відрізняються за природою:

| Аспект | Валідація формату | Доменна помилка |
|--------|------------------|-----------------|
| **Що перевіряє** | Чи дані фізично коректні | Чи операція має сенс для бізнесу |
| **Де відбувається** | Presentation Layer ([DTO](dto.md)) | Domain Layer (моделі, фабрики) |
| **Приклади** | Чи email — рядок з `@`? Чи поле присутнє? | Чи email унікальний? Чи слот вільний? |
| **Хто кидає** | DTO, Request Validator | Entity, Value Object, Factory |
| **Коли** | До того, як дані потраплять у домен | Під час виконання бізнес-операції |

```python
# Валідація формату — в DTO (Presentation)
@dataclass(frozen=True)
class CreateBookingRequest:
    start_time: str
    end_time: str

    def __post_init__(self):
        try:
            datetime.fromisoformat(self.start_time)
        except ValueError:
            raise ValidationError("Invalid datetime format")

# Доменна помилка — в Value Object (Domain)
@dataclass(frozen=True)
class TimeSlot:
    start: datetime
    end: datetime

    def __post_init__(self):
        if self.start >= self.end:
            raise InvalidTimeSlotError(self.start, self.end)
```

`ValidationError` — це «дані не пройшли вхідний контроль». `InvalidTimeSlotError` — це «бізнес-правило порушене». Вони можуть виглядати схоже, але відповідають за різне і кидаються різними шарами.

---

## Поширені міфи

### «Достатньо одного класу DomainError з текстовим повідомленням»

Текстове повідомлення — для людей. Тип помилки — для коду. Presentation Layer вирішує, який HTTP-статус повернути, на основі **типу** помилки, а не парсингу тексту. Якщо всі помилки — один клас, обробник не може відрізнити конфлікт слотів від невалідного діапазону без аналізу рядка.

### «Доменна помилка = HTTP-виняток»

Ні. `HTTPException(409)` — це деталь Presentation Layer. Домен не знає про HTTP. Якщо той самий домен використовується через CLI або gRPC — `HTTPException` не має сенсу. Домен кидає `SlotConflictError`, а Presentation маппить його в 409 для REST, `ALREADY_EXISTS` для gRPC, або повідомлення в терміналі для CLI.

### «Доменні помилки — це overengineering для маленького проєкту»

Базовий `DomainError` та 3-4 конкретних помилки — це кілька рядків коду. Натомість ви отримуєте типізовану обробку помилок, зрозумілі повідомлення і домен, який не залежить від фреймворку. Це не overengineering — це мінімальна гігієна.

### «Потрібно створити помилку для кожного можливого випадку»

Ні. Помилка потрібна, коли бізнес-ситуація потребує **різної реакції** від клієнта або системи. Якщо дві помилки завжди обробляються однаково — можливо, вони не потребують різних класів. Створюйте нову помилку, коли з'являється потреба розрізнити випадки.

### «Доменні помилки потрібні тільки для Factory»

Ні. Доменні помилки кидають усі доменні об'єкти: [Value Objects](value-objects.md) (при порушенні інваріанту), [Entities та Aggregates](entities-and-aggregates.md) (при неможливій операції), [Domain Factory](domain-factory.md) (при порушенні складних інваріантів), Domain Services. Будь-яке бізнес-правило, яке може бути порушене, виражається доменною помилкою.

---

## Джерела

- **Eric Evans** — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) — помилки як частина Ubiquitous Language, доменна модель описує і валідні стани, і причини відмови
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (2013) — практичні приклади обробки доменних помилок, розділення бізнес-винятків від технічних
- **Robert C. Martin** — *Clean Architecture* (2017) — про незалежність бізнес-логіки від механізмів доставки: домен не знає про HTTP, а помилки — частина домену
