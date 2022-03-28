# Ответы на домашнее задание к занятию "12.5 Сетевые решения CNI"

После работы с Flannel появилась необходимость обеспечить безопасность для приложения. Для этого лучше всего подойдет Calico.

## Задание 1: установить в кластер CNI плагин Calico

Для проверки других сетевых решений стоит поставить отличный от Flannel плагин — например, Calico. Требования:

* установка производится через ansible/kubespray;

```text
Calico утанавливается по-умолчанию через ansible/kubespray
```

* после применения следует настроить политику доступа к hello-world извне. Инструкции [kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/network-policies/), [Calico](https://docs.projectcalico.org/about/about-network-policy)

Проверяем что есть на кластере:
```
sudo kubectl get nodes
NAME    STATUS   ROLES                  AGE   VERSION
cp1     Ready    control-plane,master   9d    v1.23.4
node1   Ready    <none>                 9d    v1.23.4

sudo kubectl get ns
NAME              STATUS   AGE
default           Active   9d
kube-node-lease   Active   9d
kube-public       Active   9d
kube-system       Active   9d

sudo kubectl get deploy,po
No resources found in default namespace.
```

Создаём пространство имён:
```
sudo kubectl create namespace net-policy
namespace/net-policy created

sudo kubectl get ns
NAME              STATUS   AGE
default           Active   9d
kube-node-lease   Active   9d
kube-public       Active   9d
kube-system       Active   9d
net-policy        Active   39s
```

Переключаемся на пространство имён `net-policy`:
```
sudo kubectl config set-context --current --namespace=net-policy
Context "kubernetes-admin@cluster.local" modified.
```

Проверим текущий namespace:
```
sudo kubectl config view --minify | grep namespace:
    namespace: net-policy

sudo kubectl get deploy,po
No resources found in net-policy namespace.
```

Для создания вручную:

`sudo kubectl -n net-policy create deployment hello --image=k8s.gcr.io/echoserver:1.4`

`sudo kubectl -n net-policy expose deployment hello --port=80 --target-port=8080`

Создание нескольких deployment hello вызывало странные ошибки и множило поды, поэтому воспользовался примерами из лекции.

Для проверки network policy создадим два deployment: `frontend` и `backend`.

Очень удобно создавать через файл yaml:

Создаём deployment `frontend` (/templates/main/frontend.yaml):
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
  namespace: net-policy
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
        - image: praqma/network-multitool:alpine-extra
          imagePullPolicy: IfNotPresent
          name: network-multitool
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: net-policy
spec:
  ports:
    - name: web
      port: 80
  selector:
    app: frontend

```

Создаём deployment `backend` (/templates/main/backend.yaml):
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: backend
  name: backend
  namespace: net-policy
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
        - image: praqma/network-multitool:alpine-extra
          imagePullPolicy: IfNotPresent
          name: network-multitool
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: net-policy
spec:
  ports:
    - name: web
      port: 80
  selector:
    app: backend

```

Поднимаем поды:
```
sudo kubectl apply -f ./templates/main/
deployment.apps/frontend created
service/frontend created
deployment.apps/backend created
service/backend created
```

Проверяем поды:
```
sudo kubectl get deploy,po
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend    1/1     1            1           11s
deployment.apps/frontend   1/1     1            1           11s

NAME                            READY   STATUS    RESTARTS   AGE
pod/backend-f785447b9-6s9qn     1/1     Running   0          11s
pod/frontend-8645d9cb9c-2dpgh   1/1     Running   0          11s
```

Проверяем доступы:
```
sudo kubectl exec frontend-8645d9cb9c-2dpgh -- curl -s -m 1 backend
Praqma Network MultiTool (with NGINX) - backend-f785447b9-6s9qn - 10.233.90.32
# доступ есть

sudo kubectl exec backend-f785447b9-6s9qn -- curl -s -m 1 frontend
Praqma Network MultiTool (with NGINX) - backend-f785447b9-6s9qn - 10.233.90.31
# доступ есть
```

Проверяем список правил NetworkPolicy:
```
# sudo kubectl get networkpolicies         # Есть короткая запись и это здорово!
sudo kubectl get netpol
No resources found in net-policy namespace.
```

Создаём NetworkPolicy deny all режима (/templates/network-policy/00-default.yaml)

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress

```

Применяем NetworkPolicy:
```
sudo kubectl apply -f ./templates/network-policy/00-default.yaml
networkpolicy.networking.k8s.io/default-deny-ingress created
```

Проверяем список правил NetworkPolicy:
```
sudo kubectl get netpol
NAME                   POD-SELECTOR   AGE
default-deny-ingress   <none>         24s
```

Проверяем доступы:
```
sudo kubectl exec frontend-8645d9cb9c-2dpgh -- curl -s -m 1 backend
command terminated with exit code 28
# доступа нет

sudo kubectl exec backend-f785447b9-6s9qn -- curl -s -m 1 frontend
command terminated with exit code 28
# доступа нет
```


Создаём NetworkPolicy с разрешениями (/templates/network-policy/frontend.yaml)

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend
  namespace: net-policy
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress

```


Создаём NetworkPolicy с разрешениями (/templates/network-policy/backend.yaml)

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend
  namespace: net-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: frontend
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443

```

Применяем NetworkPolicy
```
sudo kubectl apply -f ./templates/network-policy/
networkpolicy.networking.k8s.io/default-deny-ingress unchanged
networkpolicy.networking.k8s.io/backend created
networkpolicy.networking.k8s.io/frontend created
```

Проверяем список правил NetworkPolicy:
```
sudo kubectl get netpol
NAME                   POD-SELECTOR   AGE
backend                app=backend    22s
default-deny-ingress   <none>         4m43s
frontend               app=frontend   22s
```

Проверяем доступ:
```
sudo kubectl exec frontend-8645d9cb9c-2dpgh -- curl -s -m 1 backend
Praqma Network MultiTool (with NGINX) - backend-f785447b9-6s9qn - 10.233.90.26
# доступ есть

sudo kubectl exec backend-f785447b9-6s9qn -- curl -s -m 1 frontend
command terminated with exit code 28
# доступа нет
```

---

## PS:

Для удаления:
```
# Удаляем все deployment:
sudo kubectl delete deployment frontend backend

# Удаляем все service:
sudo kubectl delete service frontend backend

# Удалить все network policy:
sudo kubectl delete netpol --all
networkpolicy.networking.k8s.io "backend" deleted
networkpolicy.networking.k8s.io "default-deny-ingress" deleted
networkpolicy.networking.k8s.io "frontend" deleted
```

Доступ внутрь контейнера:
```
sudo kubectl exec -it pod-name /bin/bash
```

---

## Задание 2: изучить, что запущено по умолчанию

Самый простой способ — проверить командой calicoctl get <type>. Для проверки стоит получить список нод, ipPool и profile.
Требования: 
* установить утилиту calicoctl;

```text
calicoctl устанавливается по-умолчанию на CP-ноду через ansible/kubespray.
PS: Был вопрос, где запускать команду calicoctl get <type>, на CP или рабочей ноде?
calicoctl устанавливается только на CP-ноду и для работы самой Calico на рабочих нодах не нужна.
```

* получить 3 вышеописанных типа в консоли.

```bash
sudo calicoctl get node
NAME
cp1
node1

sudo calicoctl get ipPool
NAME           CIDR             SELECTOR
default-pool   10.233.64.0/18   all()

sudo calicoctl get profile
NAME
projectcalico-default-allow
kns.default
kns.kube-node-lease
kns.kube-public
kns.kube-system
kns.net-policy
ksa.default.default
ksa.kube-node-lease.default
ksa.kube-public.default
ksa.kube-system.attachdetach-controller
ksa.kube-system.bootstrap-signer
ksa.kube-system.calico-kube-controllers
ksa.kube-system.calico-node
ksa.kube-system.certificate-controller
ksa.kube-system.clusterrole-aggregation-controller
ksa.kube-system.coredns
ksa.kube-system.cronjob-controller
ksa.kube-system.daemon-set-controller
ksa.kube-system.default
ksa.kube-system.deployment-controller
ksa.kube-system.disruption-controller
ksa.kube-system.dns-autoscaler
ksa.kube-system.endpoint-controller
ksa.kube-system.endpointslice-controller
ksa.kube-system.endpointslicemirroring-controller
ksa.kube-system.ephemeral-volume-controller
ksa.kube-system.expand-controller
ksa.kube-system.generic-garbage-collector
ksa.kube-system.horizontal-pod-autoscaler
ksa.kube-system.job-controller
ksa.kube-system.kube-proxy
ksa.kube-system.namespace-controller
ksa.kube-system.node-controller
ksa.kube-system.nodelocaldns
ksa.kube-system.persistent-volume-binder
ksa.kube-system.pod-garbage-collector
ksa.kube-system.pv-protection-controller
ksa.kube-system.pvc-protection-controller
ksa.kube-system.replicaset-controller
ksa.kube-system.replication-controller
ksa.kube-system.resourcequota-controller
ksa.kube-system.root-ca-cert-publisher
ksa.kube-system.service-account-controller
ksa.kube-system.service-controller
ksa.kube-system.statefulset-controller
ksa.kube-system.token-cleaner
ksa.kube-system.ttl-after-finished-controller
ksa.kube-system.ttl-controller
ksa.net-policy.default
```
  
