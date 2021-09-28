# farir1408_microservices
farir1408 microservices repository

## HW-12 (Lecture 16)
#### branch: `docker-2`
- [X] Работа с базовыми понятиями в docker
- [X] Docker в yandex-cloud
- [X] Работа с docker hub
- [X] Дополнительное задание: Объяснить различие между контейнером и образом
- [ ] Дополнительное задание: Автоматизировать установку и деплой приложения

<details><summary>Решение</summary>

#### Работа с базовыми понятиями в docker

* Установить [docker](https://docs.docker.com/engine/install/)

* Результат установки
```editorconfig
$ docker --version
Docker version 20.10.2, build 2291f61

docker info
Client:
    Context:    default
    Debug Mode: false
    Plugins:
        app: Docker App (Docker Inc., v0.9.1-beta3)
        buildx: Build with BuildKit (Docker Inc., v0.5.1-docker)
        scan: Docker Scan (Docker Inc., v0.5.0)
...
```

* Docker hello-world

Запустить контейнер hello-world:
```editorconfig
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
b8dfde127a29: Pull complete 
Digest: sha256:7d91b69e04a9029b99f3585aaaccae2baa80bcf318f4a5d2165a9898cd2dc0a1
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

Что произошло:
1) docker client запросил у docker engine запуск container из image helloworld
2) docker engine не нашел image hello-world локально и скачал его с Docker Hub
3) docker engine создал и запустил container из image hello-world и передал docker client вывод stdout контейнера

Проверка списка контейнеров
1) `docker ps` - выводит список запущенных контейнеров
2) `docker ps -a` - выводит список всех контейнеров

Проверка списка имеджей
1) `docker images` - выводит список образов доступных на хосте

Команда `docker run` запускает контейнер из подготовленного образа
```editorconfig
docker run -it ubuntu:18.04 /bin/bash
```

Использование команды `docker run`

`docker run = docker create + docker start + docker attach`

1) `-i` – запускает контейнер в foreground режиме ( docker attach )
2) `-t` создает TTY   
3) `-d` – запускает контейнер в background режиме   
4) `ubuntu:18.04` имя образа, из которого будет создан контейнер
5) `/bin/bash` команда, которая будет выполнена в контейнере
6) Если не указать опцию `--rm`, то docker каждый раз будет создавать новый контейнер
7) Через параметры передаются лимиты (cpu/mem/disk), ip, volumes

Запуск и остановка уже созданного контейнера осуществляются с помощью команд:
1) `docker start <containet id>`
2) `docker stop <container id>`

Подключение к запущенному контейнеру
`docker attach <container id>`

Для запуска нового процесса внутри контейнера используется
`docker exec`

Для сохранения образа из запущенного контейнера используется
`docker commit <container_id> some-name/ubuntu-test` - контейнер при этом продолжает работать

Остановка контейнера
1) `docker kill` - посылает SIGKILL (нельзя отловить и обработать)
2) `docker stop` - посылает SIGTERM (можно отловить и обработать), через 10 секунд отправляет SIGKILL

Команда `docker system df`
1) Отображает сколько дискового пространства занято образами, контейнерами и volume’ами
2) Отображает сколько из них не используется и возможно удалить

#### Docker в yandex-cloud

* Установить [yandex-cloud cli](https://cloud.yandex.ru/docs/cli/operations/install-cli)

* Выполнить команду `yc init` и залогиниться в учётной записи yandex

* Установить [docker-machine](https://docs.docker.com/machine/install-machine/)

* Создать docker-host в yandex cloud
```editorconfig
yc compute instance create \
  --name docker-host \
  --zone ru-central1-a \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-1804-lts,size=15 \
  --ssh-key ~/.ssh/otus_devops.pub
```

* Настроить локальное окружение на работу с удалённы docker host
```editorconfig
docker-machine create \
  --driver generic \
  --generic-ip-address=<ПУБЛИЧНЫЙ IP DOCKER HOST> \
  --generic-ssh-user yc-user \
  --generic-ssh-key ~/.ssh/otus_devops \
  docker-host
```

* Проверить, что docker host успешно создан
```editorconfig
docker-machine ls
```

* Переключить docker на работу с docker host
```editorconfig
eval $(docker-machine env docker-host)
```

* Для дальнейшей работы подготовить файлы для запуска mongodb

Содержимое файла `mongod.conf`
```editorconfig
# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1
```

Содержимое файла `start.sh`
```editorconfig
#!/bin/bash

/usr/bin/mongod --fork --logpath /var/log/mongod.log --config /etc/mongodb.conf

source /reddit/db_config

cd /reddit && puma || exit
```

Содержимое файла `db_config`
```editorconfig
DATABASE_URL=127.0.0.1
```

* Создать Dockerfile
```dockerfile
FROM ubuntu:18.04

# Обновить системные пакеты
RUN apt-get update
RUN apt-get install -y mongodb-server ruby-full ruby-dev build-essential git
RUN gem install bundler

# Скопировать код приложения в контейнер
RUN git clone -b monolith https://github.com/express42/reddit.git

# Скопировать файлы конфигурации
COPY mongod.conf /etc/mongod.conf
COPY db_config /reddit/db_config
COPY start.sh /start.sh

# Установка зависимостей и настройка
RUN cd /reddit && rm Gemfile.lock && bundle install
RUN chmod 0777 /start.sh

# Старк сервиса при запуске контейнера
CMD ["/start.sh"]
```

* Сборка образа
```editorconfig
$ docker build -t reddit:latest .
```

В результате каждая инструкция run/copy... будет создавать новый образ
Как видно, не считая базового образа есть 9 новых образов `none`, которые создаются
при вызове команды из Dockerfile.
```editorconfig
docker images -a
REPOSITORY      TAG       IMAGE ID       CREATED          SIZE
reddit          latest    0699a1b31565   16 seconds ago   646MB
<none>          <none>    55806c8130fc   16 seconds ago   646MB
<none>          <none>    3530d132a655   18 seconds ago   646MB
<none>          <none>    cfe5ee03398c   33 seconds ago   616MB
<none>          <none>    c23c6900b00d   33 seconds ago   616MB
<none>          <none>    0e5ecb489b90   34 seconds ago   616MB
<none>          <none>    eb7def049ed1   34 seconds ago   616MB
<none>          <none>    315c32e1a8d4   37 seconds ago   616MB
<none>          <none>    31ae58730bb6   46 seconds ago   612MB
<none>          <none>    0bf7809e0038   3 minutes ago    101MB
ubuntu          18.04     39a8cfeef173   4 weeks ago      63.1MB
tehbilly/htop   latest    4acd2b4de755   3 years ago      6.91MB
```

* Запуск контейнера
```editorconfig
$ docker run --name reddit -d --network=host reddit:latest
```

Проверку можно осуществить с помощью команд`docker-machine ls` и `docker ps`

#### Работа с docker hub

* Зарегестрироваться на [docker hub](https://hub.docker.com/)

* Залогиниться используя команду `docker login`

* Создать тег для контейнера
```editorconfig
$ docker tag reddit:latest farir/otus-reddit:1.0
```

В результате будет создан новый образ farir/otus-reddit с тегом 1.0 из образа reddit

* Загрузить образ на docker hub
```editorconfig
$ docker push farir/otus-reddit:1.0
```

Теперь при создании контейнера из образа farir/otus-reddit:1.0, образ будет скачиваться из docker hub

* Для переключения на локальное окружение
```editorconfig
$ docker - eval $(docker-machine env --unset)
```

</details>

## HW-13 (Lecture 17)
#### branch: `docker-3`
- [X] Разбить приложение на несколько компонентов
- [X] Запустить микросервисное приложение
- [X] Оптимизировать сборку образа
- [X] Создать docker-volume для хранения данных в бд
- [X] Дополнительное задание: Сборка образа на основе Alpine Linux

<details><summary>Решение</summary>

#### Разбить приложение на несколько компонентов

* Подключиться к ранее созданному docker-host
```editorconfig
$ eval $(docker-machine env docker-host)
```

* Скачать [архив с приложением](https://github.com/express42/reddit/archive/microservices.zip)
Разархивировать его в директорию `scr`
  
* Структура директории `src`
```editorconfig
src
├── README.md
├── comment
│   ├── Gemfile
│   ├── Gemfile.lock
│   ├── VERSION
│   ├── comment_app.rb
│   ├── config.ru
│   ├── docker_build.sh
│   └── helpers.rb
├── post-py
│   ├── VERSION
│   ├── docker_build.sh
│   ├── helpers.py
│   ├── post_app.py
│   └── requirements.txt
└── ui
    ├── Gemfile
    ├── Gemfile.lock
    ├── VERSION
    ├── config.ru
    ├── docker_build.sh
    ├── helpers.rb
    ├── middleware.rb
    ├── ui_app.rb
    └── views
        ├── create.haml
        ├── index.haml
        ├── layout.haml
        └── show.haml
```

Где:
1) `post-py` - сервис отвечающий за написание постов
2) `comment` - сервис отвечающий за написание комментариев
3) `ui` - веб-интерфейс, работающий с другими сервисами

Для работы также понадобится mongodb

* Создать dockerfile для каждого сервиса

Сервис `post-py`
```dockerfile
FROM python:3.6.0-alpine

WORKDIR /app
ADD . /app

RUN apk --no-cache --update add build-base && \
    pip install -r /app/requirements.txt && \
    apk del build-base

ENV POST_DATABASE_HOST post_db
ENV POST_DATABASE posts

ENTRYPOINT ["python3", "post_app.py"]
```

Сервис `comment`
```dockerfile
FROM ruby:2.2
RUN apt-get update -qq && apt-get install -y build-essential

ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

ADD Gemfile* $APP_HOME/
RUN bundle install
ADD . $APP_HOME

ENV COMMENT_DATABASE_HOST comment_db
ENV COMMENT_DATABASE comments

CMD ["puma"]
```

Сервис `ui`
```dockerfile
FROM ruby:2.2
RUN apt-get update -qq && apt-get install -y build-essential

ENV APP_HOME /app
RUN mkdir $APP_HOME

WORKDIR $APP_HOME
ADD Gemfile* $APP_HOME/
RUN bundle install
ADD . $APP_HOME

ENV POST_SERVICE_HOST post
ENV POST_SERVICE_PORT 5000
ENV COMMENT_SERVICE_HOST comment
ENV COMMENT_SERVICE_PORT 9292

CMD ["puma"]
```

#### Сборка приложения

* Скачать последнюю версию образа mongodb (в docker-host)
```editorconfig
$ docker pull mongo:latest
```

* Собрать контейнер с приложением `post-py`
```editorconfig
$ docker build -t farir/post:1.0 ./post-py
```

* Собрать контейнер с приложением `comment`
```editorconfig
$ docker build -t farir/comment:1.0 ./comment
```

* Собрать контейнер с приложением `ui`
```editorconfig
$ docker build -t farir/ui:1.0 ./ui
```

Сборка ui началась не с первого шага так как начальный образ `ruby` был уже загружен для создания образа `comment`
```editorconfig
Step 1/13 : FROM ruby:2.2
 ---> 6c8e6f9667b2
```

#### Запустить микросервисное приложение

* Создать docker сеть для контейнеров
```editorconfig
$ docker network create reddit
```

* Запустить приложения
```editorconfig
$ docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
$ docker run -d --network=reddit --network-alias=post farir/post:1.0
$ docker run -d --network=reddit --network-alias=comment farir/comment:1.0
$ docker run -d --network=reddit -p 9292:9292 farir/ui:1.0
```

* Что было сделано

1) Создана bridge-сеть для контейнеров, так как сетевые алиасы не работают в сети по умолчанию
2) Запущены контейнеры в этой сети
3) Добавлены сетевые алиасы контейнерам

* Для запуска приложения с другими сетевыми алиасами необходимо задать новые переменные для контейнеров
```editorconfig
$ docker run -d --network=reddit --network-alias=post_db_new --network-alias=comment_db_new mongo:latest
$ docker run -e POST_DATABASE_HOST=post_db_new -d --network=reddit --network-alias=post_new farir/post:1.0
$ docker run -e COMMENT_DATABASE_HOST=comment_db_new -d --network=reddit --network-alias=comment_new farir/comment:1.0
$ docker run -e POST_SERVICE_HOST=post_new -e COMMENT_SERVICE_HOST=comment_new -d --network=reddit -p 9292:9292 farir/ui:1.0
```

#### Оптимизировать сборку образа

* Размер образа ui
```editorconfig
REPOSITORY      TAG             IMAGE ID       CREATED       SIZE
farir/ui        1.0             2c014279b024   2 days ago    771MB
```

* Сервис `ui`
```dockerfile
FROM ubuntu:16.04
RUN apt-get update \
    && apt-get install -y ruby-full ruby-dev build-essential \
    && gem install bundler --no-ri --no-rdoc

ENV APP_HOME /app
RUN mkdir $APP_HOME

WORKDIR $APP_HOME
ADD Gemfile* $APP_HOME/
RUN bundle install
ADD . $APP_HOME

ENV POST_SERVICE_HOST post
ENV POST_SERVICE_PORT 5000
ENV COMMENT_SERVICE_HOST comment
ENV COMMENT_SERVICE_PORT 9292

CMD ["puma"]
```

* Размер образа `ui`
```editorconfig
REPOSITORY      TAG             IMAGE ID       CREATED          SIZE
farir/ui        2.0             af883ae457e3   43 seconds ago   462MB
```

#### Создать docker-volume для хранения данных в бд

После пересоздания контейнера mongodb его RW слой пересоздался, поэтому ранее сохранённые данные были удалены.
Что бы избежать потерю данных при пересоздании контейнера нужно использовать docker-volumes

* Создать том для mongodb
```editorconfig
$ docker volume create reddit_db
```

* Подключить том к контейнеру с mongodb
```editorconfig
$ docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db -v reddit_db:/data/db mongo:latest
```

#### Дополнительное задание: Сборка образа на основе Alpine Linux

Для сборки используется файл `/ui/Dockerfile.alpine`
```dockerfile
FROM alpine:3.14

RUN apk --update add --no-cache ruby-full ruby-dev build-base \
    && gem install bundler -v 1.17.2 --no-document

ENV APP_HOME /app
RUN mkdir $APP_HOME

WORKDIR $APP_HOME
COPY Gemfile* $APP_HOME/
RUN bundle install
COPY . $APP_HOME

ENV POST_SERVICE_HOST post
ENV POST_SERVICE_PORT 5000
ENV COMMENT_SERVICE_HOST comment
ENV COMMENT_SERVICE_PORT 9292

CMD ["puma"]
```

* Сборка нового образа
```editorconfig
$ docker build -t farir/ui:3.0 ui/ --file ui/Dockerfile.alpine
```

* Размер нового образа
```editorconfig
REPOSITORY      TAG             IMAGE ID       CREATED              SIZE
farir/ui        3.0             50a8d847f1c8   5 seconds ago        265MB
```

* Завершение

Удалить docker-machine
```editorconfig
$ docker-machine rm docker-host
```

Удалить yc instance
```editorconfig
$ yc compute instance delete docker-host
```

</details>


## HW-14 (Lecture 18)
#### branch: `docker-4`
- [X] Работа с сетями в Docker
- [X] Использование docker-compose
- [ ] Дополнительное задание: Сборка образа на основе Alpine Linux

<details><summary>Решение</summary>

#### Работа с сетями в Docker

* None драйвер

Образ `joffotron/docker-net-tools` содержит в себе все нужные утилиты для работы с сетью
```editorconfig
$ docker run -ti --rm --network none joffotron/docker-net-tools -c ifconfig
```
Результат выполнения:
```editorconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

Как видно из результата в контейнере присутствует только loopback интерфейс,
который доступен только внутри контейнера, но нет связи с внешним миром.

* Host драйвер

Запустим контейнер в сетевом пространстве хоста
```editorconfig
$ docker run -ti --rm --network host joffotron/docker-net-tools -c ifconfig
```

Вывод команды идентичен выводу команды `docker-machine ssh docker-host ipconfig`
```editorconfig
docker0   Link encap:Ethernet  HWaddr 02:42:98:E7:AE:C5  
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

eth0      Link encap:Ethernet  HWaddr D0:0D:17:54:31:FD  
          inet addr:10.128.0.24  Bcast:10.128.0.255  Mask:255.255.255.0
          inet6 addr: fe80::d20d:17ff:fe54:31fd%32720/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:226842 errors:0 dropped:0 overruns:0 frame:0
          TX packets:194559 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:136399573 (130.0 MiB)  TX bytes:18613227 (17.7 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1%32720/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:290 errors:0 dropped:0 overruns:0 frame:0
          TX packets:290 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:26494 (25.8 KiB)  TX bytes:26494 (25.8 KiB)
```

* Используя docker host network запустить несколько контейнеров

Запуск 3 раза
```editorconfig
$ docker run --network host -d nginx
```

Результат
```editorconfig
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                      PORTS     NAMES
3f15820bb7b7   nginx     "/docker-entrypoint.…"   16 seconds ago   Exited (1) 13 seconds ago             xenodochial_dijkstra
236cb5e044f8   nginx     "/docker-entrypoint.…"   22 seconds ago   Exited (1) 19 seconds ago             determined_shamir
377c172161c6   nginx     "/docker-entrypoint.…"   34 seconds ago   Up 31 seconds                         wonderful_mendel
```

Запущен только первый контейнер. Посмотрим логи 2х других контейнеров.
```editorconfig
...
 2021/09/24 11:00:02 [emerg] 1#1: bind() to [::]:80 failed (98: Address already in use)
nginx: [emerg] bind() to [::]:80 failed (98: Address already in use)
2021/09/24 11:00:02 [notice] 1#1: try again to bind() after 500ms
2021/09/24 11:00:02 [emerg] 1#1: still could not bind()
nginx: [emerg] still could not bind()
```

Так как используется host сеть, то контейнеры разделяют друг с другом одно пространство сети включая порты.
Первый контейнер успешно занял нужны адрес и порт, остальные контейнеры уже не могут занять тот же адрес и порт.

Остановить все запущенные контейнеры
```editorconfig
$ docker kill $(docker ps -q)
```

* Docker networks

Выполнить команду на докер-хосте
```editorconfig
$ sudo ln -s /var/run/docker/netns /var/run/netns
```

Просмотреть список net неймспейсов
```editorconfig
sudo ip netns
```

* Bridge драйвер

По умолчанию используется драйвер bridge, указывать флаг `--driver` не обязательно

Создать сеть
```editorconfig
$ docker network create reddit --driver bridge
```

Запустить проект используя новую сеть. Так как контейнеры ссылаются друг на друга,
то необходимо задать имена контейнерам, чтобы внутренний dns смог корректно определить ip адреса.
```editorconfig
$ docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
$ docker run -d --network=reddit --network-alias=post farir/post:1.0
$ docker run -d --network=reddit --network-alias=comment farir/comment:1.0
$ docker run -d --network=reddit -p 9292:9292 farir/ui:1.0
```

Если образы были не найдены, выполнить билд и пуш в docker-hub
```editorconfig
$ docker build -t farir/post:1.0 ./post-py
$ docker push farir/post:1.0
```

* Разделить приложение на 2 сети, чтобы ui не имел доступа к mongodb

Создать 2 сети
```editorconfig
$ docker network create back_net --subnet=10.0.2.0/24
$ docker network create front_net --subnet=10.0.1.0/24

$ docker network ls
NETWORK ID     NAME        DRIVER    SCOPE
f4a3545b18aa   back_net    bridge    local
4ec7af75507e   bridge      bridge    local
```

Запустить контейнеры в новых сетях
```editorconfig
$ docker run -d --network=front_net -p 9292:9292 --name ui farir/ui:1.0
$ docker run -d --network=back_net --name comment farir/comment:1.0
$ docker run -d --network=back_net --name post farir/post:1.0
$ docker run -d --network=back_net --name mongo_db \
    --network-alias=post_db --network-alias=comment_db mongo:latest
```

Ошибка работы приложения связана с тем, что docker при инициализации контейнера может подключить к нему только одну сеть
При этом контейнеры из соседних сетей не будут доступны как в DNS, так и для взаимодействия по сети.
Подключить дополнительные сети
```editorconfig
$ docker network connect front_net post
$ docker network connect front_net comment
```

* Сетевой стек docker-host

Установить на docker-host утилиту `bridge-utils`
```editorconfig
$ sudo apt-get update && sudo apt-get install bridge-utils
```

Идентификаторы сетей
```editorconfig
$ docker network ls
NETWORK ID     NAME        DRIVER    SCOPE
f4a3545b18aa   back_net    bridge    local
c2d0c4784012   front_net   bridge    local
```

Описание сетевых интерфейсов docker-host
```editorconfig
$ ifconfig | grep br
br-5589e761af5d: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 172.18.255.255
br-c2d0c4784012: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.1.1  netmask 255.255.255.0  broadcast 10.0.1.255
br-f4a3545b18aa: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.1  netmask 255.255.255.0  broadcast 10.0.2.255
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet 10.128.0.24  netmask 255.255.255.0  broadcast 10.128.0.255

$ brctl show br-f4a3545b18aa
bridge name     bridge id               STP enabled     interfaces
br-f4a3545b18aa         8000.024266afe2ec       no              veth9c309ee
                                                                vetha5c5cc4
                                                                vethc7d5e0e
```

Список правил ip-tables
```editorconfig
$ sudo iptables -nL -t nat
...
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  10.0.1.0/24          0.0.0.0/0           
MASQUERADE  all  --  10.0.2.0/24          0.0.0.0/0           
MASQUERADE  all  --  172.18.0.0/16        0.0.0.0/0           
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  10.0.1.2             10.0.1.2             tcp dpt:9292
...
```

Цепочка `POSTROUTING` отвечает за выпуск трафика во внешнюю сеть из bridge сети

Публикация порта контейнера в ip-tables
```editorconfig
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9292 to:10.0.1.2:9292
```

```editorconfig
$ ps ax | grep docker-proxy
28974 ?        Sl     0:02 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 9292 -container-ip 10.0.1.2 -container-port 9292
28981 ?        Sl     0:02 /usr/bin/docker-proxy -proto tcp -host-ip :: -host-port 9292 -container-ip 10.0.1.2 -container-port 9292
```

Видно, что оба процесс слушают порт 9292 (для IPv4 и IPv6) и перенаправляют трафик в контейнер.

#### Использование docker-compose

* Проблемы 
  
1) Одно приложение состоит из множества контейнеров/сервисов 
2) Один контейнер зависит от другого
3) Порядок запуска имеет значение
4) docker build/run/create … (долго и много)

