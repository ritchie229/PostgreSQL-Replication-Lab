# PostgreSQL Practice Lab

## 1. Intro

> Hereby in purposes of practical training for DBA we are aiming on deploying docker <br>
> containers running PostgreSQL, two or three containers, one master, the others replicas,<br>
> realizing thus logical and physical constant replication, will break them, and fix again, <br>
> use vacuum and auto-vacuum, reindexation etc. <br>

Streaming Replication (потоковая репликация) Это основной встроенный механизм репликации в PostgreSQL. Это репликация на уровне файлов/WAL, а не данных.<br>
- Главный сервер (Primary / Master) отправляет WAL-логи (журналы транзакций) к одному или нескольким репликам (Standby / Replica).
- Реплика принимает WAL и применяет их, оставаясь синхронизированной.
<br>Режимы:
- Asynchronous (асинхронная) – реплика немного отстает, но не блокирует запись на Primary.
- Synchronous (синхронная) – транзакция на Primary считается завершённой только после записи на реплике.


Logical Replication (логическая репликация) Работает на уровне таблиц и данных, а не в виде потоков WAL-логов.
- Primary публикует данные (publication).
- Replica подписывается на них (subscription).
<br>Возможности:
- Реплицировать только выбранные таблицы
- Репликация между разными версиями PostgreSQL
- Можно трансформировать данные в процессе
- Подходит для миграций/частичных реплик
<br>Ограничения:
- Настройка сложнее, чем у потоковой
- Не реплицируются DDL (изменения схемы) автоматически

Logical Decoding / Replication Slots - Это механизм, который позволяет «декодировать» WAL-логи для логической репликации.
Используется для
- CDC (Change Data Capture)
- Репликации в сторонние хранилища (Kafka, Elasticsearch, аналитика)
- Пользовательских обработчиков изменений
<br>Особенности
- Можно получить поток изменений
- Требует дополнительной обработки или инструментов

pglogical (расширение) - Это расширение высокого уровня, основанное на логической репликации.
<br>Преимущества
- Гибкая конфигурация
- Поддержка фильтрации данных
- Репликация между разными версиями PostgreSQL
<br>Когда использовать
- Когда нужна избирательная репликация
- При миграциях между кластерами
- Для комплексной топологии репликации

Slony-I - Старое, надёжное решение для репликации уровня SQL.
<br>Особенности
- Поддерживает сложные сценарии
- Длительное время разработки и эксплуатации
<br>Минусы
- Сложнее в настройке
- Устаревает по сравнению с нативной логической репликацией

BDR (Bi-Directional Replication) - Расширение для двунаправленной репликации между несколькими узлами.
<br>Применяется когда
- Нужна активная репликация в несколько направлений
- Интеграция нескольких мастер-узлов
<br>Сложность
- Требует специальной настройки
- Меньше стандартной поддержки

Пул реплик (Proxy / Middleware) - Не репликация как таковая, но важное средство:
- Pgpool-II
- PgBouncer
<br>Используются для:
- Балансировки нагрузки
- Управления пулами соединений
- Переключения между мастер/репликами
```
| Механизм                  | Потоковая | Логическая | Таблицы | Репликация между версиями | DDL      |
| ------------------------- | --------- | ---------- | ------- | ------------------------- | -------- |
| **Streaming Replication** | ✔         | ✖          | ✖       | ✖                         | ✖        |
| **Logical Replication**   | ✖         | ✔          | ✔       | ✔                         | ✖        |
| **pglogical**             | ✖         | ✔          | ✔       | ✔                         | ✖        |
| **Slony-I**               | ✖         | ✔          | ✔       | ✔                         | Частично |
| **BDR**                   | ✖         | ✔          | ✔       | ✔                         | Частично |

```

### ОБЩИЕ ИНСТРКУЦИИ

**Физическая репликация**
> Реплицирует WAL на уровне файлов. Полная копия кластера. Read-only. Используется для HA и DR.
<br>
**Логическая репликация**
> Реплицирует изменения таблиц. Можно выбирать таблицы. Можно писать. Используется для миграций и интеграций.
<br>
1. Physical Replication (Streaming)
Схема
```text
Primary (master)  --->  Replica (standby)
   5432               5432
```

