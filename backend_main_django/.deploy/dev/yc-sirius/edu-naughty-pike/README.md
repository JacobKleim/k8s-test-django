# Документация по деплою


## Подготовьте Pod

**Создайте файл django-pod.yaml со следующим содержимым:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: django
  namespace: edu-naughty-pike
spec:
  containers:
  - name: django
    image: django_app:test
    ports:
    - containerPort: 80
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


## Как подготовить dev окружение

Secret в Kubernetes представляет собой объект, который хранит конфиденциальные данные. Секреты можно создать несколькими способами, включая использование kubectl или путем создания манифеста YAML.

### Создание секрета через kubectl

```sh
kubectl create secret generic <secret-name> \
  --from-literal=<key1>=<value1> \
  --from-literal=<key2>=<value2> \
  --namespace=<namespace>
```

**Создание секрета из файла**

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

