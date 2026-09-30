# Этап 1: Сборка фронтенда с изменёнными переводами
FROM docker.io/node:24-trixie-slim AS compile-frontend

# Копируем исходники фронтенда из официального репозитория
# (нужно склонировать репозиторий Paperless-ngx в контекст сборки)
COPY ./paperless-ngx/src-ui /src/src-ui

# Копируем наш изменённый файл перевода поверх оригинала
COPY ./custom_locale/messages.ru_RU.xlf /src/src-ui/src/locale/messages.ru_RU.xlf

WORKDIR /src/src-ui
RUN npm install -g pnpm && pnpm install && pnpm run build

# Этап 2: Основной образ
FROM ghcr.io/paperless-ngx/paperless-ngx:latest

USER root

# Русский язык для OCR
RUN apt-get update && \
    apt-get install -y --no-install-recommends tesseract-ocr-rus && \
    rm -rf /var/lib/apt/lists/*

# Копируем собранную статику с изменёнными переводами
COPY --from=compile-frontend /src/src-ui/dist/ /usr/src/paperless/static/frontend/

# Патчим django.po (бэкенд)
COPY custom_locale/django_local_ru.po /usr/src/paperless/src/locale/ru_RU/LC_MESSAGES/django.po
RUN cd /usr/src/paperless/src/locale/ru_RU/LC_MESSAGES/ && msgfmt django.po -o django.mo

USER 70
