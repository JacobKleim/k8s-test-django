# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Как создать Secret в кластере:

### Закодируйте значения в base64

Примеры команд для Unix-подобных систем:
```sh
echo -n "123abc" | base64
echo -n "postgres://test_k8s:OwOtBep9Frut@172.20.10.2:5432/test_k8s" | base64
echo -n "192.168.59.102" | base64
echo -n "True" | base64
```
Примеры команд для Windows PowerShell:
```sh
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("123ase"))
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("postgres://test_k8s:OwOtBep9Frut@172.20.10.2:5432/test_k8s"))
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("192.168.59.102"))
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("True"))
```

### Создайте файл манифеста для Secret

Поместите файл с секретами отдельно от репозитория: он не должен быть в публично доступе.

```sh
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
data:
  SECRET_KEY: 
  DATABASE_URL: cG9zdGdyZXM6Ly90ZXN0X2s4czpPd090QmVwOUZydXRA MTcyLjIwLjEwLjI6NTQzMi90ZXN0X2s4cw==  
  ALLOWED_HOSTS: MTkyLjE2OC41OS4xMDI=  
  DEBUG: VHJ1ZQ== 
```

### Примените манифест Secret
```sh
kubectl apply -f path/to/secrets.yaml
```

### Обновите Deployment для использования Secret
```sh
kubectl apply -f path/to/deployment.yaml
```

### Примените обновленный манифест Deployment

В вашем манифесте Deployment, измените секцию env, чтобы использовать Secret:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
  labels:
    app: django
spec:
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
      - name: django-cont
        image: django_app:test
        ports:
        - containerPort: 80 
        env:
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: django-secrets
              key: SECRET_KEY
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: django-secrets
              key: DATABASE_URL
        - name: ALLOWED_HOSTS
          valueFrom:
            secretKeyRef:
              name: django-secrets
              key: ALLOWED_HOSTS
        - name: DEBUG
          valueFrom:
            secretKeyRef:
              name: django-secrets
              key: DEBUG
---

apiVersion: v1
kind: Service
metadata:
  name: django-pod-service
  labels:
    env: prod
spec:
  selector:
    app: django
  type: NodePort
  ports:
  - name: app-listener
    port: 80
    targetPort: 80
    nodePort: 30036
    protocol: TCP
```

### Перезапустите поды для применения изменений

Удалите существующие поды, чтобы они были перезапущены с новыми переменными окружения:
```sh
kubectl delete pods -l app=django
```