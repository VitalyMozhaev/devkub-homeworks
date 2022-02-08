# Ответы на домашнее задание к занятию "12.1 Компоненты Kubernetes"

Вы DevOps инженер в крупной компании с большим парком сервисов. Ваша задача — разворачивать эти продукты в корпоративном кластере. 

## Задача 1: Установить Minikube

Для экспериментов и валидации ваших решений вам нужно подготовить тестовую среду для работы с Kubernetes. Оптимальное решение — развернуть на рабочей машине Minikube.

### Как поставить на AWS:
- создать EC2 виртуальную машину (Ubuntu Server 20.04 LTS (HVM), SSD Volume Type) с типом **t3.small**. Для работы потребуется настроить Security Group для доступа по ssh. Не забудьте указать keypair, он потребуется для подключения.
- подключитесь к серверу по ssh (ssh ubuntu@<ipv4_public_ip> -i <keypair>.pem)
- установите миникуб и докер следующими командами:
  - sudo apt-get install -y apt-transport-https ca-certificates curl
  - curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
  - chmod +x ./kubectl
  - sudo mv ./kubectl /usr/local/bin/kubectl
  - sudo apt-get update && sudo apt-get install docker.io conntrack -y
  - curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
- проверить версию можно командой `minikube version`

```bash
minikube version: v1.24.0
commit: 76b94fb3c4e8ac5062daf70d60cf03ddcc0a741b
```

- переключаемся на root и запускаем миникуб: `minikube start --vm-driver=none`

```bash
minikube start --apiserver-ips=192.168.1.106 --listen-address=0.0.0.0 --vm-driver=none
```
  
- после запуска стоит проверить статус: `minikube status`

```bash
minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

- запущенные служебные компоненты можно увидеть командой: `kubectl get pods --namespace=kube-system`

```bash
kubectl get pods --namespace=kube-system
NAME                              READY   STATUS    RESTARTS   AGE
coredns-78fcd69978-5jxzm          1/1     Running   0          3m53s
etcd-vagrant                      1/1     Running   0          4m5s
kube-apiserver-vagrant            1/1     Running   0          4m7s
kube-controller-manager-vagrant   1/1     Running   0          4m5s
kube-proxy-9lzpp                  1/1     Running   0          3m53s
kube-scheduler-vagrant            1/1     Running   0          4m5s
storage-provisioner               1/1     Running   0          4m4s
```
  
### Для сброса кластера стоит удалить кластер и создать заново:
- `minikube delete`
- `minikube start --vm-driver=none`

Возможно, для повторного запуска потребуется выполнить команду: `sudo sysctl fs.protected_regular=0`

Инструкция по установке Minikube - [ссылка](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)

**Важно**: t3.small не входит во free tier, следите за бюджетом аккаунта и удаляйте виртуалку.
  
```bash
...
...
* Verifying Kubernetes components...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Enabled addons: storage-provisioner, default-storageclass
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

## Задача 2: Запуск Hello World
После установки Minikube требуется его проверить. Для этого подойдет стандартное приложение hello world. А для доступа к нему потребуется ingress.

