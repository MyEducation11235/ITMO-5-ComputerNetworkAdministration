# Лабораторная работа 2. Loki + Zabbix + Grafana

Выполнил: Проскуряков Роман Владимирович

## Часть 1. Логирование

<details>
  <summary>docker-compose.yml</summary>

```
services:
  nextcloud:
    image: nextcloud:29.0.6
    container_name: nextcloud # на это имя будет завязана настройка забикса далее, так что лучше не менять
    ports:
      - "8080:80"
    volumes:
      - nc-data:/var/www/html/data

  loki: # сервис-обработчик логов
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml # запуск с дефолтным конфигом

  promtail: # сервис-сборщик логов
    image: grafana/promtail:2.9.0
    container_name: promtail
    volumes:
      - nc-data:/opt/nc_data # та же самая директория, которая монтируется в Nextcloud
      - ./promtail_config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  grafana: # сервис визуализации
    environment:
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning # просто
      - GF_AUTH_ANONYMOUS_ENABLED=true # дефолтные
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin # настройки
    command: /run.sh
    image: grafana/grafana:11.2.0
    container_name: grafana
    ports:
      - "3000:3000"

  postgres-zabbix:
    image: postgres:15
    container_name: postgres-zabbix
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
    volumes:
      - zabbix-db:/var/lib/postgresql/data 
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "zabbix"]
      interval: 10s
      retries: 5
      start_period: 5s

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-6.4-latest # непосредственно бэкенд забикса
    container_name: zabbix-back
    ports:
      - "10051:10051"
    depends_on:
      - postgres-zabbix
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
      DB_SERVER_HOST: postgres-zabbix

  zabbix-web-nginx-pgsql:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-6.4-latest # фронтенд забикса
    container_name: zabbix-front
    ports:
      - "8082:8080" # внешний порт можно любой по своему желанию
    depends_on:
      - postgres-zabbix
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
      DB_SERVER_HOST: postgres-zabbix
      ZBX_SERVER_HOST: zabbix-back

volumes:
  nc-data:
  zabbix-db
```
</details>

<details>
  <summary>promtail_config.yml</summary>

```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push # адрес Loki, куда будут слаться логи

scrape_configs:
- job_name: system # любое имя
  static_configs:
  - targets:
      - localhost # т.к. монтируем папку с логами прямо в контейнер Loki, он собирает логи со своей локальной файловой системы
    labels:
      job: nextcloud_logs # любое имя, по этому полю будет осуществляться индексирование
      __path__: /opt/nc_data/nextcloud.log # необязательно указывать полный путь, главное сказать где искать log файлы
```
</details>

Запускаем compose файл.

`docker compose up -d`

![](ReportPhoto/dockerStart.png)

