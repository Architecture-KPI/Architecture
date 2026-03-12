# Value Objects

## Зміст

- [Вступ](#вступ)
- [Що таке Value Object](#що-таке-value-object)
- [Рівність за значенням, а не за ідентичністю](#рівність-за-значенням-а-не-за-ідентичністю)
- [Іммутабельність](#іммутабельність)
- [Value Object як місце для логіки](#value-object-як-місце-для-логіки)
- [Коли виділяти Value Object](#коли-виділяти-value-object)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

У кожному проєкті є дані, які розробники передають як примітиви: `str` для email, `float` для грошей, два `datetime` для часового слота. Код працює — до першого багу. Хтось передав від'ємну суму. Хтось створив часовий слот, де початок пізніше за кінець. Хтось порівняв два email з різним регістром і отримав «різних» користувачів.

Проблема не в неуважності — проблема в тому, що примітив не захищає свій зміст. `str` не знає, що він email. `float` не знає, що він гроші. Value Object знає.

---

## Що таке Value Object

Value Object — це об'єкт, який описує **характеристику або концепцію** домену. Він не має власної ідентичності — його визначає набір значень, які він містить.

### Проблема: примітивна одержимість (Primitive Obsession)

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass
class Booking:
    user_id: str
    resource_id: str
    start_time: datetime
    end_time: datetime
```

Що заважає створити бронювання, де `start_time > end_time`? Нічого. Перевірка перетину двох бронювань — це порівняння чотирьох `datetime` полів у кожному місці, де це потрібно. Логіка «чи перетинаються два часові слоти» — розмазана по сервісах або дубльована.

### Рішення: Value Object

```python
@dataclass(frozen=True)
class TimeSlot:
    start: datetime
    end: datetime

    def __post_init__(self):
        if self.start >= self.end:
            raise ValueError("start must be before end")

    def overlaps(self, other: 'TimeSlot') -> bool:
        return self.start < other.end and other.start < self.end

    def duration_minutes(self) -> int:
        return int((self.end - self.start).total_seconds() / 60)

    def is_in_past(self) -> bool:
        return self.end < datetime.now()
```

Тепер `TimeSlot` завжди валідний (start < end), уміє перевіряти перетин, знає свою тривалість. Код, який його використовує, стає чистішим:

```python
@dataclass
class Booking:
    user_id: str
    resource_id: str
    time_slot: TimeSlot
```

Замість `booking.start_time < other.end_time and other.start_time < booking.end_time` — просто `booking.time_slot.overlaps(other.time_slot)`.

---

## Рівність за значенням, а не за ідентичністю

Головна відмінність Value Object від [Entity](entities-and-aggregates.md): два Value Objects **рівні**, якщо їхні значення однакові. Не потрібен ID.

```python
slot_a = TimeSlot(datetime(2025, 1, 1, 10, 0), datetime(2025, 1, 1, 11, 0))
slot_b = TimeSlot(datetime(2025, 1, 1, 10, 0), datetime(2025, 1, 1, 11, 0))

assert slot_a == slot_b  # True — однакові значення
```

Це як гроші: дві купюри по 100 грн — взаємозамінні. Нам неважливо, яка саме купюра — важлива сума. Entity ж — як люди: дві особи з однаковим ім'ям — різні люди, бо мають різну ідентичність.

---

## Іммутабельність

Value Object має бути **незмінним** (immutable). Після створення його значення не міняються.

### Чому?

Уявіть, що `TimeSlot` мутабельний:

```python
slot = TimeSlot(start=datetime(2025, 1, 1, 10, 0), end=datetime(2025, 1, 1, 11, 0))
booking_a.time_slot = slot
booking_b.time_slot = slot

# пізніше хтось міняє slot
slot.end = datetime(2025, 1, 1, 12, 0)
# обидва бронювання тепер мають інший слот — несподівано
```

Мутабельний Value Object, який шерять кілька об'єктів, створює неочевидні побічні ефекти. Іммутабельність це виключає: якщо потрібен інший слот — створюємо новий:

```python
new_slot = TimeSlot(start=slot.start, end=datetime(2025, 1, 1, 12, 0))
```


## Value Object як місце для логіки

Value Object — це не «просто дані». Це повноцінний об'єкт з поведінкою, яка стосується його концепції.

### Email

```python
@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if "@" not in self.value or "." not in self.value.split("@")[-1]:
            raise ValueError(f"Invalid email: {self.value}")
        object.__setattr__(self, 'value', self.value.lower())

    def domain(self) -> str:
        return self.value.split("@")[1]
```

Без `Email` як Value Object — валідація email дублюється у кожному місці, де він приймається. З `Email` — валідація відбувається один раз, при створенні. Якщо об'єкт `Email` існує — він валідний.

---

## Коли виділяти Value Object

Ознаки того, що примітив варто замінити на Value Object:

| Ознака | Приклад |
|--------|---------|
| Примітив має обмеження, які перевіряються в різних місцях | Email, телефон, дата народження |
| Група полів завжди використовується разом | start_time + end_time → TimeSlot |
| Є логіка, що стосується цього значення | Порівняння, конвертація, форматування |
| Одне й те саме значення означає одне й те саме | 100 грн = 100 грн, незалежно від «екземпляра» |

Не потрібно перетворювати кожен `str` і `int` у Value Object. Це виправдано, коли примітив має **бізнес-зміст**, **обмеження** або **поведінку**.

---

## Поширені міфи

### «Value Object — це просто immutable клас»

Іммутабельність — необхідна, але недостатня характеристика. Конфіг теж може бути immutable, але він не Value Object. Value Object — це доменна концепція з рівністю за значенням та поведінкою, що стосується цієї концепції.

### «Value Object не може містити логіку»

Може і має. `TimeSlot.overlaps()`, `Money.add()`, `Email.domain()` — це логіка, яка належить саме цьому об'єкту. Value Object без логіки — це примітив з обгорткою.

### «Кожен примітив потрібно загорнути в Value Object»

Ні. `user_id: str` не потребує Value Object, якщо це просто ідентифікатор без бізнес-логіки. Value Object виправданий, коли є інваріанти, поведінка або коли група полів завжди використовується разом.

---

## Джерела

- **Eric Evans** — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) — оригінальне визначення Value Object (Chapter 5: «A Model Expressed in Software»)
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (2013) — практичні приклади реалізації Value Objects для різних мов
- **Martin Fowler** — [ValueObject](https://martinfowler.com/bliki/ValueObject.html) — стаття з чітким визначенням та прикладами
