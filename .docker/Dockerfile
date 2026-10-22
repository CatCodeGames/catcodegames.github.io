FROM ruby:3.3-slim

# Устанавливаем системные зависимости для компиляции гемов
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    git \
    && rm -rf /var/lib/apt/lists/*

# Гемы ставятся отдельно от сайта: Gemfile лежит в .docker, а не в корне репозитория
WORKDIR /gems
COPY Gemfile Gemfile.lock ./
ENV BUNDLE_GEMFILE=/gems/Gemfile
RUN bundle install

# Исходники сайта не копируются в образ, а подключаются томом из compose.yaml
WORKDIR /site

EXPOSE 4000 35729

# Готовый сайт пишется в /tmp/site внутри контейнера, чтобы _site не появлялась в папке проекта.
# _config.dev.yml добавляет локальные настройки поверх основного конфига.
CMD ["bundle", "exec", "jekyll", "serve", "--host", "0.0.0.0", "--livereload", "--force_polling", \
     "--destination", "/tmp/site", "--config", "_config.yml,.docker/_config.dev.yml"]
