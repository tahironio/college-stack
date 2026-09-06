# Чистая установка Ubuntu Server и запуск проектов

Эта инструкция предназначена для ситуации, когда старая Ubuntu Server не запускается и нужно развернуть всё заново через GitHub.

Используются четыре репозитория:

- `tahironio/ais-college` — AIS, Django и Gunicorn;
- `tahironio/lampa-lms` — Distantes, Next.js и SQLite;
- `tahironio/schedule-app` — Shedule, Express/Vite и PostgreSQL;
- `tahironio/college-stack` — общий Docker Compose и nginx.

Пример LAN-адреса сервера: `192.168.1.10`.

> **Главное:** эта инструкция создаёт новые пустые Docker volumes. Старые базы и файлы сами не восстановятся. Если старый диск неисправен и резервных копий нет, восстановить старые данные невозможно. Код проектов находится в GitHub, но базы, `.env`, пароли и пользовательские файлы туда не входят.

> **Никогда не выполняйте** `docker compose down -v` на рабочем сервере: ключ `-v` удаляет volumes с данными.

---

## 1. Итоговая структура

```text
/opt/college/
├── ais-college/       # GitHub: tahironio/ais-college
├── distantes/         # GitHub: tahironio/lampa-lms
├── schedule-app/      # GitHub: tahironio/schedule-app
└── college-stack/     # GitHub: tahironio/college-stack
    ├── docker-compose.yml
    ├── .env
    ├── nginx/
    └── backups/
```

Общий Compose запускает семь сервисов:

```text
nginx
ais
distantes
schedule-frontend
schedule-backend
schedule-postgres
schedule-backup
```

Сначала наружу публикуется только HTTP:

```text
22/tcp — SSH
80/tcp — HTTP
```

Порты `3000`, `5432` и внутренние Docker-порты наружу не публикуются.

---

## 2. Что делать со старыми данными

Если старый сервер не запускается:

- не пытайтесь выполнять на нём команды из этой инструкции;
- не форматируйте старый диск повторно, если хотите попробовать восстановление;
- установите новую Ubuntu на новый или исправный диск;
- сначала запустите чистый стек;
- восстановление старых данных выполняйте только после успешного запуска и только из проверенной резервной копии.

Если позже найдётся старый диск, подключите его к новой Ubuntu вторым диском. Не копируйте неизвестные файлы поверх работающих volumes без резервной копии. Раздел 14 описывает общий порядок восстановления.

Если готовых резервных копий нет, продолжайте с пустыми базами. Пользователей и данные приложений придётся создать заново.

---

## 3. Установить Ubuntu Server

Рекомендуется Ubuntu Server 24.04 LTS.

### 3.1 Подготовить установочную флешку

На рабочем компьютере:

1. Скачайте ISO-образ **Ubuntu Server 24.04 LTS** с официального сайта Ubuntu.
2. Подключите флешку размером не менее 8 ГБ.
3. Запишите ISO на флешку через Rufus или Balena Etcher.
4. Перезагрузите сервер и откройте Boot Menu/BIOS.
5. Загрузитесь с USB-флешки.

> Установка на старый диск может удалить его разделы и данные. Если есть хотя бы шанс восстановить старые данные, сначала выключите сервер, выньте старый диск или подключите новый диск для чистой установки.

### 3.2 Параметры установщика

Во время установки:

1. Выберите обычную установку Ubuntu Server.
2. Имя сервера: например, `college`.
3. Создайте обычного пользователя, например `tagir`.
4. Включите компонент **OpenSSH Server**.
5. Не устанавливайте графическую оболочку без необходимости.
6. Подключите сервер к MikroTik кабелем, если это возможно.
7. Убедитесь, что выбран правильный диск. Для полностью чистой установки можно использовать автоматическое форматирование выбранного диска.
8. Оставьте достаточно места: минимум около 40 ГБ, больше — если будут документы и медиафайлы.

