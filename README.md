# Kubernetes_ITMO
DevOps course, ITMO, kubernetes homework

## Задачи

1. В качестве web-сервера для простоты можно воспользоваться Python: “python -m http.server 8000”. 
    1. Добавьте эту команду в инструкцию CMD в Dockerfile.
2. Создать Dockerfile на основе “python:3.10-alpine”, в котором.
    1. Создать каталог “/app” и назначить его как WORKDIR.
    2. Добавить в него файл, содержащий текст “Hello world”.
    3. Обеспечить запуск web-сервера от имени пользователя с “uid 1001”.
3. Собрать Docker image с tag “1.0.0”.
4. Запустить Docker container и проверить, что web-приложение работает.
5. Выложить image на Docker Hub.
6. Создать Kubernetes Deployment manifest, запускающий container из созданного image.
    1. Имя Deployment должно быть “web”.
    2. Количество запущенных реплик должно равняться двум.
    3. Добавить использование Probes.
7. Установить manifest в кластер Kubernetes.
8. Обеспечить доступ к web-приложению внутри кластера и проверить его работу
    1. Воспользоваться командой kubectl port-forward: “kubectl port-forward --address 0.0.0.0 deployment/web 8080:8000”.
    2. Перейти по адресу http://127.0.0.1:8080/hello.html.

## Структура файлов проекта:

```shell
$ tree -I 'venv'
.
├── LICENSE
├── README.md
├── deployment.yaml
├── docker-compose.yml
└── web-server
    ├── Dockerfile
    ├── server.py
    └── templates
        └── hello.html
```

## Шаг 1

