# Entities та Aggregates

## Зміст

- [Вступ](#вступ)
- [Entity: об'єкт з ідентичністю](#entity-обєкт-з-ідентичністю)
- [Entity vs Value Object](#entity-vs-value-object)
- [Aggregate: межа консистентності](#aggregate-межа-консистентності)
- [Aggregate Root](#aggregate-root)
- [Правила проєктування Aggregates](#правила-проєктування-aggregates)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

Ви моделюєте систему бронювання. Є бронювання, є користувач, є ресурс. Усі мають поля, усі зберігаються в БД. Спокуса — зробити їх однаковими: клас з полями, getter'и, setter'и, `save()`. Але в домені ці об'єкти фундаментально різні.

Користувач з ID=42 і користувач з ID=43 — різні люди, навіть якщо в них однакове ім'я. А два часові слоти «10:00–11:00» — однакові, незалежно від того, де вони створені. Перший — Entity. Другий — [Value Object](value-objects.md). Це не просто академічна різниця — від неї залежить, як ви порівнюєте об'єкти, як зберігаєте, як захищаєте їхню цілісність.

---

## Entity: об'єкт з ідентичністю

Entity — це доменний об'єкт, який визначається **ідентичністю**, а не набором атрибутів. Навіть якщо всі поля змінилися — це все ще той самий об'єкт, поки ID залишається тим самим.

### Проблема

```python
user_a = User(name="Іван", email="ivan@example.com")
user_b = User(name="Іван", email="ivan@example.com")

# це один і той самий користувач? або два різних з однаковими даними?
```

Без чіткої ідентичності відповідь неоднозначна. Якщо це два різних записи в БД — це два різних Entity. Якщо один користувач змінив прізвище — він все ще той самий Entity.

### Рішення

Entity має унікальний ідентифікатор, і рівність визначається за ним:

```python
from dataclasses import dataclass


@dataclass
class User:
    _id: str
    _name: str
    _email: str

    @property
    def id(self) -> str:
        return self._id

    def __eq__(self, other):
        if not isinstance(other, User):
            return False
        return self._id == other._id

    def __hash__(self):
        return hash(self._id)
```

Два `User` рівні, якщо мають однаковий `id` — незалежно від імені чи email. Користувач може змінити ім'я, але залишиться тим самим Entity.

### Ключові характеристики Entity

- Має **унікальний ідентифікатор** (ID), який не змінюється протягом життя об'єкта
- **Рівність за ID**: `user_a == user_b` якщо `user_a.id == user_b.id`
- Має **життєвий цикл**: створення → зміна стану → видалення
- Може бути **мутабельним**: стан Entity змінюється з часом (статус бронювання, ім'я користувача)
- Може містити **бізнес-правила**, що стосуються її стану — наприклад, правила переходу між статусами, валідація змін. Детальніше про те, скільки логіки має бути в моделі — в [Anemic vs Rich Domain Model](anemic-vs-rich-model.md)

---

## Entity vs Value Object

| Аспект | Entity | [Value Object](value-objects.md) |
|--------|--------|-------------|
| Ідентичність | Має ID | Не має ID |
| Рівність | За ID | За значенням полів |
| Мутабельність | Може змінюватися | Іммутабельний |
| Життєвий цикл | Створення → зміни → видалення | Створив — використав — забув |
| Приклад | User, Booking, Resource | TimeSlot, Email, Money |
| Зберігання | Свій рядок/документ у БД | Частина Entity (вбудований) |

Практичне правило: якщо два об'єкти з однаковими полями — це **той самий** об'єкт, це Value Object. Якщо вони можуть бути **різними** (два бронювання на один слот для різних користувачів) — це Entity.

---

## Aggregate: межа консистентності

### Проблема

У системі бронювання є `Booking`, і в нього є `TimeSlot`. Є `Resource`, і в нього є список робочих годин. Коли створюється бронювання — потрібно перевірити, що слот не зайнятий, що ресурс існує, що час в межах робочих годин.

Хто відповідає за цю перевірку? Якщо кожен об'єкт живе сам по собі — ніхто не гарантує загальну консистентність. Можна змінити робочі години ресурсу і не перевірити існуючі бронювання. Можна створити два бронювання на один слот одночасно.

### Рішення

Aggregate — це **кластер пов'язаних об'єктів**, які розглядаються як одне ціле для змін даних. Aggregate визначає **межу консистентності** (consistency boundary): усе всередині агрегату завжди перебуває у валідному стані.

```python
from dataclasses import dataclass


@dataclass
class Booking:  # Aggregate Root
    _id: str
    _user_id: str
    _resource_id: str
    _time_slot: TimeSlot
    _status: str = "created"

    def cancel(self) -> None:
        if self._status == "cancelled":
            raise DomainError("Already cancelled")
        self._status = "cancelled"

    def confirm(self) -> None:
        if self._status != "created":
            raise DomainError("Can only confirm created bookings")
        self._status = "confirmed"
```

`Booking` — це Aggregate. Він містить `TimeSlot` (Value Object) і `BookingStatus`. Методи `cancel()` та `confirm()` гарантують, що переходи між статусами відбуваються за правилами. Зовнішній код не може встановити довільний статус напряму.

---

## Aggregate Root

Aggregate Root — це **єдина точка входу** до агрегату. Зовнішній код взаємодіє тільки з коренем, ніколи напряму з внутрішніми об'єктами.

### Правила

- Зовнішній код **звертається тільки до Root** — не до внутрішніх Entity чи Value Objects
- Root **контролює інваріанти** всього агрегату
- [Repository](repository.md) працює **тільки з Aggregate Roots** — не з внутрішніми об'єктами

### Приклад

Якщо `Order` — це Aggregate Root, а `OrderLine` — внутрішня Entity:

```python
from dataclasses import dataclass, field


@dataclass
class Order:  # Aggregate Root
    _id: str
    _lines: list[OrderLine] = field(default_factory=list)
    _status: str = "draft"

    def add_line(self, product_id: str, quantity: int, price: Money) -> None:
        if self._status != "draft":
            raise DomainError("Можна додавати позиції тільки до чернетки")
        line = OrderLine(product_id=product_id, quantity=quantity, price=price)
        self._lines.append(line)

    def total(self) -> Money:
        return sum((line.subtotal() for line in self._lines), Money.zero())
```

Зовнішній код каже `order.add_line(...)`, а не `order.lines.append(OrderLine(...))`. Root контролює, що додавання можливе (замовлення ще у статусі чернетки).

---

## Правила проєктування Aggregates

### 1. Робіть агрегати маленькими

Aggregate має бути настільки маленьким, наскільки це можливо, зберігаючи при цьому інваріанти. Великий агрегат — це проблема з конкурентним доступом і продуктивністю.

Погано: `Resource` як агрегат містить усі бронювання всередині:

```python
from dataclasses import dataclass, field


@dataclass
class Resource:
    _id: str
    _bookings: list[Booking] = field(default_factory=list)  # сотні/тисячі бронювань

    def add_booking(self, booking: Booking):
        # перевірка конфліктів через весь список
        ...
```

Краще: `Booking` і `Resource` — окремі агрегати. Перевірка конфліктів — через [Domain Factory](domain-factory.md), яка запитує репозиторій.

### 2. Посилайтесь на інші агрегати за ID

Агрегати не мають тримати прямі посилання на інші агрегати. Використовуйте ID:

```python
from dataclasses import dataclass


@dataclass
class Booking:
    _id: str
    _user_id: str          # ID, не об'єкт User
    _resource_id: str      # ID, не об'єкт Resource
    # ...
```

Чому: якщо `Booking` тримає `User` — завантаження одного бронювання тягне за собою завантаження користувача. А якщо `User` тримає список бронювань — ще й усіх бронювань. Це каскад, який швидко стає неконтрольованим.

### 3. Інваріанти визначають межі

Якщо правило «слот не може перетинатися з іншим» стосується лише бронювань одного ресурсу — це інваріант, який може перевірятися при створенні через Domain Factory. Не потрібно засовувати всі бронювання в один агрегат заради цієї перевірки.

---

## Поширені міфи

### «Entity — це те саме, що ORM Entity»

Ні. ORM Entity (наприклад, SQLAlchemy model) — це модель таблиці БД. Доменна Entity — це бізнес-об'єкт з ідентичністю та поведінкою. Вони можуть мати різні поля, різну структуру, різні типи. Маппінг між ними — відповідальність Infrastructure Layer.

### «Aggregate — це просто група пов'язаних таблиць»

Aggregate — це про бізнес-інваріанти, а не про зв'язки в БД. Дві таблиці з foreign key — це не автоматично один агрегат. Агрегат визначається тим, які об'єкти мають змінюватися **разом атомарно** для підтримки консистентності.

### «Чим більший агрегат — тим краще, бо більше інваріантів перевіряємо»

Навпаки. Великий агрегат — це вузьке горлечко: блокування при конкурентних змінах, повільне завантаження, складне тестування. Vaughn Vernon радить: «Проєктуйте маленькі агрегати».

### «Aggregate Root має мати getter'и для всіх внутрішніх об'єктів»

Ні. Root контролює доступ. Якщо зовнішньому коду потрібна інформація — Root надає її через методи, а не через прямий доступ до внутрішніх структур. Це захищає інкапсуляцію.

---

## Джерела

- **Eric Evans** — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) — оригінальні визначення Entity, Value Object, Aggregate (Chapters 5-6)
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (2013) — практичні правила проєктування агрегатів (Chapter 10: «Aggregates»), правило «маленьких агрегатів»
- **Vaughn Vernon** — [Effective Aggregate Design](https://www.dddcommunity.org/library/vernon_2011/) — серія статей про проєктування агрегатів