После первого входа через локальную консоль:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y ca-certificates curl git openssl unzip rsync ufw
sudo reboot
```

После перезагрузки войдите снова и настройте часовой пояс:

```bash
sudo timedatectl set-timezone Europe/Moscow
timedatectl
```

При необходимости задайте имя:

```bash
sudo hostnamectl set-hostname college
```

---

## 4. Настроить постоянный адрес сервера

MikroTik должен всегда выдавать Ubuntu один и тот же LAN-адрес, потому что NAT будет направлять запросы именно на него.

### Рекомендуемый способ: DHCP reservation на MikroTik

В DHCP-сервере MikroTik найдите MAC-адрес сетевой карты Ubuntu и создайте статическую аренду:

```text
IP-адрес: 192.168.1.10
```

На Ubuntu проверьте адреса:

```bash
ip -4 -br addr
ip route
hostname -I
```

Используйте адрес локальной сети, например `192.168.1.10`. Адреса `172.17.0.1`, `172.18.0.1` и похожие — это Docker-сети.

С Windows в обычной LAN/Wi-Fi-сети проверьте:

```powershell
ping 192.168.1.10
Test-NetConnection 192.168.1.10 -Port 22
```

Если Windows подключён к гостевой Wi-Fi-сети, устройства могут быть изолированы друг от друга. Используйте обычную сеть, не Guest Wi-Fi.

---

## 5. Настроить SSH

Не закрывайте локальную консоль или действующее SSH-соединение, пока новый способ входа не проверен.

Проверьте SSH:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
sudo ss -tlnp | grep ':22'
```

Ожидается прослушивание `0.0.0.0:22` и/или `[::]:22`.

### 5.1 Создать ключ на Windows

В PowerShell Windows:

```powershell
ssh-keygen -t ed25519
```

Показать публичный ключ:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

На Ubuntu:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Вставьте весь публичный ключ одной строкой, сохраните файл и выполните:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Проверьте вход по ключу из нового окна PowerShell:

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519 tagir@192.168.1.10
```

### 5.2 Запретить root и вход по паролю

Делайте это только после успешного входа по ключу.

На Ubuntu:

```bash
sudo nano /etc/ssh/sshd_config.d/99-college-hardening.conf
```

Добавьте:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Проверьте и перечитайте конфигурацию:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

Снова проверьте вход из нового окна Windows:

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519 tagir@192.168.1.10
```

Старую сессию закрывайте только после успешной проверки.

---

## 6. Настроить firewall Ubuntu

Сначала разрешите SSH, затем включите UFW:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS future'
sudo ufw enable
sudo ufw status verbose
```

Для SSH только из LAN можно заменить общее правило:

```bash
sudo ufw delete allow 22/tcp
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp comment 'LAN SSH'
```

Если SSH нужен из Интернета, оставьте порт разрешённым, но используйте только авторизацию по ключу и желательно внешний порт MikroTik `2222`, перенаправленный на внутренний порт `22`.

---

## 7. Установить Docker

Эти команды можно выполнять из любой папки:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Добавьте пользователя в группу Docker:

```bash
sudo usermod -aG docker "$USER"
sudo reboot
```

После повторного входа проверьте:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

Если появилась ошибка `permission denied`, выполните новый вход в систему и проверьте:

```bash
groups
sudo systemctl status docker --no-pager
```

Не устанавливайте одновременно Snap Docker и Docker из официального репозитория. Если Snap-версия уже установлена:

```bash
if snap list docker >/dev/null 2>&1; then
  sudo snap remove docker