Допустим:
```text
Master	10.0.0.1
Replica	10.0.0.2
```
Шаг 1. Создаём пользователя для репликации (на master)
```sql
CREATE ROLE replicator
WITH REPLICATION LOGIN PASSWORD 'secret123';
```
Шаг 2. Настраиваем postgresql.conf (на master)
```bash
vim /etc/postgresql/15/main/postgresql.conf
```

Меняем:
```ini
listen_addresses = '*'

wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB
```
Шаг 3. Разрешаем доступ в pg_hba.conf

```bash
vim /etc/postgresql/15/main/pg_hba.conf
```

Добавляем:
```ini
host  replication  replicator  10.0.0.2/32  md5
```
Шаг 4. Перезапускаем master
```bash
sudo systemctl restart postgresql
```
Шаг 5. Клонируем базу на replica
На реплике PostgreSQL должен быть остановлен
```bash
sudo systemctl stop postgresql
```

Удаляем старые данные:
```bash
rm -rf /var/lib/postgresql/15/main/*
```

Делаем basebackup:
```bash
pg_basebackup \
 -h 10.0.0.1 \
 -U replicator \
 -D /var/lib/postgresql/15/main \
 -Fp -Xs -P -R
```
Ключ -R создаст standby.signal.

Шаг 6. Запускаем replica

```bash
sudo systemctl start postgresql
```
Шаг 7. Проверяем

На master:
```sql
SELECT client_addr, state, sync_state
FROM pg_stat_replication;
```

Должна быть строка с репликой.

На replica:
```
SELECT pg_is_in_recovery();
```

Должно быть:
`t`

2. Failover вручную

Если master умер — делаем promotion:

На реплике:
```bash
pg_ctl promote
```

Или:
```sql
SELECT pg_promote();
```
Теперь это новый master.


3. Логическая репликация — практика

Схема
```text
DB1 (publisher) --> DB2 (subscriber)
```

Шаг 1. Включаем logical WAL

На publisher:
```ini
wal_level = logical
```
Перезапуск.

Шаг 2. Создаём публикацию
```sql
CREATE PUBLICATION mypub
FOR TABLE users, orders;
```

Или на всё:
```sql
CREATE PUBLICATION mypub FOR ALL TABLES;
```

Шаг 3. Создаём подписку (на subscriber)
```sql
CREATE SUBSCRIPTION mysub
CONNECTION 'host=10.0.0.1 dbname=app user=repl password=123'
PUBLICATION mypub;
```

Шаг 4. Проверяем
```sql
SELECT * FROM pg_stat_subscription;
```
------------------



## 2. Deploying Master DB Container

### Create Lab Directory

```bash
mkdir pg-lab
cd pg-lab
```

### Create first docker compose file

```bash
vim docker-compose.yml
```

```yaml
version: "3.9"

services:
  pg-master:
    image: postgres:15
    container_name: pg-master
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: labdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - pgnet

volumes:
  pgdata:

networks:
  pgnet:
```

### Run the container

```bash
docker compose up -d
docker ps
```

### Enter the container

```bash
docker exec -it pg-master bash
```

### Enter Postgres

```bash
psql -U admin -d labdb
```

### See the configs

```sql
SHOW data_directory;
SHOW config_file;
SHOW hba_file;
```
Remember the paths <br>

```sql
\q
```

### Investigate the data directory

```bash
cd /var/lib/postgresql/data
ls -lah
```

Result:<br>

```pgsql
base/
pg_wal/
postgresql.conf
pg_hba.conf
pg_stat/
```
This is the heart of PostgreSQL

### Annotations

📁 `/var/lib/postgresql/data`	Это PGDATA — сердце PostgreSQL.<br>

Самое важное:<br>

🔹 base/		→ Физические файлы таблиц по БД<br>
🔹 pg_wal/		→ WAL-журнал (основа репликации и recovery)<br>
🔹 pg_replslot/		→ replication slots (понадобится позже)<br>
🔹 postgresql.conf	→ основной конфиг<br>
🔹 pg_hba.conf		→ доступы<br>
🔹 postmaster.pid	→ значит, сервер сейчас запущен


## 3. Playaround with WAL

Будем: <br>
👉 смотреть WAL <br>
👉 управлять checkpoint <br>
👉 делать нагрузку <br>
👉 смотреть, как растёт pg_wal <br>