- развернуть через Minikube тестовое приложение по [туториалу](https://kubernetes.io/ru/docs/tutorials/hello-minikube/#%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D0%BD%D0%B8%D0%B5-%D0%BA%D0%BB%D0%B0%D1%81%D1%82%D0%B5%D1%80%D0%B0-minikube)
  
```bash
sudo kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-node created

sudo kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   0/1     1            0           23s
sudo kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   1/1     1            1           31s
  
sudo kubectl get po
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6b89d599b9-2kxfg   1/1     Running   0          116s
  
sudo kubectl get events
LAST SEEN   TYPE     REASON              OBJECT                             MESSAGE
2m21s       Normal   Scheduled           pod/hello-node-6b89d599b9-2kxfg    Successfully assigned default/hello-node-6b89d599b9-2kxfg to kuber
2m19s       Normal   Pulling             pod/hello-node-6b89d599b9-2kxfg    Pulling image "k8s.gcr.io/echoserver:1.4"
2m          Normal   Pulled              pod/hello-node-6b89d599b9-2kxfg    Successfully pulled image "k8s.gcr.io/echoserver:1.4" in 19.172427855s
119s        Normal   Created             pod/hello-node-6b89d599b9-2kxfg    Created container echoserver
118s        Normal   Started             pod/hello-node-6b89d599b9-2kxfg    Started container echoserver
2m21s       Normal   SuccessfulCreate    replicaset/hello-node-6b89d599b9   Created pod: hello-node-6b89d599b9-2kxfg
2m21s       Normal   ScalingReplicaSet   deployment/hello-node              Scaled up replica set hello-node-6b89d599b9 to 1

```
  
- установить аддоны ingress и dashboard

```bash
sudo minikube addons list
|-----------------------------|----------|--------------|--------------------------------|
|         ADDON NAME          | PROFILE  |    STATUS    |           MAINTAINER           |
|-----------------------------|----------|--------------|--------------------------------|
| ambassador                  | minikube | disabled     | third-party (ambassador)       |
| auto-pause                  | minikube | disabled     | google                         |
| csi-hostpath-driver         | minikube | disabled     | kubernetes                     |
| dashboard                   | minikube | disabled     | kubernetes                     |
| default-storageclass        | minikube | enabled ✅   | kubernetes                     |
| efk                         | minikube | disabled     | third-party (elastic)          |
| freshpod                    | minikube | disabled     | google                         |
| gcp-auth                    | minikube | disabled     | google                         |
| gvisor                      | minikube | disabled     | google                         |
| helm-tiller                 | minikube | disabled     | third-party (helm)             |
| ingress                     | minikube | disabled     | unknown (third-party)          |
| ingress-dns                 | minikube | disabled     | google                         |
| istio                       | minikube | disabled     | third-party (istio)            |
| istio-provisioner           | minikube | disabled     | third-party (istio)            |
| kubevirt                    | minikube | disabled     | third-party (kubevirt)         |
| logviewer                   | minikube | disabled     | unknown (third-party)          |
| metallb                     | minikube | disabled     | third-party (metallb)          |
| metrics-server              | minikube | disabled     | kubernetes                     |
| nvidia-driver-installer     | minikube | disabled     | google                         |
| nvidia-gpu-device-plugin    | minikube | disabled     | third-party (nvidia)           |
| olm                         | minikube | disabled     | third-party (operator          |
|                             |          |              | framework)                     |
| pod-security-policy         | minikube | disabled     | unknown (third-party)          |
| portainer                   | minikube | disabled     | portainer.io                   |
| registry                    | minikube | disabled     | google                         |
| registry-aliases            | minikube | disabled     | unknown (third-party)          |
| registry-creds              | minikube | disabled     | third-party (upmc enterprises) |
| storage-provisioner         | minikube | enabled ✅   | google                         |
| storage-provisioner-gluster | minikube | disabled     | unknown (third-party)          |
| volumesnapshots             | minikube | disabled     | kubernetes                     |
|-----------------------------|----------|--------------|--------------------------------|


sudo minikube addons enable ingress
  - Используется образ k8s.gcr.io/ingress-nginx/controller:v1.1.0
  - Используется образ k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
  - Используется образ k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
* Verifying ingress addon...
* The 'ingress' addon is enabled
  
sudo minikube addons list
|-----------------------------|----------|--------------|--------------------------------|
|         ADDON NAME          | PROFILE  |    STATUS    |           MAINTAINER           |
|-----------------------------|----------|--------------|--------------------------------|
| ambassador                  | minikube | disabled     | third-party (ambassador)       |
| auto-pause                  | minikube | disabled     | google                         |
| csi-hostpath-driver         | minikube | disabled     | kubernetes                     |
| dashboard                   | minikube | disabled     | kubernetes                     |
| default-storageclass        | minikube | enabled ✅   | kubernetes                     |
| efk                         | minikube | disabled     | third-party (elastic)          |
| freshpod                    | minikube | disabled     | google                         |
| gcp-auth                    | minikube | disabled     | google                         |
| gvisor                      | minikube | disabled     | google                         |
| helm-tiller                 | minikube | disabled     | third-party (helm)             |
| ingress                     | minikube | enabled ✅   | unknown (third-party)          |
| ingress-dns                 | minikube | disabled     | google                         |
| istio                       | minikube | disabled     | third-party (istio)            |
| istio-provisioner           | minikube | disabled     | third-party (istio)            |
| kubevirt                    | minikube | disabled     | third-party (kubevirt)         |
| logviewer                   | minikube | disabled     | unknown (third-party)          |
| metallb                     | minikube | disabled     | third-party (metallb)          |
| metrics-server              | minikube | disabled     | kubernetes                     |
| nvidia-driver-installer     | minikube | disabled     | google                         |
| nvidia-gpu-device-plugin    | minikube | disabled     | third-party (nvidia)           |
| olm                         | minikube | disabled     | third-party (operator          |
|                             |          |              | framework)                     |
| pod-security-policy         | minikube | disabled     | unknown (third-party)          |
| portainer                   | minikube | disabled     | portainer.io                   |
| registry                    | minikube | disabled     | google                         |
| registry-aliases            | minikube | disabled     | unknown (third-party)          |
| registry-creds              | minikube | disabled     | third-party (upmc enterprises) |
| storage-provisioner         | minikube | enabled ✅   | google                         |
| storage-provisioner-gluster | minikube | disabled     | unknown (third-party)          |
| volumesnapshots             | minikube | disabled     | kubernetes                     |
|-----------------------------|----------|--------------|--------------------------------|


sudo minikube dashboard
* Enabling dashboard ...
  - Используется образ kubernetesui/metrics-scraper:v1.0.7
  - Используется образ kubernetesui/dashboard:v2.3.1
* Verifying dashboard health ...
* Launching proxy ...
* Verifying proxy health ...
http://127.0.0.1:44241/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```
  
Для доступа к Dashboard на виртуалке извне необходимо:
```bash
sudo kubectl proxy --address 192.168.1.106 --accept-hosts '.*'
Starting to serve on 10.10.1.106:8001
```

![](https://github.com/VitalyMozhaev/devkub-homeworks/blob/main/12-kubernetes-01-intro/Dashboard.png)
  
## Задача 3: Установить kubectl

Подготовить рабочую машину для управления корпоративным кластером. Установить клиентское приложение kubectl.
- подключиться к minikube

```bash
# На виртуалке с кластером выводим конфигурацию:
sudo kubectl config view
# Копируем конфиг на рабочую машину (другую виртуалку) в каталог /home/user/.kube
# и переносим сертификаты в каталог /home/user/.minikube/...
# Проверяем удалённое управление кластером с рабочей машины:
kubectl get ns
NAME              STATUS   AGE
default           Active   145m
kube-node-lease   Active   146m
kube-public       Active   146m
kube-system       Active   146m
```

- проверить работу приложения из задания 2, запустив port-forward до кластера

```bash
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
service/hello-node exposed
  
kubectl get services
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-node   LoadBalancer   10.98.155.178   <pending>     8080:30276/TCP   8s
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP          174m
  
sudo minikube service hello-node
|-----------|------------|-------------|----------------------------|
| NAMESPACE |    NAME    | TARGET PORT |             URL            |
|-----------|------------|-------------|----------------------------|
| default   | hello-node |        8080 | http://192.168.1.106:30276 |
|-----------|------------|-------------|----------------------------|
* Opening service default/hello-node in default browser...
  http://192.168.1.106:30276
  
sudo kubectl cluster-info
Kubernetes control plane is running at https://192.168.1.106:8443
CoreDNS is running at https://192.168.1.106:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Проверяем работу hello-node:

```bash
curl http://192.168.1.106:30276

CLIENT VALUES:
client_address=172.17.0.1
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://192.168.1.106:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=192.168.1.106:30276
user-agent=curl/7.64.0
BODY:
-no body in request-
```

```bash
# Под запущен
kubectl get po --namespace=default
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6b89d599b9-2kxfg   1/1     Running   0          17h

# Удаляем Под
kubectl delete pod hello-node-6b89d599b9-2kxfg --namespace=default
pod "hello-node-6b89d599b9-2kxfg" deleted

# Kubernetes создаёт новый под
kubectl get po --namespace=default
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6b89d599b9-4rjwp   1/1     Running   0          14s
```
  
## Задача 4 (*): собрать через ansible (необязательное)

Профессионалы не делают одну и ту же задачу два раза. Давайте закрепим полученные навыки, автоматизировав выполнение заданий  ansible-скриптами. При выполнении задания обратите внимание на доступные модули для k8s под ansible.
 - собрать роль для установки minikube на aws сервисе (с установкой ingress)
 - собрать роль для запуска в кластере hello world