fi
```

Команду удаления выполняйте только если `snap list` действительно показал установленный Docker.

---

## 8. Скачать четыре репозитория

Создайте каталог и назначьте его владельцем обычного пользователя:

```bash
sudo mkdir -p /opt/college
sudo chown -R "$USER:$USER" /opt/college
cd /opt/college
```

Клонируйте репозитории:

```bash
git clone https://github.com/tahironio/ais-college.git ais-college
git clone https://github.com/tahironio/lampa-lms.git distantes
git clone https://github.com/tahironio/schedule-app.git schedule-app
git clone https://github.com/tahironio/college-stack.git college-stack
```

Проверьте файлы:

```bash
ls -l /opt/college/ais-college/Dockerfile
ls -l /opt/college/distantes/Dockerfile
ls -l /opt/college/schedule-app/backend/Dockerfile
ls -l /opt/college/schedule-app/frontend/Dockerfile
ls -l /opt/college/college-stack/docker-compose.yml
```

Если Dockerfile отсутствует, обновите соответствующий репозиторий:

```bash
cd /opt/college/ais-college && git pull --ff-only
cd /opt/college/distantes && git pull --ff-only
cd /opt/college/schedule-app && git pull --ff-only
cd /opt/college/college-stack && git pull --ff-only
```

Если репозитории приватные, используйте SSH Deploy key GitHub, а не пароль в URL.

---

## 9. Создать и заполнить `.env`

Compose запускается из `college-stack`:

```bash
cd /opt/college/college-stack
cp .env.example .env
chmod 600 .env
nano .env
```

При структуре из этой инструкции оставьте пути:

```env
AIS_PATH=../ais-college
DISTANTES_PATH=../distantes
SCHEDULE_PATH=../schedule-app
```

Для локальной сети:

```env
DOMAIN=college.local
AIS_ALLOWED_HOSTS=ais.college.local,localhost,127.0.0.1
```

Для реального домена, например `example.ru`:

```env
DOMAIN=example.ru
AIS_ALLOWED_HOSTS=ais.example.ru,localhost,127.0.0.1
```

Сгенерируйте четыре разных значения:

```bash
openssl rand -hex 32
openssl rand -hex 32
openssl rand -hex 24
openssl rand -hex 32
```

Вставьте их сюда:

```env
AIS_SECRET_KEY=результат-1
DISTANTES_SESSION_SECRET=результат-2
POSTGRES_PASSWORD=результат-3
JWT_SECRET=результат-4
```

Остальные обязательные параметры:

```env
AIS_ALLOWED_HOSTS=ais.college.local,localhost,127.0.0.1
DPO_API_TOKEN=замени-на-токен-или-случайную-строку
AIS_GUNICORN_WORKERS=2

POSTGRES_DB=schedule_db
POSTGRES_USER=schedule
CORS_ORIGIN=*

BACKUP_HOUR=3
BACKUP_KEEP_DAYS=14
TZ=Europe/Moscow
```

Итоговый минимальный файл должен содержать:

```env
AIS_PATH=../ais-college
DISTANTES_PATH=../distantes
SCHEDULE_PATH=../schedule-app

DOMAIN=college.local
AIS_SECRET_KEY=случайный-секрет
DISTANTES_SESSION_SECRET=другой-случайный-секрет
POSTGRES_PASSWORD=случайный-пароль
JWT_SECRET=четвертый-случайный-секрет

