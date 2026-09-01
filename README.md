# Общий стек колледжа

В этом каталоге находится единый Docker Compose для трёх приложений:

- `ais` — Django + Gunicorn + SQLite;
- `distantes` — Next.js + SQLite;
- `schedule-*` — существующий Express/Vite/PostgreSQL проект Shedule;
- `nginx` — единая внешняя точка входа на порту 80.

## Размещение на Ubuntu Server

Рекомендуемая структура:

```text
/opt/college/
├── college-stack/
├── AIS/ais-college/
├── distantes/

└── Shedule/
```

Скопируйте `.env.example` в `.env`, измените секреты и выполните из `college-stack`:

```bash
cp .env.example .env
# отредактировать .env
sudo docker compose config
sudo docker compose up -d --build
```

Для Distantes отдельный seed не требуется: проект использует собственную инициализацию SQLite.

Для проверки базы Distantes после первого запуска:

```bash
sudo docker compose logs -f distantes
```

Если в будущем в приложении Distantes появится команда seed, не запускайте её поверх важных данных без резервной копии.

## Имена приложений

В текущей конфигурации используются поддомены:

```text
ais.college.local       -> AIS

distantes.college.local -> приложение Distantes
schedule.college.local  -> Shedule
```

Для проверки в локальной сети добавьте IP сервера в `/etc/hosts` на клиентском ПК или настройте локальный DNS:

```text
192.168.1.50 ais.college.local distantes.college.local schedule.college.local
```

Для публикации в интернете нужен реальный домен, DNS-записи и доступный извне белый IP/проброс портов. Сам nginx публичную ссылку не создаёт. Если провайдер использует CGNAT, понадобится белый IP или туннель.

## Базы и данные

- AIS и Distantes намеренно остаются на отдельных SQLite-томах: это не смешивает несовместимые схемы.
- Shedule использует отдельный PostgreSQL-том `schedule_pgdata` и существующие SQL-скрипты проекта.
- Файлы AIS (`media`) и собранная статика сохраняются в именованных Docker-томах.
- Бэкап Shedule сохраняется в `college-stack/backups`.

## HTTPS

Сейчас Compose поднимает HTTP на `:80`, чтобы сначала проверить маршрутизацию в LAN. Для HTTPS добавьте сертификаты и отдельный `listen 443 ssl` в nginx-конфигурацию; для публичного домена сертификаты можно выпускать через Certbot или использовать другой TLS-терминатор.
