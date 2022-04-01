# Домашнее задание к занятию "13.3 работа с kubectl"
## Задание 1: проверить работоспособность каждого компонента
Для проверки работы можно использовать 2 способа: port-forward и exec. Используя оба способа, проверьте каждый компонент:
* сделайте запросы к бекенду;
* сделайте запросы к фронту;
* подключитесь к базе данных.

## Ответ на задание 1

Поднимаем поды (из задания ![13-kubernetes-config-01-objects](https://github.com/VitalyMozhaev/devkub-homeworks/tree/main/13-kubernetes-config-01-objects])):
```bash
sudo kubectl apply -f ./prod/
deployment.apps/backend created
service/backend created
statefulset.apps/db created
service/db created
deployment.apps/frontend created
service/frontend created
```

Проверяем ресурсы deployment, поды, сервисы, statefulset:
```bash
kubectl get deploy,po,svc,statefulset
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend    1/1     1            1           4m19s
deployment.apps/frontend   1/1     1            1           4m18s

NAME                            READY   STATUS    RESTARTS   AGE
pod/backend-58d47cccc8-htxn9    1/1     Running   0          4m19s
pod/db-0                        1/1     Running   0          4m19s
pod/frontend-847866564f-tvlm2   1/1     Running   0          4m18s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/backend    ClusterIP   10.233.52.140   <none>        9000/TCP   4m19s
service/db         ClusterIP   10.233.39.195   <none>        5432/TCP   4m19s
service/frontend   ClusterIP   10.233.42.131   <none>        8000/TCP   4m18s

NAME                  READY   AGE
statefulset.apps/db   1/1     4m19s
```

## Способ 1: kubectl exec

Проверяем доступ из backend до других подов, включая самого себя:
```bash
# до frontend
sudo kubectl exec -it backend-58d47cccc8-htxn9 -- curl -s -m 1 frontend:8000
<html><body><h1>It works!</h1></body></html>

# до backend
sudo kubectl exec -it backend-58d47cccc8-htxn9 -- curl -s -m 1 backend:9000
Praqma Network MultiTool (with NGINX) - backend-58d47cccc8-htxn9 - 10.233.90.110

# до db
sudo kubectl exec -it backend-58d47cccc8-htxn9 -- curl -s -m 1 db:5432
Praqma Network MultiTool (with NGINX) - db-0 - 10.233.90.111
```

Аналогичным образом можно проверить доступ с любого пода на любой другой.

## Способ 2: kubectl port-forward

Проверяем доступ без port-forward:
```bash
curl localhost:9000
curl: (7) Failed to connect to localhost port 9000: В соединении отказано
```

Пробрасываем порт до нужного сервиса:
```bash
sudo kubectl port-forward svc/frontend 8000
Forwarding from 127.0.0.1:8000 -> 80
Forwarding from [::1]:8000 -> 80
```

Пока запущена команда port-forward в терминале, доступ из другого терминала есть:
```bash
curl localhost:8000
<html><body><h1>It works!</h1></body></html>
```

Пробрасываем порт до нужного сервиса:
```bash
sudo kubectl port-forward svc/backend 9000
Forwarding from 127.0.0.1:9000 -> 9000
Forwarding from [::1]:9000 -> 9000
```

Пока запущена команда port-forward в терминале, доступ из другого терминала есть:
```bash
curl localhost:9000
Praqma Network MultiTool (with NGINX) - backend-58d47cccc8-htxn9 - 10.233.90.110
```

Пробрасываем порт до нужного сервиса:
```bash
sudo kubectl port-forward svc/db 5432
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
```

Пока запущена команда port-forward в терминале, доступ из другого терминала есть:
```bash
curl localhost:5432
Praqma Network MultiTool (with NGINX) - db-0 - 10.233.90.111
```

## Задание 2: ручное масштабирование

При работе с приложением иногда может потребоваться вручную добавить пару копий. Используя команду kubectl scale, попробуйте увеличить количество бекенда и фронта до 3. Проверьте, на каких нодах оказались копии после каждого действия (kubectl describe, kubectl get pods -o wide). После уменьшите количество копий до 1.

## Ответ на задание 2

Смотрим состав кластера:
```bash
sudo kubectl get nodes
NAME    STATUS   ROLES                  AGE   VERSION
cp1     Ready    control-plane,master   12d   v1.23.5
node1   Ready    <none>                 12d   v1.23.5
```

Для просмотра распределения подов при масштабировании необходимо минимум 2 ноды. Поднимим ещё одну виртуалку и внесём изменения в hosts.yaml:
```bash
all:
  hosts:
    cp1:
      ansible_host: 192.168.0.122
      ip: 192.168.0.122
      access_ip: 192.168.0.122
    node1:
      ansible_host: 192.168.0.123
      ip: 192.168.0.123
      access_ip: 192.168.0.123
    node2:
      ansible_host: 192.168.0.137
      ip: 192.168.0.137
      access_ip: 192.168.0.137
  children:
    kube_control_plane:
      hosts:
        cp1:
    kube_node:
      hosts:
        node1:
        node2:
    etcd:
      hosts:
        cp1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

Копируем публичный ключ с виртуалки, с которой производится установка, на ноды:
```bash
cat ~/.ssh/id_rsa.pub | ssh kubeuser@192.168.0.137 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Поднимаем новую ноду:
```bash
ansible-playbook -i inventory/mycluster/hosts.yaml scale.yml -b -v --become-user=root --ask-become-pass
```

После установки kubespray нода добавлена в кластер:
```bash
sudo kubectl get nodes -o wide
NAME    STATUS   ROLES                  AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION    CONTAINER-RUNTIME
cp1     Ready    control-plane,master   13d   v1.23.5   192.168.0.122   <none>        Debian GNU/Linux 10 (buster)   4.19.0-19-amd64   containerd://1.6.1
node1   Ready    <none>                 13d   v1.23.5   192.168.0.123   <none>        Debian GNU/Linux 10 (buster)   4.19.0-20-amd64   containerd://1.6.1
node2   Ready    <none>                 24h   v1.23.5   192.168.0.137   <none>        Debian GNU/Linux 10 (buster)   4.19.0-20-amd64   containerd://1.6.1
```

Поднимаем поды (из задания ![13-kubernetes-config-01-objects](https://github.com/VitalyMozhaev/devkub-homeworks/tree/main/13-kubernetes-config-01-objects])):
```bash
sudo kubectl apply -f ./prod/
deployment.apps/backend created
service/backend created
statefulset.apps/db created
service/db created
deployment.apps/frontend created
service/frontend created
```

Проверяем запущенные ресурсы:
```bash
sudo kubectl get deploy,po
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend    1/1     1            1           46s
deployment.apps/frontend   1/1     1            1           46s

NAME                            READY   STATUS    RESTARTS   AGE
pod/backend-58d47cccc8-fnj8c    1/1     Running   0          46s
pod/db-0                        1/1     Running   0          46s
pod/frontend-847866564f-ptmhs   1/1     Running   0          46s
```

Увеличиваем количество реплик для backend:
```bash
sudo kubectl scale deploy backend --replicas=3
deployment.apps/backend scaled
```

Смотрим результат:
```bash
sudo kubectl describe deploy backend
Name:                   backend
Namespace:              objects
CreationTimestamp:      Fri, 01 Apr 2022 14:07:28 +0300
Labels:                 app=backend
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=backend
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=backend
  Containers:
   backend:
    Image:      praqma/network-multitool:alpine-extra
    Port:       9000/TCP
    Host Port:  0/TCP
    Environment:
      HTTP_PORT:  9000
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   backend-58d47cccc8 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  74s   deployment-controller  Scaled up replica set backend-58d47cccc8 to 1
  Normal  ScalingReplicaSet  32s   deployment-controller  Scaled up replica set backend-58d47cccc8 to 3



sudo kubectl get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP              NODE    NOMINATED NODE   READINESS GATES
backend-58d47cccc8-fnj8c    1/1     Running   0          103s   10.233.90.115   node1   <none>           <none>
backend-58d47cccc8-rv28p    1/1     Running   0          61s    10.233.96.30    node2   <none>           <none>
backend-58d47cccc8-tt5ds    1/1     Running   0          61s    10.233.90.116   node1   <none>           <none>
db-0                        1/1     Running   0          103s   10.233.96.29    node2   <none>           <none>
frontend-847866564f-ptmhs   1/1     Running   0          103s   10.233.90.114   node1   <none>           <none>
```

Как видно, новые реплики распределяются по двум рабочим нодам.

Увеличиваем количество реплик для frontend:
```bash
sudo kubectl scale deploy frontend --replicas=3
deployment.apps/frontend scaled
```

Смотрим результат:
```bash
sudo kubectl describe deploy frontend
Name:                   frontend
Namespace:              objects
CreationTimestamp:      Fri, 01 Apr 2022 14:07:28 +0300
Labels:                 app=frontend
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=frontend
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=frontend
  Containers:
   frontend:
    Image:        httpd:2.4-alpine
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   frontend-847866564f (3/3 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  2m29s  deployment-controller  Scaled up replica set frontend-847866564f to 1
  Normal  ScalingReplicaSet  25s    deployment-controller  Scaled up replica set frontend-847866564f to 3



sudo kubectl get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP              NODE    NOMINATED NODE   READINESS GATES
backend-58d47cccc8-fnj8c    1/1     Running   0          2m46s   10.233.90.115   node1   <none>           <none>
backend-58d47cccc8-rv28p    1/1     Running   0          2m4s    10.233.96.30    node2   <none>           <none>
backend-58d47cccc8-tt5ds    1/1     Running   0          2m4s    10.233.90.116   node1   <none>           <none>
db-0                        1/1     Running   0          2m46s   10.233.96.29    node2   <none>           <none>
frontend-847866564f-47ngb   1/1     Running   0          42s     10.233.90.117   node1   <none>           <none>
frontend-847866564f-lhjb9   1/1     Running   0          42s     10.233.96.31    node2   <none>           <none>
frontend-847866564f-ptmhs   1/1     Running   0          2m46s   10.233.90.114   node1   <none>           <none>
```

Опять же видим, новые реплики распределяются по двум рабочим нодам.

Уменьшаем количество реплик до 1:
```bash
sudo kubectl scale deploy frontend --replicas=1
deployment.apps/frontend scaled

sudo kubectl get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP              NODE    NOMINATED NODE   READINESS GATES
backend-58d47cccc8-fnj8c    1/1     Running   0          4m26s   10.233.90.115   node1   <none>           <none>
backend-58d47cccc8-rv28p    1/1     Running   0          3m44s   10.233.96.30    node2   <none>           <none>
backend-58d47cccc8-tt5ds    1/1     Running   0          3m44s   10.233.90.116   node1   <none>           <none>
db-0                        1/1     Running   0          4m26s   10.233.96.29    node2   <none>           <none>
frontend-847866564f-lhjb9   1/1     Running   0          2m22s   10.233.96.31    node2   <none>           <none>


sudo kubectl scale deploy backend --replicas=1
deployment.apps/backend scaled

sudo kubectl get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
backend-58d47cccc8-rv28p    1/1     Running   0          4m25s   10.233.96.30   node2   <none>           <none>
db-0                        1/1     Running   0          5m7s    10.233.96.29   node2   <none>           <none>
frontend-847866564f-lhjb9   1/1     Running   0          3m3s    10.233.96.31   node2   <none>           <none>
```

После уменьшения реплик до 1 на каждый сервис, их расположение на нодах может поменяться.


---

PS:

Просмотр изменений с периодичностью 2 сек:
```
sudo watch 'kubectl get nodes,deploy,po,svc,statefulset -o wide'
```

Для удаления всего из конкретного namespace:
```
sudo kubectl -n objects delete all --all
```

---

Так же встретился с проблемой (создавались новые поды, а старые отваливались со статусом `evicted`).

Пригодились команды
```
# Вывод логов в реальном времени
sudo kubectl logs -f pod/frontend-847866564f-wc4rh

# Вывод логов последнего запуска
sudo kubectl logs -p pod/frontend-847866564f-zldnn

# Просмотр описания пода, статуса и событий
sudo kubectl describe pod backend-58d47cccc8-sv5vj

...
Status:       Pending
...
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  8s (x10 over 9m22s)  default-scheduler  0/3 nodes are available: 3 node(s) had taint {key1: value1}, that the pod didn't tolerate.

Так же для определения проблемы использовал:
sudo kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
NAME    TAINTS
cp1     [map[effect:NoSchedule key:node-role.kubernetes.io/master]]
node1   <none>
node2   <none>
```

Как выяснилось, ошибка возникала из-за того, что на нодах не хватало дискового пространства, в следствии чего kubelet “выселял” пользовательские поды.

После увеличения размера дискового пространства на виртуалках проблема исчезла.



