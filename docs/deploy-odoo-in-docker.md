# How to deploy odoo in Docker ?

```zsh title="conf/odoo.conf"
[options]
addons_path = /mnt/extra-addons
logfile = /var/log/odoo/odoo17.log
workers = 5
```

```zsh title="odoo_pg_pass"
odoo
```
```zsh
docker init
#choose python
```

```zsh title="Dockerfile"

FROM odoo:17.0

USER root

RUN apt-get update && \
    apt-get install -y fonts-dejavu fonts-freefont-ttf fonts-noto

USER odoo

CMD ["odoo"]
```

```zsh title="docker-compose.yaml"

services:
  web:
#    image: odoo:17.0
    build:
      context: .
    depends_on:
      - db
    ports:
      - "8069:8069"
    volumes:
      - odoo-web-data:/var/lib/odoo
      - ../conf:/etc/odoo
      - ./addons:/mnt/extra-addons
      - ../logs:/var/log/odoo  # Mount volume for Odoo logs
    environment:
      - PASSWORD_FILE=/run/secrets/postgresql_password
    secrets:
      - postgresql_password
    deploy:
      replicas: 1  # Initial number of container instances
    restart: always

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_PASSWORD_FILE=/run/secrets/postgresql_password
      - POSTGRES_USER=odoo
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - odoo-db-data:/var/lib/postgresql/data/pgdata
    secrets:
      - postgresql_password
    deploy:
      replicas: 1  # For databases, you typically don't need more than one instance.
    restart: always

  nginx:
    image: nginx:latest
    ports:
      - "9099:9099"
    volumes:
      - ../nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
    deploy:
      replicas: 1  # For databases, you typically don't need more than one instance.
    restart: always

volumes:
  odoo-web-data:
  odoo-db-data:

secrets:
  postgresql_password:
    file: ../odoo_pg_pass
```

``` title="nginx.conf"

worker_processes  1;

events {
    worker_connections  1024;
}

http{
    upstream odoo {
        server web:8069;
    }
    upstream odoochat {
      server web:8072;
    }
    map $http_upgrade $connection_upgrade {
      default upgrade;
      ''      close;
    }

    server {
      listen 80;
      proxy_read_timeout 720s;
      proxy_connect_timeout 720s;
      proxy_send_timeout 720s;

      # log
      access_log /var/log/nginx/odoo.access.log;
      error_log /var/log/nginx/odoo.error.log;

      # Redirect websocket requests to odoo gevent port
      location /websocket {
        proxy_pass http://odoochat;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;

      }

      # Redirect requests to odoo backend server
      location / {
        # Add Headers for odoo proxy mode
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://odoo;

      }

      # common gzip
      gzip_types text/css text/scss text/plain text/xml application/xml application/json application/javascript;
      gzip on;
    }
}
```

```zsh
docker compose up -d
```