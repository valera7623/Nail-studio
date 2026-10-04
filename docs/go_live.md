# Касса в проде и пилот за вечер

Ключи и живые платежи — только после одобрения кассы. Секреты в git не класть.

ЮKassa нужна для **подписки мастера** (тариф Старт). Клиент визит в боте не оплачивает — деньги мастеру напрямую.

Предпочтительно **ЮKassa** (отдельный магазин под Nail-book / `nailbook.ru`, не shopId agentops/gameforge).

Webhook ЮKassa: `POST https://nailbook.ru/yookassa/webhook`

Webhook Prodamus (запас): `POST https://nailbook.ru/prodamus/webhook`

## 0. Ключи на VPS

Не копируйте `shopId` / секрет с agentops.com.ru или gameforge.website: у каждого магазина свой webhook.

В кабинете ЮKassa: название текущего магазина → **Добавить магазин** → «На сайте» → `https://nailbook.ru`. Затем Интеграция → HTTP-уведомления: `https://nailbook.ru/yookassa/webhook` (событие `payment.succeeded`). Интеграция → Ключи API → секретный ключ.

В `/home/valera/nail-studio/.env`:

```
YOOKASSA_SHOP_ID=...
YOOKASSA_SECRET_KEY=...
PAYMENT_PROVIDER=auto
PUBLIC_BASE_URL=https://nailbook.ru
HTTP_PORT=8090
BOT_USERNAME=nailstudiobot
```

`auto` включает ЮKassa, если ключи заданы, иначе Prodamus.

Запас Prodamus:

```
PRODAMUS_PAYFORM_URL=https://payform.ru/....
PRODAMUS_SECRET=...
PRODAMUS_SHOP_ID=...
```

Пересоздать контейнер, чтобы подтянуть env:

```
cd /home/valera/nail-studio && docker compose -f docker-compose.prod.yml up -d --force-recreate --no-build
```

Проверка: в боте `/admin` строка «Касса: ЮKassa». Пока ключей нет — «Касса: нет».

Права на базу (если раньше был `readonly database`):

```
sudo chown -R valera:valera /home/valera/nail-studio/data
chmod u+rwX /home/valera/nail-studio/data
chmod u+rw /home/valera/nail-studio/data/nail.db
```

## 1. Платёж на живом боте

1. **Подписка мастера.** `/studio` → «Тариф» → Старт → оплата → тариф в кабинете.
2. **Запись клиента (без кассы).** Своя студия → ссылка клиенту → дата → время. Внутри окна запись подтверждается сразу, оплата у мастера. Вне окна — заявка.
3. **Отмена.** Клиент `/my` → отмена. Возврата через кассу платформы нет: деньги за визит не проходили через ЮKassa.

Если мастер видит «оплата подписки пока недоступна» — ключей в процессе нет или контейнер не перечитали.

## 2. Пилот «шаблон за вечер»

До рассылки — одна реальная студия (своя или дружеская).

- Владелец за вечер: название → «Ссылка записи» → «Открыть окно».
- Клиент по ссылке выбирает дату и время; студия видит запись; приходит напоминание за 24 ч или 2 ч.
- Оплата визита — у мастера.

Живой бот: **`@nailstudiobot`**. Username берётся из токена (`getMe`); в `.env` на VPS:

```
BOT_TOKEN=...          # токен @nailstudiobot из BotFather
BOT_USERNAME=nailstudiobot
PUBLIC_BASE_URL=https://nailbook.ru
```

После смены токена пересоздать контейнер. В `/studio` заново открыть «Ссылка» — QR и `t.me/nailstudiobot?start=<slug>` обновятся.

## 3. Бэкап и восстановление SQLite

Планировщик в 03:15 MSK пишет `data/backups/nail-YYYYMMDD-HHMM.db` (online backup, не `cp` живого файла). На VPS:

```
ls -lt /home/valera/nail-studio/data/backups/ | head
```

Разовая копия без ожидания крона:

```
docker compose -f docker-compose.prod.yml exec bot python scripts/backup_sqlite.py
```

Restore (бот не пишет в файл во время копирования):

```
cd /home/valera/nail-studio
docker compose -f docker-compose.prod.yml stop bot
cp data/nail.db data/nail.before-restore.db
cp data/backups/nail-YYYYMMDD-HHMM.db data/nail.db
docker compose -f docker-compose.prod.yml start bot
```

Проверка: `/admin` открывается, заявки в `/studio` на месте.

Off-site: в `.env` задайте каталог на **другом диске** (не тот же `data/`):

```
BACKUP_OFFSITE_DIR=/offsite
```

В `docker-compose.prod.yml` раскомментируйте том `/mnt/nail-offsite:/offsite`. Планировщик после локальной копии делает `copy2` туда (те же 14 файлов). Если каталог недоступен — локальный бэкап всё равно пишется, ошибка в логе.

Разовая проверка:

```
docker compose -f docker-compose.prod.yml exec bot python scripts/backup_sqlite.py
ls -lt /mnt/nail-offsite | head
```

Не Cloud Supabase и не зарубежный S3: ПДн только в РФ.

## 4. iCal после смены BOT_TOKEN

Подпись ленты — `ICAL_FEED_SECRET` (или файл `data/.ical_secret`), не токен бота. Кабинет соло-мастера кнопку iCal не показывает; лента остаётся по прямой ссылке, если `PUBLIC_BASE_URL` задан.

## 5. Traefik (HTTPS)

GitHub Actions копирует `deploy/traefik.yml` в `/home/valera/traefik/dynamic/nail.yml`. Разово, если файла ещё нет:

```
sudo mkdir -p /home/valera/traefik/dynamic
sudo cp /home/valera/nail-studio/deploy/traefik.yml /home/valera/traefik/dynamic/nail.yml
```

Проверка: `https://nailbook.ru/` отдаёт лендинг Nail-book.

## 6. Дальше не код

Пилот-метрики вне репозитория: число студий Free vs Старт, заявки/окна, конверсия заявок.
