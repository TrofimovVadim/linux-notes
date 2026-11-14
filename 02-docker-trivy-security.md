# Шпаргалка: Сборка Docker-образа и аудит безопасности через Trivy

Практический кейс DevSecOps: поэтапное устранение уязвимостей в контейнере (со **165** уязвимостей до **0**).

---

## 1. Подготовка файлов проекта

Создаём рабочую папку и простую HTML-страницу для проверки работы веб-сервера:

```bash
mkdir -p ~/my-first-sec-docker && cd ~/my-first-sec-docker
echo "<h1>My First DevSecOps Container</h1>" > index.html
```

---

## 2. Эволюция Dockerfile (от уязвимого к безопасному)

### Этап 1 (`v1`): Уязвимый образ — 165 уязвимостей
Используется устаревшая версия Nginx на базе Debian 10 (138 HIGH и 27 CRITICAL уязвимостей):

```bash
cat «EOF > Dockerfile
FROM nginx:1.21.0
COPY index.html /usr/share/nginx/html/index.html
EOF
```

### Этап 2 (`v2`): Облегчённый образ — 3 уязвимости
Переход на актуальную и минималистичную версию Nginx на базе Alpine Linux:

```bash
cat «EOF > Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EOF
```

### Этап 3 (`v3`): Полностью безопасный образ — 0 уязвимостей
Добавлено принудительное обновление системных пакетов внутри образа при сборке (`apk upgrade`), что закрывает остаточные уязвимости в системных библиотеках:

```bash
cat «EOF > Dockerfile
FROM nginx:alpine
RUN apk update && apk upgrade --no-cache
COPY index.html /usr/share/nginx/html/index.html
EOF
```

---

## 3. Сборка и локальная проверка контейнера

* **Собрать финальный образ** (точка в конце указывает на текущую папку):
  ```bash
  docker build -t my-web:v3 .
  ```

* **Запустить контейнер в фоне** на порту `8080` и проверить ответ сайта:
  ```bash
  docker run -d -p 8080:80 --name test-site my-web:v3
  curl http://localhost:8080
  ```

* **Остановить и удалить тестовый контейнер** после проверки:
  ```bash
  docker stop test-site && docker rm test-site
  ```

---

## 4. Сканирование образа на уязвимости (Trivy)

Запуск сканера **Trivy** через официальный Docker-контейнер с сохранением кэша баз на диск (`~/.cache`) и увеличенным лимитом времени (`15m`):

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/.cache:/root/.cache/ \
  aquasec/trivy image \
  --timeout 15m \
  --severity HIGH,CRITICAL \
  my-web:v3
```

### Расшифровка ключевых флагов:
* `RUN apk update && apk upgrade --no-cache` — обновляет системные пакеты внутри образа при сборке и не оставляет временный кэш.
* `--rm` — удаляет контейнер сканера сразу после проверки.
* `-v /var/run/docker.sock:...` — даёт сканеру доступ к локальным образам на хосте.
* `-v ~/.cache:/root/.cache/` — сохраняет базу уязвимостей на диск для быстрого повторного запуска.
* `--severity HIGH,CRITICAL` — показывает только высокие и критические уязвимости.

---

## 5. Полезные команды обслуживания Docker
* Список локальных образов: `docker images`
* Список запущенных контейнеров: `docker ps`
* Удалить старый образ: `docker rmi my-web:v1`
