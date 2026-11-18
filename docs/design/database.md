# Таблицы БД блока

У каждого блока своя база PostgreSQL (primary + 2 синхронные реплики под Patroni). Набор
таблиц во всех блоках и в standby-кластерах стендинов одинаковый. Общих таблиц между блоками
нет. Как устроена сама база — [`layers.md`](layers.md) §5 и
[`my-questions-answers-2.md`](my-questions-answers-2.md), ответ 13.

| Таблица | Что хранит | Кто пишет | Сколько живёт |
|---|---|---|---|
| `transactions` | Одна строка на каждое финансовое сообщение терминала со всеми его повторами | Роли Request, Response, Timeout Monitor | 3 дня, партиции по дням |
| `txn_events` | Журнал переходов состояния: источник для аудита | Те же, в той же транзакции БД, что и переход | 3 дня, затем только в аудите |
| `open_preauth` | Предавторизации старше 3 дней, по которым ещё может прийти вторая фаза | Задание обслуживания партиций | До срока второй фазы |
| `terminals` | Справочник терминалов блока: статус, разрешённые операции, профиль таймаутов | Загрузка из мастер-системы банка | Постоянно |
| `terminal_keys` | Сессионные ключи терминалов, зашифрованные мастер-ключом HSM (LMK) | Роль Request при смене ключей (`0800` Смена ключей, F70 = `101`) | Постоянно, последние 2 поколения |
| `block_takeover` | Когда и как это место стало владельцем логического блока | Failover Orchestrator, сразу после promote — когда копия блока в стендине стала рабочей базой | Постоянно |

Чего здесь нет: тела исходного запроса (в нём PIN-блок и дорожка карты — по PCI DSS их не
хранят), реестра соединений (шлюз берётся из `connection_id`), очереди задач (работа к
отправке — это строки `transactions` с истёкшим `deadline_at`).

---

## `transactions`

```sql
CREATE TABLE transactions (
  txn_id          uuid        NOT NULL,  -- наш внутренний id строки (UUIDv7). Наружу не уходит
  logical_block   text        NOT NULL,  -- 'b5'
  idem_key        text        NOT NULL,  -- TID · STAN (F11) · F7 первой передачи · класс MTI
  req_fingerprint bytea       NOT NULL,  -- SHA-256 от F3, F4, F49 и токена карты
  operation_id    varchar(32) NOT NULL,  -- 'b5.' || base32(SHA-256(idem_key))[:26]; id операции у процессинга
  stan            char(6)     NOT NULL,  -- F11
  terminal_id     char(8)     NOT NULL,  -- F41
  merchant_id     char(15)    NOT NULL,  -- F42
  op_type         text        NOT NULL,  -- PURCHASE / PREAUTH / COMPLETION / REVERSAL
  parent_txn_id   uuid,                  -- вторая фаза и реверс → исходная операция
  amount_minor    bigint      NOT NULL,  -- F4, в копейках
  currency        char(3)     NOT NULL,  -- F49
  pan_token       text        NOT NULL,  -- только токен карты, не номер
  processing_id   text,                  -- 'A' или 'B'
  proc_ref        text,                  -- id у процессинга из ответа ACCEPTED
  status          text        NOT NULL,  -- состояние, см. preprocessing.md §2.4
  proc_status     text,                  -- последний результат процессинга, даже если операция уже в реверсе
  version         int         NOT NULL DEFAULT 0, -- для перехода с проверкой версии
  deadline_at     timestamptz NOT NULL,  -- когда Timeout Monitor должен посмотреть на строку
  delivery        text,                  -- PENDING / DELIVERED / NO_CONNECTION
  connection_id   varchar(32),           -- соединение последнего запроса или повтора: куда отдать ответ
  response_iso    bytea,                 -- готовый ответ для повторов терминала
  trace_id        char(32),              -- trace id запроса, только для связи с логами
  msg_ts          timestamptz NOT NULL,  -- время из F7 (UTC); ключ партиции, одинаков у оригинала и повтора
  PRIMARY KEY (txn_id, msg_ts),
  UNIQUE (idem_key, msg_ts),             -- идемпотентность: повтор находит строку
  UNIQUE (operation_id, msg_ts)          -- результат процессинга находит строку
) PARTITION BY RANGE (msg_ts);

CREATE INDEX ON transactions (deadline_at)
  WHERE status IN ('RECEIVED','SUBMIT_UNKNOWN','ACCEPTED','AUTHORIZED','REVERSAL_PENDING');
```

**Партиции.** По дням, в работе три дня. Есть партиция `DEFAULT` для сообщений с датой вне
диапазона (сбившиеся часы терминала). Удаление старого дня — отключение партиции, а не
`DELETE`; условия удаления — [`my-questions-answers.md`](my-questions-answers.md), ответ 9.

**По каким ключам ищем**

| Кто ищет | По чему | Индекс |
|---|---|---|
| Transaction Receiver: это повтор? | `idem_key` из F41, F11, F7 и MTI | `UNIQUE (idem_key, msg_ts)` |
| Transaction Receiver: исходная для `0400` Реверс и `0220` Завершение | `idem_key`, собранный из F41 и F90 | Тот же |
| Result Receiver: результат процессинга | `operationId` из тела `result` | `UNIQUE (operation_id, msg_ts)` |
| Timeout Monitor: что просрочено | `deadline_at <= now()` | Частичный индекс по `deadline_at` |
| Поддержка, по логам | `trace_id` | Без индекса: разбор идёт в ClickHouse |

