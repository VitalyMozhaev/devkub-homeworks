# Ответы на домашнее задание к занятию "12.2 Команды для работы с Kubernetes"
Кластер — это сложная система, с которой крайне редко работает один человек. Квалифицированный devops умеет наладить работу всей команды, занимающейся каким-либо сервисом.
После знакомства с кластером вас попросили выдать доступ нескольким разработчикам. Помимо этого требуется служебный аккаунт для просмотра логов.

## Задание 1: Запуск пода из образа в деплойменте
Для начала следует разобраться с прямым запуском приложений из консоли. Такой подход поможет быстро развернуть инструменты отладки в кластере. Требуется запустить деплоймент на основе образа из hello world уже через deployment. Сразу стоит запустить 2 копии приложения (replicas=2). 

Требования:
 * пример из hello world запущен в качестве deployment
 * количество реплик в deployment установлено в 2

```bash
kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4 --replicas=2
deployment.apps/hello-node created
```

 * наличие deployment можно проверить командой kubectl get deployment

```bash
kubectl get deployment
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   2/2     2            2           90s
```

 * наличие подов можно проверить командой kubectl get pods

```bash
kubectl get po
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6b89d599b9-gm8js   1/1     Running   0          93s
hello-node-6b89d599b9-srg6k   1/1     Running   0          93s
```

## Задание 2: Просмотр логов для разработки
Разработчикам крайне важно получать обратную связь от штатно-работающего приложения и, еще важнее, об ошибках в его работе. 
Требуется создать пользователя и выдать ему доступ на чтение конфигурации и логов подов в app-namespace.

Требования: 
 * создан новый токен доступа для пользователя
 * пользователь прописан в локальный конфиг (~/.kube/config, блок users)
 * пользователь может просматривать логи подов и их конфигурацию (kubectl logs pod <pod_id>, kubectl describe pod <pod_id>)

## Ответ:

