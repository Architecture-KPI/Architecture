# ORM та доменна модель: як зберегти незалежність

## Зміст

- [Вступ](#вступ)
- [Проблема: домен залежить від ORM](#проблема-домен-залежить-від-orm)
- [Декларативний vs Імперативний маппінг](#декларативний-vs-імперативний-маппінг)
- [Окрема модель для БД](#окрема-модель-для-бд)
- [Identity Map](#identity-map)
- [Unit of Work](#unit-of-work)
- [Транзакції в Application Layer](#транзакції-в-application-layer)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

Ви побудували чисту доменну модель: [Entity](entities-and-aggregates.md) з бізнес-правилами, [Value Objects](value-objects.md) з інваріантами, [Repository](repository.md) з інтерфейсом у домені. Домен нічого не знає про інфраструктуру. А потім приходить час зберегти це в базу — і починається.

ORM хоче декоратори на класах. Або наслідування від `Base`. Або публічні поля з типами колонок. І раптом ваша чиста доменна модель залежить від SQLAlchemy, Django ORM, TypeORM або Hibernate. Зміна ORM — переписування домену. Тести домену — потребують підключення до БД.

Як цього уникнути?

---

## Проблема: домен залежить від ORM

Типовий підхід — доменна модель і ORM-модель це одне й те саме:

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class Booking(Base):  # доменна модель наслідує ORM
    __tablename__ = "bookings"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column()
    resource_id: Mapped[int] = mapped_column()
    start_time: Mapped[datetime] = mapped_column()
    end_time: Mapped[datetime] = mapped_column()
    status: Mapped[str] = mapped_column()

    def cancel(self):
        if self.status == "cancelled":
            raise ValueError("Already cancelled")
        self.status = "cancelled"
```

На перший погляд зручно: один клас, все в одному місці. Але:

- **Домен залежить від SQLAlchemy** — `from sqlalchemy` в доменному файлі
- **Типи полів диктує ORM** — `Mapped[int]` замість доменних типів
- **Value Objects зникають** — `start_time` і `end_time` це примітиви, бо ORM не вміє маппити `TimeSlot`
- **Тести домену потребують SQLAlchemy** — навіть якщо БД не потрібна для тесту бізнес-логіки
- **Зміна ORM = зміна домену** — міграція з SQLAlchemy на Tortoise чи Django ORM зачіпає бізнес-логіку

---

## Декларативний vs Імперативний маппінг

Більшість ORM пропонують два способи описати, як об'єкт маппиться на таблицю.

### Декларативний (Declarative Mapping)

Маппінг описується **прямо в класі** через декоратори, анотації або наслідування:

```python
# SQLAlchemy Declarative
class BookingEntity(Base):
    __tablename__ = "bookings"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column()
    ...
```

```typescript
// TypeORM
@Entity()
class BookingEntity {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    userId: number;
    ...
}
```

Плюси: просто, зрозуміло, вся інформація в одному місці.
Мінуси: клас **залежить від ORM**. Це підходить для ORM-моделі в Infrastructure Layer, але не для доменної моделі.

### Імперативний (Imperative / Classical Mapping)

Маппінг описується **окремо від класу**. Клас — чистий Python/TypeScript/Java об'єкт, а конфігурація маппінгу живе в інфраструктурі:

```python
# Доменна модель — чистий Python, жодних імпортів ORM
class Booking:
    def __init__(self, id: str, user_id: str, resource_id: str,
                 time_slot: TimeSlot, status: BookingStatus):
        self._id = id
        self._user_id = user_id
        self._resource_id = resource_id
        self._time_slot = time_slot
        self._status = status

    def cancel(self) -> None:
        ...
```

```python
# Інфраструктура — маппінг окремо
from sqlalchemy import Table, Column, Integer, String, DateTime, MetaData

metadata = MetaData()

bookings_table = Table(
    "bookings", metadata,
    Column("id", String, primary_key=True),
    Column("user_id", String),
    Column("resource_id", String),
    Column("start_time", DateTime),
    Column("end_time", DateTime),
    Column("status", String),
)

# mapper пов'язує чистий клас з таблицею
mapper_registry.map_imperatively(Booking, bookings_table)
```

Плюси: доменна модель не залежить від ORM. Тести домену працюють без БД.
Мінуси: більше коду, складніший setup. Не всі ORM підтримують цей підхід.

### Що обрати?

Імперативний маппінг — ідеальний для чистої архітектури, але **не всі ORM його підтримують**. На практиці частіше використовують третій підхід — окрему модель для БД.

---

## Окрема модель для БД

Найуніверсальніший підхід: **дві моделі і маппер між ними**.

```
Domain Layer:          Booking (чиста доменна модель)
                         ↕ маппер
Infrastructure Layer:  BookingEntity (ORM-модель) ↔ БД
```

### Доменна модель (Domain Layer)

Чистий об'єкт без залежностей від ORM:

```python
class Booking:
    def __init__(self, id: str, user_id: str, resource_id: str,
                 time_slot: TimeSlot, status: BookingStatus):
        self._id = id
        self._user_id = user_id
        self._resource_id = resource_id
        self._time_slot = time_slot
        self._status = status

    def cancel(self) -> None:
        if self._status == BookingStatus.CANCELLED:
            raise DomainError("Вже скасовано")
        self._status = BookingStatus.CANCELLED

    # properties, бізнес-методи...
```

### ORM-модель (Infrastructure Layer)

Декларативний маппінг — тут він доречний, бо це інфраструктурний код:

```python
class BookingEntity(Base):
    __tablename__ = "bookings"

    id: Mapped[str] = mapped_column(primary_key=True)
    user_id: Mapped[str] = mapped_column()
    resource_id: Mapped[str] = mapped_column()
    start_time: Mapped[datetime] = mapped_column()
    end_time: Mapped[datetime] = mapped_column()
    status: Mapped[str] = mapped_column()
```

### Маппер (Infrastructure Layer)

Перекладає між двома світами:

```python
class BookingMapper:
    @staticmethod
    def to_domain(entity: BookingEntity) -> Booking:
        return Booking(
            id=entity.id,
            user_id=entity.user_id,
            resource_id=entity.resource_id,
            time_slot=TimeSlot(entity.start_time, entity.end_time),
            status=BookingStatus(entity.status),
        )

    @staticmethod
    def to_entity(booking: Booking) -> BookingEntity:
        return BookingEntity(
            id=booking.id,
            user_id=booking.user_id,
            resource_id=booking.resource_id,
            start_time=booking.time_slot.start,
            end_time=booking.time_slot.end,
            status=booking.status.value,
        )
```

Маппер — єдине місце, яке знає про обидва світи. Зміна ORM — переписуємо `BookingEntity` і `BookingMapper`. Домен не зачіпаємо.

### Repository використовує маппер

```python
class PostgresBookingRepository(BookingRepository):
    def __init__(self, session: Session):
        self._session = session

    def save(self, booking: Booking) -> None:
        entity = BookingMapper.to_entity(booking)
        self._session.merge(entity)

    def find_by_id(self, booking_id: str) -> Booking | None:
        entity = self._session.get(BookingEntity, booking_id)
        if entity is None:
            return None
        return BookingMapper.to_domain(entity)
```

Назовні Repository працює з доменними моделями. Всередині — з ORM. Кордон чіткий.

---

## Identity Map

Identity Map — патерн, який гарантує: **один рядок у БД = один об'єкт у пам'яті** протягом однієї сесії.

### Проблема

```python
booking_a = repo.find_by_id("123")
booking_b = repo.find_by_id("123")

booking_a.cancel()
# booking_b досі має старий статус — два об'єкти для одного рядка
```

Без Identity Map кожен `find_by_id` створює новий об'єкт. Зміни в одному — не видно в іншому. Це призводить до конфліктів і втрати даних.

### Рішення

Identity Map кешує об'єкти за їхнім ID. При повторному запиті повертає **той самий об'єкт**:

```python
booking_a = repo.find_by_id("123")
booking_b = repo.find_by_id("123")

assert booking_a is booking_b  # той самий об'єкт у пам'яті
```

Більшість ORM реалізують Identity Map автоматично через сесію (SQLAlchemy `Session`, Hibernate `EntityManager`). Тому в рамках однієї сесії `session.get(BookingEntity, "123")` завжди повертає той самий об'єкт.

### Нюанс з окремими моделями

Якщо ви використовуєте окрему доменну модель і маппер — Identity Map ORM працює для `BookingEntity`, але не для `Booking`. Кожен виклик `BookingMapper.to_domain(entity)` створює новий доменний об'єкт. У більшості випадків це не проблема, якщо дотримуватись правила: **одна операція — одне завантаження агрегату**.

---

## Unit of Work

Unit of Work — патерн, який **відстежує зміни** в об'єктах і зберігає їх у БД одним batch'ем в кінці операції.

### Проблема

```python
class CreateBookingHandler:
    def handle(self, command):
        booking = self.factory.create(...)
        self.booking_repo.save(booking)     # INSERT
        self.resource_repo.update(resource) # UPDATE
        self.audit_repo.log(event)          # INSERT
        # три окремих звернення до БД — що якщо третє впаде?
```

Три окремих виклики. Якщо третій впаде — перші два вже в БД. Дані неконсистентні.

### Рішення

Unit of Work збирає всі зміни і застосовує їх **однією транзакцією**:

```python
class CreateBookingHandler:
    def handle(self, command):
        booking = self.factory.create(...)
        self.booking_repo.save(booking)     # ще не в БД — просто відмічено
        self.resource_repo.update(resource) # ще не в БД
        self.audit_repo.log(event)          # ще не в БД
        self.unit_of_work.commit()          # все зберігається одним flush
```

ORM сесія зазвичай і є Unit of Work. SQLAlchemy `Session`, Django `transaction.atomic()`, Hibernate `EntityManager` — усе це реалізації Unit of Work.

---

## Транзакції в Application Layer

### Проблема

Транзакція — інфраструктурна концепція (`BEGIN`, `COMMIT`, `ROLLBACK`). Але рішення про те, **що є однією атомарною операцією** — це бізнес-рішення. Де описувати транзакції?

Якщо в Repository:

```python
class PostgresBookingRepository(BookingRepository):
    def save(self, booking: Booking) -> None:
        self._session.begin()
        entity = BookingMapper.to_entity(booking)
        self._session.add(entity)
        self._session.commit()  # кожен save — окрема транзакція
```

Проблема: якщо handler робить два `save` — це дві транзакції. Між ними система може бути неконсистентною.

Якщо напряму в Handler:

```python
class CreateBookingHandler:
    def handle(self, command):
        self.session.begin()  # handler знає про SQLAlchemy Session
        ...
        self.session.commit()
```

Проблема: Application Layer залежить від інфраструктури.

### Рішення: абстракція Unit of Work

Визначити інтерфейс у Application Layer. Реалізацію — в Infrastructure:

```python
# application/ports.py
class UnitOfWork(ABC):
    @abstractmethod
    def __enter__(self) -> 'UnitOfWork': ...

    @abstractmethod
    def __exit__(self, exc_type, exc_val, exc_tb) -> None: ...

    @abstractmethod
    def commit(self) -> None: ...

    @abstractmethod
    def rollback(self) -> None: ...
```

```python
# infrastructure/unit_of_work.py
class SqlAlchemyUnitOfWork(UnitOfWork):
    def __init__(self, session_factory):
        self._session_factory = session_factory

    def __enter__(self):
        self._session = self._session_factory()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.rollback()
        self._session.close()

    def commit(self) -> None:
        self._session.commit()

    def rollback(self) -> None:
        self._session.rollback()
```

```python
# application/commands/create_booking.py
class CreateBookingHandler:
    def __init__(self, uow: UnitOfWork, factory: BookingFactory):
        self._uow = uow
        self._factory = factory

    def handle(self, command):
        with self._uow:
            booking = self._factory.create(...)
            self._uow.booking_repo.save(booking)
            self._uow.commit()
```

Handler працює з абстракцією `UnitOfWork`. Він каже: «ця операція — одна транзакція». Як саме транзакція реалізована (PostgreSQL, SQLite, in-memory) — деталь інфраструктури.

### Альтернатива: декоратор / middleware

Замість явного `commit()` в handler'і, транзакцію можна обгорнути ззовні:

```python
class TransactionalMiddleware:
    def __init__(self, uow: UnitOfWork):
        self._uow = uow

    def execute(self, handler, command):
        with self._uow:
            result = handler.handle(command)
            self._uow.commit()
            return result
```

Handler взагалі не знає про транзакції — вони додаються на рівні інфраструктури. Це чистіший підхід, але складніший у реалізації.

---

## Поширені міфи

### «Доменна модель і ORM-модель — це одне й те саме»

Ні. ORM-модель описує структуру таблиці. Доменна модель описує бізнес-концепцію. Вони можуть мати різні поля (домен використовує `TimeSlot`, БД — два окремі `datetime`), різні типи, різну структуру. Збіг полів — збіг, а не правило.

### «Окремі моделі — це зайвий бойлерплейт»

Це ціна за незалежність. Без окремих моделей зміна схеми БД ламає бізнес-логіку, а тести домену потребують підняття БД. Маппер — кілька десятків рядків коду, які захищають від годин рефакторингу.

### «Транзакції — справа Repository»

Repository відповідає за персистенцію одного агрегату. Транзакція може охоплювати кілька репозиторіїв в рамках одного use case. Це рішення Application Layer, а не Repository.

### «Unit of Work потрібен завжди»

Для простих операцій (один save) явний Unit of Work може бути зайвим. Він стає цінним, коли handler координує кілька змін, що мають бути атомарними.

---

## Джерела

- **Martin Fowler** — *Patterns of Enterprise Application Architecture* (2002) — оригінальні описи патернів Identity Map, Unit of Work, Data Mapper
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (2013) — практичні приклади збереження агрегатів через Repository та ORM
- **Robert C. Martin** — *Clean Architecture* (2017) — про незалежність бізнес-логіки від фреймворків та механізмів персистенції
- **Cosmic Python** — Harry Percival, Bob Gregory — [cosmicpython.com](https://www.cosmicpython.com/) — практичний посібник з Repository, Unit of Work та їхньої інтеграції з SQLAlchemy
