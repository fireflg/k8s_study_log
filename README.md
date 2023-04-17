# k8s_study_log
Оглавление
Мини кластер кубернетес.	1
Начало.	1
Поднимем dns  сервер.	3
Kubespray	5
Lens	8
Vscode	9
Первоначальная настройка кластера.	10
Namespaces	10
PriorityClass	11
Reloader	12

Мини кластер кубернетес.
Начало.
Создадим 7 виртуальных машин. 1 – master node, там будет расположен dns  сервер и ansible
3 – control node для etcd и kube api. 3 – worker node.
Для мастер ноды хватит таких параметров.
![image](https://user-images.githubusercontent.com/96112649/232605597-7288d6ac-5f7a-4528-999d-3133cceecf1d.png)
![image](https://user-images.githubusercontent.com/96112649/232605651-e1dfbe9b-04e2-4b50-929a-f1b9aa637a5b.png)

 
Буду использовать ubuntu20, денег на vmware workstation у меня нет, да и купить его сейчас нереально.
Вид сетевого интерфейса будет сетевой мост у всех машин, но зададим статические адреса.
 ![image](https://user-images.githubusercontent.com/96112649/232605668-742bf5e0-3f8a-4a8c-80ed-0689eef12076.png)


Для контрол и воркер нод сделаю по 2 гигабайта оперативной памяти. Но на 3-я воркер нода будет выступать в качестве резервной, так что будет 1 гиг оперативки, иначе мой пк не вытянет
В остальном примерно все тоже самое что и скринах выше.
Все же мой пк не выдержал 3 воркер ноды, поэтому конфиг будет такой. Контрол ноды желательно делать в нечетном количестве, оставлю 3.

 ![image](https://user-images.githubusercontent.com/96112649/232605685-c9fe95e3-200c-43bb-9efc-69f5b9a73e03.png)
![image](https://user-images.githubusercontent.com/96112649/232605712-04081bc8-7c81-4fd7-a380-4e8b51771cc9.png)


 

Поднимем dns  сервер.
Установим dnsmasql
sudo apt-get install dnsmasq
И resolvconf
sudo apt-get install resolvconf
Редактируем конфиги
sudo vim /etc/dnsmasq.conf
Добавляем в конец
 ![image](https://user-images.githubusercontent.com/96112649/232605757-25b4b325-4143-4907-b473-7ff3a2705a67.png)

 
Редактируем hosts
 ![image](https://user-images.githubusercontent.com/96112649/232605768-2dd3ba66-ed54-4725-b4e9-fbcac5321c28.png)

На остальных машинах, в нетплане делаем такое:  
![image](https://user-images.githubusercontent.com/96112649/232605780-2ada0397-befc-4dcb-ac8d-53b2b7c63947.png)

Не забыть убить системд-резолвед на мастере.
sudo systemctl disable systemd-resolved.service
sudo systemctl stop systemd-resolved
![image](https://user-images.githubusercontent.com/96112649/232605794-e13ebb25-1ae4-43e1-9d31-486209a870ed.png)

 
Установим ansible на хосте, но прежде прокинем ключи
Сгенерим ключ
ssh-keygen
ssh-copy-id control1
И копируем на все сервера
Устанавливаем ansible
apt-get install ansible
Проверим
 ![image](https://user-images.githubusercontent.com/96112649/232605809-32ebb867-e336-463e-9478-32fc3090280c.png)

Kubespray
https://github.com/kubernetes-sigs/kubespray
Переходим, смотрим как нужно все это установить.
git clone https://github.com/kubernetes-sigs/kubespray
Переходим в kubespray/inventory
И скопируем дефолтный конфиг
cp -R sample/ mycluster
cd mycluster
Конфигурируем инвентори файл
В секцию all, объявляем наши сервера
![image](https://user-images.githubusercontent.com/96112649/232605837-f81bc354-fac7-49bb-b700-752d1942089a.png)
 
В kube_control_plane размещаем наши control ноды и в etcd тоже. По хорошему, их можно разделять на отдельные машины, etcd ставить на хорошие nvme ssd диски, но ввиду моих ограниченных ресурсах сделаем на 3х виртуалках
 ![image](https://user-images.githubusercontent.com/96112649/232605851-b405c250-969d-4ed1-b6ec-499833f98e75.png)

В kube_node размещаем наши воркер ноды
 ![image](https://user-images.githubusercontent.com/96112649/232605869-148e9aab-c115-4507-9473-07371b51a680.png)

Сохраняем, теперь нужно пойти в директорию group_vars и поправить конфиг кластера

cd group_vars/
vim k8s_cluster/k8s-cluster.yml
Поставлю немного старую версию кубера, чтобы ничего не ломалось, проще потом почитать, что изменилось.
Так что меняем в конфиге только версию и раскоментируем вот эти параметры:
 ![image](https://user-images.githubusercontent.com/96112649/232605899-93b9a584-219a-4b44-afc2-73cd4d9028f7.png)

Настроим калико
vim k8s_cluster/k8s-net-calico.yml
 ![image](https://user-images.githubusercontent.com/96112649/232605912-5002a425-f4fc-47a4-b121-3687d0fc038e.png)

Другие настройки
vim all/all.yml
 ![image](https://user-images.githubusercontent.com/96112649/232605925-78b1ad0a-58c3-4631-8abd-7e50875b67d4.png)

На этом дела с конфигами кончились, установим pip
sudo apt-get install pip
Выходим в корень
cd ../../../
И наконец-то разворачиваем кластер
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml
Пошло что-то не так, удалил ансибл из системы и переустановил из пипа
Ещё была странная ошибка на одной ноде с калико, какого-то непонятного он жаловался на то что на ней включен vlanx
Помогло добавление в конфиг калико строчки
calico_vxlan_mode: 'Never'

Нашел это в официальной документации
Готово
 ![image](https://user-images.githubusercontent.com/96112649/232605940-391d292e-eb32-46e5-b88e-fece2c725f68.png)

Lens
Установим Lens, скачали установщик с официального сайта:
https://k8slens.dev/
Получаем activation code при регистрации и добавляем куб-конфиг.
Кубконфиг получаем так, заходим в домашнюю директорию root’а, и прописываем:
cat .kube/config
Копируем и добавляем новый кластер.
 ![image](https://user-images.githubusercontent.com/96112649/232605972-89c350ef-032f-4126-ae5a-254175099906.png)

Единственное, нужно поменять ip с 127.0.0.1, на адрес любой control node
 ![image](https://user-images.githubusercontent.com/96112649/232605990-7f15c351-e3d1-4887-8e06-57b0d28a621a.png)
![image](https://user-images.githubusercontent.com/96112649/232606012-5dbf09d1-e461-487d-bbbc-779b5065100e.png)

 
Vscode
Подключимся по ssh к какой либо ноде, для удобной работы с yaml.
Нужно установить расширение remote - ssh
![image](https://user-images.githubusercontent.com/96112649/232606029-1847a6ee-0f50-48a5-b1a3-1d2cc8c186dc.png)

Добавляем наш сервер.
 ![image](https://user-images.githubusercontent.com/96112649/232606044-ba6f731c-ee16-4a9b-8515-f5b25ec9be72.png)
![image](https://user-images.githubusercontent.com/96112649/232606059-b53bf7f1-1c1b-4297-8322-33de562814be.png)

 
Первоначальная настройка кластера.
Создадим 2 неймспейса.
В kubernetes namespaces –  это способ разделения ресурсов кластера между несколькими пользователями (с помощью квоты ресурсов).
Получить список namespaces можно с помощью команды
kubectl get namespace
 ![image](https://user-images.githubusercontent.com/96112649/232606079-16384448-93b6-43c4-921d-57690a5be955.png)

Namespaces
Создадим 2 namespace.
Открываем vscode и создаем yaml file.
``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name:  monitoring


apiVersion: v1
kind: Namespace
metadata:
  name: logging
```
apiVersion – версия Kubeapi.
Kind – сущность
Metadata – данные, позволяющие идентифицировать объект
Применяем манифест
kubectl apply -f  00-namespace.yml
 ![image](https://user-images.githubusercontent.com/96112649/232606148-022fcc1e-9ad7-4a53-9fce-c7232dbda29e.png)
![image](https://user-images.githubusercontent.com/96112649/232606163-73cb0af8-8475-4267-9b9d-8a9249f01ab0.png)
![image](https://user-images.githubusercontent.com/96112649/232606168-3bd70129-5bb4-4cc9-b147-1230c73a86b0.png)

 
 
PriorityClass
Создадим Priority classes.
PriorityClass — это объект кластера, не привязываемый к namespace. Он присваивает имя целочисленному значению приоритета.
Подробнее https://ealebed.github.io/posts/2019/%D0%BF%D1%80%D0%B8%D0%BE%D1%80%D0%B8%D1%82%D0%B5%D1%82%D0%BD%D0%BE%D1%81%D1%82%D1%8C-%D0%BF%D0%BE%D0%B4%D0%BE%D0%B2-%D0%B2-kubernetes/
![image](https://user-images.githubusercontent.com/96112649/232606200-0b543a68-f246-4881-8dcf-e4dc83618eda.png)

``` yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 2000000
globalDefault: false
description: "System components. Like a pv, controllers, docker registry etc"

apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 1005000
globalDefault: false
description: "Application productive contour"

apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000000
globalDefault: false
description: "For any case."
 ```
Reloader
Сначала нужно рассказать что такое config map – config map это сущность, которая позволяет отделить конфигурационные файлы от содержимого образа, чтобы обеспечить отказоустойчивость контейнерных приложений
``` yaml
kind: ConfigMap
apiVersion: v1
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: example-config
  namespace: default
data:
  example.property.1: hello
  example.property.2: world
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
binaryData:
  bar: L3Jvb3QvMTAw
  ```
Где:
data — содержит данные конфигурациии.
bar — указывает на файл, содержащий данные, отличные от UTF8, например двоичный файл хранилища ключей Java. Данные файла добавляются в формате Base 64.
Kubernetes не будет перезапускать/перечитывать под после изменений в config map, поэтому установим средство, которое поможет с этим
https://github.com/stakater/Reloader
Идем по пути https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml
И копируем содержимое в файл yaml.
Идем по порядку.
``` yaml
# Source: reloader/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    meta.helm.sh/release-namespace: "default"
    meta.helm.sh/release-name: "reloader"
  labels:
    app: reloader-reloader
    chart: "reloader-v1.0.22"
    release: "reloader"
    heritage: "Helm"
    app.kubernetes.io/managed-by: "Helm"
  name: reloader-reloader
  namespace: monitoring
  ```
Добавляем сущность ServiceAccount.
Service Account (учетная запись службы) — это учетная запись, которая позволяет компоненту приложения напрямую обращаться к API без предоставления учетных данных пользователя.
Тут нужно обратить внимание на namespace, не будем создавать отдельный namespace под reloader, закинем его в мониторинг.
``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    meta.helm.sh/release-namespace: "default"
    meta.helm.sh/release-name: "reloader"
  labels:
    app: reloader-reloader
    chart: "reloader-v1.0.22"
    release: "reloader"
    heritage: "Helm"
    app.kubernetes.io/managed-by: "Helm"
  name: reloader-reloader-role
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
      - configmaps
    verbs:
      - list
      - get
      - watch
  - apiGroups:
      - "apps"
    resources:
      - deployments
      - daemonsets
      - statefulsets
    verbs:
      - list
      - get
      - update
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - deployments
      - daemonsets
    verbs:
      - list
      - get
      - update
      - patch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
```
ClusterRole – это тот же объект что и Role только применяется ко всему кластеру.
А Role – это некий набор прав на объекты кластера Kubernetes. Role ничего и никому не разрешает. Это просто список.
То есть мы указываем,что есть такая роль и что она может.
``` yaml
# Source: reloader/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1

kind: ClusterRoleBinding
metadata:
  annotations:
    meta.helm.sh/release-namespace: "default"
    meta.helm.sh/release-name: "reloader"
  labels:
    app: reloader-reloader
    chart: "reloader-v1.0.22"
    release: "reloader"
    heritage: "Helm"
    app.kubernetes.io/managed-by: "Helm"
  name: reloader-reloader-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: reloader-reloader-role
subjects:
  - kind: ServiceAccount
    name: reloader-reloader
    namespace: monitoring
```
По аналогии с ClusterRole есть ClusterRoleBinding. Он как раз и занимается назначенем роли
В roleRef указываем права из какой роли будут разрешены.
subjects — кому будут разрешены эти права (или назначена эта роль).
И добавим само приложение.
Deployment -  это абстракция Kubernetes, которая позволяют нам управлять тем, что всегда присутствует в жизненном цикле приложения. Речь идёт об управлении изменениями приложений. Приложения, которые не изменяются, это, так сказать, «мёртвые» приложения. Если же приложение «живёт», то можно столкнуться с тем, что периодически изменяются требования к нему, расширяется его код, этот код упаковывается и разворачивается. При этом на каждом шаге данного процесса могут совершаться ошибки.

Ресурс вида Deployment позволяет автоматизировать процесс перехода от одной версии приложения к другой. Это делается без прерывания работы системы, а если в ходе этого процесса произойдёт ошибка, у нас будет возможность быстро вернуться к предыдущей, рабочей версии приложения.

``` yaml
# Source: reloader/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    meta.helm.sh/release-namespace: "default"
    meta.helm.sh/release-name: "reloader"
  labels:
    app: reloader-reloader
    chart: "reloader-v1.0.22"
    release: "reloader"
    heritage: "Helm"
    app.kubernetes.io/managed-by: "Helm"
    group: com.stakater.platform
    provider: stakater
    version: v1.0.22
  name: reloader-reloader
  namespace: monitoring
spec:
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: reloader-reloader
      release: "reloader"
  template:
    metadata:
      labels:
        app: reloader-reloader
        chart: "reloader-v1.0.22"
        release: "reloader"
        heritage: "Helm"
        app.kubernetes.io/managed-by: "Helm"
        group: com.stakater.platform
        provider: stakater
        version: v1.0.22
    spec:
      containers:
      - image: "ghcr.io/stakater/reloader:v1.0.22"
        imagePullPolicy: IfNotPresent
        name: reloader-reloader

        ports:
        - name: http
          containerPort: 9090
        livenessProbe:
          httpGet:
            path: /live
            port: http
          timeoutSeconds: 5
          failureThreshold: 5
          periodSeconds: 10
          successThreshold: 1
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /metrics
            port: http
          timeoutSeconds: 5
          failureThreshold: 5
          periodSeconds: 10
          successThreshold: 1
          initialDelaySeconds: 10
        resources:
          limits:
            cpu: "100m"
            memory: "512Mi"
          requests:
            cpu: "10m"
            memory: "128Mi"

        securityContext:
          {}
      securityContext: 
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: reloader-reloader
 ```
 Что пока что для меня важно это поле spec
Spec – это требуемое состояние объекта.
replicas: 1 – значит, что будет 1 под
Lables; Метки — это пары ключ-значение, которые добавляются к объектам, как поды. Метки предназначены для идентификации атрибутов объектов, которые имеют значимость и важны для пользователей, но при этом не относятся напрямую к основной системе. Метки можно использовать для группировки и выбора подмножеств объектов. Метки могут быть добавлены к объектам во время создания и изменены в любое время после этого. Каждый объект может иметь набор меток в виде пары ключ-значение. Каждый ключ должен быть уникальным в рамках одного и того же объекта.
matchLabels  - это словарь пары ключ-значение
Так же есть поле template – храним состояние подов по шаблону.
В нем есть же поле spec где мы храним, собственно состояние.
Наш контейнер, pull policy и т.д и т.п.
imagePullPolicy: IfNotPresent
Говорит о том, что если образ будет пулиться с удаленного ресурса, только в том случае, если он не будет обнаружен локально.
Так же я добавил ограничение использования ресурсов, в поле spec->resources->limits
        ``` yaml
        resources:
          limits:
            cpu: "100m"
            memory: "512Mi"
          requests:
            cpu: "10m"
            memory: "128Mi"
      ```
 ![image](https://user-images.githubusercontent.com/96112649/232606329-ea690339-9110-4ac0-9b3f-f0e972d34764.png)
![image](https://user-images.githubusercontent.com/96112649/232606343-f16c7896-33a8-4180-b22a-cb487ef6121e.png)

Под запускается
 ![image](https://user-images.githubusercontent.com/96112649/232606355-e5d7d99f-a4ea-4685-b72d-8a68e747771b.png)
![image](https://user-images.githubusercontent.com/96112649/232606370-c4317119-87bf-4320-9b4e-e27c4805a8c0.png)

 
Забыл указать приорити класс-нейм.
 
Снова сделал kubectl apply
Посмотрим yaml pod’а
kubectl get po -n monitoring
kubectl get pod reloader-reloader-6b5bcd8ddb-7t6h7 -n monitoring -o yaml
 
