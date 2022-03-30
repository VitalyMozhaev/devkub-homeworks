# Ответы на домашнее задание к занятию "13.1 контейнеры, поды, deployment, statefulset, services, endpoints"
Настроив кластер, подготовьте приложение к запуску в нём. Приложение стандартное: бекенд, фронтенд, база данных. Его можно найти в папке 13-kubernetes-config.

## Задание 1: подготовить тестовый конфиг для запуска приложения
Для начала следует подготовить запуск приложения в stage окружении с простыми настройками. Требования:
* под содержит в себе 2 контейнера — фронтенд, бекенд;
* регулируется с помощью deployment фронтенд и бекенд;
* база данных — через statefulset.

## Ответ на задание 1 (Stage):

Создаём пространство имён:
```bash
sudo kubectl create ns objects
namespace/objects created
```

Проверяем пространства имён:
```bash
sudo kubectl get ns
NAME              STATUS   AGE
default           Active   19d
kube-node-lease   Active   19d
kube-public       Active   19d
kube-system       Active   19d
net-policy        Active   9d
objects           Active   92s
```

Переключаемся на пространство имён `objects`:
```bash
sudo kubectl config set-context --current --namespace=objects
Context "kubernetes-admin@cluster.local" modified.
```

Проверяем текущий namespace:
```bash
sudo kubectl config view --minify | grep namespace:
    namespace: objects
```

Создаём структуру каталогов:
```bash
cd
mkdir -p objects/{stage,prod} && cd objects
```

Создаём deployment `frontend + backend` (/objects/stage/frontback.yaml):
```bash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontback
  name: frontback
  namespace: objects
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontback
  template:
    metadata:
      labels:
        app: frontback
    spec:
      containers:
        - name: frontend
          image: httpd:2.4-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
        - name: backend
          image: praqma/network-multitool:alpine-extra
          imagePullPolicy: IfNotPresent
          env:
            - name: HTTP_PORT
              value: "9000"
          ports:
            - containerPort: 9000
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: frontback
  namespace: objects
spec:
  ports:
    - name: stage
      port: 8000
      targetPort: 80
  selector:
    app: frontback

```

Создаём deployment `db` (/objects/stage/db.yaml):
```bash
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: db
  name: db
  namespace: objects
spec:
  serviceName: "db"
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: praqma/network-multitool:alpine-extra
          imagePullPolicy: IfNotPresent
          env:
            - name: HTTP_PORT
              value: "5432"
          ports:
            - containerPort: 5432
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: db
  namespace: objects
spec:
  ports:
    - name: stage
      port: 5432
  selector:
    app: db

```

Поднимаем поды на stage:
```bash
sudo kubectl apply -f ./stage/
statefulset.apps/db created
service/db created
deployment.apps/frontback created
service/frontback created
```

Проверяем deployment, поды, сервисы, statefulset:
```bash
sudo kubectl get deploy,po,svc,statefulset
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/frontback   1/1     1            1           26m

NAME                             READY   STATUS    RESTARTS   AGE
pod/db-0                         1/1     Running   0          19s
pod/frontback-67dcb678f4-2zh56   2/2     Running   0          26m

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/db          ClusterIP   10.233.40.71    <none>        5432/TCP   19s
service/frontback   ClusterIP   10.233.39.157   <none>        8000/TCP   26m

NAME                  READY   AGE
statefulset.apps/db   1/1     19s
```

---

Удаляем:
```bash
# Удаляем все deployment:
sudo kubectl delete deployment frontback

# Удаляем все service:
sudo kubectl delete service frontback db

# Удаляем statefulset:
sudo kubectl delete statefulset db
```

## Задание 2: подготовить конфиг для production окружения
Следующим шагом будет запуск приложения в production окружении. Требования сложнее:
* каждый компонент (база, бекенд, фронтенд) запускаются в своем поде, регулируются отдельными deployment’ами;
* для связи используются service (у каждого компонента свой);
* в окружении фронта прописан адрес сервиса бекенда;
* в окружении бекенда прописан адрес сервиса базы данных.

## Ответ на задание 2 (Prod):

Создаём deployment `frontend` (/objects/prod/frontend.yaml):
```bash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
  namespace: objects
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: httpd:2.4-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: objects
spec:
  ports:
    - name: prod
      port: 8000
      targetPort: 80
  selector:
    app: frontend

```

Создаём deployment `backend` (/objects/prod/backend.yaml):
```bash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: backend
  name: backend
  namespace: objects
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: praqma/network-multitool:alpine-extra
          imagePullPolicy: IfNotPresent
          env:
            - name: HTTP_PORT
              value: "9000"
          ports:
            - containerPort: 9000
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: objects
spec:
  ports:
    - name: prod
      port: 9000
  selector:
    app: backend

```

Создаём deployment `db` (/objects/prod/db.yaml):
```bash
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: db
  name: db
  namespace: objects
spec:
  serviceName: "db"
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: praqma/network-multitool:alpine-extra
          imagePullPolicy: IfNotPresent
          env:
            - name: HTTP_PORT
              value: "5432"
          ports:
            - containerPort: 5432
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: db
  namespace: objects
spec:
  ports:
    - name: prod
      port: 5432
  selector:
    app: db

```

Поднимаем поды на prod:
```bash
sudo kubectl apply -f ./prod/
deployment.apps/backend created
service/backend created
statefulset.apps/db created
service/db created
deployment.apps/frontend created
service/frontend created
```

Проверяем deployment, поды, сервисы, statefulset:
```bash
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend    1/1     1            1           19s
deployment.apps/frontend   1/1     1            1           2m53s

NAME                            READY   STATUS    RESTARTS   AGE
pod/backend-58d47cccc8-jkbkh    1/1     Running   0          19s
pod/db-0                        1/1     Running   0          7s
pod/frontend-847866564f-5nrwv   1/1     Running   0          2m53s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/backend    ClusterIP   10.233.32.37    <none>        9000/TCP   19s
service/db         ClusterIP   10.233.20.107   <none>        5432/TCP   7s
service/frontend   ClusterIP   10.233.24.255   <none>        8000/TCP   2m53s

NAME                  READY   AGE
statefulset.apps/db   1/1     7s
```

---

Удаляем:
```bash
# Удаляем все deployment:
sudo kubectl delete deployment frontend backend

# Удаляем все service:
sudo kubectl delete service frontend backend db

# Удаляем statefulset:
sudo kubectl delete statefulset db
```

## Задание 3 (*): добавить endpoint на внешний ресурс api
Приложению потребовалось внешнее api, и для его использования лучше добавить endpoint в кластер, направленный на это api. Требования:
* добавлен endpoint до внешнего api (например, геокодер).

```
К сожалению, из-за отсутствия времение не получается проработать дополнительные задания.
```
