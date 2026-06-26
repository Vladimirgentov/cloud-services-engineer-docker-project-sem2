# Momo Store Docker Project

Проектная работа по дисциплине **«Docker-контейнеризация и хранение данных»**.

В проекте настроена контейнеризация приложения **Momo Store** с помощью Docker и Docker Compose.

## Состав приложения

Приложение состоит из двух сервисов:

- `backend` — backend API на Go;
- `frontend` — frontend на Vue.js, который отдаётся через nginx.

## Структура проекта

```text
.
├── backend/
│   ├── Dockerfile
│   └── .dockerignore
├── frontend/
│   ├── Dockerfile
│   └── .dockerignore
├── docker-compose.yml
├── .github/
│   └── workflows/
│       └── deploy.yaml
└── README.md
Требования

Для запуска нужны:

Docker;
Docker Compose;
Git.

Проверка версий:

docker version
docker compose version
git --version
Быстрый запуск

Собрать и запустить приложение:

docker compose up -d --build

Проверить состояние контейнеров:

docker compose ps

После запуска frontend доступен по адресам:

http://localhost/
http://localhost/momo-store/

Backend API доступен через nginx-прокси:

http://localhost/api/products
http://localhost:8081/products

Остановить приложение:

docker compose down

Остановить приложение и удалить volume:

docker compose down -v
Docker Compose

В docker-compose.yml описаны два сервиса:

СервисНазначениеВнутренний портВнешний порт
backendGo API8081не публикуется напрямую
frontendnginx + Vue.js frontend + proxy до backend8080, 808180, 8081

Backend внутри Docker-сети доступен по имени:

backend:8081

Frontend публикует наружу:

80:8080
8081:8081
Backend Dockerfile

Для backend используется multi-stage build.

Первый stage:

базовый образ golang:1.22-alpine;
скачивание Go-зависимостей;
сборка приложения из ./cmd/api;
сборка статического бинарного файла с CGO_ENABLED=0.

Финальный stage:

базовый образ alpine:3.20;
копируется только собранный бинарный файл;
добавляется непривилегированный пользователь app;
приложение запускается не от root;
открыт порт 8081;
настроен HEALTHCHECK.
Frontend Dockerfile

Для frontend также используется multi-stage build.

Первый stage:

базовый образ node:16-alpine;
установка зависимостей через npm ci;
сборка production-версии frontend.

Финальный stage:

базовый образ nginxinc/nginx-unprivileged:1.27-alpine;
копируются только статические файлы из dist;
nginx запускается не от root;
frontend отдаётся по пути /momo-store/;
nginx проксирует запросы /api/ на backend;
настроен HEALTHCHECK.
Безопасность контейнеров

В проекте реализованы следующие меры:

контейнеры запускаются не от root;
включён cap_drop: ALL;
включён security_opt: no-new-privileges:true;
включён read_only: true;
временные директории вынесены в tmpfs;
заданы лимиты CPU и памяти;
секреты не хранятся в Dockerfile и Docker-образах.
Проверка работы

Проверить итоговую Compose-конфигурацию:

docker compose config

Собрать образы:

docker compose build

Запустить приложение:

docker compose up -d

Проверить frontend:

curl -I http://localhost/momo-store/

Проверить backend через nginx-прокси:

curl http://localhost/api/products

Проверить backend через опубликованный порт:

curl http://localhost:8081/products
Масштабирование backend

Backend можно масштабировать, так как у него нет фиксированного container_name и он не публикует порт напрямую на host machine.

Пример запуска трёх экземпляров backend:

docker compose up -d --scale backend=3

Вернуть один экземпляр:

docker compose up -d --scale backend=1
CI/CD

Workflow находится в файле:

.github/workflows/deploy.yaml

Pipeline запускается при push в ветки:

main;
docker-project.

Основные этапы pipeline:

checkout репозитория;
проверка наличия основных файлов;
проверка docker compose config;
сборка образов через Docker Compose;
сканирование backend-образа через Trivy;
сканирование frontend-образа через Trivy.
Итог

В проекте реализованы:

Dockerfile для backend;
Dockerfile для frontend;
multi-stage builds;
Docker Compose конфигурация;
внутренняя Docker-сеть;
volume для backend;
healthchecks;
restart policy;
ограничения ресурсов;
запуск контейнеров не от root;
read-only filesystem;
ограничение capabilities;
Trivy security scan в GitHub Actions.
