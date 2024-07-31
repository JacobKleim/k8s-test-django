# Документация по деплою

## Как собрать и опубликовать Docker образ

### Сборка образа

1. Перейдите в корневую директорию проекта:
    ```sh
    cd /path/to/your/project
    ```

2. Убедитесь, что у вас есть Dockerfile и файл requirements.txt.

3. Получите хэш текущего коммита git:
    ```sh
    COMMIT_HASH=$(git rev-parse --short HEAD)
    ```

4. Соберите Docker образ:
    ```sh
    docker build -t your_dockerhub_username/django_site:$COMMIT_HASH .
    ```

### Публикация образа

1. Войдите в Docker Hub:
    ```sh
    docker login
    ```

2. Отправьте образ в Docker Hub:
    ```sh
    docker push your_dockerhub_username/django_site:$COMMIT_HASH
    ```

### Загрузка образа

Чтобы загрузить старый образ по хэшу коммита, выполните:
```sh
docker pull your_dockerhub_username/django_site:<COMMIT_HASH>
```


## Как подготовить окружение

Secret в Kubernetes представляет собой объект, который хранит конфиденциальные данные. Секреты можно создать несколькими способами, включая использование kubectl или путем создания манифеста YAML. 

**Для нашего сайта понадобятся секреты SECRET_KEY, DATABASE_URL, ALLOWED_HOSTS, DEBUG и файл с сертификатом для подключения к нашей БД PostgreSQL.**

### Создание секрета через kubectl

```sh
kubectl create secret generic <secret-name> \
  --from-literal=<key1>=<value1> \
  --from-literal=<key2>=<value2> \
  --namespace=<namespace>
```

### Создание секрета из файла

Если у вас есть файл, который вы хотите использовать для создания секрета, используйте команду

```sh
kubectl create secret generic <secret-name> \
  --from-file=<path-to-file> \
  --namespace=<namespace>
```

### Создание секрета из манифеста YAML

Вы также можете создать секрет, используя манифест YAML. Создайте файл secret.yaml со следующим содержимым:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-database-secret
  namespace: edu-naughty-pike
type: Opaque
data:
  host: <base64-encoded-host>
  port: <base64-encoded-port>
  name: <base64-encoded-database-name>
  username: <base64-encoded-username>
  password: <base64-encoded-password>
```

**Примените манифест:**

```sh
kubectl apply -f path/to/secret.yaml
```

### Монтирование секрета

**Можно одновременно монтировать секреты из переменных окружения и из файлов:**

Секрет из переменной окружения
```yaml
env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: my-database-secret
          key: host
```

Секрет из файла
```yaml
  volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
volumes:
- name: secret-volume
secret:
    secretName: my-database-secret
```

**Пример манифеста:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: edu-naughty-pike
spec:
  containers:
  - name: my-app-container
    image: my-app-image
    env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: my-database-secret
          key: host
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: my-database-secret
```


## Подготовьте Pod

**Создайте файл django-pod.yaml со следующим содержимым:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: django
  namespace: your_namespace
  labels:
    app: django
spec:
  containers:
  - name: django
    image: docker_username/django_site:<tag>
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
          name: postgres
          key: url
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
    volumeMounts:
    - name: pg-root-cert
      mountPath: /etc/postgresql/certs
      readOnly: true
  volumes:
  - name: pg-root-cert
    secret:
      secretName: pg-root-cert      
  restartPolicy: Never
```

**Примените манифест:**

```sh
kubectl apply -f path/to/django-pod.yaml
```


## Создайте Service

**Создайте файл django-service.yaml со следующим содержимым:**

В nodePort используйте порт, который вам выделен.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: django-service
  namespace: edu-naughty-pike
spec:
  selector:
    app: django
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30321
  type: NodePort
```

**Примените манифест:**
```sh
kubectl apply -f  path/to/django-service.yaml
```


## Настройка Ingress

**Создайте файл django-ingress.yaml.yaml со следующим содержимым:**

В host используйте домен, который вам выделен.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-ingress
  namespace: edu-naughty-pike
spec:
  rules:
  - host: edu-naughty-pike.sirius-k8s.dvmn.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: django-service
            port:
              number: 80
```

**Примените манифест:**
```sh
kubectl apply -f  path/to/django-ingress.yaml
```