* Установить [docker-compose](https://docs.docker.com/compose/install/)
```editorconfig
$ docker-compose --version
docker-compose version 1.27.4, build 40524192
```

* Описать инфраструктуру в файле docker-compose.yml
```dockerfile
version: '3.3'
services:
  post_db:
    image: mongo:3.2
    volumes:
      - post_db:/data/db
    networks:
      - reddit
  ui:
    build: ./ui
    image: ${USERNAME}/ui:1.0
    ports:
      - 9292:9292/tcp
    networks:
      - reddit
  post:
    build: ./post-py
    image: ${USERNAME}/post:1.0
    networks:
      - reddit
  comment:
    build: ./comment
    image: ${USERNAME}/comment:1.0
    networks:
      - reddit

volumes:
  post_db:

networks:
  reddit:
```

Запустить сборку docker-compose
```editorconfig
$ export USERNAME=farir

$ docker-compose up -d
...
 
$ docker-compose ps
Name                  Command             State                    Ports
----------------------------------------------------------------------------------------------
src_comment_1   puma                          Up
src_post_1      python3 post_app.py           Up
src_post_db_1   docker-entrypoint.sh mongod   Up      27017/tcp
src_ui_1        puma                          Up      0.0.0.0:9292->9292/tcp,:::9292->9292/tcp
```

* Изменить базовое имя проекта
```editorconfig
$ docker-compose -p <project_prefix> up -d
```

</details>