**Почему три идентификатора.** `txn_id` — внутренний первичный ключ: на него ссылаются
`parent_txn_id` и `txn_events`. `idem_key` — то, что можно собрать из сообщения терминала.
`operation_id` — то же самое в виде, пригодном для процессинга: фиксированной длины, без
номера терминала в открытом виде и с префиксом блока для Callback Router. Подробно —
[`my-questions-answers-2.md`](my-questions-answers-2.md), ответы 2 и 7.

**Как пишем.** Переход состояния — `UPDATE … SET status = :new, version = version + 1 WHERE
txn_id = :id AND version = :v`. Не обновилась ни одна строка — кто-то успел раньше,
перечитываем. Переходы `RECEIVED`, финальный статус и `REVERSAL_PENDING` пишутся с
`synchronous_commit = on`, служебные отметки (`ACCEPTED`, `DELIVERED`, смена `connection_id`) —
с `local`.

---

## `txn_events`

```sql
CREATE TABLE txn_events (
  txn_id      uuid        NOT NULL,
  seq         int         NOT NULL,  -- номер события внутри транзакции, без пропусков
  from_status text,
  to_status   text        NOT NULL,
  cause       text        NOT NULL,  -- что вызвало переход: RESULT, TIMEOUT, TERMINAL_0400, RECONCILIATION…
  trace_id    char(32),
  prev_hash   bytea       NOT NULL,  -- хеш предыдущего события той же транзакции: правка задним числом видна
  at          timestamptz NOT NULL,
  PRIMARY KEY (txn_id, seq)
) PARTITION BY RANGE (at);
```

Пишется в той же транзакции БД, что и `UPDATE transactions`. Отсюда Debezium читает журнал
БД и отправляет события в Kafka → ClickHouse и WORM-хранилище.

---

## `open_preauth`

```sql
CREATE TABLE open_preauth (LIKE transactions INCLUDING ALL);
```

Перед отключением партиции старше 3 дней сюда переносятся строки в `AUTHORIZED`: по ним ещё
может прийти `0220` Завершение. Строка живёт до срока второй фазы, затем Timeout Monitor
отправляет реверс (`PREAUTH_EXPIRED`).

---

## `terminals`

```sql
CREATE TABLE terminals (
  terminal_id     char(8)   PRIMARY KEY,  -- F41
  merchant_id     char(15)  NOT NULL,     -- F42
  mcc             char(4)   NOT NULL,     -- F18 подставляем отсюда
  merchant_name   text      NOT NULL,     -- F43 подставляем отсюда
  status          text      NOT NULL,     -- ACTIVE / BLOCKED
  allowed_ops     text[]    NOT NULL,     -- PURCHASE, PREAUTH, REFUND…
  timeout_profile text      NOT NULL,     -- профиль TMS: из него считаем T2 = срок терминала − 5 с
  updated_at      timestamptz NOT NULL
);
```

Предлагаю: источник — мастер-система банка, загрузка изменений раз в минуту. Роль Request
держит справочник в памяти и перечитывает изменения по `updated_at`. В каждом блоке лежат все
терминалы, а не только свои: справочник небольшой (100 000 строк), а после смены кольца
терминал окажется в другом блоке без переноса данных.

---

## `terminal_keys`

```sql
CREATE TABLE terminal_keys (
  terminal_id  char(8)  NOT NULL,
  generation   int      NOT NULL,  -- текущее и предыдущее поколение: ответ на старом ключе ещё проверяется
  key_type     text     NOT NULL,  -- MAC / PIN
  key_under_lmk bytea   NOT NULL,  -- ключ, зашифрованный мастер-ключом HSM; в открытом виде его нет нигде
  kcv          char(6)  NOT NULL,  -- контрольное значение ключа
  created_at   timestamptz NOT NULL,
  PRIMARY KEY (terminal_id, generation, key_type)
);
```

Новые ключи выпускает HSM при `0800` Смена ключей (F70 = `101`), примерно раз в сутки на
терминал.

---

## `block_takeover`

```sql
CREATE TABLE block_takeover (
  epoch          bigint      PRIMARY KEY, -- revision etcd той записи карты, которая сделала это место владельцем
  logical_block  text        NOT NULL,    -- 'b5'
  mode           text        NOT NULL,    -- PLANNED / EMERGENCY
  taken_at       timestamptz NOT NULL,    -- момент promote: копия стала рабочей базой и начала принимать запись
  replayed_up_to timestamptz,             -- EMERGENCY: время последней транзакции, дошедшей из старого primary
  reconciled_at  timestamptz              -- EMERGENCY: когда сверка закончена; до этого действует «окно потерь»
);
```

Это единственный способ, которым слой управления сообщает что-то приложениям блока: через
их собственную базу, а не через etcd. Запись делает Failover Orchestrator сразу после
promote (перевода копии в рабочую базу). Transaction Receiver читает последнюю строку при старте и раз в 5 с. Если последняя
строка `EMERGENCY` и `reconciled_at` пуст, включается проверка «окна потерь» (ответ 5 в
[`my-questions-answers-2.md`](my-questions-answers-2.md)).

---

## Что лежит не в БД блока

| Данные | Где |
|---|---|
| Кольцо, карта блоков, правила A / B, история переводов терминалов | etcd маршрутизации (5 узлов) |
| Кто сейчас primary в кластере БД блока | etcd блока (DCS Patroni, 3 узла) |
| История операций старше 3 дней | ClickHouse и WORM-хранилище аудита |
| Запрет на `operationId`, по которому был реверс без операции | У процессинга |
| LMK, ключи в открытом виде | Только внутри HSM |
