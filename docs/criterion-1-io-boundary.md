# Критерий №1 — отделение I/O от бизнес-логики

> Деталь концепции. Обзор и место критерия — в [`CONCEPT.md`](../CONCEPT.md).

Три последовательных прохода. Каждый следующий дороже, но точнее.

## Проход A — статический, по AST и графу импортов (без ИИ)

Решает 70–80% случаев бесплатно.

### Каталог I/O-маркеров по языкам

**Go**
- Сеть/HTTP: `net/http`, `net`, `google.golang.org/grpc`, `google.golang.org/protobuf`
- БД: `database/sql`, `gorm.io/*`, `*mongo*`, `github.com/jackc/pgx`, `github.com/jmoiron/sqlx`
- Кэш/брокеры: `github.com/redis/*`, `github.com/segmentio/kafka-go`, `github.com/streadway/amqp`, `github.com/nats-io/*`
- Файловая система: `os`, `io`, `bufio`, `path/filepath`
- Недетерминизм: `time.Now`, `time.Since`, `math/rand`, `crypto/rand`
- Сырые типы в сигнатурах: `*sql.DB`, `*http.Client`, `*http.Request`, `*os.File`, `io.Reader`/`io.Writer` как параметр доменной функции

**Java**
- Сеть/HTTP: `java.net.*`, `java.net.http.*`, `org.springframework.web.*`, `okhttp3.*`, `org.apache.http.*`, `feign.*`
- БД: `java.sql.*`, `javax.sql.*`, `jakarta.persistence.*`, `org.hibernate.*`, `org.springframework.data.*`, `org.jdbi.*`
- Кэш/брокеры: `redis.clients.jedis.*`, `io.lettuce.*`, `org.apache.kafka.*`, `com.rabbitmq.*`, `org.springframework.amqp.*`
- ФС: `java.io.File`, `java.nio.file.*`, `java.io.InputStream`/`OutputStream` как параметр доменного метода
- Недетерминизм: `java.time.Clock` (если не инжектится), `java.time.LocalDateTime.now()`, `java.util.Random`, `Math.random`, `UUID.randomUUID` внутри логики
- Сырые типы в сигнатурах: `javax.sql.DataSource`, `EntityManager`, `JdbcTemplate`, `RestTemplate`, `WebClient`
- Аннотационные сигналы нарушения: `@RestController`, `@Repository`, `@Service` со встроенной БД-логикой; `@Autowired` сырого DataSource в доменный сервис

**Python**
- Сеть/HTTP: `requests`, `httpx`, `aiohttp`, `urllib`, `urllib3`, `grpc`
- БД: `psycopg`, `psycopg2`, `sqlalchemy`, `asyncpg`, `pymongo`, `mysqlclient`, `pymysql`, `sqlite3`
- Кэш/брокеры: `redis`, `aioredis`, `kafka`, `confluent_kafka`, `pika`, `aio_pika`, `nats`
- ФС: `open()`, `pathlib.Path.read_*`/`write_*`, `os.*`, `shutil.*`, `io.*`
- Недетерминизм: `datetime.now`, `datetime.utcnow`, `time.time`, `random.*`, `secrets.*`, `uuid.uuid4` внутри логики
- Сырые типы в сигнатурах: `Connection`, `Engine`, `Session`, `httpx.Client`, `requests.Session`

**TypeScript**
- Сеть/HTTP: `fetch`, `axios`, `node-fetch`, `got`, `undici`, `@nestjs/axios`
- БД: `pg`, `mysql2`, `mongodb`, `mongoose`, `typeorm`, `prisma`, `knex`, `drizzle-orm`
- Кэш/брокеры: `ioredis`, `redis`, `kafkajs`, `amqplib`, `nats`, `bullmq`
- ФС: `fs`, `node:fs`, `node:fs/promises`, `node:path`
- Недетерминизм: `Date.now`, `new Date()`, `Math.random`, `crypto.randomUUID` внутри логики
- Сырые типы в сигнатурах: `Pool`, `Client` (из `pg`), `PrismaClient`, `AxiosInstance`, `MongoClient`
- DOM/браузерные: `window`, `document`, `localStorage`, `fetch` напрямую в доменных функциях

### Правила-блокеры (общие для всех языков)

1. Модуль, классифицированный как «логика», **не должен** иметь транзитивных импортов из I/O-каталога.
2. В контрактах модулей логики **запрещены**:
   - сырые типы клиентов (`*sql.DB`, `DataSource`, `Session`, `Pool`, …);
   - больше одной data-структуры на входе (правило «один data-аргумент на модуль»);
   - возврат через исключение/panic вместо `Result<T, E>` (`(T, error)` в Go, `Either`/`Result` в TS, `Result`/sealed-class в Java, кортеж/`Result`-like в Python).
3. Скрытое состояние: глобальные переменные, синглтоны, `package init`, `static` с состоянием, мутабельные модульные переменные.
4. Юнит-тесты модулей логики, использующие mock-фреймворки (`gomock`, `Mockito`, `unittest.mock`, `jest.mock` с заменой зависимостей), — сигнал, что зависимости втащены внутрь логики. По автору моков быть не должно.

### Каталог разрешённых импортов в логике

Чтобы статика не давала ложноположительных, ведётся явный список «чистых» библиотек по языкам:

- Go: `errors`, `strings`, `strconv`, `sort`, `math`, `fmt` без `Print*`, доменные пакеты проекта.
- Java: `java.lang.*` (кроме `System.out`/`err`), `java.util.*` (immutable-коллекции и `Optional`), `java.math.*`, доменные пакеты.
- Python: `dataclasses`, `typing`, `enum`, `functools`, `itertools`, `decimal`, `re`, доменные пакеты.
- TypeScript: чистые утилиты (`zod` для валидации, `neverthrow` для `Result`, lodash-fp), доменные модули.

Выход прохода A: классификация модулей + список «жёстких» нарушений с `file:line`.

## Проход B — семантический, через LLM

Только для модулей, которые статика не отнесла однозначно, или для модулей со скрытой зависимостью (интерфейс с неизвестной реализацией).

Промпт-чеклист с обязательной цитатой строки:

```
Дано: код модуля + его публичный контракт.
Ответь Y/N с цитатой строки:
1. Функция модуля формулируется одной фразой? Если да — какой?
2. Контракт: ровно одна data-структура на входе?
3. Возврат — Result<T,E> или эквивалент, без исключений вверх?
4. Внутри модуля есть: сеть, БД, ФС, брокер, время, случайность,
   глобальное состояние? Перечисли вхождения.
5. Все зависимости — автономные I/O-объекты (Store/Client/Publisher),
   а не сырые клиенты?
6. Модуль возвращает управление вызвавшему (нет «прыжков в сторону»)?
```

LLM не имеет права судить «по ощущению» — каждый ответ привязан к цитате. Несоответствие хотя бы одному пункту = нарушение.

## Проход C — сверка с проектной документацией

Если в репозитории есть `docs/contracts-graph.md`, `messages.md`, `intent.md`:

- LLM строит фактический граф вызовов из кода и сверяет с задекларированным.
- Расхождения — отдельный класс нарушений: «код разошёлся со спецификацией».
- По дисциплине автора это означает «остановиться и сообщить» — RRA именно это и делает.

## Итоговая оценка по критерию №1

Структурированный результат, не одна цифра:

- **Жёсткие нарушения** (`*sql.DB` в сигнатуре доменной функции, `@Repository` в slice-логике, `Date.now()` внутри чистой функции) — блокирующие, репозиторий не проходит.
- **Мягкие** — список с локациями.
- **Покрытие графом** — какая доля модулей классифицирована однозначно.
