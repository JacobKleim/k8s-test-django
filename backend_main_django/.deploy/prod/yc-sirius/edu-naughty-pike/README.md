# Документация по деплою на prod

## Подготовка к деплою

**Подготовьте докер образ, который будете использовать в манифестах**

**Получение доступа к кластеру**

- Убедитесь, что у вас есть доступ к production-кластеру Kubernetes.

**Создание секретов и конфигов**

- Создайте секреты для базы данных и других чувствительных данных:

Образ с Django считывает настройки из переменных окружения.

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. По умолчанию на продакшене установлена в `False` [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

`Файл с сертификатом` для подключения к нашей БД PostgreSQL.

## Деплой приложения

**Применение манифестов Kubernetes**

Примените файлы манифестов для создания подов, сервисов и других ресурсов:
```sh
kubectl apply -f path/to/your/pod.yaml
kubectl apply -f path/to/your/service.yaml
kubectl apply -f path/to/your/ingress.yaml
```

**Настройка сетевых правил**

- Убедитесь, что сетевые правила (Ingress или LoadBalancer) настроены правильно для доступа к вашему сайту.


## Выполнение миграций и создание суперпользователя

**Запуск миграций**

```sh
kubectl exec -it <pod-name> -- python3 manage.py migrate
```

**Создание суперпользователя**

```sh
kubectl exec -it <pod-name> -- python3 manage.py createsuperuser
```

## Проверка и мониторинг

**Проверьте, что ваш сайт доступен по адресу:**

```sh
curl http://<your-production-host>
```

**Мониторинг логов и метрик**

```sh
kubectl logs <pod-name>
```