### See the currect WAL configurations
Enter Postgres <br>
```bash
psql -U admin -d labdb
```
Check configs <br>
```sql
SHOW wal_level;			- replica
SHOW checkpoint_timeout;	- 5min
SHOW max_wal_size;		- 1GB
SHOW min_wal_size;		- 80MB
SHOW archive_mode;		- off
SHOW archive_command;		- disabled
```
Check WAL size: <br>
```bash
du -sh /var/lib/postgresql/data/pg_wal	- 16M
```

### Create Load

```sql
CREATE TABLE test_wal (
  id SERIAL PRIMARY KEY,
  data TEXT
);
```
```sql
INSERT INTO test_wal (data)
SELECT md5(random()::text)
FROM generate_series(1, 500000);
```

### Check WAL size and Checkpoint Statistics

```bash
du -sh /var/lib/postgresql/data/pg_wal
```

```sql
SELECT * FROM pg_stat_bgwriter;
```
```sql
SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint FROM pg_stat_bgwriter;
```
Output: <br>
```
checkpoints_timed 	= 3	→ Чекпоинты по таймеру (5 минут)
checkpoints_req		= 1	→ Принудительный checkpoint (из-за WAL size, WAL вырос → дошёл до max_wal_size → Postgres форснул checkpoint.)
buffers_checkpoint	= 1008	→ Сколько страниц сброшено на диск
```


### Comparing the results

```
wal_level = replica        ✅ для physical replication
checkpoint_timeout = 5min  ✅ стандарт
max_wal_size = 1GB         ✅ безопасно
min_wal_size = 80MB
archive_mode = off         ❌ нет PITR
```
> До INSERT: 	pg_wal = 16M <br>
> практически пусто — только служебные сегменты <br>
> После INSERT: 	pg_wal = 97M <br>
> +80 MB WAL от одной операции. <br>



## 4. Breaking WAL (test)

✔ уменьшим max_wal_size <br>
✔ увеличим нагрузку <br>
✔ увидим постоянные checkpoint <br>
✔ поймём, почему БД тормозит <br>

### posrtgresql.conf

```bash
vim /var/lib/postgresql/data/postgresql.conf

```
Put:
```conf
max_wal_size = 64MB
checkpoint_timeout = 30s
```

### Restart Postgres and Check

```bash
docker restart pg-master
```
or
```bash
pg_ctl restart -D /var/lib/postgresql/data
```

```sql
SHOW max_wal_size;		- 64MB
SHOW checkpoint_timeout;	- 30s
```

### Load again and See Results

```sql
INSERT INTO test_wal (data)
SELECT md5(random()::text)
FROM generate_series(1, 300000);
```

```sql
SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint
FROM pg_stat_bgwriter;
```
Output:
```
checkpoints_timed	= 12
checkpoints_req		= 3
buffers_checkpoint	= 7996
```

```bash
du -sh /var/lib/postgresql/data/pg_wal 	→ 97M
```

### Comparing 

> max_wal_size = 64MB      ❌ очень мало <br>
> checkpoint_timeout = 30s ❌ очень часто <br>

Было:
```ini
checkpoints_timed = 3
checkpoints_req   = 1
buffers = 1008
```
Стало:
```ini
checkpoints_timed = 12		За короткое время — 12 чекпоинтов(причина checkpoint_timeout = 30s).
checkpoints_req   = 3		3 раза WAL переполнялся (причина max_wal_size = 64MB).
buffers = 7996   		Каждый checkpoint писал ~60MB на диск.
```

> <br>
> Checkpoint — это момент, когда PostgreSQL говорит: 🧠 «Так, всё, что я держу в памяти — срочно сохранить на диск».<br>

Во время работы Postgres:
- данные сначала меняются в RAM (shared_buffers)
- на диск они пишутся не сразу
- чтобы было быстро
Checkpoint = массовая запись на диск.
```
checkpoints_timed = 3		- Это значит: ✔️ 3 раза checkpoint сработал по таймеру.
checkpoint_timeout = 5min	- Каждые 5 минут → checkpoint.
```
Работаешь с БД 	→ прошло ~15 минут → checkpoints_timed = 3.

> <br>
> max_wal_size — это не “жёсткий лимит”. Это триггер для checkpoint. 

------------

# Physical Replication

##  Master DB Prep

### Bring all back

On host machine <br>
```bash
vim /var/lib/docker/volumes/pg-lab_pgdata/_data/postgresql.conf
```
or On Container <br>
```bash
vim /var/lib/postgresql/data/postgresql.conf
```

