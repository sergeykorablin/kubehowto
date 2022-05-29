# Настройка кластера Kubernetes

## Предисловие с предупреждением
 
Перед началом погружения в Kubernetes стоит посмотреть на возможности [Docker Swarm](https://docs.docker.com/engine/swarm/). Возможно он покроет ваши потребности по управлению контейнерами и вы сэкономите много времени, сил и нервов на изучение kubernetes, на его установку и поддержку.

Если вы хотите быстро познакомиться с основными возможностями и концепциями kubernetes - обратите внимание на [minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/) и не читайте дальше эту статью! :laughing:

Если вы хотите погрузиться в kubernetes немного глубже - добро пожаловать, надеюсь информация изложенная ниже будет вам полезна.

В этой статье постараюсь описать установку собственного кластера kunernetes. Статья в первую очередь написана для себя, чтобы суммировать знания которые я получил за время изучения этой темы из разных статей, how-to и обучающих видео. Так же опишу проблемы с которыми пришлось столкнуться, некоторые из них на данный момент мне решить не удалось.

Скажу честно, временами при изучении kubernetes мой мозг просто закипал от обилия информации, концепций, объектов и связей между ними. Состояние менялись от ярости до бессилия и обратно) В такие моменты важно отвлекаться на что-то другое, отдыхать и давать своему мозгу время на создание новые нейронных связей.

Какиvb знанияvb неорбходимо обладать чтобы не сильно страдать изучая Kubernetes: базовое понимание работы с Linux, системами виртуализации, представление о контейнерах (например Docker), понимание принципов работы сетей и сетевых протоколов.

## Что за кубер и зачем это всё

Kubernetes - это оркестратор контейнеров разработанный компанией Google для создания  распределенных приложений. На самом деле kubernetes выполняет намного больше функций чем просто оркестровка, например: автообнаружение, балансировка нагрузки между узлами кластера и копиями приложений, мониторинг и поддержание состояния приложенией, хранение и распространение секретных данных, организация доступа к хранилищам и т.д.

Kebernetes довольно сложная система т.к. предназначена для решения сложных задач. Как многие  open source системы эта система модульная, в неё могут входить разное количество компонентов. Часто вы сами будете выбирать, какие из нескольких видов компонентов вы будете использовать для построения своего кластера и будете ли использовать вообще.

## Базовые понятия

* pod
* service
* deployment
* replica set
* stateful set 
* namespace
* config map
* secret

## Подготовка узлов

Кластер будет состоять из 3-х узлов: одного мастера и двух воркеров.
В моем случае узлами будут виртуальными машинами под управлением Hyper-V, вы можете воспользоваться любой другой системой виртуализации (KVM, VirtuakBox, VMWare).

> Для мастер узла необходимо минимум 2 ядра и 2GB ОЗУ иначе вы не сможете инициализировать кластер 

Кроме этого Kubernetes требует чтобы на всех узлах кластера был отключен swap. 
Для отключения выполняем команду
```console
$ sudo swapoff -a
```
и комментируем соответствующую запись в /etc/fstab

В качестве гостевой операционной системы буду использовать Ubuntu 20.04. Вы можете использовать другие ОС, поддерживаются rhel-based и debian-based. 

> **Note**
> Изначально я пытался собрать кластер на базе Ubuntu 22.04, но служебные сервисы (поды) рандомно падали и перезапускались, почему это происходит выяснить не получилось.

Сбросим настройки containerd
```console
$ containerd config default | sudo tee /etc/containerd/config.toml
$ sudo systemctl restart containerd
```
...

## Установка kubelet, kubeadm, kubectl

Для установки необходимых компонентов добавим apt репозиторий и gpg ключ на всех узлах

```console
$ sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

$ echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

На всех узлах необходимо установить kubelet, kubeadm. На мастер узле дополнительно kubectl
```console
$ sudo apt update
$ sudo apt install -y kubelet kubeadm kubectl
```


## Инициализация кластера и сети
Перед началом инициализации кластера необходимо сделать выбор CNI плагина который будет отвечать за работу внутренней сети кластера. 
> CNI - Container Network Interface

Плагинов существует большое количество от разных вендоров с разным функционалом. По [ссылке](https://kubernetes.io/docs/concepts/cluster-administration/networking/) можете познакомиться со списком и кратким описанием.

В этой статье будет использовать простой Flannel для организации сети в кластере. Flanell может использовать несколько [бэкендов](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md) для пересылки пакетов включая VXLAN. 

Обратите внимание, Flanel не поддерживает сетевые политики. Выбирайте CNI плагин в соответствии с вашими потребностями.

Для инициализации кластреа выполняем на мастер узле:
```console
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Копируем конфиг в вашу домашнюю директорию:
```console
$ mkdir ~/.kube
$ cp /etc/kubernetes/admin.conf ~/.kube/config
```

Аргумент --pod-network-cidr=10.244.0.0/16 задает частную подсеть, из которой будут назначаться IP-адреса подов. Flannel использует указанную подсеть по умолчанию. 

