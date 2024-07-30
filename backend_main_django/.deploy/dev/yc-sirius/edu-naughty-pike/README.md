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