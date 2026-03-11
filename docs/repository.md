# Repository Pattern

## Зміст

- [Вступ](#вступ)
- [Що таке Repository і яку проблему він вирішує](#що-таке-repository-і-яку-проблему-він-вирішує)
- [Repository — це доменний патерн](#repository--це-доменний-патерн)
- [Інтерфейс репозиторію: що має бути, а що — ні](#інтерфейс-репозиторію-що-має-бути-а-що--ні)
- [Repository vs DAO](#repository-vs-dao)
- [Repository та CQS: хто відповідає за читання?](#repository-та-cqs-хто-відповідає-за-читання)
- [Реалізація: де і як](#реалізація-де-і-як)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

Слово «репозиторій» в розробці перевантажене. Git-репозиторій, npm registry, generic `BaseRepository<T>` з десятком CRUD-методів — усе це називають «репозиторієм». В результаті коли хтось каже «зроби репозиторій», незрозуміло: це обгортка над ORM? Місце для SQL-запитів? Клас з методом `find_by_email_and_status_and_date_range()`?

Repository Pattern — це конкретний тактичний патерн з Domain-Driven Design. Він має чітку мету, чіткі межі і чітко визначену зону відповідальності. І ця зона значно вужча, ніж те, у що його зазвичай перетворюють.

---

## Що таке Repository і яку проблему він вирішує

### Проблема

Доменна модель не повинна знати, де і як вона зберігається. Це основа DIP і шарової архітектури. Але домену потрібна можливість «зберегти бронювання» і «знайти бронювання за ID». Як це зробити, якщо домен не може імпортувати ORM?

Без репозиторію доменна логіка змішується з персистенцією:

```python
class CreateBookingHandler:
    def handle(self, command):
        booking = self.factory.create(...)

        # доменний handler знає про SQL, транзакції, ORM
        session = Session()
        entity = BookingEntity(
            id=booking.id,
            user_id=booking.user_id,
            start_time=booking.time_slot.start,
        )
        session.add(entity)
        session.commit()
```

Домен залежить від інфраструктури. Тести потребують БД. Зміна ORM ламає бізнес-логіку.

### Рішення

Repository — це **абстракція колекції доменних об'єктів**. Для домену він виглядає як in-memory колекція: можна додати об'єкт, можна знайти за ідентифікатором. Як саме дані зберігаються — домен не знає і не хоче знати.

```python
# domain/repositories.py — інтерфейс в домені
class BookingRepository(ABC):
    @abstractmethod
    def save(self, booking: Booking) -> None: ...

    @abstractmethod
    def find_by_id(self, booking_id: str) -> Booking | None: ...
```

```python
# application/create_booking.py — handler працює з абстракцією
class CreateBookingHandler:
    def __init__(self, repo: BookingRepository, factory: BookingFactory):
        self._repo = repo
        self._factory = factory

    def handle(self, command):
        booking = self._factory.create(...)
        self._repo.save(booking)  # як зберігається — не знаю і не хочу
```

Тепер handler тестується з `InMemoryBookingRepository`, а в продакшені працює `PostgresBookingRepository`. Домен чистий.

### Визначення

Ерік Еванс у «Domain-Driven Design» формулює так:

> Repository забезпечує ілюзію in-memory колекції всіх об'єктів певного типу. Він інкапсулює механізм збереження та пошуку, надаючи доменному шару простий інтерфейс для роботи з агрегатами.

Ключове слово: **агрегати**. Repository працює з доменними сутностями (агрегатами), а не з таблицями, рядками або довільними проєкціями.

---

## Repository — це доменний патерн

Repository належить домену — не інфраструктурі, не Application Layer. Це означає:

**Інтерфейс** репозиторію визначається в Domain Layer. Домен каже: «мені потрібна можливість зберегти Booking і знайти його за ID». Це мова домену, не мова SQL.

**Реалізація** живе в Infrastructure Layer. Інфраструктура каже: «ось як я це зроблю через PostgreSQL / MongoDB / файл».

```
Domain Layer:          BookingRepository (interface)
                              ↑
Infrastructure Layer:  PostgresBookingRepository (implementation)
```

Repository — це **порт** в термінах гексагональної архітектури. Домен визначає контракт, інфраструктура його реалізує. Залежність направлена від інфраструктури до домену, а не навпаки.

---

## Інтерфейс репозиторію: що має бути, а що — ні

Це місце, де більшість помилок. Інтерфейс репозиторію має відображати **потреби домену в роботі з агрегатами**. Не більше.

### Що належить репозиторію

```python
class BookingRepository(ABC):
    @abstractmethod
    def save(self, booking: Booking) -> None: ...

    @abstractmethod
    def find_by_id(self, booking_id: str) -> Booking | None: ...

    @abstractmethod
    def find_overlapping(self, resource_id: str, time_slot: TimeSlot) -> list[Booking]: ...
```

- `save` — зберегти агрегат (створення або оновлення)
- `find_by_id` — знайти агрегат за ідентифікатором
- `find_overlapping` — доменна потреба: перевірити конфлікт слотів для бізнес-правила

Ці методи потрібні **домену** для виконання бізнес-логіки.

### Що НЕ належить репозиторію

```python
# НЕ РОБІТЬ ТАК
class BookingRepository(ABC):
    @abstractmethod
    def find_all_with_user_name_and_resource(self) -> list[BookingDetailDTO]: ...

    @abstractmethod
    def count_by_status(self, status: str) -> int: ...

    @abstractmethod
    def find_for_calendar_view(self, month: int) -> list[CalendarEntryDTO]: ...

    @abstractmethod
    def search(self, query: str, page: int, size: int) -> PaginatedResult: ...
```

Чому це не належить репозиторію:

- `find_all_with_user_name_and_resource` — повертає DTO з денормалізованими даними для UI. Це потреба **клієнта** (Presentation), а не домену.
- `count_by_status` — агрегація для дашборду або аналітики. Домену не потрібно підраховувати бронювання за статусом для виконання бізнес-правил.
- `find_for_calendar_view` — проєкція під конкретний UI-компонент. Домен не знає про календарі.
- `search` з пагінацією — інфраструктурна оптимізація для відображення списків.

Усе це — **логіка читання (query)**. Вона не працює з доменними агрегатами, не виконує бізнес-правил, і не належить доменному репозиторію.

### Правило

Запитайте себе: «чи потрібен цей метод домену для виконання бізнес-правила або збереження агрегату?»

- Так → належить репозиторію
- Ні, це для відображення / звітності / пошуку → не належить

---

## Repository vs DAO

Repository часто плутають з DAO (Data Access Object). Різниця принципова:

| Аспект | Repository | DAO |
|--------|-----------|-----|
| Рівень абстракції | Доменні агрегати | Таблиці / рядки БД |
| Мова інтерфейсу | Мова домену (`save(booking)`) | Мова даних (`insert(row)`, `query(sql)`) |
| Що повертає | Доменні моделі | ORM-сутності, рядки, dict |
| Де живе інтерфейс | Domain Layer | Infrastructure Layer |
| Приховує | Деталі персистенції від домену | Конкретну БД від решти коду |

DAO — це про **доступ до даних**. Repository — це про **роботу з доменними об'єктами**. DAO може бути деталлю реалізації всередині Repository:

```python
class PostgresBookingRepository(BookingRepository):
    def __init__(self, session: Session):
        self._session = session  # ORM session — це, по суті, DAO

    def save(self, booking: Booking) -> None:
        entity = BookingMapper.to_entity(booking)
        self._session.merge(entity)

    def find_by_id(self, booking_id: str) -> Booking | None:
        entity = self._session.get(BookingEntity, booking_id)
        if entity is None:
            return None
        return BookingMapper.to_domain(entity)
```

Repository використовує ORM (DAO-рівень) всередині, але назовні виглядає як колекція доменних об'єктів.

---

## Repository та CQS: хто відповідає за читання?

Це ключове питання. Якщо репозиторій — для доменних агрегатів, то хто обслуговує запити на читання (отримати список бронювань для UI, показати календар, вивести статистику)?

### Проблема «роздутого» репозиторію

Коли все читання пхають в репозиторій, він перетворюється на звалище:

```python
class BookingRepository(ABC):
    # доменні потреби
    def save(self, booking: Booking) -> None: ...
    def find_by_id(self, booking_id: str) -> Booking | None: ...
    def find_overlapping(self, resource_id: str, slot: TimeSlot) -> list[Booking]: ...

    # потреби UI
    def find_all_for_user_page(self, user_id: str) -> list[BookingListItem]: ...
    def find_for_admin_dashboard(self) -> DashboardData: ...
    def search_with_filters(self, filters: dict) -> PaginatedResult: ...
    def get_monthly_stats(self, month: int) -> MonthlyStats: ...
```

Половина методів — для домену, половина — для UI. Різні причини для змін, різні споживачі, різні моделі даних. Це порушення SRP: клас має дві причини для змін — зміна бізнес-логіки та зміна потреб UI.

### Рішення: Read Repository / Query Service

Розділити на два інтерфейси з різними відповідальностями:

```python
# domain/repositories.py — для домену (write + domain reads)
class BookingRepository(ABC):
    @abstractmethod
    def save(self, booking: Booking) -> None: ...

    @abstractmethod
    def find_by_id(self, booking_id: str) -> Booking | None: ...

    @abstractmethod
    def find_overlapping(self, resource_id: str, slot: TimeSlot) -> list[Booking]: ...
```

```python
# application/queries/ — для читання (query handlers)
class BookingReadRepository(ABC):
    @abstractmethod
    def find_user_bookings(self, user_id: str) -> list[BookingListDTO]: ...

    @abstractmethod
    def find_available_slots(self, resource_id: str, date: date) -> list[TimeSlotDTO]: ...
```

- `BookingRepository` — доменний патерн, працює з агрегатами, живе в Domain Layer
- `BookingReadRepository` — інтерфейс для Query Handler'ів, повертає DTO/Read Models, живе в Application Layer

Це природно вписується в CQS: команди використовують доменний `BookingRepository`, запити — `BookingReadRepository`. Різні потреби, різні інтерфейси, різні реалізації за потреби.

### Чи може Read Repository теж роздутися?

Так. Якщо `BookingReadRepository` обслуговує і сторінку бронювань користувача, і адмін-дашборд, і календар — це знову різні споживачі з різними причинами для змін. За принципом ISP, кожен Query Handler міг би залежати від свого вузького інтерфейсу (`UserBookingsReader`, `SlotAvailabilityReader` тощо), а одна реалізація — імплементувати їх усі.

На практиці розділення write/read — вже значний крок. Подальше дроблення read-інтерфейсів — це рішення, яке приймається коли кількість запитів та їхніх споживачів починає рости.

---

## Реалізація: де і як

### Структура

```
src/
├── domain/
│   └── repositories/
│       └── booking_repository.py      # інтерфейс (для домену)
├── application/
│   └── queries/
│       └── booking_read_repository.py # інтерфейс (для читання)
└── infrastructure/
    └── repositories/
        ├── postgres_booking_repo.py       # реалізація доменного
        └── postgres_booking_read_repo.py  # реалізація для читання
```

### Один клас — два інтерфейси?

Технічно, одна реалізація може імплементувати обидва інтерфейси:

```python
class PostgresBookingRepository(BookingRepository, BookingReadRepository):
    ...
```

Це допустимо на старті. Але з часом write та read частини мають тенденцію рости в різних напрямках. Коли це станеться — розділити реалізацію буде просто, бо інтерфейси вже розділені.

### In-Memory реалізація для тестів

Одна з головних переваг Repository — тести домену працюють без БД:

```python
class InMemoryBookingRepository(BookingRepository):
    def __init__(self):
        self._bookings: dict[str, Booking] = {}

    def save(self, booking: Booking) -> None:
        self._bookings[booking.id] = booking

    def find_by_id(self, booking_id: str) -> Booking | None:
        return self._bookings.get(booking_id)

    def find_overlapping(self, resource_id: str, slot: TimeSlot) -> list[Booking]:
        return [
            b for b in self._bookings.values()
            if b.resource_id == resource_id and b.time_slot.overlaps(slot)
        ]
```

Тест створює `InMemoryBookingRepository`, передає його в handler — і вся доменна логіка тестується за мілісекунди без жодної залежності від інфраструктури.

---

## Поширені міфи

### «Repository — це обгортка над ORM»

Ні. ORM — це деталь реалізації, яка може жити всередині Repository. Але Repository — це доменний контракт. Його інтерфейс описується мовою домену (`save(booking)`, `find_overlapping(resource, slot)`), а не мовою даних (`insert`, `select`, `join`).

### «Кожна сутність потребує Repository»

Ні. Repository створюється для **агрегатів** — кореневих доменних об'єктів. Якщо `TimeSlot` — Value Object всередині `Booking`, йому не потрібен окремий репозиторій. Він зберігається і завантажується разом з `Booking`.

### «Generic Repository з CRUD-методами — це нормально»

`BaseRepository<T>` з `find_all()`, `find_by_id()`, `save()`, `delete()` — це DAO, а не Repository. Він не відображає потреби домену, не говорить мовою домену, і провокує додавати в нього все підряд. Доменний репозиторій має **специфічні методи** для конкретного агрегату.

### «Всю роботу з БД треба робити через Repository»

Тільки роботу з **доменними агрегатами**. Запити для UI, звіти, аналітика, пошук — це не Repository. Це Read Repository, Query Service, або прямий запит в БД з Query Handler'а.

### «Repository має повертати DTO для API»

Ні. Доменний Repository повертає доменні моделі. Маппінг у DTO — відповідальність Presentation або Application Layer.

---

## Джерела

- **Eric Evans** — *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) — оригінальне визначення Repository як тактичного патерну DDD (Chapter 6: «The Life Cycle of a Domain Object»)
- **Vaughn Vernon** — *Implementing Domain-Driven Design* (2013) — практичні приклади реалізації Repository для різних технологій
- **Martin Fowler** — *Patterns of Enterprise Application Architecture* (2002) — порівняння Repository та інших патернів доступу до даних (Repository, DAO, Table Data Gateway)
