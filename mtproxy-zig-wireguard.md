# MTProxy (mtproto.zig) + WireGuard туннель

Гайд для случая когда хостер блокирует исходящий трафик до DC Telegram.
Решение: поднять MTProxy на российском VPS, а трафик до DC пустить через WireGuard туннель на зарубежный VPS.

```
Пользователь (РФ) → MTProxy / Selectel (РФ) → WireGuard → VPS за рубежом → Telegram DC
```

---

## Требования

- **VPS A** — российский, для MTProxy (клиенты подключаются сюда). Желательно с публичным IPv6 /64 — это сильно повышает стабильность (см. раздел про IPv6 hopping).
- **VPS B** — зарубежный, для WireGuard-выхода (должен иметь TCP доступ до DC Telegram)

### Проверка VPS B перед установкой

```bash
for ip in 149.154.167.51 149.154.175.51 149.154.167.41 149.154.175.100 91.108.56.130; do
  echo -n "$ip: "
  timeout 3 bash -c "echo > /dev/tcp/$ip/443" 2>&1 && echo "TCP OK" || echo "fail"
done
```

Все должны быть `TCP OK`. Иначе VPS B не подходит.

> **Важно:** TCP доступ не гарантирует что MTProto трафик пройдёт. Некоторые хостеры (например Aeza Швеция) пропускают TCP но режут MTProto через DPI. В этом случае прокси будет работать нестабильно — подключается, держится несколько часов, потом DPI обучается и начинает резать.

---

## Шаг 1 — WireGuard на VPS B (зарубежный)

```bash
apt install -y wireguard

# Генерация ключей
wg genkey | tee /root/wg-server-private | wg pubkey > /root/wg-server-public
cat /root/wg-server-private   # запиши
cat /root/wg-server-public    # запиши
```

Создай конфиг `/etc/wireguard/wg1.conf`:

```ini
[Interface]
PrivateKey = <ПРИВАТНЫЙ_КЛЮЧ_VPS_B>
Address = 10.99.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <ПУБЛИЧНЫЙ_КЛЮЧ_VPS_A>
AllowedIPs = 10.99.0.2/32
```

```bash
# IP forwarding
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
sysctl -p

# Запуск
systemctl enable --now wg-quick@wg1
```

### NAT на VPS B

Узнай внешний интерфейс:
```bash
ip route | grep default
# например: default via 10.0.0.1 dev enp0s3
```

Добавь NAT (подставь свой интерфейс вместо `enp0s3`):
```bash
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

Добавь маршрут обратно в туннель:
```bash
ip route add 10.99.0.2/32 dev wg1
```

Сохрани правила чтобы не пропали после перезагрузки:
```bash
apt install -y iptables-persistent
iptables-save > /etc/iptables/rules.v4

# Маршрут через crontab
echo "@reboot ip route add 10.99.0.2/32 dev wg1" >> /etc/crontab
```

---

## Шаг 2 — WireGuard на VPS A (российский, MTProxy)

```bash
apt install -y wireguard

# Генерация ключей
wg genkey | tee /root/wg-client-private | wg pubkey > /root/wg-client-public
cat /root/wg-client-private   # запиши
cat /root/wg-client-public    # передай на VPS B в [Peer]
```

Создай конфиг `/etc/wireguard/wg1.conf`:

```ini
[Interface]
PrivateKey = <ПРИВАТНЫЙ_КЛЮЧ_VPS_A>
Address = 10.99.0.2/24
Table = off

[Peer]
PublicKey = <ПУБЛИЧНЫЙ_КЛЮЧ_VPS_B>
Endpoint = <IP_VPS_B>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

> `Table = off` — важно! Без этого WireGuard перехватит весь трафик сервера.

```bash
systemctl enable --now wg-quick@wg1
```

### Проверка туннеля

```bash
ping -c 3 10.99.0.1                    # VPS B через туннель
ping -I awg0 -c 3 149.154.167.51       # DC Telegram через туннель
```

Если пинг DC не проходит — скорее всего нет маршрута на VPS B:
```bash
# На VPS B
ip route add 10.99.0.2/32 dev wg1
```

---

## Шаг 3 — Установка mtproto.zig на VPS A

```bash
# Установка mtbuddy
curl -fsSL https://raw.githubusercontent.com/sleep3r/mtproto.zig/main/deploy/bootstrap.sh | sudo bash

# Установка прокси
mtbuddy install --port 443 --domain ya.ru --yes
```

> Если порт 443 занят Docker-контейнером от старой установки:
> ```bash
> docker stop mtproto-proxy && docker rm mtproto-proxy
> systemctl restart mtproto-proxy
> ```

### Проверка работы ядра AmneziaWG (нужен для туннеля)

```bash
# Проверь версию ядра
uname -r

# Установи заголовки и dkms модуль
apt install -y linux-headers-generic linux-image-generic
# если ядро старое — перезагрузись: reboot

apt install -y amneziawg-dkms
dkms install amneziawg/1.0.0 -k $(uname -r)
modprobe amneziawg

# Проверка
lsmod | grep amnezia
```

---

## Шаг 4 — Подключение туннеля к mtproto.zig

```bash
mtbuddy setup tunnel /etc/wireguard/wg1.conf
```

Команда автоматически:
- устанавливает AmneziaWG
- настраивает policy routing (SO_MARK=200 → table 200 → awg0)
- обновляет конфиг прокси (`[upstream].type = "tunnel"`)
- перезапускает сервис

### Проверка