Устанавливаем Flannel:
```console
$ kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
>Если вы хотите использовать для подов подсеть отлитчную от 10.244.0.0/16 - вам неоходимо указать указать её в параметре --pod-network-cidr при инициализации кластера, скачать и изменить подсеть в kube-flannel.yml, затем применить измененный yaml-файл: 
>
>$ kubectl apply -f kube-flannel.yml



## Подключение воркеров
После выполнения ```kubeadm init``` мы должны увидеть в выводе команду, которую необходимо использовать для присоединения узлов к кластеру. 

На всех воркерах выполняем команду:
```console
$ kubeadm join ...
```

Выведем список всех узлов кластера
```console
$ kubеctl get nodes
NAME     STATUS     ROLES           AGE     VERSION
kube-1   NotReady   control-plane   3d22h   v1.24.0
kube-2   Ready      <none>          3d22h   v1.24.0
kube-3   Ready      <none>          3d11h   v1.24.0
```

## Запуск первого приложения

Давайте создадим развёртывание (Deployment), используя готовый образ echoserver, представляющий простой HTTP-сервер, и сделаем его доступным на порту 8080:
```console
$ kubectl create deployment hello --image=k8s.gcr.io/echoserver:1.10
deployment.apps/hello created
```
Должно появиться два объекта Deployment и Pod:
```console
$ kubectl get deployment
deployment
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
hello             1/1     1            1           4s

$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
hello2-584c7785f7-f42hn             1/1     Running   0          10s
```

Чтобы получить доступ к объекту Deployment hello извне, создаем объект сервиса (Service):
```console
$ kubectl expose deployment hello --type=NodePort --port=8080
```
Тип сервиса ```NodePort``` указывает, что надо открыть порт на всех воркерах и связать его с Deployment-ом hello.
```console
$ kubectl get svc
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
hello           NodePort    10.100.60.224    <none>        8080:30059/TCP   25s
```
Откройте в браузере ```http://<ip любого воркера>:30059/``` Если всё настроено правильно - должны увидеть вывод echoserver с некой технической информацией.

Удалим все созданные объекты
```console
$ kubectl delete service hello

$ kubectl delete deployment hello
```

Создадим файл hello-deployment.yaml с таким содержанием:
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: hello
  labels:
    app: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: echoserver
          image: k8s.gcr.io/echoserver:1.10
```

Применим деплоймент:
```console
$ kubеctl apply -f hello-deployment.yaml
```

## Сервисы и доступ к приложению
ClusterIP

NodePort

LoadBalancer

...

## Балансировщик для входящих соединений

### MetalLB
...

https://metallb.universe.tf/installation/
```console
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
```

### ingress-nginx
...

## Ручная настройка SSL 
Если у вас есть готовый сертификат - вы можете использовать его для терминации ssl.
Создаем каталог ```certs```, копируем в него сертификат tls.crt и ключ tls.key и создаем tls секрет:
```console
kubectl create secret tls dev-example-com-tls -n dev --key=certs/tls.key" --cert=certs/tls.crt"
```

ВЫведем список всех секретов в namespace dev
```console
kubectl get secret -n dev
NAME              TYPE                DATA   AGE
dev-example-com-tls   kubernetes.io/tls   2      5m
```

Можно посмотреть что содержит секрет выполниво команду:
```console
$ kubectl describe secret dev-example-com-tls -n dev
Name:         dev-example-com-tls
Namespace:    dev
Labels:       <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  5591 bytes
tls.key:  1679 bytes
```

Далее добавляем секцию tls в настройки Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hello-svc
  namespace: dev
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  tls:
  # указываем секрет с парой ключ/сертификат из предыдуго шага
  - secretName: dev-example-com-tls
    hosts:
    - dev.example.com
  rules:
  - host: dev.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 80
```

