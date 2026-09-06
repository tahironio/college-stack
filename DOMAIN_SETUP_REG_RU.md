# Подключение домена Reg.ru к проектам college-stack

Эта инструкция описывает, как направить реальный домен, зарегистрированный в Reg.ru, на три проекта на Ubuntu Server:

- `ais.ВАШ_ДОМЕН` → AIS;
- `distantes.ВАШ_ДОМЕН` → Distantes;
- `schedule.ВАШ_ДОМЕН` → Shedule.

Пример ниже использует:

```text
Основной домен: college-example.ru
Публичный IP:   109.168.145.74
IP сервера LAN: 192.168.1.10
WAN MikroTik:   pppoe-out1
```

Замените эти значения на свои. В DNS указывают **публичный IP**, а в MikroTik — **локальный IP Ubuntu-сервера**.

> Важно: Reg.ru отвечает за регистрацию домена и DNS-зону. Reg.ru не подключает контейнеры автоматически. Схема работает так: DNS направляет поддомен на публичный IP → MikroTik передаёт порты на Ubuntu → nginx выбирает контейнер по имени поддомена.

---

## 1. Что должно быть готово

До изменения DNS проверьте:

- домен оплачен и находится в вашем личном кабинете Reg.ru;
- Ubuntu Server доступен в локальной сети;
- Docker Compose запускает `college-stack` без ошибок;
- у Ubuntu постоянный LAN-адрес, например `192.168.1.10`;
- публичный IPv4 действительно принадлежит вашему MikroTik;
- провайдер не использует CGNAT;
- вы можете зайти на сервер по SSH из внешней сети;
- на MikroTik есть доступ к настройкам NAT и firewall.

Проверка стека на Ubuntu:

```bash
cd /opt/college/college-stack
docker compose ps
sudo ss -tulpn
```

В текущей конфигурации nginx публикует только HTTP:

```text
Ubuntu: 80/tcp → nginx
```

Текущий Compose **ещё не настроен на HTTPS**. Поэтому сначала подключите домен по HTTP и проверьте маршрутизацию. HTTPS будет отдельным этапом.

---

## 2. Определить правильный DNS-сервис домена

В Reg.ru откройте личный кабинет и найдите:

```text
Домены → нужный домен → Управление → DNS-серверы и управление зоной
```

Перед добавлением записей посмотрите, какие DNS-серверы назначены домену.

### Вариант A — DNS управляется в личном кабинете Reg.ru

Если указаны:

```text
ns1.reg.ru
ns2.reg.ru
```

или DNS-серверы Reg.ru для зоны, записи добавляются в личном кабинете Reg.ru. Переходите к разделу 3.

### Вариант B — DNS управляется хостингом Reg.ru

Если указаны:

```text
ns1.hosting.reg.ru
ns2.hosting.reg.ru
```

записи могут добавляться в разделе DNS хостинга. Откройте управление DNS-записями именно у этого хостинга.

### Вариант C — указаны DNS другого провайдера

Если стоят DNS-серверы Cloudflare, другого хостинга или другого регистратора, записи Reg.ru не используются. Добавлять A-записи нужно в панели того DNS-провайдера, чьи NS указаны у домена.

> Нельзя просто добавить запись в Reg.ru и ожидать результата, если домен делегирован на чужие DNS-серверы.

Официальная инструкция Reg.ru: [настройка ресурсных записей в личном кабинете](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/nastroyka-resursnykh-zapisey-dns/nastroyka-resursnykh-zapisey-v-lichnom-kabinete).

---

## 3. Добавить A-записи поддоменов в Reg.ru

В карточке домена откройте управление DNS-зоной и нажмите **Добавить запись**.

Для каждого проекта создайте отдельную A-запись.

### Запись для AIS

```text
Тип:       A
Поддомен:  ais
IP Address: 109.168.145.74
TTL:       значение по умолчанию
```

Результат:

```text
ais.college-example.ru → 109.168.145.74
```

### Запись для Distantes

```text
Тип:       A
Поддомен:  distantes
IP Address: 109.168.145.74
TTL:       значение по умолчанию
```

Результат:

```text
distantes.college-example.ru → 109.168.145.74
```

### Запись для Shedule

```text
Тип:       A
Поддомен:  schedule
IP Address: 109.168.145.74
TTL:       значение по умолчанию
```

Результат:

