# 3x-ui + Caddy + Reality Self Steal в Docker
## Пошаговая инструкция по развёртыванию связки 3x-ui + Caddy + Reality (Self Steal) в Docker.
### Подготовка

Предполагается, что:
- Настроен и защищён доступ к серверу по SSH
- Установлен и настроен firewall (открыты порты 80, 443 и 8443)
- Зарегистрирован и делегирован домен (например, example.com), указывающий на ваш сервер \
Нет своего домена, не страшно, можно использовать бесплатные домены предоставляемые сервисами: dynu.com, freedns.afraid.org, duckdns.org и т.п., главное, что бы он был и указывал на сервер

### Установка Docker

- Инструкции по установке Docker: https://docs.docker.com/engine/install/

- Быстрая установка:
```bash
bash <(wget -qO- https://get.docker.com) @ -o get-docker.sh
```

- Запуск Docker без root
```bash
sudo groupadd docker
```
```bash
sudo usermod -aG docker $USER
```
```bash
newgrp docker
```

- Проверьте, что Docker работает корректно:
```bash
docker run hello-world
```

### Создание необходимых директорий и файлов
- Создайте директории:

```bash
mkdir -p /opt/3x-ui-setup/{3x-ui,caddy/templates}
```
- Создайте файл `docker-compose.yml`:
```bash
nano /opt/3x-ui-setup/docker-compose.yml
```
```bash
services:
  caddy:
    image: caddy:2.11
    container_name: caddy
    restart: always
    network_mode: host
    volumes:
      - ./caddy/data:/data
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
      - ./caddy/templates:/srv

  3xui:
    image: ghcr.io/mhsanaei/3x-ui:latest
    container_name: 3xui_app
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./3x-ui/db/:/etc/x-ui/
    environment:
      XRAY_VMESS_AEAD_FORCED: "false"
      XUI_ENABLE_FAIL2BAN: "true"
    tty: true
```

- Создайте файл `Caddyfile`:
```bash
nano /opt/3x-ui-setup/caddy/Caddyfile
```
```bash
{
    https_port 4123
    default_bind 127.0.0.1
    servers {
        listener_wrappers {
            proxy_protocol {
                allow 127.0.0.1/32
            }
            http_redirect
            tls
        }
    }
    auto_https disable_redirects
}

https://example.com {
    root * /srv
    file_server
}

http://example.com {
    bind 0.0.0.0
    redir https://example.com{uri} permanent
}

https://example.com:8443 {

    bind 0.0.0.0

    # Подписка
    handle /sub/* {
        reverse_proxy 127.0.0.1:2096
    }

    # Панель
    handle {
        reverse_proxy 127.0.0.1:2053
    }
}

:4123 {
    tls internal
    respond 204
}

:80 {
    bind 0.0.0.0
    respond 204
}
```
- Замените **example.com** на ваш реальный домен в `Caddyfile`
 ```bash
  sed -i 's/example.com/ваш.домен.com/g' /opt/3x-ui-setup/caddy/Caddyfile
  ```
Или внимательно заменить вручную в редакторе.
- Для маскировки сервера используется [Confluence](https://github.com/Jolymmiles/confluence-marzban-home)\
Добавьте страницу для маскировки:
```bash
wget -qO- https://raw.githubusercontent.com/Jolymmiles/confluence-marzban-home/main/index.html  | envsubst > /opt/3x-ui-setup/caddy/templates/index.html
```

### Запустите Docker Compose
```bash
docker compose -f /opt/3x-ui-setup/docker-compose.yml up -d
```

### Первый вход в панель
- Откройте в браузере: https://example.com:8443
- Логин: admin
- Пароль: admin
> [!WARNING]
> Обязательно, сразу же измените стандартные логин и пароль: `Panel Settings -> Authentication`

### Изменение путей к панели и подписке
**Настройка пути до панели**
- Перейдите `Panel Settings -> General -> URI Path`
- Измените / на что то свое, например: /admin-secret-path/
- Сохраните настройки.
- Теперь панель будет доступна по адресу: https://example.com:8443/admin-secret-path

**Настройка пути до подписки (если планируется использовать)**
- Перейдите в `Panel Settings → Subscription -> URI Path (sub)`
- Измените /sub/ на что то свое, например: /sub-secret-path/
- `Panel Settings → Subscription -> Reverse Proxy URI`
- Измените Reverse Proxy URI на https://example.com:8443/sub-secret-path/
- Сохраните настройки и перезапустите панель.

> [!CAUTION]
> Без изменения `Caddyfile` подписки открываться не будут

- Измените путь в `Caddyfile`:
```bash
nano /opt/3x-ui-setup/caddy/Caddyfile
```
```bash
handle /sub/*
```
на:
```bash
handle /sub-secret-path/*
```
- Перезапустите контейнеры:
```bash
docker compose -f /opt/3x-ui-setup/docker-compose.yml down && docker compose -f /opt/3x-ui-setup/docker-compose.yml up -d
```
> [!CAUTION]
> Необходимо использовать собственное уникальное значение для admin-secret-path и sub-secret-path.

### Создание подключения Reality (Self Steal)

#### Создайте новый inbound в панели 3x-ui
При создании inbound используйте следующии параметры:
- Protocol: vless
- Port: 443
- Flow: xtls-rprx-vision
- Transmission: tcp
- Security: Reality
- Xver: 1
- uTLS: chrome
- Target: 127.0.0.1:4123
- SNI: example.com
- PrivateKey Public Key: сгенерировать нажав Get New Cert
- ShortID: сгенерировать
- Sniffing - enable: HTTP TLS QUIC FAKEDNS отмечены
> [!CAUTION]
> Замените **example.com** на ваш домен.>
- Inbound должен выглядеть приблизительно [так](panel.png) 


#### Thanks:
 [Akiyamov](https://github.com/Akiyamov) 
+ Caddy + Reality (Self Steal) в Docker
