# CQS та CQRS

## Зміст

- [Вступ](#вступ)
- [CQS: принцип розділення](#cqs-принцип-розділення)
- [Command та Command Handler](#command-та-command-handler)
- [Query та Query Handler](#query-та-query-handler)
- [Навіщо Handler замість Service](#навіщо-handler-замість-service)
- [Тестування команд і запитів](#тестування-команд-і-запитів)
- [Від CQS до CQRS](#від-cqs-до-cqrs)
- [CQRS: різні моделі для читання і запису](#cqrs-різні-моделі-для-читання-і-запису)
- [Коли CQRS виправданий](#коли-cqrs-виправданий)
- [Поширені міфи](#поширені-міфи)
- [Джерела](#джерела)

---

## Вступ

У вас є `BookingService` з десятком методів: `create`, `cancel`, `confirm`, `get_by_id`, `get_available_slots`, `get_user_bookings`, `search`... Частина методів змінює стан, частина — тільки читає. Але всі живуть в одному класі, залежать від одних і тих самих репозиторіїв, і тестуються однаково — через інтеграційні тести з БД.

А тепер з'являється вимога: читання має бути швидким (кешування, денормалізовані view), а запис — надійним (транзакції, валідація, доменна логіка). Оптимізувати одне заважає інше — бо вони склеєні в одному сервісі, з одними моделями, одними репозиторіями.

CQS і CQRS — це про те, як розклеїти.

---

## CQS: принцип розділення

CQS (Command-Query Separation) — принцип, сформульований Бертраном Меєром:

> Кожен метод має бути або **командою** (змінює стан, нічого не повертає), або **запитом** (повертає дані, не змінює стан), але не обома одночасно.

Це принцип на рівні **методів і класів**. Він не вимагає різних БД чи моделей — тільки чіткого розділення: метод або змінює щось, або повертає щось.

### Навіщо?

Якщо метод і змінює стан, і повертає дані — стає незрозуміло: він викликається заради ефекту чи заради результату? Чи безпечно його викликати повторно? Чи можна його кешувати?

```python
# Порушення CQS — що робить цей метод?
def create_and_return_booking(self, data) -> Booking:
    booking = Booking(...)
    self.repo.save(booking)      # побічний ефект
    return booking               # і повертає дані
```

Після розділення:
- **Command**: `create_booking(data) -> booking_id` — створює, повертає лише ID
- **Query**: `get_booking(id) -> BookingDTO` — тільки читає, ніколи не змінює

Виклик Query — завжди безпечний. Можна кешувати, повторювати, оптимізувати. Command — завжди усвідомлена дія зі зміною стану.

---

## Command та Command Handler

Command — [DTO](dto.md), що описує **намір виконати дію**, яка змінює стан системи. Кожна команда обробляється одним `CommandHandler`.

```python
@dataclass(frozen=True)
class CreateBookingCommand:
    user_id: str
    resource_id: str
    start_time: datetime
    end_time: datetime

class CreateBookingCommandHandler:
    def __init__(self, factory: BookingFactory, repo: BookingRepository):
        self._factory = factory
        self._repo = repo

    def handle(self, command: CreateBookingCommand) -> str:
        booking = self._factory.create(
            command.user_id, command.resource_id,
            command.start_time, command.end_time,
        )
        self._repo.save(booking)
        return booking.id
```

Характеристики:
- Command — простий об'єкт з даними, **без логіки**
- Handler проходить через доменний шар, використовує [Domain Factory](domain-factory.md) та [Repository](repository.md)
- Command **не повертає дані** (допустимий виняток — ID створеної сутності)
- Один Command = один Handler = одна бізнес-операція

---

## Query та Query Handler

Query — DTO, що описує, **які дані потрібно отримати**. Не змінює стан.

```python
@dataclass(frozen=True)
class GetAvailableSlotsQuery:
    resource_id: str
    date: date

class GetAvailableSlotsQueryHandler:
    def __init__(self, read_repo: BookingReadRepository):
        self._read_repo = read_repo

    def handle(self, query: GetAvailableSlotsQuery) -> list[TimeSlotDTO]:
        return self._read_repo.find_available_slots(
            query.resource_id, query.date
        )
```

Характеристики:
- Query містить параметри пошуку/фільтрації
- Handler може читати з БД **напряму**, минаючи доменний шар — йому не потрібна бізнес-логіка
- Повертає [DTO](dto.md) / Read Models, оптимізовані під потреби клієнта — не доменні моделі
- **Не мутує стан** — ніколи

### Read Models

Query Handler повертає не доменні моделі, а Read Models — [DTO](dto.md), спроєктовані під конкретний запит:

| Тип моделі | Призначення |
|------------|-------------|
| Доменна модель | Бізнес-логіка, інваріанти, поведінка |
| Read Model | Дані для відображення, оптимізовані під конкретний UI/API |

Read Model може містити денормалізовані дані з кількох [сутностей](entities-and-aggregates.md) — наприклад, ім'я користувача поруч з бронюванням, щоб клієнту не потрібно робити додаткові запити.

---

## Навіщо Handler замість Service

Замість одного великого сервісу з десятком методів — кожна операція отримує свій Handler:

```
BookingService.create()          →  CreateBookingCommandHandler
BookingService.cancel()          →  CancelBookingCommandHandler
BookingService.get_available()   →  GetAvailableSlotsQueryHandler
BookingService.get_by_id()       →  GetBookingByIdQueryHandler
```

### Проблема з Service

```python
class BookingService:
    def __init__(self, booking_repo, resource_repo, user_repo,
                 notification_service, analytics_service, logger):
        # 6 залежностей — і це ще не все
        ...

    def create(self, ...): ...
    def cancel(self, ...): ...
    def confirm(self, ...): ...
    def get_by_id(self, ...): ...
    def get_available_slots(self, ...): ...
    def get_user_bookings(self, ...): ...
    def search(self, ...): ...
```

Сервіс на 7 методів з 6 залежностями. `get_available_slots` не потребує `notification_service`, але залежить від нього через конструктор. Кожна зміна в одному методі — ризик зачепити інший. Тести потребують мокання всіх залежностей, навіть непотрібних. Це порушення і [SRP](solid.md#single-responsibility-principle-srp), і [ISP](solid.md#interface-segregation-principle-isp).

### Рішення: один Handler = одна операція

```python
class CreateBookingCommandHandler:
    def __init__(self, factory: BookingFactory, repo: BookingRepository):
        # лише потрібні залежності
        ...

class GetAvailableSlotsQueryHandler:
    def __init__(self, read_repo: BookingReadRepository):
        # одна залежність
        ...
```

Переваги:
- Кожен Handler — маленький клас з однією відповідальністю
- Мінімум залежностей — лише те, що потрібно для конкретної операції
- Додати нову операцію = створити новий Handler, не чіпаючи існуючі ([OCP](solid.md#openclosed-principle-ocp))
- Простіше тестувати — менше моків, фокусований тест

---

## Тестування команд і запитів

Команди та запити тестуються **по-різному**, бо роблять різне.

### Команди: given-when-then (поведінкові)

```python
def test_cancel_booking():
    # Given: існує активне бронювання
    repo = InMemoryBookingRepository()
    booking = create_test_booking(status=BookingStatus.CONFIRMED)
    repo.save(booking)
    handler = CancelBookingCommandHandler(repo)

    # When: скасовуємо
    handler.handle(CancelBookingCommand(booking_id=booking.id))

    # Then: статус змінився
    updated = repo.find_by_id(booking.id)
    assert updated.status == BookingStatus.CANCELLED
```

Тестуємо **побічний ефект**: що змінилося в системі після виконання команди?

### Запити: fixture → query → assert

```python
def test_get_available_slots():
    # Fixture: є бронювання на 10:00-11:00
    read_repo = InMemoryBookingReadRepository()
    read_repo.add_booking(resource_id="room-1", start=10, end=11)
    handler = GetAvailableSlotsQueryHandler(read_repo)

    # Query
    result = handler.handle(GetAvailableSlotsQuery(
        resource_id="room-1", date=date.today()
    ))

    # Assert: слот 10:00-11:00 не в результатах
    assert not any(s.start_hour == 10 for s in result)
```

Тестуємо **результат**: чи повернув запит правильні дані?

---

## Від CQS до CQRS

CQS — це **принцип**: розділи методи на команди і запити.

CQRS (Command-Query Responsibility Segregation) — це **архітектурний патерн**: розділи не тільки методи, а й **моделі**, і потенційно **сховища** для читання і запису.

```
CQS:   один Repository, розділені методи (Command/Query Handlers)
CQRS:  різні моделі, можливо різні БД для write і read
```

### Еволюція

```
Етап 1 (CQS):
  Command Handler → BookingRepository (write)
  Query Handler   → BookingReadRepository (read)
  Одна БД, різні інтерфейси

Етап 2 (CQRS з проєкціями):
  Command Handler → BookingRepository → Write DB (normalized)
  Events → Projection Builder → Read DB (denormalized views)
  Query Handler → Read DB

Етап 3 (CQRS з різними сховищами):
  Command Handler → PostgreSQL (write)
  Events → Projection → Elasticsearch (read/search)
  Query Handler → Elasticsearch
```

Кожен етап — не обов'язковий. Більшості систем достатньо етапу 1 (CQS).

---

## CQRS: різні моделі для читання і запису

### Проблема, яку вирішує CQRS

В CQS Command і Query Handler'и працюють з **однією БД**. Це достатньо для більшості випадків. Але є ситуації, коли виникає конфлікт:

- **Нормалізована** схема зручна для запису (немає дублювання, легко підтримувати інваріанти), але **повільна для складних читань** (JOIN через 5 таблиць для одного списку)
- **Денормалізована** схема зручна для читання (один запит — усі дані), але **складна для запису** (потрібно оновлювати кілька місць при кожній зміні)

### Рішення: окремі моделі

Write Model — нормалізована, оптимізована під бізнес-операції та інваріанти:

```
bookings (id, user_id, resource_id, start_time, end_time, status)
users (id, name, email)
resources (id, name, type)
```

Read Model — денормалізована, оптимізована під конкретні запити:

```
booking_list_view (booking_id, user_name, resource_name, start_time, end_time, status)
available_slots_view (resource_id, date, slot_start, slot_end)
```

Коли Command змінює дані у write-моделі, **проєкція** (projection) оновлює read-модель. Зазвичай це відбувається через [доменні події](domain-events.md):

```
Command → Write DB → Domain Event → Projection → Read DB
                                                     ↑
Query Handler reads from ─────────────────────────────┘
```

### Eventual Consistency

Між записом у Write DB і оновленням Read DB є затримка. Дані в read-моделі **тимчасово неконсистентні** — це [eventual consistency](domain-events.md#eventual-consistency). Для більшості read-сценаріїв (списки, пошук, аналітика) це прийнятно.

---

## Коли CQRS виправданий

**CQS достатньо**, коли:
- Читання і запис працюють з однією БД без проблем з продуктивністю
- Read-запити не потребують складних JOIN або денормалізації
- Система невелика, команда не хоче додаткової складності

**CQRS виправданий**, коли:
- Патерни читання і запису **радикально різні** (наприклад, пишемо нормалізовано, читаємо через складні проєкції)
- Потрібна **масштабована читання** (read-репліки, кеші, пошукові індекси)
- Запис — рідкісний, читання — масове (типово для аналітики, каталогів, дашбордів)
- Система вже використовує [доменні події](domain-events.md), і побудова проєкцій додається природно

**CQRS не потрібен**, якщо:
- Це простий CRUD
- Read і write використовують одні й ті самі дані в одному форматі
- Вас влаштовує один `SELECT ... JOIN` для читання

---

## Поширені міфи

### «CQS і CQRS — це одне й те саме»

Ні. CQS — **принцип** на рівні методів: метод або змінює стан, або повертає дані. CQRS — **архітектурний патерн**: різні моделі (і можливо сховища) для запису та читання. CQS можна впровадити за годину. CQRS може зайняти тижні.

### «CQRS вимагає Event Sourcing»

Ні. CQRS і Event Sourcing — ортогональні патерни. CQRS можна робити без подій (проєкції оновлюються синхронно). Event Sourcing можна робити без CQRS. Вони часто згадуються разом, бо добре поєднуються — але не залежать одне від одного.

### «CQRS вимагає дві різні бази даних»

Ні. CQRS вимагає різних **моделей** для читання і запису. Вони можуть жити в одній БД (різні таблиці: нормалізовані для write, VIEW або денормалізовані таблиці для read). Різні БД — оптимізація, а не вимога.

### «Command не може повертати нічого»

Суворе прочитання CQS — так. На практиці повернення ID створеного об'єкта — загальноприйнятний компроміс. Клієнту потрібен ID, щоб зробити наступний запит. Заставляти його вгадувати або окремо запитувати — зайве ускладнення.

### «Query Handler може викликати доменну логіку»

Може, але не повинен. Query Handler читає дані — і все. Якщо для читання потрібна «логіка», зазвичай це означає, що read-модель недостатньо підготовлена. Логіка — справа команд і домену, не запитів.

### «Потрібно починати з CQRS»

Ні. Починайте з CQS — розділення Command/Query Handler'ів на рівні Application Layer. Це дешево і дає більшість переваг. CQRS з'явиться як природна еволюція, коли read і write почнуть конфліктувати.

---

## Джерела

- **Bertrand Meyer** — *Object-Oriented Software Construction* (1988, 2nd ed. 1997) — оригінальне формулювання принципу CQS
- **Greg Young** — [CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) — ключовий документ, що описує патерн CQRS
- **Martin Fowler** — [CQRS](https://martinfowler.com/bliki/CQRS.html) — стаття з поясненням, коли CQRS виправданий, а коли — зайвий
- **Robert C. Martin** — *Clean Architecture* (2017) — CQS як частина розділення відповідальностей на рівні use cases