```text
schedule.college-example.ru → 109.168.145.74
```

Итоговая DNS-зона должна содержать примерно следующее:

| Тип | Поддомен | Значение |
|---|---|---|
| A | `ais` | `109.168.145.74` |
| A | `distantes` | `109.168.145.74` |
| A | `schedule` | `109.168.145.74` |

Не добавляйте к IP порт:

```text
Правильно:   109.168.145.74
Неправильно: 109.168.145.74:80
```

DNS не хранит информацию о порте. Все три поддомена могут указывать на один IP, потому что nginx различает их по заголовку `Host`.

> Не создавайте одновременно A- и CNAME-запись для одного и того же поддомена. Для собственного сервера здесь нужна A-запись.

Reg.ru указывает, что обновление ресурсных записей обычно занимает от 15 минут до 1 часа, а после смены DNS-серверов может занимать до 24 часов.

---

## 4. Настроить `.env` на Ubuntu

Подключитесь к серверу по SSH и откройте environment-файл общего Compose:

```bash
cd /opt/college/college-stack
nano .env
```

Укажите ваш домен без `http://`, `https://` и без имени поддомена:

```env
DOMAIN=college-example.ru
```

Для AIS укажите реальный поддомен:

```env
AIS_ALLOWED_HOSTS=ais.college-example.ru,localhost,127.0.0.1
```

Проверьте, что пути соответствуют фактическим каталогам:

```env
AIS_PATH=../ais-college
DISTANTES_PATH=../distantes
SCHEDULE_PATH=../schedule-app
```

Минимальный фрагмент `.env` для реального домена:

```env
AIS_PATH=../ais-college
DISTANTES_PATH=../distantes
SCHEDULE_PATH=../schedule-app

DOMAIN=college-example.ru
AIS_ALLOWED_HOSTS=ais.college-example.ru,localhost,127.0.0.1

AIS_SECRET_KEY=ваш-случайный-секрет
DISTANTES_SESSION_SECRET=другой-случайный-секрет
POSTGRES_PASSWORD=пароль-postgresql
JWT_SECRET=секрет-jwt
DPO_API_TOKEN=ваш-токен-или-временное-значение

POSTGRES_DB=schedule_db
POSTGRES_USER=schedule
CORS_ORIGIN=*
TZ=Europe/Moscow
```

Если проект Shedule использует HTTPS-origin в своих настройках, позже замените `CORS_ORIGIN=*` на точный адрес фронтенда:

```env
CORS_ORIGIN=https://schedule.college-example.ru
```

Не публикуйте `.env` в GitHub:

```bash
chmod 600 .env
grep -E '^(DOMAIN|AIS_ALLOWED_HOSTS|AIS_PATH|DISTANTES_PATH|SCHEDULE_PATH)=' .env
```

Последняя команда выводит только безопасные настройки, но не запускайте `cat .env` и не отправляйте его содержимое в чат или GitHub.

---

## 5. Пересоздать конфигурацию nginx

Шаблон nginx использует переменную `DOMAIN` и сам создаёт такие server names:

```text
ais.${DOMAIN}
distantes.${DOMAIN}
schedule.${DOMAIN}
```

Проверьте Compose и пересоберите только nginx:

```bash
cd /opt/college/college-stack
docker compose config --quiet
docker compose up -d --force-recreate nginx
```

Если проектный код тоже обновлялся:

```bash
docker compose up -d --build
```

Проверьте сгенерированную конфигурацию внутри контейнера:

```bash
docker compose exec nginx nginx -T
```

В выводе должны присутствовать:

```text
server_name ais.college-example.ru;
server_name distantes.college-example.ru;
server_name schedule.college-example.ru;
```

Если там осталось `college.local`, значит `.env` был изменён не в `/opt/college/college-stack` или контейнер nginx не был пересоздан.

---

## 6. Настроить проброс портов на MikroTik

DNS направляет запросы на публичный IP, но входящий трафик должен попасть с MikroTik на Ubuntu.

В этой схеме:

```text
Интернет: 109.168.145.74:80
       ↓
MikroTik: WAN-интерфейс pppoe-out1
       ↓
Ubuntu: 192.168.1.10:80
       ↓
Docker nginx
```

### 6.1 NAT для HTTP

В терминале MikroTik RouterOS v6 выполните:

```routeros
/ip firewall nat add chain=dstnat in-interface=pppoe-out1 protocol=tcp dst-port=80 action=dst-nat to-addresses=192.168.1.10 to-ports=80 comment="college HTTP"
```

Если в firewall есть финальное правило `drop` для цепочки `forward`, добавьте разрешение выше него:

```routeros
/ip firewall filter add chain=forward in-interface=pppoe-out1 protocol=tcp connection-nat-state=dstnat dst-address=192.168.1.10 dst-port=80 action=accept comment="allow college HTTP"
```

Проверьте в WinBox/WebFig:

```text
IP → Firewall → NAT
```

У правила должны быть:

```text
Chain:        dstnat
In. Interface: pppoe-out1
Protocol:     tcp
Dst. Port:    80
Action:       dst-nat
To Addresses: 192.168.1.10
To Ports:     80
```

Если WAN-интерфейс называется не `pppoe-out1`, используйте фактическое имя интерфейса.

### 6.2 HTTPS-порт пока не добавлять

Текущий `docker-compose.yml` содержит:

```yaml
ports:
  - "80:80"
```

Порт `443` и сертификаты пока не подключены. Не добавляйте проброс 443 только ради того, чтобы “порт был открыт”: nginx пока не умеет принимать HTTPS.

После отдельной настройки TLS потребуется:

```text
MikroTik 443 → Ubuntu 192.168.1.10:443
```

---

## 7. Дождаться DNS и проверить записи

Проверяйте DNS с Windows:

```powershell
nslookup ais.college-example.ru
nslookup distantes.college-example.ru
nslookup schedule.college-example.ru
```

В ответе должен быть публичный IP:

```text
109.168.145.74
```

Также можно использовать PowerShell:

```powershell
Resolve-DnsName ais.college-example.ru -Type A
Resolve-DnsName distantes.college-example.ru -Type A
Resolve-DnsName schedule.college-example.ru -Type A
```

Проверяйте именно из внешней сети: например, с телефона через мобильный Интернет. Не подключайтесь к Wi-Fi сервера во время теста публичного адреса.

На внешнем компьютере:

```powershell
Test-NetConnection ais.college-example.ru -Port 80
Test-NetConnection distantes.college-example.ru -Port 80
Test-NetConnection schedule.college-example.ru -Port 80
```

Затем откройте в браузере:

```text
http://ais.college-example.ru
http://distantes.college-example.ru
http://schedule.college-example.ru
```

Для проверки без браузера:

```bash
curl -I http://ais.college-example.ru/
curl -I http://distantes.college-example.ru/
curl -I http://schedule.college-example.ru/
```

Ожидаемый ответ может быть `200`, `301` или `302`. Ответ `502 Bad Gateway` означает, что DNS и nginx уже достигнуты, но целевой контейнер не отвечает.

---

## 8. Почему все три проекта используют один IP

Это нормально:

```text
ais.college-example.ru       → 109.168.145.74
distantes.college-example.ru → 109.168.145.74
schedule.college-example.ru  → 109.168.145.74
```

Nginx получает запросы на одном порту и смотрит на имя хоста:

```text
Host: ais.college-example.ru       → контейнер ais
Host: distantes.college-example.ru → контейнер distantes
Host: schedule.college-example.ru  → контейнер schedule-frontend
```

Нельзя направить DNS на `192.168.1.10`: это приватный адрес, он недоступен из Интернета.

---

## 9. HTTPS — следующий обязательный этап

После проверки HTTP не передавайте реальные пароли и персональные данные через Интернет без HTTPS.

Для трёх поддоменов нужен сертификат, покрывающий:

```text
ais.college-example.ru
distantes.college-example.ru
schedule.college-example.ru
```

Можно использовать:

- один сертификат с тремя SAN-именами;
- wildcard-сертификат `*.college-example.ru`.

Текущая конфигурация не содержит необходимых частей для TLS:

- нет публикации `443:443` в Compose;
- нет volume с сертификатами;
- нет `listen 443 ssl` в nginx;
- нет HTTP→HTTPS redirect;
- нет ACME location для автоматического выпуска сертификата.

Поэтому **не выполняйте вслепую** команды Certbot из чужих инструкций: сертификат может выпуститься, но nginx-контейнер не будет знать, где находятся ключи.

После успешного HTTP-теста нужно отдельно изменить:

1. `docker-compose.yml` — добавить `443:443` и volume сертификатов;
2. `nginx/default.conf.template` — добавить HTTPS server blocks;
3. конфигурацию выпуска и продления сертификатов;
4. MikroTik — пробросить TCP 443;
5. `DOMAIN` и `AIS_ALLOWED_HOSTS` оставить с реальным доменом;
6. настройки приложений — trusted origins, cookies и CORS.

Официальная документация Reg.ru по DNS находится [здесь](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/nastroyka-resursnykh-zapisey-dns/nastroyka-resursnykh-zapisey-v-lichnom-kabinete), а общий раздел по установке SSL — [здесь](https://help.reg.ru/support/ssl-sertifikaty/3-etap-ustanovka-ssl-sertifikata/).

---

## 10. Частые проблемы

### В Reg.ru нет кнопки добавления DNS-записи

Проверьте DNS-серверы домена. Если используются NS другого провайдера, редактируйте DNS там. Если домен недавно делегирован на другие NS, дождитесь обновления до 24 часов.

### `nslookup` показывает старый IP

Возможные причины:

- DNS-кэш Windows или роутера;
- TTL старой записи;
- DNS-запись изменена не у того провайдера;
- домен делегирован на другие DNS-серверы;
- зона ещё не обновилась.

Очистите кэш Windows:

```cmd
ipconfig /flushdns
```

Проверяйте также с мобильного Интернета и через несколько публичных DNS-проверок.

### DNS правильный, но сайт не открывается

Проверьте по порядку:

```bash
cd /opt/college/college-stack
docker compose ps
sudo ss -tulpn | grep ':80'
docker compose logs --tail=100 nginx
```

На MikroTik проверьте счётчики NAT-правила. Если счётчик не увеличивается, запрос не попадает на правило: неправильный WAN-интерфейс, другой публичный IP или нет входящего firewall-разрешения.

### Открывается 404 от nginx

Проверьте `DOMAIN` и пересоздайте nginx:

```bash
cd /opt/college/college-stack
grep '^DOMAIN=' .env
docker compose up -d --force-recreate nginx
docker compose exec nginx nginx -T
```

404 обычно означает, что заголовок Host не совпал с `server_name`.

### Открывается `502 Bad Gateway`

Проверьте целевой контейнер:

```bash
cd /opt/college/college-stack
docker compose ps
docker compose logs --tail=200 ais distantes schedule-backend
```

Проверьте DNS-имена внутри Docker-сети:

```bash
docker compose exec nginx getent hosts ais distantes schedule-frontend schedule-backend
```

### Из LAN домен не открывается, а с телефона открывается

Это может быть отсутствие hairpin NAT на MikroTik. Для локальной сети используйте локальный DNS или hosts:

```text
192.168.1.10 ais.college-example.ru distantes.college-example.ru schedule.college-example.ru
```

Публичный DNS при этом можно оставить с IP `109.168.145.74`.

### Внешний SSH работает, а сайт нет

SSH и HTTP используют разные NAT-правила. Наличие правила для TCP 2222/22 не означает, что HTTP настроен. Нужны отдельные правила для TCP 80 и позже TCP 443.

### Провайдер блокирует входящие порты

Если DNS правильный, но на MikroTik нет входящих пакетов, проверьте правила провайдера и тип подключения. Если используется CGNAT, обычный port forwarding не сработает. Тогда нужен белый IPv4, VPN/tunnel или внешний reverse proxy.

---

## 11. Финальный чек-лист

- [ ] Домен активен в Reg.ru.
- [ ] DNS-записи редактируются именно у текущего DNS-провайдера.
- [ ] `ais` A-запись указывает на публичный IP.
- [ ] `distantes` A-запись указывает на публичный IP.
- [ ] `schedule` A-запись указывает на публичный IP.
- [ ] На Ubuntu в `.env` указано `DOMAIN=ваш-домен.ru`.
- [ ] `AIS_ALLOWED_HOSTS` содержит реальный AIS-поддомен.
- [ ] Nginx пересоздан после изменения `.env`.
- [ ] MikroTik направляет TCP 80 на `192.168.1.10:80`.
- [ ] Правило `forward accept` стоит выше финального `drop`.
- [ ] Все три адреса проверены из внешней сети.
- [ ] PostgreSQL и внутренние Docker-порты не открыты наружу.
- [ ] HTTPS настроен до использования реальных персональных данных.
- [ ] `.env`, ключи и сертификаты не отправлены в GitHub.
