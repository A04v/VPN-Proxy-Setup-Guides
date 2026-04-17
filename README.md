# VPN & Proxy Setup Guides

Гайды по настройке прокси и VPN для обхода блокировок.

---

## Гайды

### MTProxy для Telegram

| Файл | Описание | Статус |
|------|----------|--------|
| [mtproxy-fake-tls.md](mtproxy-fake-tls.md) | MTProxy (mtg, Go) + Fake TLS. Простая установка через Docker. Требует хостера с прямым доступом до DC Telegram. | резерв |
| [mtproxy-zig-wireguard.md](mtproxy-zig-wireguard.md) | MTProxy (mtproto.zig) + WireGuard туннель. Решение для хостеров которые блокируют DC Telegram — трафик идёт через зарубежный VPS. | **актуальный** |

### VPN

| Файл | Описание |
|------|----------|
| [hiddify-setup.md](hiddify-setup.md) | Hiddify — панель управления VPN (VLESS+Reality, Hysteria2, TUIC) |
| [amneziawg-setup.md](amneziawg-setup.md) | AmneziaWG — модифицированный WireGuard с маскировкой трафика |

---

## Архитектура (апрель 2026)

Прямое подключение MTProxy к DC Telegram с российских хостеров заблокировано.
Рабочая схема:

```
Пользователь (РФ)
    ↓
MTProxy на российском VPS (порт 443, Fake TLS)
    ↓
WireGuard туннель
    ↓
Зарубежный VPS (NAT)
    ↓
Telegram DC (Амстердам)
```

### Проверка хостера перед установкой

```bash
# На сервере где планируется MTProxy
for ip in 149.154.167.51 149.154.175.51 149.154.167.41 149.154.175.100 91.108.56.130; do
  echo -n "$ip: "
  timeout 3 bash -c "echo > /dev/tcp/$ip/443" 2>&1 && echo "TCP OK" || echo "fail"
done
```

- Все `TCP OK` → хостер пропускает, можно ставить mtg без туннеля
- Есть `fail` → нужен туннель, смотри [mtproxy-zig-wireguard.md](mtproxy-zig-wireguard.md)

---

## Инфраструктура

| Сервер | Роль | Хостер | Локация |
|--------|------|--------|---------|
| VPS A | MTProxy (клиентский вход) | Selectel | Санкт-Петербург |
| VPS B | WireGuard выход + Hiddify VPN | Aeza | Швеция |
