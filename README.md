# memos-docker
### `memos.domain.com` указан как пример
### Up memos server note on docker
### После установки докера и докер compose 

### 1. Создадим папку для проекта
```
mkdir -p /etc/docker/memos
cd /etc/docker/memos
```
### 2. Создаем `docker-compose.yml` и поднимаем контейнер 

```
nano docker-compose.yml
```

#### Cодержимое `docker-compose.yml`
```yml
services:
  memos:
    image: neosmemo/memos:stable
    container_name: memos
    ports:
     - "127.0.0.1:5230:5230"
    volumes:
      - /root/.memos:/var/opt/memos
    environment:
      - MEMOS_MODE=prod
      - MEMOS_PORT=5230
    restart: unless-stopped
    networks:
      proxy:                              # Подключаем контейнер к внешней сети "proxy" (используется Traefik)
    labels:                               # Метки для интеграции с Traefik (обратный прокси)
      - "traefik.enable=true"                                      # Включаем обработку контейнера Traefik
      - "traefik.http.routers.memos.entrypoints=web"             # Определяем HTTP-вход (порт 80)
      - "traefik.http.routers.memos.rule=Host(`memos.domain.com`)"  # Трафик на этот домен будет направляться в данный контейнер
      - "traefik.http.middlewares.memos-https-redirect.redirectscheme.scheme=https"         # Middleware для редиректа с HTTP на HTTPS
      - "traefik.http.routers.memos.middlewares=memos-https-redirect"          # Применяем middleware редиректа к HTTP-маршруту
      - "traefik.http.routers.memos-secure.entrypoints=websecure"          # Определяем HTTPS-вход (порт 443)
      - "traefik.http.routers.memos-secure.rule=Host(`memos.domain.com`)"         # HTTPS-маршрут для того же домена
      - "traefik.http.routers.memos-secure.tls=true"          # Включаем TLS (HTTPS)
      - "traefik.http.routers.memos-secure.service=memos"     # Привязываем HTTPS-маршрут к сервису memos
      - "traefik.http.services.memos.loadbalancer.server.port=5230"          # Указываем внутренний порт, на котором memos слушает в контейнере
      - "traefik.docker.network=proxy"          # Указываем, что Traefik должен искать контейнер в сети "proxy"

networks:
  proxy:                                  # Определение внешней сети для взаимодействия с Traefik
    external: false                       
```
#### Поднимаем контейнер через команду `docker compose up -d` так же смотрим статус контейнера ниже пример того чего мы должны увидеть
```bash
root@server-gw-git:/etc/docker/memos# docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED        STATUS                    PORTS                                                               NAMES
2a2e7864c6e6   neosmemo/memos:stable       "/usr/local/memos/en…"   18 hours ago   Up 18 hours               127.0.0.1:5230->5230/tcp                                            memos
root@server-gw-git:/etc/docker/memos# 
```
### 3. Настраиваем nginx т.к безопаснее будет заходить на сайт по домен
##### Если у вас не установлен `nginx` и `certbot` то устанавливаем и включаем 
```
apt install certbot python3-certbot-nginx nginx
systemctl enable --now nginx
```
##### Настройка самого nginx
##### Создаем файл 
```
nano /etc/nginx/sites-available/memos.domain.com
```
#### Содержимое файла `memos.domain.com`
```
server {
    server_name memos.domain.com;

    location / {
        client_max_body_size 512M;
        proxy_pass http://127.0.0.1:5230;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
    }
}
```
### 4. Обязательно делаем ссылку без нее не будет работать и перезапускаем nginx
```bash
ln -s /etc/nginx/sites-available/memos.domain.com /etc/nginx/sites-enabled/
systemctl restart nginx
```

### 5. Получение сертификата
Пишем команду `certbot --nginx -d memos.domain.ru` после этого вы можете переходить на доменное имя `memos.domain.com`