> **Warning**
> Тут я столкнулся с одной непонятной проблемой. При добавлении wildcard сертификата Ingress контролер писал, что не может найти секрет с указаным именем и использовал дефолтный самоподписанный сертификат. Как только я менял сертификат на выданный для конкретного fqdn - всё начинало работать. Не разобрался в чем причина такого поведения. 
>
>На github есть [тикет](https://github.com/kubernetes/ingress-nginx/issues/7349) на эту тему, разбор ни чем не закончился, топикстартер пропал.

## Настройка cert-manager для автоматического получение сертификатов

Теперь настроим автоматический выпуск сертификата.
Для выпуска сертификатов будем использовать cert-manager, по ссылкам можете найти [официальный сайт](https://cert-manager.io/) проекта и [документацию](https://cert-manager.io/docs/).

Устанавливаем cert-manager:
```console
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml

namespace/cert-manager created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
...
validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
```
### Issuer и ClusterIssuer

Далее нам необходимо настроить issuer (Издателя), он бывает двух типов: 
* ClusterIssuer - глобальный издатель, может создавать tls секреты в любом namespace-е.
* Issuer - namespace-зависим, настраивается для каждого namespase-а в котором будет создавать и обновлять tls секреты.

Методы проверки домена могут быть DNS-01 и HTTP-01. Чтобы использовать проверу dns-01 ваш dns-провайдер должен быть в списке поддерживаемых, см. список в официальной документации.
В примере будем использовать HTTP-01, для этого 80-й порт ingress контроллера должен быть доступен из интернет по доменному имени dev.example.com
В моем случае кластер находится в локальной сети и не имеет публичного ip-адреса, поэтому на отдельном сервере который торчит в интернет настраиваю nginx в качестве реверс-прокси для http://dev.example.com.

Настройки nginx ```/etc/nginx/sites-enabled/kuber-acme```:
```nginx
server {
  listen 80;
  listen [::]:80;
  # Домен для которого будем выпускать сертификат. Можно указать несколько
  server_name dev.example.com;

  # Переадресуем запросы только на http://dev.example.com/.well-known/acme-challenge/...
  location /.well-known/acme-challenge/ {
        # ip 192.168.1.110 - адрес Ingress контроллера
        proxy_pass http://192.168.1.110/.well-known/acme-challenge/;
        # Важно передать Host в заголовке чтобы Ingress контроллер правильно маршрутизировал запрос
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
  }
}
```
Перезапускаем или делаем reload nginx:
```console
$ sudo systemctl reload nginx
```

Описываем Issuer (Издателя):
```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  # Имя любое, используется дальше в других ресурсах
  name: letsencrypt-issuer
  namespace: dev
spec:
  acme:
    # укажите ваш почтовый адрес
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret который будет использован для хранения приватного ключа.
      name: letsencrypt-issuer-tls
    solvers:
    - http01:
        ingress:
          class: nginx
```
>В данном примере мы используем https://acme-**staging**-v02.api.letsencrypt.org/directory для выпуска сертификата. Сертификт не будт валидным, но мы можем спокойно менять настройки не боясь, что letsencrypt нас забанит если в конфиге будут ошибки и он несколько раз не сможет проверить домен. 
>
>Если со staging всё получилось - поменяйте server на https://acme-v02.api.letsencrypt.org/directory

Есть два варианта выпуска серфтификата:
1. Создать ресурс Certificate с описанием нужного сертификата
2. Добавить аннотацию в Ingress и позволить cert-manager самому разобраться какой сертификат нужен

### Вариант 1
Создаем описание сертификата:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
 name: dev-example-ca-tls
 namespace: dev
spec:
 # имя tls секрета для хранения пары сертификат/ключ
 secretName: dev-example-com-tls
 issuerRef:
   kind: Issuer
   # имя issuer 
   name: letsencrypt-issuer
 commonName: "dev.example.com"
 dnsNames:
 # один или несколько fqdn
 - dev.example.com
 ```
При создании Certificate диспетчер сертификатов создает ресурс CertificateRequest, содержащий запрос на сертификат, ссылку на Issuer и другие параметры на основе спецификации ресурса Certificate. 

Этот сертификат укажет cert-manager попытаться использовать Issuer с именем letsencrypt-issuer для получения пары ключ/сертификат для домена dev.example.com. В случае успеха полученный ключ и сертификат будут сохранены в секрете с именем ```dev-example-com-tls``` с ключами ```tls.key``` и ```tls.crt``` соответственно.

Подробней про жизненный цикл сертификата можете почитать по [ссылке](https://cert-manager.io/docs/concepts/certificate/#certificate-lifecycle)

Добавляем в описание Ingress tls серкцию с названием secreta в котором должен храниться сертификат
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hello-svc
  namespace: dev
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  tls:
    # название секрета в котором должен храниться сертификат и ключ
    - secretName: dev-example-com-tls
      hosts:
        - dev.example.com
  rules:
  - host: dev.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 80
```

### Вариант 2
Можно не создавать ресурс Certificate, а только добавить аннотацию в Ingress с названием Issuer - тогда cert-manager автоматически создаст ресурс Certificate на основе информации Ingress и  выпустит сертификат.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hello-svc
  namespace: dev
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    # Указываем имя Issuer для автоматического получения сертификата 
    cert-manager.io/issuer: "letsencrypt-issuer"
spec:
  tls:
    - secretName: dev-example-com-tls
    ...
```

## Хранилища данных
Persistent Volume

Persistent Volume Claim

Изначально предполагалось, что в контейнерах должны работать stateless приложения, которые можно запускать и останавливать без сохранения данных. Позже оказалось, что в виде контейнеров удобно запускать и другие приложения, например базы данных, которые должны где-то хранить свои данные, поэтому появились тома (volumes).

В этом примере настроим на мастер узле сервер nfs и добавим его в кластер, чтобы запущенные поды могли хранить свои данные в постоянном хранилище

На мастер узле
```console
$ sudo apt install nfs-kernel-server

$ sudo mkdir /var/nfs
```
Добавим в ```/etc/exports``` строку
```conf
/var/nfs    192.168.1.102(rw,sync) 192.168.1.103(rw,sync)
```


На воркер узлах
```console
$ sudo apt install nfs-common
```

...

## Пользователи и авторизация RBAC

...


## Мониторинг с помощью Prometheus и Grafana

[Prometheus](https://prometheus.io/) это open-source фреймворк для мониторинга. It provides out-of-the-box monitoring capabilities for the Kubernetes container orchestration platform. 
...

```console
$ kubectl create namespace monitoring
```
