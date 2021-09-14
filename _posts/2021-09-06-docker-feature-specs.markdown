---
layout: post
title:  "Feature-спеки внутри Docker контейнера"
date:   2021-09-06 11:37:40 +0300
categories: docker rails
---
Чистые функции хороши тем, что не нужно бояться что при вызове такой функции мы можем получить непредсказуемый результат. Функция просто делает свою работу, не оставляя никаких сайд-эффектов и при этом мы можем быть уверены что при тех же аргументах мы получим тот же результат, независимо от контекста. Docker контейнеры используют подобную идеологию - мы можем быть уверены что наш докер контейнер будет работать одинаково на любой хост-машине.

## зачем?

Для меня основной причиной является изоляция внешних сервисов, используемых в приложении:

- `PostgreSQL`
- `Elasticsearch`
- `Chromedriver`
- etc

Если версию Ruby можно достаточно удобно менеджить с помощью, например, `rbenv`, то внешние для приложения сервисы не так удобно переключать между различными приложениями.

## выносим приложение и сервисы в контейнер

Начнем с написания `Dockerfile`. Для `Rails` приложения нам понадобятся `Ruby` + `PostgreSQL` клиент для подключения к базе данных + `Node` и `Yarn` для наших ассетов. Также потребуется установить `bundler` для зависимостей. 

```
FROM ruby:2.6.8-slim-buster

RUN apt-get update -qq && apt-get install -yq --no-install-recommends \
    build-essential \
    curl \
    git \
    gnupg2 \
    shared-mime-info \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Add PostgreSQL to sources list
ARG PG_MAJOR
RUN curl -sSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - \
    && echo 'deb http://apt.postgresql.org/pub/repos/apt/ buster-pgdg main' $PG_MAJOR > /etc/apt/sources.list.d/pgdg.list


# Add NodeJS to sources list
ARG NODE_MAJOR
RUN curl -sL https://deb.nodesource.com/setup_$NODE_MAJOR.x | bash -


# Add Yarn to the sources list
ARG YARN_VERSION
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo 'deb http://dl.yarnpkg.com/debian/ stable main' > /etc/apt/sources.list.d/yarn.list


# Install Dependencies
RUN apt-get update -qq && apt-get install -yq --no-install-recommends \
    libpq-dev \
    postgresql-client-$PG_MAJOR \
    nodejs \
    yarn=$YARN_VERSION-1 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN rm -f .bundle/config
RUN gem install bundler -v 1.17.3

RUN mkdir /app
WORKDIR /app
```

Зависимости и исходный код будет содержаться в волюмах, чтобы размер образа был меньше. Далее следует приступить к написанию `docker-compose` для оркестрации нашего приложения и вспомогательных сервисов (база данных, Redis, Elasticsearch):
{% highlight yml %}
version: "3.4"

