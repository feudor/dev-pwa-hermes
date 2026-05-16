# Установка: Nginx + Cloudflare — открыть приложение только для 1 внешнего IP

## Что это
Файл-шаблон инструкций (и конфиг-шаблон) для случая:
- Frontend PWA обслуживается через **Nginx**
- перед Nginx стоит **Cloudflare**
- нужно разрешить доступ **только** с одного внешнего IP-адреса пользователя

Тут нет секретов: IP и host — обычные настройки.

---

## 0) Что подготовить
1) Найди свой внешний IP-адрес (тот, с которого ты обычно заходишь).
   - Впиши его в конфиг ниже вместо `__YOUR_EXTERNAL_IP__`.
2) Убедись, что Cloudflare проксирует трафик на этот сервер.
3) Убедись, что в Nginx корректно настроен `real_ip_header`, чтобы Nginx видел реальный IP клиента (а не IP Cloudflare).

---

## 1) Nginx: real IP из Cloudflare
В конфиге (обычно в `http { ... }` блоке) добавь:

```nginx
# --- Cloudflare real client IP ---
real_ip_header CF-Connecting-IP;

# Cloudflare IP ranges (нужно обновлять периодически)
# Источник: https://www.cloudflare.com/ips/
# Пример — подставь актуальные ranges.
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 104.16.0.0/13;
set_real_ip_from 104.24.0.0/14;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 131.0.72.0/22;
```

> ВАЖНО: Cloudflare меняет/расширяет диапазоны. В идеале подгружать автоматически или обновлять вручную.

Проверь, что после этого `ngx_http_realip_module` реально работает (для дебага можно временно логировать `$remote_addr`).

---

## 2) Nginx: allowlist только для твоего IP
Дальше создай отдельный конфиг-серверный блок/локацию для PWA.

Например, файл: `/etc/nginx/conf.d/pwa-hermes-2026-05.conf`

### Вариант: ограничить доступ к всему приложению по `location /`
```nginx
server {
  # server_name — твой домен/поддомен
  server_name __YOUR_DOMAIN__;

  # Разрешить только твой внешний IP
  allow __YOUR_EXTERNAL_IP__;
  deny  all;

  location / {
    # Если PWA статика — укажи root/alias
    # root /var/www/pwa-hermes;

    # Если proxy на backend
    # proxy_pass http://127.0.0.1:PORT;

    # пример: proxy
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

> Примечание: `allow/deny` будет сравнивать **$remote_addr**, которое мы “подменили” на real client IP через `real_ip_header CF-Connecting-IP`.

---

## 3) Проверка
1) Проверь конфиг:
```bash
sudo nginx -t
```
2) Перезагрузи Nginx:
```bash
sudo systemctl reload nginx
```
3) Открой приложение:
   - из твоего IP: должен быть доступ
   - из любого другого: должен быть 403

---

## 4) Где именно положить
- Положи этот конфиг в отдельный файл в `conf.d/` или `sites-available/` — как у тебя принято.
- Важно: `real_ip_header` + `set_real_ip_from` должны быть в контексте `http {}` или выше.

---

## 5) Смысл по Cloudflare
Если Cloudflare включён как proxy (orange cloud):
- Nginx “сам по себе” увидел бы IP Cloudflare
- поэтому обязательно нужно настроить `CF-Connecting-IP` как real client IP

---

## TODO перед запуском
- [ ] Подставить `__YOUR_DOMAIN__`
- [ ] Подставить `__YOUR_EXTERNAL_IP__`
- [ ] Подставить/обновить актуальные `set_real_ip_from` ranges из https://www.cloudflare.com/ips/
- [ ] Указать правильный `proxy_pass` или `root/alias`
