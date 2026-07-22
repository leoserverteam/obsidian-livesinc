# Obsidian LiveSync self-host

Самохостинг синхронизации Obsidian через плагин
[obsidian-livesync](https://github.com/vrtmrz/obsidian-livesync) на базе CouchDB,
за reverse-proxy nginx на домене `obsidian.leonet.site`.

## Состав

- `docker-compose.yml` — контейнер CouchDB + одноразовый init-контейнер,
  создающий системные базы (`_users`, `_replicator`, `_global_changes`).
- `couchdb/local.ini` — конфигурация CouchDB (CORS, лимиты размера документа под вложения).
- `nginx/obsidian.leonet.site.conf` — конфиг сайта nginx (443 + редирект с 80),
  проксирует на CouchDB `127.0.0.1:5984`. Сертификаты подключаются через
  `include snippets/sslpath.conf;` — путь/файл сертификатов настраивается отдельно
  в общей конфигурации nginx на сервере.
- `env.sample` — шаблон переменных окружения.

## Деплой на сервере

1. Склонировать репозиторий на сервер, например в `/opt/obsidian-livesync`.
2. Создать `.env` на основе `env.sample`:

   ```bash
   cp env.sample .env
   # отредактировать COUCHDB_USER / COUCHDB_PASSWORD
   ```

3. Поднять контейнеры:

   ```bash
   docker compose up -d
   ```

   При первом запуске `couchdb-init` создаст системные базы данных и завершится
   (это нормально, статус `Exited (0)`).

4. Подключить конфиг nginx. Скопировать/симлинкнуть файл в конфигурацию nginx на хосте:

   ```bash
   ln -s /opt/obsidian-livesync/nginx/obsidian.leonet.site.conf /etc/nginx/sites-enabled/obsidian.leonet.site.conf
   ```

   Убедиться, что в `/etc/nginx/snippets/sslpath.conf` прописаны реальные пути до
   сертификата и ключа (`ssl_certificate`, `ssl_certificate_key`), т.к. этот
   файл переиспользуется несколькими конфигами.

5. Проверить конфиг и перезагрузить nginx:

   ```bash
   nginx -t && systemctl reload nginx
   ```

6. Проверить, что CouchDB отвечает:

   ```bash
   curl -u admin:пароль https://obsidian.leonet.site/
   ```

   Должен вернуться JSON вида `{"couchdb":"Welcome",...}`.

## Настройка плагина Obsidian LiveSync

В плагине (Remote Database Configuration) указать:

- **URI**: `https://obsidian.leonet.site`
- **Username** / **Password**: значения из `.env`
- **Database name**: любое имя (например, `obsidian`), плагин создаст базу автоматически

После этого включить синхронизацию и повторить настройку на других устройствах,
указав те же данные подключения.

## Обновление CouchDB

```bash
docker compose pull
docker compose up -d
```

## Бэкап

Данные CouchDB лежат в `./couchdb/data` (примонтировано как volume) —
достаточно бэкапить эту директорию (контейнер можно не останавливать,
CouchDB устойчив к бэкапу "на горячую", но для консистентности рекомендуется
использовать `docker compose stop couchdb` перед копированием при критичных бэкапах).
