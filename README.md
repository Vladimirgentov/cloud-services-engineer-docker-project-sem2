\# Momo Store Docker Project



Проектная работа по дисциплине \*\*«Docker-контейнеризация и хранение данных»\*\*.



В проекте настроена контейнеризация приложения \*\*Momo Store\*\* с использованием Docker и Docker Compose.



Приложение состоит из двух частей:



\* `backend` — backend API на Go.

\* `frontend` — frontend на Vue.js, отдаётся через nginx.



\## Структура проекта



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

├── .gitignore

└── README.md

```



\## Требования



Для запуска проекта нужны:



\* Docker;

\* Docker Compose;

\* Git.



Проверка версий:



```bash

docker version

docker compose version

git --version

```



\## Быстрый запуск



Собрать и запустить приложение:



```bash

docker compose up -d --build

```



Проверить состояние контейнеров:



```bash

docker compose ps

```



После запуска приложение доступно по адресам:



```text

http://localhost/

http://localhost/momo-store/

```



Backend API доступен через nginx-прокси:



```text

http://localhost/api/products

http://localhost:8081/products

```



Остановить приложение:



```bash

docker compose down

```



\## Сервисы Docker Compose



В `docker-compose.yml` описаны два сервиса:



| Сервис     | Назначение                                 | Внутренний порт |            Внешний порт |

| ---------- | ------------------------------------------ | --------------: | ----------------------: |

| `backend`  | Go API                                     |            8081 | не публикуется напрямую |

| `frontend` | nginx + Vue.js frontend + proxy до backend |      8080, 8081 |                80, 8081 |



Backend внутри Docker-сети доступен по имени:



```text

backend:8081

```



Frontend публикует наружу:



```text

80:8080

8081:8081

```



Порт `80` используется для frontend, а порт `8081` используется для доступа к backend через nginx-прокси.



\## Проверка работы приложения



Проверить редирект с `/` на `/momo-store/`:



```bash

curl -I http://localhost/

```



Ожидаемый результат:



```text

HTTP/1.1 302 Moved Temporarily

Location: /momo-store/

```



Проверить frontend:



```bash

curl -I http://localhost/momo-store/

```



Ожидаемый результат:



```text

HTTP/1.1 200 OK

```



Проверить backend через frontend nginx proxy:



```bash

curl http://localhost/api/products

```



Проверить backend через опубликованный порт `8081`:



```bash

curl http://localhost:8081/products

```



Проверить health endpoint backend:



```bash

curl -i http://localhost:8081/health

```



\## Сборка образов вручную



Backend:



```bash

docker build -t momo-store-backend:1.0.0 ./backend

```



Frontend:



```bash

docker build -t momo-store-frontend:1.0.0 ./frontend

```



Посмотреть собранные образы:



```bash

docker images | grep momo-store

```



Для PowerShell:



```powershell

docker images | Select-String "momo-store"

```



\## Размеры образов



После сборки размеры образов можно проверить командой:



```bash

docker images momo-store-backend:1.0.0

docker images momo-store-frontend:1.0.0

```



Таблица для фиксации результата:



| Образ                       | Назначение                           |               Размер |

| --------------------------- | ------------------------------------ | -------------------: |

| `momo-store-backend:1.0.0`  | Backend Go API                       |  45.3 MB |

| `momo-store-frontend:1.0.0` | Frontend nginx + Vue.js static files |  78.3 MB |



Размеры образов уменьшены за счёт multi-stage build: инструменты сборки остаются только в build-stage и не попадают в финальный runtime-образ.



\## Backend Dockerfile



Для backend используется multi-stage build.



Первый stage:



\* базовый образ `golang:1.22-alpine`;

\* скачивание Go-зависимостей;

\* сборка приложения из `./cmd/api`;

\* сборка статического бинарного файла с `CGO\_ENABLED=0`.



Финальный stage:



\* базовый образ `alpine:3.20`;

\* в образ копируется только собранный бинарный файл;

\* добавляется непривилегированный пользователь `app`;

\* приложение запускается не от root;

\* открыт только порт `8081`;

\* настроен `HEALTHCHECK`.



Основные меры оптимизации:



\* multi-stage build;

\* лёгкий финальный образ;

\* кэширование Go-зависимостей;

\* исключение лишних файлов через `.dockerignore`;

\* удаление инструментов разработки из финального образа.



\## Frontend Dockerfile



Для frontend также используется multi-stage build.



Первый stage:



\* базовый образ `node:16-alpine`;

\* установка зависимостей через `npm ci`;

\* сборка production-версии frontend.



Финальный stage:



\* базовый образ `nginxinc/nginx-unprivileged:1.27-alpine`;

\* в образ копируется только директория `dist`;

\* nginx запускается не от root;

\* frontend отдаётся по пути `/momo-store/`;

\* nginx проксирует запросы `/api/` на backend;

\* настроен `HEALTHCHECK`.



Основные меры оптимизации:



\* multi-stage build;

\* лёгкий runtime-образ;

\* отсутствие Node.js и npm в финальном образе;

\* исключение `node\_modules`, `dist`, `.git` и других лишних файлов через `.dockerignore`.



\## Docker Compose



В `docker-compose.yml` настроены:



\* сборка backend и frontend образов;

\* внутренняя Docker-сеть `app-net`;

\* volume `backend-data`;

\* зависимости между сервисами через `depends\_on`;

\* healthchecks для backend и frontend;

\* политики перезапуска `unless-stopped`;

\* ограничения CPU и памяти;

\* запуск контейнеров с ограниченными правами;

\* read-only root filesystem;

\* tmpfs для временных директорий.



Проверить итоговую Compose-конфигурацию:



```bash

