# Strapi + PostgreSQL + nginx в Docker

https://github.com/lostdiablodream/strapi/blob/main/strapi-docker.tar.gz

Лабораторная работа на Strapi 5.62.0, Node.js 22 и PostgreSQL 16. Браузер открывает статический фронтенд nginx, который обращается к публичному Strapi API. Strapi хранит записи в PostgreSQL, а загруженные файлы — в отдельном named volume.

## Быстрый запуск

Требуются Docker с Docker Compose и, только для локального этапа, Node.js 22+.

1. Создайте `.env` из шаблона и замените все секреты на собственные случайные значения:

   ```bash
   cp .env.example .env
   ```

2. Создайте внешнюю сеть и запустите сервисы:

   ```bash
   docker network create lab-strapi-network
   docker compose up -d --build
   docker compose ps
   ```

   Если сеть уже существует, первую команду пропускайте. Для автоматического создания шести демонстрационных статей, трёх рубрик и изображения в новой пустой БД первый запуск можно выполнить так:

   ```bash
   SEED_DEMO_DATA=true docker compose up -d --build
   ```

3. Откройте:
   - фронтенд: http://localhost:8080;
   - Strapi API: http://localhost:1337/api/articles?populate=*;
   - админку: http://localhost:1337/admin.

На новой БД создайте администратора в админке. Права Public `find` и `findOne` для обеих коллекций выставляются при запуске приложения.

## Модель и контент

`Article` содержит обязательное уникальное поле `title`, Rich text `content`, числовой `rating`, дату `published_on`, Enumeration `type`, одиночное изображение `cover` и связь many-to-one с `Category`. У `Category` есть обязательные уникальные `name` и `slug`, описание и обратная связь one-to-many.

Демонстрационное заполнение выполняется только при `SEED_DEMO_DATA=true` и только если таблица статей пуста. Файл `database/seed/tube-mic.jpg` переиспользован из прошлой практики. После первичного заполнения оставьте `SEED_DEMO_DATA=false`, чтобы проверка удаления томов показывала действительно пустую БД.

## Проверка PostgreSQL

В одиночном контейнере этапа 1 используется имя `lab-postgres`, поэтому команда одинакова до и после перехода на Compose:

```bash
docker exec lab-postgres psql -U <DATABASE_USERNAME> -d <DATABASE_NAME> -c '\dt'
docker exec lab-postgres psql -U <DATABASE_USERNAME> -d <DATABASE_NAME> -c 'SELECT id, title, rating, type FROM articles;'
docker exec lab-postgres psql -U <DATABASE_USERNAME> -d <DATABASE_NAME> -c 'SELECT * FROM files;'
```

Таблица `articles_category_lnk` хранит связи статей с рубриками, `files` — метаданные изображений, а сами файлы — в `public/uploads` и томе `lab-uploads`.

## Персистентность

Пересоздание контейнеров без удаления томов:

```bash
docker compose down
docker compose up -d
```

Записи, администратор и изображения сохраняются в `lab-pgdata` и `lab-uploads`.

Удаление контейнеров и томов:

```bash
docker compose down -v
docker compose up -d
```

`down -v` удаляет данные БД и загруженные файлы. После него Strapi снова предложит создать администратора, а пользовательский контент будет отсутствовать, если не включать seed. Не используйте эту команду с нужными данными.

## Локальный этап

Для воспроизведения пути «локальный Strapi → контейнерный PostgreSQL» остановите Compose, создайте отдельный контейнер с проброшенным портом `5432:5432` и тем же именованным томом, установите зависимости и запустите проект:

```bash
cd strapi-app
npm ci
SEED_DEMO_DATA=true npm run develop
```

В `strapi-app/.env` для локального запуска `DATABASE_HOST=127.0.0.1`; внутри Compose приложение использует hostname `postgres`.

## Полезные команды

```bash
docker compose logs -f strapi
docker compose restart strapi
docker compose build
docker images lab-strapi:1.0
docker compose down
```

Все пароли и секреты Strapi подставляются в Compose через `${...}` из `.env`; `.env` исключён из Docker-контекста и Git.

Если корневой `.env` ещё не создан, для уже настроенного экземпляра из этого каталога команды можно выполнять с явным файлом генератора: `docker compose --env-file strapi-app/.env ps` (и аналогично для `up`, `down`, `logs`). Для нового развёртывания используйте собственный корневой `.env` из шаблона. Не меняйте логин или пароль PostgreSQL на существующем инициализированном томе: смена переменной не обновляет учётную запись внутри уже созданной БД.