В качестве фреймворка для сервера использован [Python Flask](https://flask.palletsprojects.com/). Код `web-server/server.py`:

```python
from flask import Flask, render_template

app = Flask(__name__)


@app.route('/')
@app.route('/hello.html')
def hello_world():
    """Hello world function

    Returns template:
        hello.html
    """
    return render_template('hello.html')

```

hello.html:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>
      Hello World!
    </title>
  </head>
  <body>
    Hello World!
  </body>
</html>
```

## Шаг 2

В файле `web-server/Dockerfile` описана сборка и описан запуск сервера:

```Dockerfile
# Based image
FROM python:3.10-alpine

# Variables required for enviroment creation
ARG USER=app
ARG UID=1001
ARG GID=1001

# Framework installation
RUN pip install --no-cache-dir Flask==2.2.*

# Creating OS user and home directory
RUN addgroup -g ${GID} -S ${USER} \
   && adduser -u ${UID} -S ${USER} -G ${USER} \
   && mkdir -p /app \
   && chown -R ${USER}:${USER} /app
USER ${USER}

# Entering home dir /app
WORKDIR /app


# Enviroment variables required for launching web-application
ENV FLASK_APP=server.py \
   FLASK_RUN_HOST="0.0.0.0" \
   FLASK_RUN_PORT="8000" \
   PYTHONUNBUFFERED=1

# Copying application code to home directory
COPY --chown=$USER:$USER . /app

# Publishing the port that the application is listening on
EXPOSE 8000

# Application launch command
CMD ["flask", "run"]
```

## Шаг 3

Используя приведённую команду ниже создаём образ - Docker image:

```shell
sudo docker build -t beonewithyuri/web-server:1.0.0 --network host -t beonewithyuri/web-server:latest web-server
```

Список image:

```shell
$ sudo docker image ls
REPOSITORY                    TAG       IMAGE ID       CREATED             SIZE
beonewithyuri/server          1.0.0     30221e994b70   About an hour ago   60.7MB
beonewithyuri/web-server      1.0.0     30221e994b70   About an hour ago   60.7MB
beonewithyuri/test            1.0.0     c5f5f0e729ef   2 hours ago         49.7MB
beonewithyuri/test            latest    34d04760f1b6   2 hours ago         49.7MB
```

Теперь для удобства создадим файл `docker-compose.yml`, где опишем конфигурацию сборки и запуска приложений.

```yaml
# Версия API Docker compose
version: "3"

# Раздел, в котором описываются приложения (сервисы).
services:

  # Раздел для описания приложения 'server'.
  server:

    # Имя image tag
    image: beonewithyuri/web-server:1.0.0
 
    # Параметры сборки Docker image.
    build: 
      # Путь к Dockerfile,
      context: web-server/
      # Использовать host-сеть при сборке,
      network: host

    # Перенаправление портов из Docker container на host-машину.
    ports:
      - 8000:8000

    # Имя user, используемого в image,
    user: "1001"

    # Используемый тип сети при запуске container.
    network_mode: host

    # Проверка готовности приложения к работе. Параметр "--spider" означает: не загружать url, 
    # а только проверить его наличие.
    healthcheck:
        test: wget --no-verbose --tries=1 --spider http://localhost:8000 || exit 1
        interval: 5s
        timeout: 5s
        retries: 5
```

Соберем Docker images при помощи docker compose.

```shell
$ sudo docker compose build
[+] Building 1.3s (10/10) FINISHED                                                                                                                                                                                                                                                                         
 => [internal] load build definition from Dockerfile                                                                                                                                                                                                                                                  0.0s
 => => transferring dockerfile: 493B                                                                                                                                                                                                                                                                  0.0s
 => [internal] load .dockerignore                                                                                                                                                                                                                                                                     0.0s
 => => transferring context: 2B                                                                                                                                                                                                                                                                       0.0s
 => [internal] load metadata for docker.io/library/python:3.10-alpine                                                                                                                                                                                                                                 1.1s
 => [internal] load build context                                                                                                                                                                                                                                                                     0.0s
 => => transferring context: 591B                                                                                                                                                                                                                                                                     0.0s
 => [1/5] FROM docker.io/library/python:3.10-alpine@sha256:def82962a6ee048e54b5bec2fcdfd4aada4a907277ba6b0300f18c836d27f095                                                                                                                                                                           0.0s
 => CACHED [2/5] RUN pip install --no-cache-dir Flask==2.2.*                                                                                                                                                                                                                                          0.0s
 => CACHED [3/5] RUN addgroup -g 1001 -S app    && adduser -u 1001 -S app -G app    && mkdir -p /app    && chown -R app:app /app                                                                                                                                                                      0.0s
 => CACHED [4/5] WORKDIR /app                                                                                                                                                                                                                                                                         0.0s
 => [5/5] COPY --chown=app:app . /app                                                                                                                                                                                                                                                                 0.1s
 => exporting to image                                                                                                                                                                                                                                                                                0.1s
 => => exporting layers                                                                                                                                                                                                                                                                               0.0s
 => => writing image sha256:11c8183314a47681ff73a843876933705e27fde722f02c108aba287960855e1b                                                                                                                                                                                                          0.0s
 => => naming to docker.io/beonewithyuri/web-server:1.0.0 
 ```
## Шаг 4

Запустим веб-приложение при помощи docker compose.

```shell
$ docker compose up
[+] Running 2/0
 ✔ Container kubernetes_itmo-server-1                                Recreated                                                                                                                                                                                                                        0.1s 
 ! server Published ports are discarded when using host network mode                                                                                                                                                                                                                                  0.0s 
Attaching to kubernetes_itmo-server-1
kubernetes_itmo-server-1  |  * Serving Flask app 'server.py'
kubernetes_itmo-server-1  |  * Debug mode: off
kubernetes_itmo-server-1  | WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
kubernetes_itmo-server-1  |  * Running on all addresses (0.0.0.0)
kubernetes_itmo-server-1  |  * Running on http://127.0.0.1:8000
kubernetes_itmo-server-1  |  * Running on http://172.21.30.20:8000
kubernetes_itmo-server-1  | Press CTRL+C to quit
kubernetes_itmo-server-1  | 127.0.0.1 - - [13/May/2023 21:49:28] "GET / HTTP/1.1" 200 -
```

В отдельном терминале проверим веб-приложение.

```shell
$ curl http://127.0.0.1:8000/hello.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>
      Hello World!
    </title>
  </head>
  <body>
    Hello World!
  </body>
</html>
```

Веб-приложение работает корректно.

## Шаг 5

Выложим на dockerhub.
Получим Access Token в `https://hub.docker.com/settings/security`

Авторизуемся в Docker registry. В качестве пароля введем Access Token.

```shell
$ sudo docker login -u beonewithyuri

Password:
Login Succeeded
```

Отправим Docker image в Registry Docker Hub.

```shell
$ sudo docker push beonewithyuri/web-server:1.0.0
```

Откроем web-интерфейс Docker Hub по адресу `https://hub.docker.com/` и найдем загруженный image

![image](image.png)