services:
  db:
    image: postgres:13.2-alpine
    environment:
      - POSTGRES_PASSWORD=password
    ports:
      - 5432:5432
    volumes:
      - db_data:/var/lib/postgresql/data

  redis:
    image: redis:6.2.0
    ports:
      - 6379:6379

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.1
    volumes:
      - elastic_data:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false

  # for selenium capybara feature specs
  chrome:
    image: selenium/standalone-chrome-debug:3.141.59
    ports:
      - 4444:4444

  app: &app
    image: rails_app:1.0.0
    stdin_open: true
    tty: true
    build:
      context: .
      args:
        - PG_MAJOR=13
        - NODE_MAJOR=12
        - YARN_VERSION=1.22.5

  backend: &backend
    <<: *app
    environment:
      - RAILS_ENV=${RAILS_ENV:-development}
      - NODE_ENV=${NODE_ENV:-development}
      - REDIS_URL=${REDIS_URL:-redis://redis:6379}
      - ELASTICSEARCH_URL=${ELASTICSEARCH_URL:-http://elasticsearch:9200}
      - DATABASE_URL=${DATABASE_URL:-postgres://postgres:password@db:5432}
      - DATABASE_CLEANER_ALLOW_REMOTE_DATABASE_URL=${DATABASE_CLEANER_ALLOW_REMOTE_DATABASE_URL:-true}
      - WEBPACKER_DEV_SERVER_HOST=webpacker
      - HUB_URL=http://chrome:4444/wd/hub
    depends_on:
      - db
      - redis
      - elasticsearch
      - chrome
    volumes:
      - .:/app:cached
      - ruby_bundle:/usr/local/bundle
      - rails_cache:/app/tmp/cache
      - node_modules:/app/node_modules
      - packs:/app/public/packs

  web:
    <<: *backend
    command: bin/docker-entrypoint
    ports:
      - 3000:3000
      - 4000:4000

  sidekiq:
    <<: *backend
    command: bundle exec sidekiq -C config/sidekiq.yml

  webpacker:
    <<: *app
    command: bin/webpack-dev-server
    ports:
      - 3035:3035
    volumes:
      - .:/app:cached
      - ruby_bundle:/usr/local/bundle
      - rails_cache:/app/tmp/cache
      - node_modules:/app/node_modules
      - packs:/app/public/packs
    environment:
      - RAILS_ENV=${RAILS_ENV:-development}
      - NODE_ENV=${NODE_ENV:-development}
      - WEBPACKER_DEV_SERVER_HOST=0.0.0.0

volumes:
  db_data:
  elastic_data:
  ruby_bundle:
  rails_cache:
  node_modules:
  packs:

{% endhighlight %}

Из интересного:
- Мы открыли 4 000-ый порт в нашем приложении. Это нужно для `Rspec` + `Capybara` тестов. Мы будем запускать сервер для тестов по этому порту.
- У нас есть контейнер `chrome` в котором находится браузер `chrome` для `feature` тестов. 

Интересные переменные окружения:
- `DATABASE_CLEANER_ALLOW_REMOTE_DATABASE_URL` - по умолчанию, в целях безопасности `database_cleaner` не очищает базу данных если ее адрес резолвится за пределы `localhost`-а, но с помощью этой переменной мы разраешаем такое поведение. Это допустимо, потому что данный `docker-compose` будет использоваться исключительно для разработки и тестирования (+ внутри `Docker` контейнеров)
- `HUB_URL` - мы выносим наш браузер для `feature` тестов в отдельный контейнер. Это адрес подключения к нашему `chrome` контейнеру для `capybara`
- `DATABASE_URL` - адрес нашей базы данных
- `REDIS_URL` - адрес для `Redis`. Аналогично с `DATABASE_URL`.

Далее следует подготовить наш код к тому, что он будет работать внутри контейнера в dev и test окружении:
- `Sidekiq` может автоматически подключаться к нужному урлу если он задан в переменной `REDIS_URL`, поэтому для `Sidekiq` ничего менять не нужно ([link](https://github.com/mperham/sidekiq/wiki/Using-Redis#using-an-env-variable))
- Нужно немного подправить `database.yml`:
{% highlight yml %}
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  url: <%= ENV["DATABASE_URL"] %>

development:
  <<: *default
  database: rails_app_development

test:
  <<: *default
  database: rails_app_test

production:
  <<: *default
{% endhighlight %}
- Далее подправить `cable.yml` для `ActionCable`:
{% highlight yml %}
development:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: rails_app_development

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: rails_app_production
{% endhighlight %}
- Нужно также передать `capybara` что мы будем подключаться к `Chrome` который находится в другом контейнере + запускать тестовый сервер на том порту, который мы открыли в `docker-compose.yml` (4000):
{% highlight ruby %}
CHROME_HUB_URL = ENV["HUB_URL"]

Capybara.register_driver :container_chrome_headless do |app|
  capabilities = ::Selenium::WebDriver::Remote::Capabilities.chrome(
    chromeOptions: { args: %w[no-sandbox headless disable-gpu] }
  )

  config = CHROME_HUB_URL.present? ? { browser: :remote, url: CHROME_HUB_URL } : { browser: :chrome }

  Capybara::Selenium::Driver.new(app, config.merge(desired_capabilities: capabilities))
end

Capybara.configure do |config|
  config.javascript_driver = CHROME_HUB_URL.present? ? :container_chrome_headless : :selenium_chrome_headless
  config.match = :prefer_exact
  config.disable_animation = true
  config.always_include_port = true

  if CHROME_HUB_URL.present?
    config.server_host = "0.0.0.0"
    config.server_port = 4_000
    config.app_host = "http://web:4000"
  end
end
{% endhighlight %}

Готово! Можем приступать к запуску наших контейнеров

## запуск

1. Сбилдим используемые образы командой `docker-compose build`
2. Установим фронтенд зависимости командой `docker-compose run --rm web yarn`
3. Установим бэкенд зависимости командой `docker-compose run --rm web bundle install`
4. Настроим базу данных для дев-окружения командой `docker-compose run --rm web bin/rails db:setup`
5. Настроим базу данных для тест-окружения командой `docker-compose run --rm -e RAILS_ENV=test web bin/rails db:test:prepare`
6. Запустим наш проект командой `docker-compose up -d`
7. Проверим тесты - `docker-compose exec -e RAILS_ENV=test web bin/rspec`

Все, включая feature тесты должны работать внутри контейнеров. Пример проекта: [link](https://github.com/RuslanKhabibullin/stripe_test)
