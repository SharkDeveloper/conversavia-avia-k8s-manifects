FROM ghcr.io/paperless-ngx/paperless-ngx:latest

USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends tesseract-ocr-rus && \
    rm -rf /var/lib/apt/lists/*
# Копируем наш отредактированный django.po
COPY django_local_ru.po /usr/src/paperless/src/locale/ru_RU/LC_MESSAGES/django.po

# Компилируем .po в .mo
RUN cd /usr/src/paperless/src/locale/ru_RU/LC_MESSAGES/ && \
    msgfmt django.po -o django.mo

USER 70
