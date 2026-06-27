# syntax=docker/dockerfile:1.7

ARG NODE_VERSION=16

FROM node:${NODE_VERSION}-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

ARG VUE_APP_API_URL=/api
ENV VUE_APP_API_URL=${VUE_APP_API_URL}

RUN npm run build

FROM nginxinc/nginx-unprivileged:1.27-alpine

USER root

RUN cat > /etc/nginx/conf.d/default.conf <<'EOF'
server {
    listen 8080;
    server_name _;
    absolute_redirect off;

    resolver 127.0.0.11 valid=30s ipv6=off;
    set $backend_upstream backend:8081;

    root /usr/share/nginx/html;
    index index.html;

    location = / {
        return 302 /momo-store/;
    }

    location /momo-store/ {
        try_files $uri $uri/ /momo-store/index.html;
    }

    location /api/ {
        rewrite ^/api/(.*)$ /$1 break;
        proxy_pass http://$backend_upstream;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    listen 8081;
    server_name _;

    resolver 127.0.0.11 valid=30s ipv6=off;
    set $backend_upstream backend:8081;

    location / {
        proxy_pass http://$backend_upstream;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF

RUN mkdir -p /usr/share/nginx/html/momo-store

COPY --from=builder /app/dist/ /usr/share/nginx/html/momo-store/

RUN chown -R 101:101 /usr/share/nginx/html /etc/nginx/conf.d

USER 101:101

EXPOSE 8080 8081

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/momo-store/ || exit 1