Put: <br>
```conf
wal_level = replica
max_wal_size = 1GB
checkpoint_timeout = 5min
max_wal_senders = 5
max_replication_slots = 5
```

### Create user for replica

```sql
CREATE ROLE replicator
WITH REPLICATION LOGIN PASSWORD 'repl123';
```
### Providing Access to replicator

in `pg_hba.conf` put: <br>
```conf
host replication replicator 0.0.0.0/0 md5
```

### Restart pg-master

```bash
docker restart pg-master
```

```
Postgres (master)
  ├─ пишет WAL
  ├─ отдаёт WAL
  └─ ждёт standby
```
Time to create stanby.<br>

## Deploy Standby Container

👉 создадим второй контейнер <br>
👉 очистим data-dir  <br>
👉 скопируем данные через pg_basebackup  <br>
👉 подключим к мастеру  <br>
👉 увидим streaming replication  <br>

### Modify docker-compose.yml

add:
```yaml
  pg-standby:
    image: postgres:15
    container_name: pg-standby
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: adminpass
      POSTGRES_DB: labdb
    volumes:
      - pgstandby:/var/lib/postgresql/data
    networks:
      - pgnet
    depends_on:
      - pg-master
```
```yaml
volumes:
  pgdata:
  pgstandby:
```
```bash
docker compose up -d
```

### Stopping Standby Postgres

#### On baremetal server
```bash
pg_ctl stop -D /var/lib/postgresql/data
rm -rf /var/lib/postgresql/data/*
```

```bash
pg_basebackup \
  -h pg-master \
  -U replicator \
  -D /var/lib/postgresql/data \
  -Fp -Xs -P -R
```
`Пароль: repl123`

```
| Опция | Значение            |
| ----- | ------------------- |
| -h    | мастер              |
| -U    | юзер                |
| -D    | куда писать         |
| -Fp   | plain files         |
| -Xs   | WAL streaming       |
| -P    | прогресс            |
| -R    | auto standby config |
```
> <br>
> -R — создаёт recovery config



#### On containers

Stop the container <br>
```bash
docker stop pg-standby
```
And run an "Empty shell" <br>
```bash
docker run --rm -it \
  --network pg-lab_pgnet \
  -v pg-lab_pgstandby:/var/lib/postgresql/data \
  --entrypoint bash \
  postgres:15
```
Remove all data <br>
```bash
rm -rf /var/lib/postgresql/data/*	# Удаляем данные на слейв
```

Do BaseBackup - настраиваем БД на получение <br>

```bash
pg_basebackup \
  -h pg-master \
  -U replicator \
  -D /var/lib/postgresql/data \
  -Fp -Xs -P -R
```
`Пароль: repl123`

```bash
exit
```
> <br>
> The temporary container will get removed.<br>

Run Compose: <br>

```bash 
docker compose up -d 	# После pg_basebackup при старте postgres (контейнера слейв) начинает работать в режиме реплики
```

#### Final for Standby


On Master: <br>

```sql
SELECT * FROM pg_stat_replication;	# Важно state=streaming
```
Output: <br>
```bash
labdb=# SELECT pid,usename, application_name, client_addr, client_port, backend_start, state, sync_state, reply_time FROM pg_stat_replication;
 pid |  usename   | application_name | client_addr | client_port |         backend_start         |   state   | sync_state |          reply_time
-----+------------+------------------+-------------+-------------+-------------------------------+-----------+------------+-------------------------------
 348 | replicator | walreceiver      | 172.23.0.3  |       53768 | 2026-02-03 10:01:13.734202+00 | streaming | async      | 2026-02-03 11:39:38.834629+00
```
On Standby: <br>

```sql
SELECT pg_is_in_recovery(); 	# Проверка режима реплики (ждет - не ждет)

```
Output: <br>
```bash
labdb=# SELECT pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t

```
```bash
docker exec -it pg-standby bash | grep standby.signal	#На реплике должен появиться файл standby.signal
```
>
> MASTER (rw)  ── WAL ──▶  STANDBY (ro) <br>
>

Еще одна проверка:

```sql
SELECT
  client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
```



### 📌SELECT pg_is_in_recovery();

```
| Где     | Результат |
| ------- | --------- |
| Master  | f         |
| Standby | t         |
```
---------


# Logical Replication

## Master DB Prep

### Check wal_level


