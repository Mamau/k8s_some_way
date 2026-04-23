## DaemonSet
***
Какие значения можно использовать в манифесте, описано в документации по [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/)  
  
В отличии от Deployment разворачивается на каждой ноде по одному поду при условии что этот под можно запустить (taints).
### Пример конфигурации DaemonSet
***
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: test-ds
  labels:
    app.kubernetes.io/name: &name test-ds
    app.kubernetes.io/instance: &instance test-ds
    app.kubernetes.io/version: &version v0.0.1
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: *name
      app.kubernetes.io/instance: *instance
      app.kubernetes.io/version: *version
  template:
    metadata:
      labels:
        app.kubernetes.io/name: *name
        app.kubernetes.io/instance: *instance
        app.kubernetes.io/version: *version
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
              name: http
              protocol: TCP
          resources:
            requests:
              memory: 100Mi
              cpu: 100m
            limits:
              memory: 100Mi
              cpu: 100m
```
Манифест `DaemonSet` полностью копирует структуру манифеста `Deployment`. Единственное отличие - в отсутствии поля .spec.replicas,
так как это не применимо.  
  
Обратите внимание, что в манифесте не определяется `Service`. Это сделано намерено. Предполагается, что приложение, которое
запускается при помощи `DaemonSet`, предназначено только для сбора и отсылки некой информации. В качестве примера можно привести
системы сбора логов k8s. Там на каждой ноде запускается агент, который читает логи, находящиеся в директории `/var/log`. 
И отправляет их куда то.  
  
Или сбор метрик при помощи node-exporter. Точно также, на каждой ноде запускается агент, который собирает данные о машине и 
отдает их по запросу агентов сбора метрик. Агенты обращаются непосредственно к конкретному экземпляру node-exporter, 
не использую `Service`. Как он узнает ip - узнаем позже.  
  
Применим манифест:
```shell
kubectl -n work apply -f 5_daemonset_taints_nodeselector/06-daemonset.yaml 
```
Посмотрим
```shell
kubectl -n work get all
```
### Зараженные ноды (Taints и Toleration's)
***
Посмотри внимательно на ноды
```shell
kubectl -n work get pods -o wide
```
```text
NAME            READY   STATUS    RESTARTS   AGE     IP                NODE                        NOMINATED NODE   READINESS GATES
test-ds-6f9jr   1/1     Running   0          2m28s   192.168.129.85    ubuntu-4gb-hel1-29-node-2   <none>           <none>
test-ds-sq592   1/1     Running   0          2m28s   192.168.125.148   ubuntu-4gb-hel1-30-node-3   <none>           <none>
```
Вспомни какие есть ноды в нашем кластере
```shell
kubectl get nodes
```
```text
NAME                        STATUS   ROLES           AGE   VERSION
ubuntu-4gb-hel1-28-node-1   Ready    control-plane   13d   v1.35.3
ubuntu-4gb-hel1-29-node-2   Ready    <none>          12d   v1.35.3
ubuntu-4gb-hel1-30-node-3   Ready    <none>          12d   v1.35.3
```
Что не сходится. По идее поды DaemonSet должны быть запущены на всех нодах, а здесь видно что они запущены только на двух.  
  
При назначении пода на ноду, на работу планировщика влияют разные факторы. В нашем случае на control ноды кластера были установлены 
Taints (зараза). Посмотреть какие taints установлены можно командой:
```shell
kubectl get nodes -o custom-columns=Name:.metadata.name,TAINTS:.spec.taints
```
```text
Name                        TAINTS
ubuntu-4gb-hel1-28-node-1   [map[effect:NoSchedule key:node-role.kubernetes.io/control-plane]]
ubuntu-4gb-hel1-29-node-2   <none>
ubuntu-4gb-hel1-30-node-3   <none>
```
Как видно из вывода команды, на ноде `ubuntu-4gb-hel1-28-node-1` установлен taints: `node-role.kubernetes.io/control-plane`. 
  
По умолчанию планировщик не распределяет пода ны "зараженные" ноды. Поэтому поды нашего DaemonSet были созданы только на worker нодах.
  
Для того чтобы поды могли запускаться на "зараженной" ноде, необходимо добавить толерантность к конкретному типу заразы.
  
> зараза состоит из трех частей: key=[value]:Effect. В нашем случае это:
- `key` - ключ taint (например, node-role.kubernetes.io/control-plane)
- `value` - значение taint. Не обязателен к определению, Если не указан, то любое значение будет считаться совпадением.
- `Effect` - действие.
  - `NoSchedule` - запрещает планирование под на ноде. Поды запущенные до применения taint не удаляются.
  - `NoExecute` - запрещает планирование под на ноде. Поды запущенные до применения taint будут удалены с ноды.
  - `PreferNoSchedule` - это "предпочтительная" или "мягкая" версия NoSchedule. Планировщик будет пытаться не размешать на узле поды, но это не гарантированно.
  
Чтобы игнорировать taint `node-role.kubernetes.io/control-plane:NoSchedule` для подов DaemonSet необходимо добавить в манифест толерантность
к конкретному типу заразы в спецификации пода. В нашем случае это будет выглядеть так:
```yaml
spec:
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
```
Применить изменения без удаления не получится
```shell
kubectl -n work delete -f 5_daemonset_taints_nodeselector/06-daemonset.yaml
```
Применим новый
```shell
kubectl -n work apply -f 5_daemonset_taints_nodeselector/07-daemonset.yaml
```
Посмотрим
```shell
kubectl -n work get pods -o wide
```
Теперь на всех трех нодах
```text
NAME            READY   STATUS    RESTARTS   AGE   IP                NODE                        NOMINATED NODE   READINESS GATES
test-ds-bq45d   1/1     Running   0          31s   192.168.129.86    ubuntu-4gb-hel1-29-node-2   <none>           <none>
test-ds-mrps8   1/1     Running   0          31s   192.168.125.149   ubuntu-4gb-hel1-30-node-3   <none>           <none>
test-ds-nbztk   1/1     Running   0          31s   192.168.100.196   ubuntu-4gb-hel1-28-node-1   <none>           <none>
```
Добавить свою заразу на ноду можно так:
```shell
kubectl taint nodes ubuntu-4gb-hel1-29-node-2 test-taint=:NoExecute
```
Посмотрим
```shell
kmy -n work get pods -o wide
```
```text
NAME            READY   STATUS    RESTARTS   AGE    IP                NODE                        NOMINATED NODE   READINESS GATES
test-ds-mrps8   1/1     Running   0          3m1s   192.168.125.149   ubuntu-4gb-hel1-30-node-3   <none>           <none>
test-ds-nbztk   1/1     Running   0          3m1s   192.168.100.196   ubuntu-4gb-hel1-28-node-1   <none>           <none>
```
Видим что пропал второй под - эффект NoExecute отработал - под удален.  
  
Уберем taint с ноды
```shell
kubectl taint nodes ubuntu-4gb-hel1-29-node-2 test-taint=:NoExecute-
```
Посмотрим
```shell
kmy -n work get pods -o wide
```
Под вернулся
```text
NAME            READY   STATUS    RESTARTS   AGE     IP                NODE                        NOMINATED NODE   READINESS GATES
test-ds-mrps8   1/1     Running   0          5m16s   192.168.125.149   ubuntu-4gb-hel1-30-node-3   <none>           <none>
test-ds-nbztk   1/1     Running   0          5m16s   192.168.100.196   ubuntu-4gb-hel1-28-node-1   <none>           <none>
test-ds-xrvq4   1/1     Running   0          8s      192.168.129.87    ubuntu-4gb-hel1-29-node-2   <none>           <none>
```
Effect NoExecute опасный. Он удалит с ноды все поды, не имеющие толерантности к taint. На самом деле на этой ноде были некоторые 
служебные поды, которые не желательно удалять. Поэтому рекомендуется использовать `NoSchedule`.  
  
Удалим DaemonSet
```shell
kubectl -n work delete -f 5_daemonset_taints_nodeselector/07-daemonset.yaml
```
  
### Node Selector
***
Использовать taints для распределения подов DaemonSet не лучший способ управления. Например нам необходимо разместить поды 
на строго определенные ноды кластера. В этом случае можно использовать `NodeSelector`. В качестве параметра используемого 
для отбора нод, можно указать (labels), установленные на нодах.  
  
Пометим ноды node-1 и node-2 меткой `special: ds-only`:
```shell
kubectl label nodes ubuntu-4gb-hel1-28-node-1 special=ds-only && \
kubectl label nodes ubuntu-4gb-hel1-29-node-2 special=ds-only
```
Посмотрим на каких нодах установлены метки `special: ds-only`:
```shell
kubectl get nodes --show-labels | grep 'special=ds-only'
```
```text
buntu-4gb-hel1-28-node-1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=,special=ds-only
ubuntu-4gb-hel1-29-node-2   Ready    <none>          13d   v1.35.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=ubuntu-4gb-hel1-29-node-2,kubernetes.io/os=linux,special=ds-only
```
Добавим `nodeSelector` в определение пода:
```yaml
spec:
  nodeSelector:
    special: ds-only
