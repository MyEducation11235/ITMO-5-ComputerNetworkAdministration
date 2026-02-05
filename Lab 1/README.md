# Лабораторная работа 1. HA Postgres Cluster

Выполнил: Проскуряков Роман Владимирович

## Часть 1. Поднимаем Postgres

<details>
  <summary>Dockerfile</summary>

```
FROM postgres:15

# Ставим нужные для Patroni зависимости
RUN apt-get update -y && \
	apt-get install -y netcat-openbsd python3-pip curl python3-psycopg2 python3-venv iputils-ping

# Используем виртуальное окружение, доустанавливаем, собственно, Patroni
RUN python3 -m venv /opt/patroni-venv && \
	/opt/patroni-venv/bin/pip install --upgrade pip && \
	/opt/patroni-venv/bin/pip install patroni[zookeeper] psycopg2-binary

# Копируем конфигурацию для двух узлов кластера Patroni
COPY postgres0.yml /postgres0.yml
COPY postgres1.yml /postgres1.yml

ENV PATH="/opt/patroni-venv/bin:$PATH"

USER postgres

#CMD не задаем, т.к. все равно будем переопределять его далее в compose
```
</details>

<details>
  <summary>Конечный docker-compose.yml</summary>

```
services:
  pg-master:
    build: .
    image: localhost/postgres:patroni # имя для кастомного образа из Dockerfile, можно задать любое
    container_name: pg-master # Будущий адрес первой ноды
    restart: always
    hostname: pg-master
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      PGDATA: '/var/lib/postgresql/data/pgdata'
    expose:
      - 8008
    ports:
      - 5433:5432
    volumes:
      - pg-master:/var/lib/postgresql/data
    command: patroni /postgres0.yml

  pg-slave:
    build: .
    image: localhost/postgres:patroni # имя для кастомного образа из Dockerfile, можно задать любое
    container_name: pg-slave # Будущий адрес второй ноды
    restart: always
    hostname: pg-slave
    expose:
      - 8008
    ports:
      - 5434:5432
    volumes:
      - pg-slave:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      PGDATA: '/var/lib/postgresql/data/pgdata'
    command: patroni /postgres1.yml

  zoo:
    image: confluentinc/cp-zookeeper:7.7.1
    container_name: zoo # Будущий адрес зукипера
    restart: always
    hostname: zoo
    ports:
      - 2181:2181
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  haproxy:
    image: haproxy:3.0
    container_name: postgres_entrypoint # Это будет адрес подключения к БД, можно выбрать любой
    ports:
      - 5432:5432 # Это будет порт подключения к БД, можно выбрать любой
      - 7000:7000
    depends_on: # Не забываем убедиться, что сначала все корректно поднялось
      - pg-master
      - pg-slave
      - zoo
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    
volumes:
  pg-master:
  pg-slave:
```
</details>

<details>
  <summary>postgres0.yml</summary>

```
scope: my_cluster # Имя нашего кластера
name: postgresql0 # Имя первой ноды

restapi: # Адреса первой ноды
  listen: pg-master:8008
  connect_address: pg-master:8008

zookeeper:
  hosts:
    - zoo:2181 # Адрес Zookeeper

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 10485760
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_segments: 8
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "always"
        archive_timeout: 1800s
        archive_command: mkdir -p /tmp/wal_archive && test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f
  pg_hba:
  - host replication replicator 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-master:5432 # Адрес первой ноды
  data_dir: /var/lib/postgresql/data/postgresql0 # Место хранения данных первой ноды
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication: # логопасс для репликаци, при желании можно поменять
      username: replicator
      password: rep-pass
    superuser: # админский логопасс, при желании можно поменять (в том числе в файле compose)
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: '.'

watchdog:
  mode: off

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```
</details>

<details>
  <summary>postgres1.yml</summary>

```
scope: my_cluster # Имя нашего кластера
name: postgresql1 # Имя второй ноды

restapi: # Адреса первой ноды
  listen: pg-slave:8008
  connect_address: pg-slave:8008

zookeeper:
  hosts:
    - zoo:2181 # Адрес Zookeeper

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 10485760
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        wal_keep_segments: 8
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "always"
        archive_timeout: 1800s
        archive_command: mkdir -p /tmp/wal_archive && test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f
  pg_hba:
  - host replication replicator 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-slave:5432 # Адрес первой ноды
  data_dir: /var/lib/postgresql/data/postgresql1 # Место хранения данных первой ноды
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass1
  authentication:
    replication: # логопасс для репликаци, при желании можно поменять
      username: replicator
      password: rep-pass
    superuser: # админский логопасс, при желании можно поменять (в том числе в файле compose)
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: '.'

watchdog:
  mode: off

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```
</details>

<details>
  <summary>haproxy.cfg</summary>