```sql
SHOW wal_level;		# меняем replica → logical (in postgresql.conf)
```
```bash
docker restart pg-master
```

### Add pg-logical in docker-compose.yml

```yaml
  pg-logical:
    image: postgres:15
    container_name: pg-logical
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: adminpass
      POSTGRES_DB: labdb
    volumes:
      - pglogical:/var/lib/postgresql/data
    networks:
      - pgnet
```
```yaml
volumes:
  pgdata:
  pgstandby:
  pglogical:
```

```bash
docker compose up -d
```
### Prep PUBLICATION on master

Create table:<br>
```sql
CREATE TABLE logical_test (
    id SERIAL PRIMARY KEY,
    data TEXT
);
```
Create publication:<br>
```sql
CREATE PUBLICATION lab_pub
FOR TABLE logical_test;
```
Check:<br>
```sql
\dRp+
```

### Prep pg-logical

> [!IMPORTANT]
> Создаем идентичную таблицу заранее:<br>

```sql
CREATE TABLE logical_test (
    id SERIAL PRIMARY KEY,
    data TEXT
);
```
Право Select над табл. logical_test  юзеру replicator:<br>
```sql
(ALTER ROLE replicator WITH LOGIN REPLICATION;)
GRANT SELECT ON logical_test TO replicator;
```
или над всеми таблицами:<br>
```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicator;
```
И на будущее (чтобы новые таблицы тоже автоматически давались):<br>
```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA public
GRANT SELECT ON TABLES TO replicator;
```

Создаём подписку:<br>
```sql
CREATE SUBSCRIPTION lab_sub
CONNECTION 'host=pg-master port=5432 dbname=labdb user=replicator password=repl123'
PUBLICATION lab_pub;
```
> <br>
> All insertions and drops in table logical_test should reflect on db pg-logical <br>
>



### WORKAROUND

on Master:<br>
```
SHOW wal_level;				# sould be logical
SELECT * FROM pg_replication_slots;	# plugin = pgoutput
SELECT * FROM pg_publication;		# logical_test
SELECT * FROM pg_publication_tables;	# logical_test

SHOW max_replication_slots;		# max_replication_slots = 10
SHOW max_logical_replication_workers;	# max_logical_replication_workers = 4
SHOW max_worker_processes;		# max_worker_processes = 10

```

on pg-logical:<br>
```
\dRs			# lab_sub row should be there
or
SELECT * FROM pg_subscription \gx

SHOW max_logical_replication_workers;
SHOW max_worker_processes;

ALTER SUBSCRIPTION lab_sub REFRESH PUBLICATION;		# Форсинг
```
On host <br>
```bash
docker logs pg-logical --tail 100
```



#### ✅ Physical replication <br>
> Настроен streaming replication через pg_basebackup. <br>
> На мастере включен wal_level=replica, max_wal_senders, создан replication user. <br>
> Сделан basebackup, создан standby.signal, дальше WAL начал стримиться. <br>
> Проверка через pg_stat_replication и pg_is_in_recovery. <br>

#### ✅ Logical replication <br>
> Через publication и subscription. <br>
> Включен wal_level=logical. <br>
> Создан publication на таблицы, потом subscription. <br>
> Следить за replication slot. <br>
> Права — replication user должен иметь право на SELECT (GRANT). <br>




------------------




## FAILOVER of MASTER

> 👉 «убьём» master<br>
> 👉 сделаем standby новым master<br>
> 👉 проверим данные<br>
> 👉 подключимся к нему <br>

#### Проверяем состояние на мастере
```sql
SELECT client_addr, state, sync_state FROM pg_stat_replication;
```

#### Создаём контрольную запись

```sql
INSERT INTO repl_test(msg) VALUES ('before failover');
```

#### Убиваем master

```bash
docker stop pg-master
```

#### docker stop pg-master

```bash
docker exec -it pg-standby psql -U admin -d labdb
```

```sql
SELECT pg_is_in_recovery();
```
Reply: `t`


#### SELECT Promote (делаем новый Master из Standby)

> «Прекратить быть репликой и стать мастером» <br>
> Перестает ждать WAL <br>
> Удаляет standby.signal <br>
> Становится Primary <br>


```sql
SELECT pg_promote();
```

```sql
SELECT pg_is_in_recovery();
```
Reply: `f`


#### pg_basebackup (делаем новый Standby из Master)