Проверим, установлен ли RBAC Manager (он нужен для создания создание RoleBinding'ов):
```bash
kubectl cluster-info dump | grep authorization-mode
                            "--authorization-mode=Node,RBAC",
```

Создаём новый namespace:
```bash
kubectl create ns app-namespace
namespace/app-namespace created
```
Создаём deployment:
```bash
kubectl create deployment app-deployment --image=k8s.gcr.io/echoserver:1.4 --namespace=app-namespace --replicas=2
deployment.apps/app-deployment created

kubectl get po
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6b89d599b9-7jlfn   1/1     Running   0          164m
hello-node-6b89d599b9-98rwl   1/1     Running   0          164m

kubectl get po -n app-namespace
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-644d6775c5-cflld   1/1     Running   0          44s
app-deployment-644d6775c5-j8pm5   1/1     Running   0          44s
```
Создаём нового пользователя:
```bash
kubectl create serviceaccount developer
serviceaccount/developer created
```
Создаём роль:
```bash
kubectl create clusterrole developerRole --verb=get --verb=list --verb=watch --resource=pods --resource=pods/log
clusterrole.rbac.authorization.k8s.io/developerRole created
```
Привязываем роль к пользователю:
```bash
kubectl create rolebinding developer --serviceaccount=default:developer --clusterrole=developerRole -n app-namespace
rolebinding.rbac.authorization.k8s.io/developer created
```
Запишем токен в креды:
```bash
kubectl config set-credentials kuberuser --token=$(kubectl describe secrets "$(kubectl describe serviceaccount developer | grep -i Tokens | awk '{print $2}')" | grep token: | awk '{print $2}')
User "kuberuser" set.
```
Установим новый контекст:
```bash
kubectl config set-context developerlogs --cluster=minikube --user=kuberuser
Context "developerlogs" created.

(EDIT: Можно было сразу добавить --namespace=app-namespace для установления namespace по-умолчанию для контекста.)

kubectl config get-contexts
CURRENT   NAME            CLUSTER    AUTHINFO    NAMESPACE
          developerlogs   minikube   kuberuser
*         minikube        minikube   minikube    default

# Сейчас доступ ещё есть:
kubectl get po
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6b89d599b9-7jlfn   1/1     Running   0          179m
hello-node-6b89d599b9-98rwl   1/1     Running   0          179m
```
Переключимся на новый контекст:
```bash
kubectl config use-context developerlogs
Switched to context "developerlogs".
```
Проверяем:
```bash
# Доступ теперь ограничен ролью
kubectl get po
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:developer" cannot list resource "pods" in API group "" in the namespace "default"

# Но внутри разрешённого ролью namespace доступ есть:
kubectl get po -n app-namespace
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-644d6775c5-cflld   1/1     Running   0          17m
app-deployment-644d6775c5-j8pm5   1/1     Running   0          17m

# Однако удалить pod пользователь не имеет права:
kubectl delete pod app-deployment-644d6775c5-j8pm5 -n app-namespace
Error from server (Forbidden): pods "app-deployment-644d6775c5-j8pm5" is forbidden: User "system:serviceaccount:default:developer" cannot delete resource "pods" in API group "" in the namespace "app-namespace"
```
Проверим чтение логов в подах:
```bash
kubectl logs app-deployment-644d6775c5-cflld
Error from server (Forbidden): pods "app-deployment-644d6775c5-cflld" is forbidden: User "system:serviceaccount:default:developer" cannot get resource "pods" in API group "" in the namespace "default"

# Если не указывать namespace, то используется namespace "default", который не доступен по ограничению прав.
# Выберем namespace app-namespace
kubectl logs app-deployment-644d6775c5-cflld -n app-namespace

# Вернул пустоту, т.к. логов пока нет.
# Можно просматривать логи в интерактивном режиме:
kubectl logs -f app-deployment-644d6775c5-cflld -n app-namespace
```
Посмотрим конфигурацию (доступ есть):
```bash
kubectl describe pod app-deployment-644d6775c5-cflld -n app-namespace
Name:         app-deployment-644d6775c5-cflld
Namespace:    app-namespace
Priority:     0
Node:         kuber/192.168.1.106
Start Time:   Thu, 17 Feb 2022 13:47:02 +0300
Labels:       app=app-deployment
              pod-template-hash=644d6775c5
Annotations:  <none>
Status:       Running
IP:           172.17.0.7
IPs:
  IP:           172.17.0.7
Controlled By:  ReplicaSet/app-deployment-644d6775c5
Containers:
  echoserver:
    Container ID:   docker://ea156232c100ab9134ef78c1a1b37df6e2bb293e2b42b2e52042181d7a6a4a52
    Image:          k8s.gcr.io/echoserver:1.4
    Image ID:       docker-pullable://k8s.gcr.io/echoserver@sha256:5d99aa1120524c801bc8c1a7077e8f5ec122ba16b6dda1a5d3826057f67b9bcb
    ...
    ...
```
Пользователь прописан в config
```bash
kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/hwuser/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Mon, 07 Feb 2022 13:50:41 MSK
        provider: minikube.sigs.k8s.io
        version: v1.25.1
      name: cluster_info
    server: https://192.168.1.106:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: kuberuser
  name: developerlogs
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Mon, 07 Feb 2022 13:50:41 MSK
        provider: minikube.sigs.k8s.io
        version: v1.25.1
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: developerlogs
kind: Config
preferences: {}
users:
- name: kuberuser
  user:
    token: REDACTED
- name: minikube
  user:
    client-certificate: /home/hwuser/.minikube/profiles/minikube/client.crt
    client-key: /home/hwuser/.minikube/profiles/minikube/client.key
```


## Задание 3: Изменение количества реплик
Поработав с приложением, вы получили запрос на увеличение количества реплик приложения для нагрузки. Необходимо изменить запущенный deployment, увеличив количество реплик до 5. Посмотрите статус запущенных подов после увеличения реплик. 

Требования:
 * в deployment из задания 1 изменено количество реплик на 5

```bash
sudo kubectl scale --replicas=5 deployment/hello-node
deployment.apps/hello-node scaled
```

 * проверить что все поды перешли в статус running (kubectl get pods)

```bash
kubectl get po
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6b89d599b9-689q5   1/1     Running   0          85s
hello-node-6b89d599b9-7jlfn   1/1     Running   0          10s
hello-node-6b89d599b9-98rwl   1/1     Running   0          10s
hello-node-6b89d599b9-kws8l   1/1     Running   0          10s
hello-node-6b89d599b9-lzt2d   1/1     Running   0          85s
```