```bash
mtbuddy status
awg show awg0                              # туннель, transfer должен расти
ip route get 149.154.167.51 mark 200       # должен показать dev awg0
```

---

## Итоговый конфиг прокси

`/opt/mtproto-proxy/config.toml`:

```toml
[server]
port = 443
max_connections = 512
idle_timeout_sec = 300
handshake_timeout_sec = 60

[upstream]
type = "tunnel"

[upstream.tunnel]
interface = "awg0"

[censorship]
tls_domain = "ya.ru"
mask = true
desync = false
drs = true
fast_mode = true
mask_port = 8443

[general]
use_middle_proxy = false

[access.users]
user = "<32-hex-секрет>"
```

> **Не добавляй `log_level = "debug"`** в конфиг — это вызывает высокую нагрузку на CPU из-за mutex contention в логгере.

---

## Ссылка для подключения

После установки mtbuddy показывает ссылку:

```
tg://proxy?server=<IP_VPS_A>&port=443&secret=ee<секрет>
```

Или смотри в конфиге:
```bash
cat /opt/mtproto-proxy/config.toml
```

---

## IPv6 hopping — лучшее решение против блокировок

Если VPS A имеет публичный IPv6 /64 префикс — настрой автоматическую ротацию адресов. ТСПУ не блокирует IPv6 /64 целиком из-за риска collateral damage. При обнаружении блокировки скрипт автоматически меняет IPv6 адрес и обновляет DNS через Cloudflare API.

Нужен домен на Cloudflare с AAAA записью.

```bash
mtbuddy ipv6-hop --auto --prefix <твой_IPv6_/64_префикс> --threshold 5
```

Клиенты автоматически подхватывают новый адрес при следующем DNS lookup — без смены ссылки прокси.

---

## Диагностика

### Проверка статуса

```bash
mtbuddy status
journalctl -u mtproto-proxy -f
```

### Метрики в логах

```
conn stats: active=X/512 hs_inflight=Y accepted+=Z closed+=W
drops: hs_timeout+=N
```

- `active` — активные соединения
- `hs_inflight` — handshake в процессе
- `hs_timeout` — handshake не завершился (проблема с DC или DPI режет)
- `relay c2s failed` в debug логах — клиент оборвал соединение (NAT timeout на роутере или DPI)

### Если `hs_timeout` растёт

```bash
# Трафик через туннель
awg show awg0

# Пинг DC через туннель с VPS A
ping -I awg0 -c 3 149.154.167.51

# Прослушать трафик на VPS B — должны видеть пакеты до DC
tcpdump -i wg1 -n 'dst net 149.154.0.0/16 or dst net 91.108.0.0/16'

# Проверить что NAT работает на VPS B
iptables -t nat -L POSTROUTING -n -v | grep enp0s3
```

### Если прокси работал и внезапно перестал

ТСПУ периодически усиливает блокировки волнами — может не работать несколько часов, потом восстанавливается. Это нормально для апреля 2026. Проверь:

```bash
# Доступен ли IP снаружи (должен ответить CN=ya.ru — Nginx маскировка)
curl -v https://<IP_VPS_A>:443 --max-time 5

# Живой ли туннель
ping -I awg0 -c 3 149.154.167.51

# Все DC через туннель
for ip in 149.154.167.51 149.154.175.51 149.154.167.41 149.154.175.100 91.108.56.130; do
  echo -n "$ip: "
  ping -I awg0 -c 2 -W 3 $ip 2>/dev/null | tail -1
done
```

### Полезные команды

```bash
# Статус сервиса
systemctl status mtproto-proxy

# Перезапуск
systemctl restart mtproto-proxy

# Обновление mtproto.zig
mtbuddy update

# Конфиг прокси
cat /opt/mtproto-proxy/config.toml
```

---

## Известные проблемы

### `error: AddressInUse` при старте
Порт 443 занят другим процессом (обычно старый Docker-контейнер):
```bash
ss -tlnp | grep 443
docker stop mtproto-proxy && docker rm mtproto-proxy
systemctl restart mtproto-proxy
```

### `Error: Unknown device type` для awg0
Не загружен модуль ядра AmneziaWG:
```bash
modprobe amneziawg
```

### Модуль не собирается (`kernel headers not found`)
```bash
apt install -y linux-headers-$(uname -r)
# если пакет не найден — ядро устарело:
apt install -y linux-headers-generic linux-image-generic && reboot
```

### Пинг через туннель идёт но `hs_timeout` всё равно есть
Ответы от DC не возвращаются — нет маршрута на VPS B:
```bash
# На VPS B
ip route add 10.99.0.2/32 dev wg1
```

### VPS B режет MTProto через DPI
Симптом: TCP до DC проходит, curl даёт `SSL_ERROR_SYSCALL`, прокси работает несколько часов потом перестаёт. DPI на хостере обучился и начал резать MTProto внутри туннеля. Решение — найти другой VPS B где этого нет, или настроить IPv6 hopping на VPS A.

### Соединение обрывается через ~30 секунд на WiFi
Симптом: на мобильном интернете работает, на WiFi отваливается. Причина — NAT timeout на домашнем роутере. Частично решается увеличением `idle_timeout_sec = 300` в конфиге. Полное решение — IPv6 на VPS A.

### `Using AWG endpoint IPv4 for middle-proxy NAT translation: <IP_VPS_B>`
Известный баг mtproto.zig в tunnel режиме — прокси берёт IP из AWG endpoint вместо `public_ip` из конфига. Не влияет на работу в режиме `use_middle_proxy = false`.