Он остановлен. <br>
Запускаем shell без entrypoint и удаляем PG_DATA (RESET)<br>

```bash
docker run --rm -it  --network pg-lab_pgnet -v pg-lab_pgdata:/var/lib/postgresql/data   --entrypoint bash   postgres:15
rm -rf /var/lib/postgresql/data/*
```
```bash
pg_basebackup \
  -h pg-master \
  -U replicator \
  -D /var/lib/postgresql/data \
  -Fp -Xs -P -R
Пароль на юзер replicator: `repl123`
ls /var/lib/postgresql/data/standby.signal 	# Проверка наличия файла standby.signal
```
Выходим и запускаем контейнер:<br>
```bash
exit
docker start pf-master
```

> Точно так же можно любой инстанс назначить слейвом.<br>
> Для ручного failover используем pg_promote, он завершает recovery и переводит standby в primary.<br>


------------


##  SLOTs / Слоты


Без слотов: Postgres удаляет сегменты WAL (по 16МБ), которых считает не нужными для восстановления после сбоя.<br>
Реплика outage - WAL не отдал данные и стерся, репилка после восстановления уже ничего не получит.<br>
Со слотами: Postgres спрашивает: Все реплики это уже прочитали? И только потом удаляет. WAL хранится пока реплика не прочитает. <br>
Slots — опасны. Если реплика умерла → WAL растёт → диск кончается. Поэтому DBA обязан мониторить slots.<br>

### Physical Slot:

#### Включение

Master:
```sql
SELECT * FROM pg_create_physical_replication_slot('standby_slot1');
```
```
SELECT slot_name, active, restart_lsn, slot_type FROM pg_replication_slots \gx 
slot_name   | standby_slot1			# слот создан
active      | f					# слот не авктивен, никто не подключен
restart_lsn |					# WAL не привязан к слоту
```

Опасно когда:
```
active = f
restart_lsn != NULL
```

Slave:
```ini
primary_slot_name = standby_slot1
```

Для этого:
```bash
docker run --rm -it --network pg-lab_pgnet -v pg-lab_pgdata:/var/lib/postgresql/data --entrypoint bash  postgres:15
rm -rf /var/lib/postgresql/data/*
pg_basebackup -h pg-master -U replicator -D /var/lib/postgresql/data -Fp -Xs -P -R -S standby_slot1	# (--slot=standby_slot1) привязка к slot.
exit
docker start pg-master
```
Возможно потребуется на мастере:
```bash
docker exec -it pg-standby bash
echo "primary_slot_name = 'standby_slot1'" >> /var/lib/postgresql/data/postgresql.auto.conf
exit
docker restart pg-standby
```

На мастер станет:
```
active = t
restart_lsn = 0/xxxx
slot_type = physical
```


> **ВАЖНО**
```bash
pg_basebackup \					# Запомни!!!
  --slot=standby_slot1

du -sh /var/lib/postgresql/data/pg_wal 		# Размер WAL
SELECT slot_name, active, restart_lsn, slot_type FROM pg_replication_slots; 	# Наличие и Статус
```
> Если реплика умерла, а WAL очищен, после оживления догонять не будет никогда!<br>
> Только пересборка:
- удалить слот (`SELECT pg_drop_replication_slot('standby_slot1');`)
- пересоздать слот (`SELECT * FROM pg_create_physical_replication_slot('standby_slot1');`)
- pg_basebackup (`pg_basebackup -h pg-master -U replicator -D /var/lib/postgresql/data -Fp -Xs -P -R --slot=standby_slot1`)




### LOGICAL thru SLOT (after FAILOVER)

Нормальная схема в жизни:
```bash
Primary
 ├── physical slot → standby
 └── logical slot → analytics / kafka
```

On MASTER:
```sql
ALTER SYSTEM SET wal_level = logical;
или:
vim /var/lib/postgresql/data/postgresql.conf  # wal_level = logical
или лучше:
echo "wal_level = logical" >> /var/lib/postgresql/data/postgresql.auto.conf

потом:
SELECT pg_reload_conf();
или:
docker restart pg-master
или на baremetal:
pg_ctl restart -D /var/lib/postgresql/data
```
Проверка:
```sql
SHOW max_replication_slots;
SHOW max_logical_replication_workers;
SHOW max_worker_processes;
SELECT * FROM pg_publication_tables \gx

```