```
Применяем манифест
```shell
kubectl -n work apply -f 5_daemonset_taints_nodeselector/08-daemonset.yaml
```
Смотрим
```shell
kubectl -n work get pods -o wide
```
```text
NAME            READY   STATUS    RESTARTS   AGE   IP                NODE                        NOMINATED NODE   READINESS GATES
test-ds-54k6z   1/1     Running   0          13s   192.168.100.197   ubuntu-4gb-hel1-28-node-1   <none>           <none>
test-ds-fm6ks   1/1     Running   0          13s   192.168.129.88    ubuntu-4gb-hel1-29-node-2   <none>           <none>
```
Удалим метки с ноды
```shell
kubectl label nodes ubuntu-4gb-hel1-28-node-1 special- && \
kubectl label nodes ubuntu-4gb-hel1-29-node-2 special-
```
Посмотрим на ноды DaemonSet после удаления меток
```shell
kubectl -n work get ds
```
Видим что ниодного пода не запущено
```text
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR     AGE
test-ds   0         0         0       0            0           special=ds-only   3m4s
```
Демон сет следит за ситуацией и если вы удаляете какието метки то он также удалит поды.
  
Удалим манифест
```yaml
kubectl -n work delete -f 5_daemonset_taints_nodeselector/08-daemonset.yaml
```
```text
daemonset.apps "test-ds" deleted
```