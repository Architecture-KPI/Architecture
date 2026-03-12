# Доменні помилки

## Зміст

- [Вступ](#вступ)
- [Проблема: помилки без структури](#проблема-помилки-без-структури)
- [Що таке доменна помилка](#що-таке-доменна-помилка)
- [Структура помилок](#структура-помилок)
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

Домен знає про HTTP-статуси. Зміна фреймворку (FastAPI → Flask → gRPC) ламає бізнес-логіку. Доменні тести потребують HTTP-фреймворк. Порушено DIP — домен залежить від інфраструктури.

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
| **Мова домену** | Повідомлення описує бізнес-ситуацію, не технічну деталь |
| **Живе в Domain Layer** | Не імпортує нічого з інфраструктури, фреймворків, HTTP |
| **Відокремлена від generic exceptions** | `DomainError`, а не `ValueError` чи `Exception` — легко перехопити всі бізнес-помилки |
| **Несе контекст** | Повідомлення містить дані, потрібні для розуміння, що сталося |

---

## Структура помилок

Достатньо одного базового класу `DomainError`. Не потрібно створювати окремий клас для кожної бізнес-ситуації — повідомлення в помилці описує, що саме пішло не так:

```python
class DomainError(Exception):
    pass
```

Використання:

```python
raise DomainError("Slot is already taken for this resource")
raise DomainError("Booking is already cancelled")
raise DomainError(f"Invalid time range: {start} >= {end}")
raise DomainError(f"Resource {resource_id} not found")
```

Один клас — один `except DomainError` в Presentation Layer. Просто, зрозуміло, без зайвої ієрархії.

### Коли додавати підкласи

Підкласи потрібні **тільки якщо** Presentation Layer має по-різному реагувати на різні помилки (наприклад, різні HTTP-статуси):

```python
class DomainError(Exception):
    pass

class NotFoundError(DomainError):
    pass
```

`NotFoundError` → HTTP 404, все інше `DomainError` → HTTP 409 або 400. Не більше 2-3 підкласів — якщо їх більше, скоріш за все, ви ускладнюєте без потреби.

### Де живуть помилки

```
src/
├── domain/
│   ├── errors.py           # DomainError, NotFoundError
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
            raise DomainError(f"Invalid time range: {self.start} >= {self.end}")
```

Якщо `TimeSlot` існує — він валідний. Це гарантія, яку дає Value Object.

### Entities та Aggregates

[Entity](entities-and-aggregates.md) кидає помилку, коли бізнес-операція неможлива в поточному стані:

```python
class Booking:
    def cancel(self) -> None:
        if self._status == "cancelled":
            raise DomainError("Booking is already cancelled")
        self._status = "cancelled"
```

### Domain Factory

[Factory](domain-factory.md) кидає помилку, коли створення об'єкта порушує складні інваріанти:

```python
class BookingFactory:
    def create(self, user_id, resource_id, start, end) -> Booking:
        time_slot = TimeSlot(start, end)  # може кинути DomainError

        if not self._resource_repo.find_by_id(resource_id):
            raise NotFoundError(f"Resource {resource_id} not found")

        if self._booking_repo.find_overlapping(resource_id, time_slot):
            raise DomainError("Slot is already taken for this resource")

        return Booking(...)
```

### Domain Service

Якщо бізнес-правило залучає кілька агрегатів — його перевіряє Domain Service, і він теж кидає доменну помилку.

---

## Хто обробляє доменні помилки

Домен кидає помилку. Але хто вирішує, що з нею робити? **Presentation Layer**.

### Маппінг у HTTP-статуси (FastAPI)

Presentation Layer перехоплює доменні помилки і транслює їх у відповідь, зрозумілу клієнту. У FastAPI це робиться через **exception handlers**:

```python
# presentation/exception_handlers.py
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from starlette.requests import Request


async def domain_error_handler(request: Request, exc: Exception) -> JSONResponse:
    return JSONResponse(status_code=409, content={"detail": str(exc)})


async def not_found_error_handler(request: Request, exc: Exception) -> JSONResponse:
    return JSONResponse(status_code=404, content={"detail": str(exc)})


def register_exception_handlers(app: FastAPI) -> None:
    app.add_exception_handler(DomainError, domain_error_handler)
    app.add_exception_handler(NotFoundError, not_found_error_handler)
```

`NotFoundError` матчиться першим (бо конкретніший), решта `DomainError` ловиться базовим handler'ом. Два класи — два handler'и — нічого зайвого.

### Чому саме Presentation Layer?

Домен каже: «слот зайнятий». Це бізнес-факт, і він не знає, через який канал прийшов запит. Що з цим робити — залежить від контексту:

- **REST API** — повертає HTTP 409
- **gRPC** — повертає `ALREADY_EXISTS`
- **CLI** — виводить повідомлення в термінал
- **Тест** — перевіряє, що виняток кинуто

Якщо домен знає про HTTP — він не зможе працювати в інших контекстах. Розділення дає гнучкість: один і той самий домен обслуговує REST, gRPC, CLI і тести без змін.

---

## Помилки як частина Ubiquitous Language

Доменні помилки — це не технічна деталь. Вони описують ситуації, про які знає бізнес. Навіть з одним класом `DomainError` — **повідомлення** має говорити мовою домену:

```python
# Добре — мова домену
raise DomainError("Slot is already taken for this resource")
raise DomainError("Cannot cancel past booking")

# Погано — технічна мова
raise DomainError("Constraint violation")
raise DomainError("Invalid state transition")
```

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
            raise DomainError(f"Invalid time range: {self.start} >= {self.end}")
```

`ValidationError` — це «дані не пройшли вхідний контроль». `DomainError` — це «бізнес-правило порушене». Вони можуть виглядати схоже, але відповідають за різне і кидаються різними шарами.

---

## Поширені міфи

### «Потрібно створити окремий клас для кожної помилки»

Ні. Окремі класи потрібні тільки коли Presentation Layer має по-різному реагувати (наприклад, 404 vs 409). Якщо всі бізнес-помилки обробляються однаково — достатньо одного `DomainError` з описовим повідомленням.

### «Доменна помилка = HTTP-виняток»

Ні. `HTTPException(409)` — це деталь Presentation Layer. Домен не знає про HTTP. Якщо той самий домен використовується через CLI або gRPC — `HTTPException` не має сенсу. Домен кидає `DomainError`, а Presentation маппить його в 409 для REST, `ALREADY_EXISTS` для gRPC, або повідомлення в терміналі для CLI.

### «Доменні помилки потрібні тільки для Factory»

Ні. Доменні помилки кидають усі доменні об'єкти: [Value Objects](value-objects.md) (при порушенні інваріанту), [Entities та Aggregates](entities-and-aggregates.md) (при неможливій операції), [Domain Factory](domain-factory.md) (при порушенні складних інваріантів), Domain Services. Будь-яке бізнес-правило, яке може бути порушене, виражається доменною помилкою.

---

## Джерела

- **Eric Evans** — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) — помилки як частина Ubiquitous Language, доменна модель описує і валідні стани, і причини відмови
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (2013) — практичні приклади обробки доменних помилок, розділення бізнес-винятків від технічних
- **Robert C. Martin** — *Clean Architecture* (2017) — про незалежність бізнес-логіки від механізмів доставки: домен не знає про HTTP, а помилки — частина домену
