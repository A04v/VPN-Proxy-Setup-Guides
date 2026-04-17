# MTProxy (mtproto.zig) + WireGuard туннель

Гайд для случая когда хостер блокирует исходящий трафик до DC Telegram.
Решение: поднять MTProxy на российском VPS, а трафик до DC пустить через WireGuard туннель на зарубежный VPS.

```
Пользователь (РФ) → MTProxy / Selectel (РФ) → WireGuard → VPS за рубежом → Telegram DC
```

---

## Требования

- **VPS A** — российский, для MTProxy (клиенты подключаются сюда)
- **VPS B** — зарубежный, для WireGuard-выхода (должен иметь TCP доступ до DC Telegram)

### Проверка VPS B перед установкой

```bash
for ip in 149.154.167.51 149.154.175.51 149.154.167.41 149.154.175.100 91.108.56.130; do
  echo -n "$ip: "
  timeout 3 bash -c "echo > /dev/tcp/$ip/443" 2>&1 && echo "TCP OK" || echo "fail"
done
```

Все должны быть `TCP OK`. Иначе VPS B не подходит.

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

> **Важно:** правило iptables и маршрут не сохраняются после перезагрузки.
> Добавь их в `/etc/rc.local` или используй `iptables-persistent`.

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
ping -c 3 10.99.0.1   # должен отвечать
ping -I wg1 -c 3 149.154.167.51   # DC Telegram через туннель
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
awg show awg0          # туннель, transfer должен расти
ip route get 149.154.167.51 mark 200   # должен показать dev awg0
```

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

## Диагностика

### Проверка статуса

```bash
mtbuddy status
journalctl -u mtproto-proxy -f
```

### Метрики в логах

```
conn stats: active=X/512 hs_inflight=Y accepted+=Z closed+=W
```

- `active` — активные соединения
- `hs_inflight` — handshake в процессе (если много и не падает — хорошо)
- `hs_timeout` в строке drops — handshake не завершился (проблема с DC)

### Если `hs_timeout` растёт — проверить туннель

```bash
# Трафик через туннель
awg show awg0

# Пинг DC через туннель с VPS A
ping -I awg0 -c 3 149.154.167.51

# Прослушать трафик на VPS B
tcpdump -i wg1 -n icmp
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
