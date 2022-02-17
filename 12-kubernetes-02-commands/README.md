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
