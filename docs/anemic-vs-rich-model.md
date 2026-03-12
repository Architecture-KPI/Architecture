# Anemic vs Rich Domain Model

## Зміст

- [Вступ](#вступ)
- [Anemic Domain Model](#anemic-domain-model)
- [Rich Domain Model](#rich-domain-model)
- [Порівняння](#порівняння)
- [Коли який підхід обирати](#коли-який-підхід-обирати)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

Ви створюєте модель `Booking`. Вона має поля: `id`, `user_id`, `resource_id`, `time_slot`, `status`. Питання: де живе логіка зміни статусу? Де перевірка, що скасувати можна лише активне бронювання? Де правило, що підтвердити можна лише щойно створене?

Є два полярних підходи. В одному модель — контейнер даних, а логіка — в зовнішніх сервісах. В іншому модель сама знає свої правила і захищає свій стан. Обидва підходи мають ім'я, і обидва мають право на існування — залежно від контексту.

---

## Anemic Domain Model

Анемічна модель — це об'єкт, який містить **лише дані** (але може містити тревіальні гетери, сетери) . Уся бізнес-логіка знаходиться в зовнішніх сервісах.

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass
class Booking:
    id: int
    user_id: int
    resource_id: int
    start_time: datetime
    end_time: datetime
    _status: str = "pending"

    @property
    def status(self) -> str:
        return self._status

    @status.setter
    def status(self, value: str) -> None:
        self._status = value
```

> Тривіальні геттери/сеттери через `@property` — просто повертають або присвоюють значення без жодної логіки. Анемічна модель **може** їх містити, але сама по собі не додає валідації чи бізнес-правил.

```python
class BookingService:
    def cancel(self, booking: Booking):
        if booking.status == "cancelled":
            raise ValueError("Already cancelled")
        if booking.start_time < datetime.now():
            raise ValueError("Cannot cancel past booking")
        booking.status = "cancelled"

    def confirm(self, booking: Booking):
        if booking.status != "created":
            raise ValueError("Can only confirm created bookings")
        booking.status = "confirmed"
```

Модель — просто структура з полями. Сервіс знає всі правила і маніпулює станом моделі ззовні.

### Що з цим не так?

Мартін Фаулер називає це «антипатерном», і ось чому:

- **Інваріанти не захищені**: будь-хто може написати `booking.status = "confirmed"` напряму, минаючи перевірки
- **Логіка розмазана**: якщо два сервіси працюють з `Booking` — правила дублюються або, гірше, розходяться

### Коли це працює?

При всій критиці, Anemic Model — це реальність більшості проєктів. І вона працює нормально, коли:
- Простий домен - мало бізнес правил
- Потрібна вища швидкість розробки
- Команда не знайома з DDD-підходами (краще Anemic, що працює, ніж Rich, що зроблений неправильно)
- В ФП стилі домінує саме цей підхід

**Важливо:** оскільки модель не захищає свій стан, відповідальність за коректність лягає на розробників — всі зміни мають проходити через сервіси, а не через пряме присвоєння полів.

---

## Rich Domain Model

Багата модель **інкапсулює і дані, і поведінку**. Об'єкт сам відповідає за свої інваріанти — зовнішній код не може привести його до невалідного стану.

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass
class Booking:
    _id: int
    _user_id: int
    _resource_id: int
    _start_time: datetime
    _end_time: datetime
    _status: str = "created"

    @property
    def id(self) -> int:
        return self._id

    @property
    def status(self) -> str:
        return self._status

    def cancel(self) -> None:
        if self._status == "cancelled":
            raise DomainError("Already cancelled")
        if self._start_time < datetime.now():
            raise DomainError("Cannot cancel past booking")
        self._status = "cancelled"

    def confirm(self) -> None:
        if self._status != "created":
            raise DomainError("Can only confirm created bookings")
        self._status = "confirmed"
```

Зверніть увагу:
- Поля **приватні** (`self._status`), зовнішній код не може їх змінити напряму
- Доступ до даних — через **properties** (read-only)
- Зміна стану — тільки через **методи**, які перевіряють правила

### Що це дає?

- **Інваріанти захищені**: неможливо привести `Booking` до невалідного стану, не пройшовши через метод з перевірками
- **Логіка в одному місці**: всі правила для `Booking` — в класі `Booking`. Знайти, зрозуміти, змінити — просто
- **Виразний код**: `booking.cancel()` замість `service.cancel(booking)` — код читається як бізнес-операція

---

## Порівняння

| Аспект | Anemic | Rich |
|--------|--------|------|
| Де бізнес-логіка | У зовнішніх сервісах | У самій моделі |
| Захист інваріантів | Зовнішній (хтось має викликати правильний сервіс) | Внутрішній (модель не дозволяє невалідний стан) |
| Ризик дублювання | Високий (два сервіси — два місця для правил) | Низький (одне місце) |
| Складність на старті | Нижча | Вища |
| Складність при зростанні | Зростає швидко (логіка розповзається) | Зростає повільніше (логіка локалізована) |

---

## Коли який підхід обирати

**Anemic Model** виправдана, коли:
- Домен простий — мало бізнес-правил
- Проєкт маленький і не планує рости
- Команда не знайома з DDD-підходами (краще Anemic, що працює, ніж Rich, що зроблений неправильно)

**Rich Model** виправдана, коли:
- Є складні бізнес-правила, які потрібно захистити
- Модель має **стан з переходами** (статуси, lifecycle)
- Правила ризикують дублюватися в різних сервісах
- Проєкт росте, і ціна помилки в бізнес-логіці — висока

---

## Поширені міфи

### «Anemic Model — це завжди антипатерн»

Фаулер назвав його антипатерном у контексті **складних доменів**, де бізнес-логіка — головна цінність коду. Для проектів з простим доменом і малою кількість бізнес правил, Anemic Model — прагматичний вибір.

### «Rich Model — це коли модель робить ВСЕ»

Ні. Rich Model інкапсулює логіку, що стосується **її власного стану**. Вона не робить HTTP-запити, не відправляє email, не працює з БД. Якщо операція потребує зовнішніх залежностей — це [Domain Factory](domain-factory.md), Domain Service або Application Service, а не метод моделі.

### «Rich Model не потрібна, бо є сервіси»

Сервіси оркеструють. Модель — знає правила. `BookingService` каже «потрібно скасувати бронювання» і викликає `booking.cancel()`. Якщо правила скасування живуть у сервісі, а не в моделі — при появі другого сервісу, який теж скасовує, правила доведеться дублювати або виносити в ще один «спільний» сервіс.

### «Приватні поля і properties — це overengineering»

Приватні поля — це не ритуал. Це **механізм захисту інваріантів**. Якщо `status` публічний — будь-хто може написати `booking.status = "confirmed"` і обійти всі перевірки. Приватне поле з property — це спосіб сказати: «читати можна, змінювати — тільки через мої методи».

---

## Джерела

- **Martin Fowler** — [AnemicDomainModel](https://martinfowler.com/bliki/AnemicDomainModel.html) — стаття, де Anemic Model названа антипатерном, з поясненням чому
- **Eric Evans** — *Domain-Driven Design* (2003) — Rich Domain Model як основа тактичного DDD
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (2013) — практичні приклади Rich Model з інваріантами та методами
- **Robert C. Martin** — *Clean Architecture* (2017) — про розділення бізнес-правил і механізмів доставки
