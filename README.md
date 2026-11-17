# TrustTunnel + сайт-заглушка + Xray (WireGuard для российских сайтов)

Инструкция по развёртыванию собственного VPN-сервера на базе [TrustTunnel](https://github.com/TrustTunnel/TrustTunnel), который:

1. **Маскируется под обычный сайт** («Бухгалтерия, отчёты») на вашем домене.
2. **Направляет российские сайты через WireGuard** (выход в РФ), а весь остальной трафик пускает напрямую.

> ⚠️ **Перед публикацией репозитория** замените реальные домен, IP-адреса и ключи на плейсхолдеры (`example.com` и т. п.). Публичный репозиторий с вашим настоящим доменом сводит маскировку на нет. Приватные ключи и пароли в git не коммитьте никогда (см. раздел «Безопасность»).

---

## Содержание

- [Как это работает](#как-это-работает)
- [Требования](#требования)
- [Шаг 1. DNS и сертификат](#шаг-1-dns-и-сертификат)
- [Шаг 2. Сайт-заглушка](#шаг-2-сайт-заглушка)
- [Шаг 3. Установка TrustTunnel](#шаг-3-установка-trusttunnel)
- [Шаг 4. Настройка hosts.toml и vpn.toml](#шаг-4-настройка-hoststoml-и-vpntoml)
- [Шаг 5. Xray с WireGuard и маршрутизацией](#шаг-5-xray-с-wireguard-и-маршрутизацией)
- [Шаг 6. Подключение TrustTunnel к Xray](#шаг-6-подключение-trusttunnel-к-xray)
- [Шаг 7. Запуск и клиент](#шаг-7-запуск-и-клиент)
- [Проверка](#проверка)
- [Известные ограничения](#известные-ограничения)
- [Решение проблем](#решение-проблем)
- [Безопасность](#безопасность)
- [Рекомендуемая структура репозитория](#рекомендуемая-структура-репозитория)

---

## Как это работает

```
                          ┌──────────────────────────────── VPS ────────────────────────────────┐
                          │                                                                      │
  браузер ──https──►  :443 TrustTunnel                                                           │
  (проверка)              │   ├─ example.com          → reverse proxy → nginx 127.0.0.1:8080     │
                          │   │                          (сайт-заглушка «Бухгалтерия»)           │
                          │   └─ reports.example.com  → VPN-хост                                 │
                          │                                  │                                   │
  клиент TrustTunnel ──►  │                                  ▼                                   │
                          │                         SOCKS5 127.0.0.1:1081 (Xray)                 │
                          │                                  │                                   │
                          │                     ┌────────────┴────────────┐                      │
                          │          российские сайты                остальное                   │
                          │          (geosite/geoip ru)                  │                       │
                          │                 │                            ▼                       │
                          │                 ▼                        direct                      │
                          │        WireGuard (RU-сервер)                                         │
                          └──────────────────────────────────────────────────────────────────────┘
```

**Почему два имени.** В конфигурации TrustTunnel имя хоста может встречаться только в одной секции, поэтому сайт-заглушка и VPN живут на разных именах одного домена:

| Имя | Роль |
|---|---|
| `example.com` | сайт-заглушка (`reverse_proxy_hosts`) |
| `reports.example.com` | реальный VPN-хост (`main_hosts`) |

Оба имени указывают на один и тот же сервер и порт 443.

**Почему Xray между TrustTunnel и WireGuard.** Параметр `forward_protocol.socks5` в TrustTunnel отправляет *весь* туннелируемый трафик на один SOCKS5-адрес, выборочной маршрутизации у него нет. Поэтому он пересылает всё в локальный Xray, а Xray по правилам решает, что пустить через WireGuard, а что напрямую. Заодно WireGuard-выход Xray поддерживает и TCP, и UDP.

---

## Требования

- VPS с публичным IP (Linux, systemd), root-доступ.
- Домен (в примерах `example.com`) с возможностью добавлять A/AAAA-записи.
- Открытые порты: **443/tcp**, **443/udp** (QUIC), **80/tcp** (только для выпуска и продления сертификата).
- Рабочий WireGuard-сервер с выходом в РФ: его публичный ключ, `endpoint` и адрес вашего клиента в туннеле.
- Установленные `nginx`, `certbot`, `curl`.

---

## Шаг 1. DNS и сертификат

Создайте A (и при необходимости AAAA) записи для обоих имён на IP вашего VPS:

```
example.com          A   <IP сервера>
reports.example.com  A   <IP сервера>
```

Выпустите **один сертификат на оба имени** (порт 80 должен быть свободен):

```bash
sudo certbot certonly --standalone \
  -d example.com -d reports.example.com \
  --deploy-hook "systemctl restart trusttunnel"
```

Файлы окажутся в `/etc/letsencrypt/live/example.com/` (`fullchain.pem`, `privkey.pem`). Автопродление certbot настраивает сам, а `--deploy-hook` перезапускает TrustTunnel после обновления. Подробности по перезагрузке сертификатов смотрите в `CERT_RENEWAL.md` в репозитории TrustTunnel.

---

## Шаг 2. Сайт-заглушка

Nginx слушает **только loopback**, наружу его выставлять не нужно:

`/etc/nginx/conf.d/decoy.conf`
```nginx
server {
    listen 127.0.0.1:8080;
    server_name example.com;
    root /var/www/decoy;
    index index.html;
}
```

В `/var/www/decoy` положите настоящий статический сайт: главная, «Услуги», «Отчётность», «Контакты», `favicon.ico`, `robots.txt`. Чем правдоподобнее и «живее» сайт, тем лучше маскировка при ручной проверке.

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I http://127.0.0.1:8080
```

---

## Шаг 3. Установка TrustTunnel

```bash
curl -fsSL https://raw.githubusercontent.com/TrustTunnel/TrustTunnel/refs/heads/master/scripts/install.sh | sh -s -
cd /opt/trusttunnel
sudo ./setup_wizard
```

В мастере укажите:

- адрес прослушивания `0.0.0.0:443`;
- логин и пароль пользователя;
- сертификат: **«Provide path to existing certificate»**, пути из Шага 1;
- хост: `reports.example.com`.

---

## Шаг 4. Настройка hosts.toml и vpn.toml

**`/opt/trusttunnel/hosts.toml`**
```toml
[[main_hosts]]
hostname = "reports.example.com"
cert_chain_path = "/etc/letsencrypt/live/example.com/fullchain.pem"
private_key_path = "/etc/letsencrypt/live/example.com/privkey.pem"

[[reverse_proxy_hosts]]
hostname = "example.com"
cert_chain_path = "/etc/letsencrypt/live/example.com/fullchain.pem"
private_key_path = "/etc/letsencrypt/live/example.com/privkey.pem"
```

**`/opt/trusttunnel/vpn.toml`** (добавьте или раскомментируйте):
```toml
[reverse_proxy]
server_address = "127.0.0.1:8080"
path_mask = "/"

# Чтобы неавторизованные запросы не отдавали 407, как у прокси
non_connect_auth_failure_status_code = 404
```

`ping_enable` и `speedtest_enable` оставьте выключенными (значение по умолчанию).

> Описание параметров `path_mask` в документации краткое. Проверьте на практике, что заглушка открывается целиком (см. [Проверка](#проверка)). Если нет, попробуйте другое значение маски и смотрите логи.

---

## Шаг 5. Xray с WireGuard и маршрутизацией

### Установка

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

Конфиг: `/usr/local/etc/xray/config.json`. Для получения ключевой пары клиента можно использовать команду `xray wg`; публичный ключ клиента нужно добавить на WireGuard-сервер.

### Конфигурация

```json
{
  "log": { "loglevel": "warning" },
  "dns": {
    "servers": [
      {
        "address": "77.88.8.8",
        "domains": ["geosite:category-ru", "domain:ru", "domain:su", "domain:xn--p1ai"]
      },
      "1.1.1.1"
    ]
  },
  "inbounds": [
    {
      "tag": "from-tt",
      "listen": "127.0.0.1",
      "port": 1081,
      "protocol": "socks",
      "settings": { "auth": "noauth", "udp": true },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"],
        "routeOnly": true
      }
    }
  ],
  "outbounds": [
    { "tag": "direct", "protocol": "freedom" },
    {
      "tag": "wg-ru",
      "protocol": "wireguard",
      "settings": {
        "secretKey": "<ПРИВАТНЫЙ_КЛЮЧ_КЛИЕНТА>",
        "address": ["10.2.0.2/32"],
        "peers": [
          {
            "publicKey": "<ПУБЛИЧНЫЙ_КЛЮЧ_СЕРВЕРА_WG>",
            "endpoint": "ru-server.example:51820",
            "allowedIPs": ["0.0.0.0/0"],
            "keepAlive": 25
          }
        ],
        "mtu": 1380,
        "noKernelTun": true
      }
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "domain": ["geosite:category-ru", "domain:ru", "domain:su", "domain:xn--p1ai"],
        "outboundTag": "wg-ru"
      },
      { "type": "field", "ip": ["geoip:ru"], "outboundTag": "wg-ru" },
      { "type": "field", "network": "tcp,udp", "outboundTag": "direct" }
    ]
  }
}
```

### Что здесь важно

- **Порядок правил.** Сначала домены (`.ru`, `.su`, `.рф` в punycode и список `geosite:category-ru`), затем `geoip:ru` для российских IP у сайтов на других доменах. Всё остальное идёт напрямую.
- **`sniffing` + `routeOnly`.** По TLS/HTTP определяется домен, даже если TrustTunnel передал только IP. Исходный адрес назначения при этом сохраняется, домен нужен только для выбора маршрута.
- **DNS.** Для российских доменов используется российский резолвер (`77.88.8.8`), чтобы CDN отдавал российские узлы. Внутри WireGuard-туннеля Xray работает только с IP и сам резолвит имена встроенным DNS.
- **`noKernelTun: true`.** WireGuard работает в юзерспейсе и не меняет таблицы маршрутизации хоста. Название и поведение этого поля менялись между версиями Xray, поэтому проверьте конфиг командой ниже и при ошибке сверьтесь с документацией вашей версии.
- **`mtu`.** 1380 взят с запасом. При зависающих загрузках уменьшайте.
- **Списки.** `geosite:category-ru` входит в списки v2fly, но убедитесь, что он есть в вашем `geosite.dat`. При необходимости можно использовать расширенные российские списки из сторонних сборок (например, `russia-v2ray-rules-dat`), предварительно проверив, какие категории в них есть.

### Проверка и запуск

```bash
xray run -test -c /usr/local/etc/xray/config.json
sudo systemctl enable --now xray
```

Обновление геобаз: `bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install-geodata`.

---

## Шаг 6. Подключение TrustTunnel к Xray

В `/opt/trusttunnel/vpn.toml` замените блок `[forward_protocol]`:

```toml
[forward_protocol.socks5]
address = "127.0.0.1:1081"
extended_auth = false
```

Этот параметр применяется только после перезапуска сервиса.

---

## Шаг 7. Запуск и клиент

```bash
sudo cp /opt/trusttunnel/trusttunnel.service.template /etc/systemd/system/trusttunnel.service
sudo systemctl daemon-reload
sudo systemctl enable --now trusttunnel
```

Файрвол (пример для ufw):

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
```

Экспорт конфигурации клиента:

```bash
cd /opt/trusttunnel
./trusttunnel_endpoint vpn.toml hosts.toml -c myuser -a reports.example.com
```

Команда выдаёт ссылку `tt://...`, её можно вставить в приложение AdGuard TrustTunnel (iOS/Android) или превратить в QR-код. Для CLI-клиента добавьте `--format toml`.

---

## Проверка

**1. Заглушка и VPN-хост снаружи**

```bash
curl -I https://example.com            # должна открыться «Бухгалтерия»
curl -I https://reports.example.com    # не должно быть 407 и признаков прокси
curl -I --resolve example.com:443:<IP> https://example.com
```

Дополнительно откройте оба имени в браузере. Обратите внимание, что отдаёт `reports.example.com` при обычном заходе: пустая страница или ошибка могут выдать, что это не сайт.

**2. Xray отдельно от TrustTunnel**

```bash
curl --socks5-hostname 127.0.0.1:1081 https://2ip.ru       # IP RU-сервера WireGuard
curl --socks5-hostname 127.0.0.1:1081 https://ifconfig.me  # IP вашего VPS
```

**3. С клиента через TrustTunnel**

- российский сайт показывает российский IP;
- зарубежный сайт показывает IP VPS;
- работают DNS, QUIC (HTTP/3) и ping (см. ограничения ниже).

---

## Известные ограничения

- **UDP и ICMP через SOCKS5-forward.** В документации TrustTunnel не описано, как именно UDP и ICMP пересылаются через `forward_protocol.socks5`. Обязательно проверьте DNS, QUIC и ping с клиента. Если UDP не работает, ищите причину в этом звене.
- **Один набор правил на всех.** Через `forward_protocol.socks5` идёт весь трафик всех пользователей одного endpoint. Индивидуальных правил на пользователя нет; для этого потребуются отдельные экземпляры TrustTunnel.
- **`path_mask` и поведение reverse proxy** описаны в документации кратко, проверяйте реальное поведение.
- **WireGuard** не предназначен для обхода DPI: его трафик имеет узнаваемые признаки. Учитывайте это при выборе маршрута до RU-сервера.
- Конфигурации составлены по документации на момент написания. Параметры TrustTunnel и Xray меняются от версии к версии, сверяйтесь с `CONFIGURATION.md` (TrustTunnel) и [документацией Xray](https://xtls.github.io/).

---

## Решение проблем

| Симптом | Что проверить |
|---|---|
| Заглушка не открывается или открывается частично | `path_mask`, статус nginx на `127.0.0.1:8080`, логи TrustTunnel с уровнем debug |
| `reports.example.com` отвечает как прокси (407) | параметр `non_connect_auth_failure_status_code`, версия TrustTunnel |
| Российские сайты показывают зарубежный IP | порядок правил маршрутизации, наличие `geosite:category-ru` в `geosite.dat`, работа sniffing |
| Сайты через WireGuard зависают | уменьшите `mtu`, проверьте доступность `endpoint` по UDP |
| Ошибка Xray на поле `noKernelTun` | версия Xray, документация WireGuard-выхода для вашей версии |
| Не обновился сертификат | свободен ли порт 80, отработал ли `--deploy-hook`, `certbot renew --dry-run` |
| Не работает QUIC/UDP с клиента | открыт ли 443/udp в файрволе; см. раздел про ограничения SOCKS5-forward |

Полезные команды:

```bash
sudo systemctl status trusttunnel xray nginx
sudo journalctl -u trusttunnel -u xray -n 100 --no-pager
```

---

## Безопасность

- **Не коммитьте** приватные ключи WireGuard, пароли пользователей, ссылки `tt://...`, файлы `credentials.toml` и содержимое `/etc/letsencrypt`.
- Храните настоящие конфиги вне репозитория или в приватном репозитории; в публичном держите только `*.example`-шаблоны с плейсхолдерами.
- SOCKS5-порт Xray (`127.0.0.1:1081`) должен слушать **только loopback**, без внешнего доступа.
- Регулярно обновляйте TrustTunnel, Xray и геобазы.

Пример `.gitignore`:

```gitignore
# секреты и рабочие конфиги
*.key
*.pem
credentials.toml
config.json
vpn.toml
hosts.toml
*.tt
```

---

## Рекомендуемая структура репозитория

```
.
├── README.md
├── .gitignore
├── configs/
│   ├── hosts.toml.example
│   ├── vpn.toml.example
│   ├── xray-config.json.example
│   └── nginx-decoy.conf.example
└── decoy-site/
    ├── index.html
    ├── services.html
    ├── reports.html
    ├── contacts.html
    └── favicon.ico
```

---

## Ссылки

- TrustTunnel: <https://github.com/TrustTunnel/TrustTunnel>
- Документация Xray (WireGuard outbound): <https://xtls.github.io/config/outbounds/wireguard.html>
- Xray-install: <https://github.com/XTLS/Xray-install>
- Certbot: <https://certbot.eff.org/>
