FROM ubuntu:18.04

# Обновить системные пакеты
RUN apt-get update
RUN apt-get install -y mongodb-server ruby-full ruby-dev build-essential ruby-bundler git

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
