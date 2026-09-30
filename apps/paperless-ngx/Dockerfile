FROM ghcr.io/paperless-ngx/paperless-ngx:latest

USER root

RUN apt-get update && \
    apt-get install -y --no-install-recommends tesseract-ocr-rus && \
    rm -rf /var/lib/apt/lists/*

# Патч бэкенда (как у тебя было)
COPY django_local_ru.po /usr/src/paperless/src/locale/ru_RU/LC_MESSAGES/django.po
RUN cd /usr/src/paperless/src/locale/ru_RU/LC_MESSAGES/ && \
    msgfmt django.po -o django.mo

# Патч фронтенда: заменяем строки прямо в скомпилированных JS-бандлах
RUN find /usr/src/paperless/static/frontend/ru-RU \
         /usr/src/paperless/src/documents/static/frontend/ru-RU \
    -type f -name '*.js' -exec sed -i \
        -e 's/Корреспондент/Шифр/g' \
        -e 's/Корреспонденты/Шифры/g' \
        -e 's/Корреспондента/Шифра/g' \
        -e 's/Корреспондентов/Шифров/г' \
        -e 's/Корреспонденту/Шифру/g' \
        -e 's/Корреспондентом/Шифром/g' \
        -e 's/Корреспонденте/Шифре/g' \
        {} +

USER 70