AIS_ALLOWED_HOSTS=ais.college.local,localhost,127.0.0.1
DPO_API_TOKEN=случайная-строка
AIS_GUNICORN_WORKERS=2
POSTGRES_DB=schedule_db
POSTGRES_USER=schedule
CORS_ORIGIN=*
BACKUP_HOUR=3
BACKUP_KEEP_DAYS=14
TZ=Europe/Moscow
```

> Значения `случайный-секрет`, `другой-случайный-секрет` и остальные placeholders в этом примере нужно заменить результатами команд `openssl`. Не оставляйте placeholders в рабочем `.env`.

Правила:

- `.env` нельзя отправлять в GitHub;
- не используйте один и тот же секрет в разных параметрах;
- после инициализации PostgreSQL не меняйте `POSTGRES_PASSWORD` без отдельной смены пароля внутри базы;
- `CORS_ORIGIN=*` является разрешающей настройкой, при необходимости укажите точный origin;
- файл должен иметь права `600`.

Проверьте Compose без вывода секретов:

```bash
docker compose config --quiet
```

---

## 10. Запустить общий Compose

Если старый Shedule Compose когда-либо запускался отдельно, остановите только его контейнеры из каталога Shedule, без удаления volumes:

```bash
cd /opt/college/schedule-app
docker compose down
```

На чистом сервере этот шаг обычно не нужен.

Запустите общий стек строго из `college-stack`:

```bash
cd /opt/college/college-stack
docker compose up -d --build
```

Первая сборка может занять несколько минут.

Проверьте:

```bash
docker compose ps
```

Посмотрите логи:

```bash
docker compose logs --tail=100 nginx
docker compose logs --tail=100 ais
docker compose logs --tail=100 distantes
docker compose logs --tail=100 schedule-backend
docker compose logs --tail=100 schedule-postgres
docker compose logs --tail=100 schedule-backup
```

Ожидается, что все семь сервисов запущены, а PostgreSQL имеет статус `healthy`.

---

## 11. Проверить приложения на сервере

Проверка маршрутизации nginx по Host-заголовку:

```bash
curl -I -H 'Host: ais.college.local' http://127.0.0.1/
curl -I -H 'Host: distantes.college.local' http://127.0.0.1/
curl -I -H 'Host: schedule.college.local' http://127.0.0.1/
```

Ответ `200`, `301`, `302` или ответ приложения означает, что маршрут работает. `502 Bad Gateway` означает, что nginx не может достучаться до целевого сервиса.

Проверьте имена сервисов внутри Docker-сети:

```bash
docker compose exec nginx getent hosts ais distantes schedule-frontend schedule-backend
```

Проверьте опубликованные порты:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
sudo ss -tulpn
```

На host обычно должны быть опубликованы SSH `22` и nginx `80`. PostgreSQL `5432` наружу публиковаться не должен.

---

## 12. Настроить имена на Windows-клиенте

Для `college.local` нужны записи в hosts или локальный DNS.

Запустите **Командную строку от имени администратора**:

```cmd
notepad C:\Windows\System32\drivers\etc\hosts
```

В конец файла добавьте одну строку:

```text
192.168.1.10 ais.college.local distantes.college.local schedule.college.local
```

Сохраните файл именно как `hosts`, не `hosts.txt`.

Если Windows пишет, что нет разрешения на сохранение, Notepad запущен не от имени администратора.

Очистите DNS-кэш:

```cmd
ipconfig /flushdns
```

Откройте:

```text
http://ais.college.local
http://distantes.college.local
http://schedule.college.local
```

На Linux-клиенте аналогичная запись добавляется в `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

---

## 13. Настроить MikroTik RouterOS v6

Пути подключения различаются:

```text
Из LAN:       ssh tagir@192.168.1.10
Из Интернета: ssh -p 2222 tagir@PUBLIC_IP
```

### 13.1 Проверить внешний IP

На Ubuntu:

```bash
curl -4 ifconfig.me
```

В MikroTik сравните этот адрес с WAN-адресом роутера. Если WAN-адрес находится в диапазонах ниже, это не прямой публичный IPv4:

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
100.64.0.0/10
```

Это может означать CGNAT или наличие ещё одного роутера провайдера.

### 13.2 Проброс SSH внешнего порта 2222

В MikroTik определите фактический WAN-интерфейс. В вашей предыдущей конфигурации использовался `pppoe-out1`; замените его, если имя другое.

NAT-правило:

```routeros
/ip firewall nat add chain=dstnat in-interface=pppoe-out1 protocol=tcp dst-port=2222 action=dst-nat to-addresses=192.168.1.10 to-ports=22 comment="college SSH"
```

Разрешающее правило в `forward` должно находиться выше финального `drop`:

```routeros
/ip firewall filter add chain=forward in-interface=pppoe-out1 protocol=tcp connection-nat-state=dstnat dst-address=192.168.1.10 dst-port=22 action=accept comment="allow college SSH"
```

Проверяйте из внешней сети, например через мобильный Интернет:

```powershell
ssh -p 2222 -i $env:USERPROFILE\.ssh\id_ed25519 tagir@PUBLIC_IP
```

Не проверяйте публичный IP из той же LAN, если hairpin NAT не настроен.

### 13.3 Проброс HTTP

После успешной проверки LAN добавьте:

```routeros
/ip firewall nat add chain=dstnat in-interface=pppoe-out1 protocol=tcp dst-port=80 action=dst-nat to-addresses=192.168.1.10 to-ports=80 comment="college HTTP"
/ip firewall filter add chain=forward in-interface=pppoe-out1 protocol=tcp connection-nat-state=dstnat dst-address=192.168.1.10 dst-port=80 action=accept comment="allow college HTTP"
```

Не открывайте наружу порты `3000`, `5432` и Docker bridge-порты.

---

## 14. Восстановление данных, если найдётся резервная копия

Восстановление выполняйте только после успешного запуска чистого стека и проверки, что резервная копия действительно читается.

### 14.1 PostgreSQL

Положите SQL-дамп в `/opt/college/college-stack/backups/` и выполните:

```bash
cd /opt/college/college-stack
cat backups/schedule-YYYY-MM-DD.sql \
  | docker compose exec -T schedule-postgres \
      sh -c 'psql -U "$POSTGRES_USER" "$POSTGRES_DB"'
```

Если дамп содержит команды создания ролей или базы, сначала изучите его содержимое. Не импортируйте непроверенный дамп в рабочую базу.

### 14.2 SQLite

Остановите приложения:

```bash
cd /opt/college/college-stack
docker compose stop ais distantes
```

Проверьте имена volumes:

```bash
docker volume ls | grep college-stack
```

Пример восстановления AIS из архива:

```bash
sudo docker run --rm \
  -v college-stack_ais_db:/target \
  -v "$PWD/backups:/backup:ro" \
  alpine sh -c 'rm -rf /target/* && tar xzf /backup/ais-db-YYYY-MM-DD.tar.gz -C /target'
```

Не заменяйте volume без проверенной копии текущего состояния. Distantes восстанавливается аналогично из правильного архива.

Запустите приложения:

```bash
docker compose up -d ais distantes
```

---

## 15. Бэкапы и обновления

Сервис `schedule-backup` пишет PostgreSQL-бэкапы в:

```text
/opt/college/college-stack/backups/
```

Проверка:

```bash
cd /opt/college/college-stack
docker compose logs --tail=100 schedule-backup
ls -lh backups/
```

Бэкапы на том же диске не защищают от поломки диска. Регулярно копируйте `backups` на другой компьютер или внешний диск.

Перед обновлением:

```bash
cd /opt/college/college-stack
docker compose exec -T schedule-postgres \
  sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' \
  > backups/schedule-before-update-$(date +%F-%H%M).sql
```

Обновите код:

```bash
cd /opt/college/ais-college && git pull --ff-only
cd /opt/college/distantes && git pull --ff-only
cd /opt/college/schedule-app && git pull --ff-only
cd /opt/college/college-stack && git pull --ff-only
```

Проверьте и пересоберите:

```bash
docker compose config --quiet
docker compose up -d --build
docker compose ps
```

Не используйте для обычного обновления:

```bash
git reset --hard
docker compose down -v
```

---

## 16. Частые ошибки

### `docker: command not found`

Docker не установлен или пользователь ещё не перезашёл после добавления в группу:

```bash
docker --version
docker compose version
groups
sudo systemctl status docker --no-pager
```

### `failed to read dockerfile: open Dockerfile: no such file or directory`

Проверьте пути и наличие файлов:

```bash
cd /opt/college/college-stack
docker compose config | grep -A4 -E 'build:|context:'
ls -la /opt/college/ais-college/Dockerfile
ls -la /opt/college/distantes/Dockerfile
ls -la /opt/college/schedule-app/backend/Dockerfile
ls -la /opt/college/schedule-app/frontend/Dockerfile
```

Затем обновите репозитории через `git pull --ff-only`.

### Контейнер завершается

```bash
docker compose ps -a
docker compose logs --tail=200 ИМЯ_СЕРВИСА
```

Причины: ошибка приложения, отсутствующая переменная `.env`, права на volume или занятый порт.

### nginx выдаёт `502 Bad Gateway`