Создаём аккаунт в Nextcloud (http://127.0.0.1:8080)

![](ReportPhoto/NextcloudStart.png)

проверяем, что логи “пошли” в нужный нам файл `/var/www/html/data/nextcloud.log`

`docker exec -it nextcloud bash`

`cat data/nextcloud.log`

![](ReportPhoto/nextCloudLog.png)

Проверяем в логах promtail, что он “подцепил” нужный нам log-файл: должны быть строчки, содержащие `msg="Seeked /opt/nc_data/nextcloud.log ..."`

`docker compose logs promtail`

![](ReportPhoto/promtailInitLogs.png)

## Часть 2. Мониторинг

<details>
  <summary>template.yml</summary>

```
zabbix_export:
  version: '6.4'
  template_groups:
    - uuid: a571c0d144b14fd4a87a9d9b2aa9fcd6
      name: Templates/Applications
  templates:
    - uuid: a615dc391a474a9fb24bee9f0ae57e9e
      template: 'Test ping template'
      name: 'Test ping template'
      groups:
        - name: Templates/Applications
      items:
        - uuid: a987740f59d54b57a9201f2bc2dae8dc
          name: 'Nextcloud: ping service'
          type: HTTP_AGENT
          key: nextcloud.ping
          value_type: TEXT
          trends: '0'
          preprocessing:
            - type: JSONPATH
              parameters:
                - $.body.maintenance
            - type: STR_REPLACE
              parameters:
                - 'false'
                - healthy
            - type: STR_REPLACE
              parameters:
                - 'true'
                - unhealthy
          url: 'http://{HOST.HOST}/status.php'
          output_format: JSON
          triggers:
            - uuid: a904f3e66ca042a3a455bcf1c2fc5c8e
              expression: 'last(/Test ping template/nextcloud.ping)="unhealthy"'
              recovery_mode: RECOVERY_EXPRESSION
              recovery_expression: 'last(/Test ping template/nextcloud.ping)="healthy"'
              name: 'Nextcloud is in maintenance mode'
              priority: DISASTER
```
</details>

Подключаемся к веб-интерфейсу Zabbix (http://localhost:8082). Креды *Admin* | *zabbix*

Делаем import кастомного шаблона (template.yml) в zabbix для мониторинга nextcloud.

![](ReportPhoto/templateImport.png)

Чтобы Zabbix и Nextcloud могли общаться по своим коротким именам внутри докеровской сети, в некстклауде необходимо “разрешить” это имя. Для этого
нужно зайти на контейнер некстклауда под юзером www-data

`docker exec -u www-data -it nextcloud bash`

и выполнить команду

`php occ config:system:set trusted_domains 1 --value="nextcloud"`

Создаём хоста в Zabbix. Чтобы он знал, что ему слушать.

![](ReportPhoto/CreateHost.png)

Получаем первые данные с хоста *healhy*

![](ReportPhoto/monitoring1.png)

Попробуем нарушить работу тестового сервиса.

`docker exec -u www-data -it nextcloud bash`

![](ReportPhoto/ProblemConsole.png)

Создаём проблему:

`php occ maintenance:mode --on`

![](ReportPhoto/Problem.png)

Возвращаем как было.

`php occ maintenance:mode --off`

![](ReportPhoto/ProblemResolved.png)

## Часть 3. Визуализация

В Grafana установим плагин Zabbix

Команда устанавливает слишком новую версию Zabbix

~~`docker exec -it grafana bash -c "grafana cli plugins install alexanderzobnin-zabbix-app"`~~

#### Вместо этого установим версию совместимую с grafana:11.2.0

Заходим в контейнер

`docker exec -it grafana bash`

Плагины Grafana лежат в папке `/var/lib/grafana/plugins`

`cd /var/lib/grafana/plugins`

Качаем архив нужной версии (5.2.1 для grafana:11.2.0)

`wget https://github.com/grafana/grafana-zabbix/releases/download/v5.2.1/alexanderzobnin-zabbix-app-5.2.1.zip`

![](ReportPhoto/installZabbixDowngrade.png)

Разархивируем архив

`unzip alexanderzobnin-zabbix-app-5.2.1.zip`

![](ReportPhoto/unzip.png)

После установки плагина любым из способов, нужно выполнить рестарт

`docker restart grafana`

![](ReportPhoto/grafaneResart.png)

#### Заходим в grafana (http://localhost:3000)

Проверяем, что установлена нужная версия

![](ReportPhoto/showWersion.png)

Активируем плагин Zabbix (Enable)

![](ReportPhoto/ActiviteZabbix.png)

#### Подключаем Zabbix к Grafana (http://zabbix-front:8080/api_jsonrpc.php)

![](ReportPhoto/addZabbix.png)

Успешно

![](ReportPhoto/zabbixSuccess.png)

#### Подключаем Loki к Grafana (http://loki:3100)

![](ReportPhoto/AddLoki.png)

Успешно

![](ReportPhoto/lokiSuccess.png)

В Explore получаем логи Zabbix

![](ReportPhoto/zabbixLogs.png)

В Explore получаем логи loki

![](ReportPhoto/lokiLogs.png)

дашборд датасурсов Zabbix - график доступности по времени

![](ReportPhoto/zabbixDasboard.png)

дашборд датасурсов Loki - таблица с логами

![](ReportPhoto/lokiDashboard.png)

## Ответы на вопросы

### 1. Чем SLO отличается от SLA?

*SLA (Service Level Agreement)* — формальный документ, договор между поставщиком услуги и клиентом. Фиксирует минимально приемлемый уровень работы сервиса и последствия за его невыполнение.

Пример: Облачный сервис гарантирует 99.9% доступности в месяц. Если доступность упадет ниже, клиент получит 10% от месячной платы.

*SLO (Service Level Objective)* — внутренний, измеримый и конкретный целевой показатель для ключевых характеристик сервиса. Позволяет определить, насколько надежным должен быть сервис, чтобы удовлетворять пользователей и с запасом выполнять условия SLA. Это ориентир для принятия инженерных решений.

Пример: "Мы ставим себе внутреннюю цель (SLO) — 99.95% доступности, чтобы с уверенностью гарантировать клиентам (SLA) 99.9%."

### 2. Чем отличается инкрементальный бэкап от дифференциального?

*Инкрементальный* — копирует только изменения, сделанные с момента предыдущего бэкапа ЛЮБОГО ТИПА. Для восстановления нужна цепочка архивов: последний полный и ВСЕ последующие инкрементальные.

*Дифференциальный* — копирует все изменения, сделанные с момента последнего ПОЛНОГО бэкапа. Для восстановления нужны два архива: последний полный и последний дифференциальный.

### 3. В чем разница между мониторингом и observability?

*Мониторинг* — это набор инструментов и практик для слежения за известными метриками и состояниями системы.

*Observability (наблюдаемость)* — это свойство системы, позволяющее по её внешним выводам понимать её внутреннее состояние и находить неизвестные и непредвиденные проблемы.

## Вывод

В ходе лабораторной работы была развернута и настроена система логирования на базе Loki, подключён Zabbix для мониторинга сервиса Nextcloud, выполнена интеграция с Grafana и созданы дашборды. Все компоненты успешно взаимодействуют между собой, обеспечивая наблюдение за сервисом.