On MASTER:
```sql
SELECT * FROM pg_replication_slots WHERE slot_type='logical';	# Проверяем есть ли старый логический слот
SELECT pg_drop_replication_slot('lab_sub');			# Если есть, удаляем - крайний случай
DROP PUBLICATION lab_pub; 					# Удаляем старую публикацию
CREATE PUBLICATION lab_pub FOR ALL TABLES;			# Создаём заново
```
On PG-LOGICAL:
```sql
DROP SUBSCRIPTION lab_sub;		# Удаляем старую подписку (восстановление после фейловер, можно `WITH (force);`)
SELECT * FROM pg_stat_subscription;	# Проверка подписок
delete from pg_stat_subscription where subid = 16399;	# Чистка метаданных - крайний случай
DELETE FROM pg_subscription WHERE oid = 16399;		# Чистка метаданных - крайний случай

-- или вообще дропаем /data/ логической репликации, стопаем логик, далее...
docker run --rm -it --network pg-lab_pgnet -v pg-lab_pglogical:/var/lib/postgresql/data --entrypoint bash  postgres:15
rm -rf /var/lib/postgresql/data/*

-- на мастере слота не должно быть
SELECT pg_drop_replication_slot('lab_sub');
SELECT pg_drop_replication_slot('pg_16449_sync_16390_7602668465704763430');

-- логик запускаем и вручную создаём таблицы...
CREATE TABLE <schema.table_name> (data data_types);
CREATE SUBSCRIPTION lab_sub 
	CONNECTION 'host=pg-master user=replicator password=repl123 dbname=labdb' 
	PUBLICATION lab_pub WITH (create_slot = true, slot_name = lab_sub, copy_data = true);
-- если торопимся
ALTER SUBSCRIPTION lab_sub REFRESH PUBLICATION;			# пересинхронизация

```
> Postgres сам создаст logical slot.<br>
> Create all relations manually and identically!!! Структура должна быть 1 в 1, как на мастере. Используй `\d after_failover` чтобы увидеть названия и типы
> "Physical replication использует physical replication slots, а logical replication — logical slots, которые обычно создаются автоматически при создании subscription. Они независимы и могут работать параллельно."

----------------

## Всяко разное

Размер таблицы:
```sql
select pg_size_pretty(pg_total_relation_size('wal_bomb'));	# + indexes
select pg_size_pretty(pg_relation_size('wal_bomb'));		# - indexes
select pg_size_pretty(pg_indexes_size('wal_bomb'));		# only indexes
```
Список всех таблиц с размерами:
```sql
SELECT 
    relname AS "Table",
    pg_size_pretty(pg_total_relation_size(relid)) AS "Size"
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```
Размер WAL:
```bash
du -sh /var/lib/postgresql/data/pg_wal
```
**Сброс WAL:**
```bash
pg_ctl stop
/usr/lib/postgresql/15/bin/pg_resetwal -f /var/lib/postgresql/data/
```
или
```bash
docker run --rm --user postgres -v pg-lab_pgdata:/var/lib/postgresql/data  --entrypoint pg_resetwal postgres:15 -f /var/lib/postgresql/data
```
Размеры WAL уменьшатся. Далее необходимо запустить PostgreSQL и:
```sql
pg_checksums --check /var/lib/postgresql/data
REINDEX DATABASE postgres;
VACUUM FULL;
SELECT * FROM pg_stat_database_conflicts;
```




Снятие дампа: `pg_dump -h localhost -U username -d dbname -F c -f dumpfile.dump`

```bash
pg_dump -U username -d dbname -f backup.sql		# Создание SQL-файла (простой текст)
	psql -U username -d dbname -f backup.sql
pg_dump -U username -d dbname -F c -f backup.dump	# Создание сжатого дампа (рекомендуемый формат -F c)
	pg_restore -U username -d dbname -v backup.dump
pg_dumpall -U username -f all_databases.sql		# Создание дампа всех баз данных (включая роли и пользователей):
	psql -f all_databases.sql.sql postgres
pg_dumpall | gzip > backup.gz
	gunzip -c backup.gz | psql -f - postgres
```
Снатие дампа с Докер:
```bash
docker run --rm \
  -v pg-lab_pgdata:/data \
  -v $(pwd):/backup \
  alpine \
  cp -a /data /backup/pgdata_backup_$(date +%F)
```


 