docker compose config

```



Запустить приложение:



```bash

docker compose up -d --build

```



Посмотреть состояние:



```bash

docker compose ps

```



Остановить приложение:



```bash

docker compose down

```



Удалить также volume:



```bash

docker compose down -v

```



\## Масштабирование backend



Backend можно масштабировать горизонтально, так как у него нет фиксированного `container\_name` и он не публикует порт напрямую на host machine.



Запустить 3 экземпляра backend:



```bash

docker compose up -d --scale backend=3

```



Проверить контейнеры:



```bash

docker compose ps

```



Ожидаемый результат:



```text

momo-store-backend-1

momo-store-backend-2

momo-store-backend-3

momo-store-frontend-1

```



Проверить работу приложения после масштабирования:



```bash

curl http://localhost/api/products

curl http://localhost:8081/products

```



Вернуть один экземпляр backend:



```bash

docker compose up -d --scale backend=1

```



\## Безопасность контейнеров



В проекте реализованы следующие меры безопасности:



\* контейнеры запускаются не от root;

\* в backend используется отдельный пользователь `app`;

\* frontend использует `nginx-unprivileged`;

\* включён `cap\_drop: ALL`;

\* включён `security\_opt: no-new-privileges:true`;

\* включён `read\_only: true`;

\* временные директории вынесены в `tmpfs`;

\* открыты только необходимые порты;

\* заданы лимиты CPU и памяти;

\* в финальные образы не попадают инструменты сборки;

\* секреты не хранятся внутри Dockerfile и Docker-образов.



\## Работа с секретами



В текущей версии приложения чувствительные секреты для запуска не требуются.



В проекте соблюдены базовые правила:



\* пароли и токены не записываются в Dockerfile;

\* секреты не добавляются в Docker-образы;

\* секреты не должны коммититься в Git;

\* для будущих секретов следует использовать Docker Secrets, GitHub Secrets или отдельные `.env` файлы, исключённые из Git.



Пример безопасного подхода для локальных переменных окружения:



```text

.env

.secrets/

```



Такие файлы должны быть добавлены в `.gitignore`.



\## Проверка уязвимостей



В GitHub Actions добавлен отдельный job `security\_scan`, который выполняет сканирование Docker-образов с помощью Trivy.



Сканируются образы:



```text

docker-project-backend:latest

docker-project-frontend:latest

```



Локально можно выполнить проверку так, если Trivy установлен на машине:



```bash

trivy image momo-store-backend:1.0.0

trivy image momo-store-frontend:1.0.0

```



Либо через Docker:



```bash

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image momo-store-backend:1.0.0

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image momo-store-frontend:1.0.0

```



В GitHub Actions используется режим:



```yaml

exit-code: 0

severity: CRITICAL,HIGH

```



Это позволяет увидеть найденные уязвимости в отчёте pipeline, но не блокирует выполнение workflow из-за уязвимостей базовых образов.



\## CI/CD



Файл workflow:



```text

.github/workflows/deploy.yaml

```



В pipeline сохранены исходные jobs и добавлен отдельный job для security scan.



Основные этапы pipeline:



1\. checkout репозитория;

2\. настройка Docker Buildx;

3\. логин в Docker Hub;

4\. сборка и публикация backend-образа;

5\. сборка и публикация frontend-образа;

6\. проверка сборки через Docker Compose;

7\. сканирование образов с помощью Trivy.



Workflow запускается при push в ветку:



```text

main

```



\## Проверенные команды



В процессе выполнения проекта были проверены команды:



```bash

docker build -t momo-store-backend:1.0.0 ./backend

docker build -t momo-store-frontend:1.0.0 ./frontend

docker compose config

docker compose up -d --build

docker compose ps

curl -I http://localhost/

curl -I http://localhost/momo-store/

curl http://localhost/api/products

curl http://localhost:8081/products

docker compose up -d --scale backend=3

docker compose up -d --scale backend=1

docker compose down

```



\## Коммиты



Рекомендуемый порядок коммитов:



```bash

git add backend/.dockerignore frontend/.dockerignore

git commit -m "Add dockerignore files"



git add backend/Dockerfile

git commit -m "Add backend Dockerfile"



git add frontend/Dockerfile

git commit -m "Add frontend Dockerfile"



git add docker-compose.yml

git commit -m "Add docker compose configuration"



git add .github/workflows/deploy.yaml

git commit -m "Add Trivy image scanning"



git add README.md

git commit -m "Add project README"

```



\## Итог



В проекте реализованы:



\* Dockerfile для backend;

\* Dockerfile для frontend;

\* multi-stage builds;

\* оптимизация размера образов;

\* Docker Compose конфигурация;

\* внутренняя сеть для сервисов;

\* volume для backend;

\* healthchecks;

\* restart policy;

\* ограничения ресурсов;

\* запуск контейнеров не от root;

\* read-only filesystem;

\* ограничение capabilities;

\* горизонтальное масштабирование backend;

\* Trivy security scan в GitHub Actions.



