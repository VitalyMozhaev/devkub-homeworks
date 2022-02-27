# Ответы на домашнее задание к занятию "12.4 Развертывание кластера на собственных серверах, лекция 2"
Новые проекты пошли стабильным потоком. Каждый проект требует себе несколько кластеров: под тесты и продуктив. Делать все руками — не вариант, поэтому стоит автоматизировать подготовку новых кластеров.

## Задание 1: Подготовить инвентарь kubespray
Новые тестовые кластеры требуют типичных простых настроек. Нужно подготовить инвентарь и проверить его работу. Требования к инвентарю:
* подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды;
* в качестве CRI — containerd;
* запуск etcd производить на мастере.

## Ответ на задание 1:

Подготовлены 3 виртуальные машины на Debian 10:
- виртуалка, с которой производится установка
- cp 1 (10.10.1.106)
- node 1 (10.10.1.118)

Создаём пользователя на рабочей ноде и мастер ноде (Control Plane), и даём права:
```bash
sudo adduser kubeuser
sudo usermod -aG sudo kubeuser
```

Устанавливаем необходимые пакеты:
```bash
sudo apt install apt-transport-https ca-certificates curl -y
# Проверяем python
python3 --version
# Устанавливаем pip3
sudo apt install python3-pip -y
# Устанавливаем git
sudo apt install git -y
```

Копируем проект из github и переходим в каталог kubespray:
```bash
git clone https://github.com/kubernetes-sigs/kubespray
cd kubespray/
```

Устанавливаем зависимости:
```bash
sudo pip3 install -r requirements.txt
# При установке зависимостей возникла ошибка, оказалось не все нужные пакеты установлены на нодах.
# Исправил установкой следующих пакетов:
sudo apt install --fix-broken
sudo apt-get install libssl-dev libffi-dev python-dev -y

# Пробуем повторно установить зависимости
sudo pip3 install -r requirements.txt
# Теперь всё хорошо
```

Создаём из примера свой inventory с названием mycluster:
```bash
cp -rfp inventory/sample inventory/mycluster
```

### Конфигурируем с запуском билдера.
Подготовим список IP адресов и скопируем их через пробел в первую команду:
```bash
declare -a IPS=(10.10.1.106 10.10.1.118)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

Правим полученный предыдущей командой файл inventory/mycluster/hosts.yaml
```bash
all:
  hosts:
    cp1:
      ansible_host: 10.10.1.106
      ip: 10.10.1.106
      access_ip: 10.10.1.106
    node1:
      ansible_host: 10.10.1.118
      ip: 10.10.1.118
      access_ip: 10.10.1.118
  children:
    kube_control_plane:
      hosts:
        cp1:
    kube_node:
      hosts:
        node1:
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

Создаём ssh ключи на локальной машине (в моём случае на виртуалке, с которой производится установка) выполняем:
```bash
ssh-keygen
```

Копируем созданный публичный ключ с виртуалки, с которой производится установка, на ноды:
```bash
cat ~/.ssh/id_rsa.pub | ssh kubeuser@10.10.1.106 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh kubeuser@10.10.1.118 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Для установки на Debian 10 исправил в файле \roles\kubernetes\preinstall\vars\ubuntu.yml
```bash
required_pkgs:
  - python3-apt
```

Containerd должен быть прописан в файле \kubespray\inventory\mycluster\group_vars\k8s_cluster\k8s_cluster.yml
```bash
container_manager: containerd
# Прописан по-умолчанию
```

Запускаем установку:
```bash
ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b -v --become-user=root --ask-become-pass
```

Встретился с проблемой:
```bash
TASK [bootstrap-os : Fetch /etc/os-release] ************************************
fatal: [cp1]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt: "}
fatal: [node1]: FAILED! => {"msg": "Timeout (12s) waiting for privilege escalation prompt: "}
```

Оказалось, имя виртуальной машины не соответствовало и команда повышения прав sudo выводила ошибку:
```bash
sudo: unable to resolve host cp1
sudo: unable to resolve host node1
```

Пришлось зайти на каждую машину и исправить имя
```bash
# На cp1:
hostname
cp1
sudo mcedit /etc/hosts
# Прописать:
127.0.1.1 cp1

# На node1:
hostname
node1
sudo mcedit /etc/hosts
# Прописать:
127.0.1.1 node1
```

После этого проблема ушла и всё отработало без ошибок:
```bash
...
# Много много вывода результатов работы ansible ...
...
PLAY RECAP *********************************************************************
cp1                        : ok=723  changed=146  unreachable=0    failed=0    skipped=1168 rescued=0    ignored=4
localhost                  : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node1                      : ok=505  changed=88   unreachable=0    failed=0    skipped=667  rescued=0    ignored=1
```

Проверяем кластер на виртуалке cp1:
```bash
sudo kubectl get nodes
[sudo] пароль для kubeuser:
NAME    STATUS   ROLES                  AGE   VERSION
cp1     Ready    control-plane,master   13m   v1.23.4
node1   Ready    <none>                 11m   v1.23.4
```

## Задание 2 (*): подготовить и проверить инвентарь для кластера в AWS
Часть новых проектов хотят запускать на мощностях AWS. Требования похожи:
* разворачивать 5 нод: 1 мастер и 4 рабочие ноды;
* работать должны на минимально допустимых EC2 — t3.small.