```
global
  maxconn 100

defaults
  log global
  mode tcp
  retries 3
  timeout client 30m
  timeout connect 4s
  timeout server 30m
  timeout check 5s

listen stats
  mode http
  bind *:7000
  stats enable
  stats uri /

listen postgres
  bind *:5432 # Выбранный порт из docker-compose.yml
  option httpchk GET /health
  http-check expect status 200 # Описываем нашу проверку доступности (в данном случае обычный HTTP-пинг)
  default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
  server postgresql_pg_master_5432 pg-master:5432 maxconn 100 check port 8008 # Адрес первой ноды постгреса
  server postgresql_pg_slave_5432 pg-slave:5432 maxconn 100 check port 8008 # Адрес второй ноды постгреса
```
</details>

Собираем докер образ из докер файла
	
`docker build --no-cache -f Dockerfile -t localhost/postgres:patroni .`

[<img src="ReportPhoto/build.png" />]()

Запустим контейнеры через docker compose

`docker compose up -d`

Эта команда запустит:
* pg-master (Postgres + Patroni, порт 5433)
* pg-slave (Postgres + Patroni, порт 5434)
* zoo (Zookeeper, порт 2181)

`docker compose ps`

[<img src="ReportPhoto/compose.png" />]()

Узнаём какая из нод стала главной:

`docker compose logs pg-master | grep leader`

`docker compose logs pg-slave | grep leader`

[<img src="ReportPhoto/whoLeader.png" />]()

Основной является *pg-slave*

Проверяем, что зукипер запустился

`docker logs zoo | grep 2181`

[<img src="ReportPhoto/zooStarted.png" />]()

### Вопрос о разнице между expose и ports в Docker Compose
expose делает порт доступным только для других контейнеров в той же сети Docker, но не публикует его на хост-машине. ports публикует порт на хост-машине, позволяя внешним подключениям. В данной лабораторной порт 8008 используется Patroni для внутреннего API, поэтому его достаточно экспонировать только между контейнерами. Порты 5433 и 5434 проброшены на хост для подключения клиентов к БД.

### Вопрос о пересборке образа при изменениях в Docker Compose
При обычном docker-compose up образ не пересобирается, если он уже существует. Если изменить файлы postgresX.yml, которые копируются в образ через COPY в Dockerfile, потребуется пересборка с docker-compose up --build. Если же изменить только содержимое Dockerfile, пересборка также потребуется. Однако если конфигурационные файлы монтируются через volumes, изменения применяются без пересборки.

# Часть 2 Проверяем репликацию

Подключаемся к основной бд (pg-slave) и редиактируем её

```
CREATE TABLE test_table (id int, data varchar);
INSERT INTO test_table VALUES('2', 'data for replic');
SELECT * FROM test_table
```

Изменения прошли успешно.

[<img src="ReportPhoto/editLeader.png" />]()

Проверяем на реплике (pg-master).

`SELECT * FROM test_table`

Получили те же данные.

[<img src="ReportPhoto/lookingReplic.png" />]()

Пробуем изменить реплику (pg-master) напрямую

`INSERT INTO test_table VALUES('3', 'data error')`

Получаем ошибку:
`ERROR:  cannot execute INSERT in a read-only transaction`

[<img src="ReportPhoto/errorEdit.png" />]()

# Часть 3 Делаем высокую доступность

Перезапускаем проект после изменений в `docker-compose.yml`. Для этого:

Удаляем ранее поднятые контейнеры

`docker compose down`

И запускаем заново

`docker compose up -d`

и проверяем, что всё работает так же как до этого.

[<img src="ReportPhoto/restart.png" />]()

Через несколько секунд проверяем, что HAproxy тоже начал работать

`docker logs postgres_entrypoint`

[<img src="ReportPhoto/haproxyLogs.png" />]()

Проверяем что через порт HAProxy получаем те же данные, что записаны в бд.

[<img src="ReportPhoto/haproxyConnected.png" />]()

Принудительно отключаем ноду текущего мастера (pg-slave):

`docker stop pg-slave`

И видим, что реплика перехватила подхватила работу и стала лидирующей нодой.

[<img src="ReportPhoto/stopMaster.png" />]()

Пытаемся добавить новую таблицу sql. Она добавляется на единственную активную ноду.

```
CREATE TABLE new_table (id int, data varchar);
INSERT INTO new_table VALUES('2', 'new');
SELECT * FROM test_new
```

[<img src="ReportPhoto/replicUpdated.png" />]()

Запускаем старого мастера (pg-slave) обратно

`docker start pg-slave`

[<img src="ReportPhoto/restartMaster.png" />]()

Проверяем, что он получил изменения, сделанные в моменте пока он не работал.

[<img src="ReportPhoto/oldmasterUpdated.png" />]()

Теперь старый масте стал репликой. Роли поменялись местами.

[<img src="ReportPhoto/oldmasterNowIsReplic.png" />]()

## Вывод

В ходе работы был успешно развернут отказоустойчивый кластер PostgreSQL. Использование Patroni в связке с ZooKeeper позволило автоматизировать процесс перевыборов лидера при отказе, а HAProxy обеспечил прозрачное для пользователя переключение между узлами. Кластер сохраняет работоспособность и целостность данных даже при выходе из строя одного из серверов БД. Однако приходится платить за это избыточностью данных.