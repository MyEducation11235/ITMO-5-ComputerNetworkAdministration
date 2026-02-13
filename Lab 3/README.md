# Лабораторная работа 3. Ansible + Caddy

Выполнил: Проскуряков Роман Владимирович

## Часть 1. Установка и настройка Ansible

Создаём виртуальную среду для работы

`python3 -m venv lab3-venv`

Активируем её

`source lab3-venv/bin/activate`

Устанавливаем ansible

`pip install ansible`

Проверяем, что всё установилось

`ansible --version`

installAnsible.png

После создания конфигов, вроверяем что они подцепились

withConfig.png

Проверяем, что сервер с Ansible подключился к “клиенту” (в нашем случае это одна и та же машина, localhost)

checkAnsimble.png

Создаем текстовый файл с производным содержимым, через модуль shell

`ansible my_servers -c local -m shell -a 'echo test_file_content > ./test.txt'`

Проверяем, что по нужному пути создался нужный файл с нужным именем

`ls`

Удаляем файл через модуль file

`ansible my_servers -c local -m file -a 'path=./test.txt state=absent'`

createDelTest.png

## Часть 2. Установка Caddy

Для начала создадим в рабочей директории папку roles

`mkdir roles && cd roles`

и в ней инициализируем исходное конфигурационное “дерево”

`ansible-galaxy init caddy_deploy`

`tree`

caddyDeploy.png

<details>
  <summary>roles/caddy_deploy/tasks/main.yml</summary>

```
---
# tasks file for caddy_deploy

- name: Install prerequisites
  apt:
    pkg:
    - debian-keyring
    - debian-archive-keyring
    - apt-transport-https
    - curl

- name: Add key for Caddy repo
  apt_key:
    url: https://dl.cloudsmith.io/public/caddy/stable/gpg.key
    state: present
    keyring: /usr/share/keyrings/caddy-stable-archive-keyring.gpg

- name: add Caddy repo
  apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
    state: present
    filename: caddy-stable

- name: add Caddy src repo
  apt_repository:
    repo: "deb-src [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
    state: present
    filename: caddy-stable

- name: Install Caddy webserver
  apt:
    name: caddy
    update_cache: yes
    state: present

- name: Restart Caddy service
  ansible.builtin.systemd:
    name: caddy
    state: restarted
```
</details>

<details>
  <summary>caddy_deploy.yml</summary>

```
---
- name: Install and configure Caddy webserver # Любое описание
  hosts: my_servers # хосты из файла inventory/hosts, где будем выполнять наш плейбук
  connection: local # аналог -c local, но для плейбуков
  become: true

  roles:
    - caddy_deploy # собственно, роль для выполнения
```
</details>

Запускаем плейбук с запросом пароля для sudo

`ansible-playbook caddy_deploy.yml --ask-become-pass`

playBookStart.png

page1.png

## Часть 3. Домен и настройка Caddyfile

duckdns.org не работал, да и внешнего ip у меня нет. Попробую вместо своего доменого имени использовать localhost. Это же тоже своего рода доменое имя?)

<details>
  <summary>roles/caddy_deploy/templates/Caddyfile.j2</summary>

```
{{ domain_name }} {
  root * /usr/share/caddy
  file_server

  log {
    output file {{ log.file }}
    format json
    level {{ log.level }}
  }
}
```
</details>

Пишем путь `/var/log/caddy/cadde_access.log` вместо `/var/log/caddy_access.log`, чтобы избежать ошибки недостатка прав как ниже.

configError.png

<details>
  <summary>roles/caddy_deploy/vars/main.yml</summary>

```
---
# vars file for caddy_deploy

domain_name: localhost

log: # Можно поиграться со значениями
  file: /var/log/caddy/cadde_access.log
  level: "INFO"
```
</details>

<details>
  <summary>Генерирующийся Caddyfile</summary>

```
localhost {
  root * /usr/share/caddy
  file_server

  log {
    output file /var/log/caddy/caddy_access.log
    format json
    level INFO
  }
}
```
</details>

<details>
  <summary>Добавляем в roles/caddy_deploy/tasks/main.yml</summary>

```
- name: Install Caddy webserver
...

- name: Create config file
  template:
    src: templates/Caddyfile.j2 # Откуда берем
    dest: /etc/caddy/Caddyfile # Куда кладем
```
</details>

Перезапускаем deploy и заходим уже по https на наш localhost (именно на него, а не на 127.0.0.1) 

Сертификат нам никто не выдавал поэтому предупреждает нас.

noSertificate.png

Но в целом работает

https.png

## Расширение функционала

<details>
  <summary>roles/caddy_deploy/files/index.html</summary>

```
<!DOCTYPE html>
<html>
<head>
    <title>Мой Caddy</title>
</head>
<body>
    <h1>Hello World from Caddy!</h1>
    <p>Сервер работает под управлением Ansible.</p>
</body>
</html>
```
</details>

<details>
  <summary>Новый roles/caddy_deploy/templates/Caddyfile.j2</summary>

```
{{ domain_name }} {
  root * /var/www/{{ domain_name }}
  file_server

  # Добавляем заголовки безопасности (пример)
  header / {
    X-Content-Type-Options nosniff
    X-Frame-Options DENY
    Referrer-Policy no-referrer-when-downgrade
  }

  log {
    output file {{ log.file }}
    format json
    level {{ log.level }}
  }
}
```
</details>

<details>
  <summary>Добавляем в roles/caddy_deploy/tasks/main.yml</summary>

```
- name: Install Caddy webserver
...

- name: Ensure web root directory exists
  file:
    path: "/var/www/{{ domain_name }}"
    state: directory
    owner: caddy
    group: caddy
    mode: '0755'

- name: Copy custom index.html
  copy:
    src: files/index.html
    dest: "/var/www/{{ domain_name }}/index.html"
    owner: caddy
    group: caddy
    mode: '0644'

- name: Create config file
...
```
</details>

После запуска deploy:

* В каталоге `/var/www/localhost` появится `index.html` и он будет отображаться на сайте

page2.png

* Caddy будет обслуживать его с добавленными заголовками

`curl -k -I https://localhost`

headers.png

## Плейбук для управления файлом

Напишем плейбук который выпонит: 

* создаст файл ./test.txt с содержимым test_file_content

* изменяет содержимое файла на другое

* выводит текущее содержимое файла

* удаляет файл

<details>
  <summary>file_management.yml</summary>

```
---
- name: Управление файлом test.txt
  hosts: localhost
  gather_facts: no

  tasks:
    - name: Создать файл с начальным содержимым
      ansible.builtin.copy:
        dest: ./test.txt
        content: "test_file_content"

    - name: Изменить содержимое файла
      ansible.builtin.copy:
        dest: ./test.txt
        content: "новое содержимое файла"

    - name: Прочитать содержимое файла
      ansible.builtin.command: cat ./test.txt
      register: file_content

    - name: Вывести содержимое файла
      ansible.builtin.debug:
        var: file_content.stdout

    - name: Удалить файл
      ansible.builtin.file:
        path: ./test.txt
        state: absent
```
</details>

fileMenegment.png

# Вывод

Как по мне проще написать какой нибудь bush файл - больше контроля, меньше неожиданных ошибок, да и писать проще, но наверное, я не понял, чем универсальнее использование ansible.
Caddy - довольно удобно и быстро, особенно для тестовых проектов.



