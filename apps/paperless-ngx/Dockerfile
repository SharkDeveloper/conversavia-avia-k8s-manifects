FROM ghcr.io/paperless-ngx/paperless-ngx:latest

USER root

RUN apt-get update && \
    apt-get install -y --no-install-recommends tesseract-ocr-rus && \
    rm -rf /var/lib/apt/lists/*

# Патч бэкенда
COPY django_local_ru.po /usr/src/paperless/src/locale/ru_RU/LC_MESSAGES/django.po
RUN cd /usr/src/paperless/src/locale/ru_RU/LC_MESSAGES/ && \
    msgfmt django.po -o django.mo

# Патч фронтенда
RUN find /usr/src/paperless/src/documents/static/frontend/ru-RU \
    -type f -name '*.js' -exec env LC_ALL=C.UTF-8 sed -i \
        -e 's/Корреспондент/Шифр/g' \
        -e 's/Корреспонденты/Шифры/g' \
        -e 's/Корреспондента/Шифра/g' \
        -e 's/Корреспондентов/Шифров/g' \
        -e 's/Корреспонденту/Шифру/g' \
        -e 's/Корреспондентом/Шифром/g' \
        -e 's/Корреспонденте/Шифре/g' \
        {} +

# Удаляем предсжатые версии, чтобы nginx отдал пропатченный main.js
RUN find /usr/src/paperless/static/frontend/ru-RU \
         /usr/src/paperless/src/documents/static/frontend/ru-RU \
    -type f \( -name '*.js.br' -o -name '*.js.gz' \) -delete

USER 70