```bash
docker compose ps
docker compose logs --tail=200 nginx ais distantes schedule-backend
docker compose exec nginx getent hosts ais distantes schedule-backend schedule-frontend
```

Внутри nginx нельзя использовать `localhost` для других контейнеров. Используйте имена Compose-сервисов.

### Порт 80 занят

```bash
sudo ss -tulpn | grep ':80'
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Остановите старый Caddy, Apache, nginx или отдельный Shedule Compose. На host-порту 80 должен работать только один reverse proxy.

### SSH из LAN зависает или закрывается

На Ubuntu:

```bash
ip -4 -br addr
sudo ss -tlnp | grep ':22'
sudo ufw status verbose
sudo journalctl -fu ssh.service
```

На Windows:

```powershell
ipconfig
Test-NetConnection 192.168.1.10 -Port 22
ssh -4 -vvv tagir@192.168.1.10
```

Проверьте, что Windows находится в обычной LAN, а не в гостевой Wi-Fi, адрес сервера правильный и не занят другим устройством.

### Публичный SSH не подключается

Проверьте:

1. публичный IP назначен именно MikroTik;
2. в NAT указан правильный WAN-интерфейс;
3. NAT направляет запрос на `192.168.1.10`;
4. правило `accept` в `forward` находится выше финального `drop`;
5. UFW разрешает внутренний порт 22;
6. проверка выполняется из внешней сети;
7. внешний порт совпадает с SSH-командой.

На Ubuntu можно смотреть входящие пакеты:

```bash
sudo tcpdump -ni any tcp port 22
```

Если пакетов нет — проблема до Ubuntu, в MikroTik или у провайдера. Если пакеты есть — проверяйте UFW и журнал SSH.

### Имена сайтов не открываются

Добавьте записи в hosts или настройте DNS. nginx выбирает приложение по Host-заголовку.

### После сборки пропали данные

`docker compose up -d --build` обычно не удаляет volumes. Проверьте:

```bash
docker volume ls | grep college-stack
docker compose config --volumes
```

Не используйте `docker compose down -v`.

### Закончился диск

```bash
df -h
docker system df
```

Можно удалить неиспользуемые образы после проверки:

```bash
docker image prune
```

Не выполняйте `docker volume prune`: volumes содержат базы и файлы приложений.

---

## 17. HTTPS и публичный домен

Текущая конфигурация запускает HTTP на порту 80. Для публичного HTTPS нужны дополнительные изменения:

1. реальный домен и DNS-записи;
2. проброс портов 80 и 443 на MikroTik;
3. получение сертификатов;
4. подключение сертификатов к nginx;
5. `listen 443 ssl` и перенаправление HTTP на HTTPS;
6. настройки trusted origins в Django и URL приложений.

Одного проброса порта 443 недостаточно: текущий nginx не настроен на TLS.

Не передавайте реальные пароли через публичный HTTP. Сначала настройте HTTPS.

---

## 18. Финальная проверка

- [ ] Ubuntu обновлена.
- [ ] У сервера постоянный LAN-адрес.
- [ ] SSH по ключу работает из нового окна.
- [ ] Root-вход отключён.
- [ ] Парольный SSH-вход отключён после проверки ключа.
- [ ] UFW разрешает нужные порты.
- [ ] Docker и Docker Compose установлены.
- [ ] Клонированы все четыре репозитория.
- [ ] Dockerfile AIS и Distantes существуют.
- [ ] `.env` создан, заполнен и имеет права `600`.
- [ ] `docker compose config --quiet` проходит без ошибки.
- [ ] Все семь сервисов запущены.
- [ ] PostgreSQL имеет статус `healthy`.
- [ ] Проверки nginx через `curl` работают.
- [ ] hosts или DNS настроены на клиенте.
- [ ] Все три сайта открываются из LAN.
- [ ] PostgreSQL не опубликован наружу.
- [ ] SSH NAT проверен из внешней сети.
- [ ] Бэкапы хранятся вне сервера.
- [ ] HTTPS настроен до передачи реальных данных через Интернет.
- [ ] `.env`, ключи, сертификаты, базы и backups не отправлены в GitHub.
