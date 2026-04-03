# Установка Hiddify Manager на VPS + домен на FreeDNS

## Что это и зачем

Hiddify Manager — панель управления VPN-сервером с поддержкой 20+ протоколов:
VLESS + Reality, Hysteria2, TUIC, Shadowsocks и другие. Оптимизирована для
обхода блокировок в России, Китае и Иране.

GitHub: https://github.com/hiddify/Hiddify-Manager  
Клиентское приложение: https://github.com/hiddify/hiddify-app

---

## Требования к VPS

- ОС: Ubuntu 22.04 или 24.04 (чистая установка, без других сервисов)
- CPU: 1 ядро минимум
- RAM: 2 ГБ минимум (4 ГБ рекомендуется)
- Диск: 10 ГБ и выше
- Публичный IPv4-адрес
- SSH-доступ (root)
- Виртуализация: KVM

> ⚠️ Hiddify занимает порты 80 и 443 через HAProxy. Не устанавливай его
> на сервер где уже есть другие веб-сервисы или панели.

---

## Шаг 1. Доступ к серверу

Подключись по SSH. Если нет SSH-клиента — используй веб-браузер:
https://app.shellngn.com/ → введи IP, порт 22, логин root, пароль.

---

## Шаг 2. Установка Hiddify Manager

Выполни одну команду на сервере:

**Стабильная версия (release):**

```bash
bash <(curl -L https://i.hiddify.com/release)
```

**Бета-версия (новее, рекомендуется):**

```bash
bash <(curl -L https://i.hiddify.com/beta)
```

Установка занимает 5–15 минут. В конце выведет:

```
Admin panel URL: https://ВАШ_IP/СЕКРЕТНЫЙ_ПУТЬ/admin/
Login: admin
Password: СЛУЧАЙНАЯ_СТРОКА
```

**Сохрани эти данные сразу** — они нужны для входа в панель.

> После установки сервер некоторое время недоступен по прямому IP —
> это нормально. Hiddify включает режим маскировки: без знания секретного
> пути панель не открывается.

---

## Шаг 3. Бесплатный домен на FreeDNS

Домен нужен для стабильной работы Reality и для удобства — чтобы
не зависеть от IP при его смене.

### Регистрация

1. Перейди на https://freedns.afraid.org
2. Нажми **Sign Up Free** → заполни форму → подтверди email

### Создание поддомена

1. Войди в аккаунт
2. Перейди в **Subdomains** → **Add a subdomain**
3. Заполни поля:

| Поле | Значение |
|---|---|
| Type | A |
| Subdomain | любое имя (например: `vpn`, `proxy`, `myvpn`) |
| Domain | выбери из списка (`mooo.com`, `strangled.net`, `chickenkiller.com`) |
| Destination | IP-адрес твоего VPS |
| TTL | 300 |

4. Нажми **Save**

### Проверка что DNS обновился

Подожди 5–30 минут, затем проверь:

```bash
ping твойподдомен.mooo.com
```

Должен отвечать IP твоего сервера. Пока не резолвится — не переходи дальше.

---

## Шаг 4. Добавление домена в Hiddify

1. Открой панель по ссылке из установки
2. Перейди в **Domains → Create**
3. Заполни:

| Поле | Значение |
|---|---|
| Domain | полный поддомен (например `vpn.mooo.com`) |
| Alias | любое имя (например `Main`) |
| Mode | **Direct** (обязательно!) |
| Show in configs | включить |
| Force HTTPS | включить |

4. Нажми **Create → Apply Changes**

---

## Шаг 5. Настройка Reality

Reality — основной протокол для обхода блокировок. Маскирует трафик
под реальный HTTPS к популярному сайту.

1. Перейди в **Settings → Reality**
2. Включи Reality
3. Настройки:

| Параметр | Значение |
|---|---|
| Short IDs | оставь `auto` |
| uTLS fingerprint | `chrome` или `random` |
| Fallback domain | `www.microsoft.com` или `www.apple.com` |

4. Нажми **Apply Changes**

---

## Шаг 6. Включение протоколов

Перейди в **Settings → Proxy** и включи:

- **VLESS + Reality** — основной, самый устойчивый к блокировкам
- **Hysteria2** — быстрый, работает поверх UDP/QUIC
- **TUIC v5** — хорош для нестабильных соединений

Нажми **Apply Changes**.

---

## Шаг 7. Создание пользователей

1. Перейди в **Users → Create**
2. Заполни:

| Поле | Значение |
|---|---|
| Name | имя пользователя |
| Usage limit | `999999` ГБ |
| Package Days | `9999` |

3. Нажми **Save**

Для каждого человека создавай отдельного пользователя — так удобнее
отслеживать использование и при необходимости отключать доступ.

---

## Шаг 8. Подключение клиентов

### Скачать приложение Hiddify

| Платформа | Ссылка |
|---|---|
| Android | Google Play или GitHub Releases |
| Windows | GitHub Releases (Portable .zip) |
| macOS | GitHub Releases (.dmg) |
| iOS | App Store |

GitHub Releases: https://github.com/hiddify/hiddify-app/releases

### Подключиться

1. В панели Hiddify открой страницу пользователя
2. Скопируй **Subscription link** или отсканируй QR-код
3. В приложении Hiddify: **Add profile → Subscription → вставь ссылку → Update**
4. Выбери протокол (Reality или Hysteria2) и нажми **Connect**

---

## Обновление Hiddify

```bash
bash <(curl -L https://i.hiddify.com/beta)
```

Та же команда что и при установке — обнаружит существующую установку
и обновит до последней версии.

---

## Что делать если IP заблокировали

1. В панели провайдера смени IP сервера (или создай новый)
2. Обнови A-запись в FreeDNS: **Subdomains → Edit** → смени Destination на новый IP
3. Подожди 5–30 минут пока DNS обновится
4. Конфиги пользователей обновятся автоматически если они используют
   Subscription link с доменом (не IP)

---

## Полезные команды на сервере

```bash
# Статус всех сервисов Hiddify
cd /opt/hiddify-manager && bash hiddify.sh status

# Перезапуск
cd /opt/hiddify-manager && bash hiddify.sh restart

# Логи
cd /opt/hiddify-manager && bash hiddify.sh log
```

---

## Итоговая таблица

| Параметр | Значение |
|---|---|
| Установка | `bash <(curl -L https://i.hiddify.com/beta)` |
| Путь на сервере | `/opt/hiddify-manager` |
| Протоколы | VLESS+Reality, Hysteria2, TUIC v5 |
| Домен | FreeDNS (freedns.afraid.org) |
| Клиент | Hiddify App (все платформы) |
| Обновление | Та же команда установки |
