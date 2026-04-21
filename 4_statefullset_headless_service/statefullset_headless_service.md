## StatefulSet
***
> В отличии от Deployment, StatefulSet гарантирует уникальность имени каждого пода в кластере kubernetes.
> И строго определяет порядок запуска, обновления и удаления подов.
  
### Пример конфигурации StatefulSet
***
Пример манифеста StatefulSet:  
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: test-st
  labels:
    app.kubernetes.io/name: &name test-st
    app.kubernetes.io/instance: &instance test-st
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 3
  serviceName: "test-st-headless"
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
Нужен сервис для обращания к подам по сети, потомучто обычный сервис это сетевое взаимодействие, а headless это днс имена
```yaml
apiVersion: apps/v1
kind: Service
metadata:
  name: test-st
  labels:
    app.kubernetes.io/name: &name test-st
    app.kubernetes.io/instance: &instance test-st
    app.kubernetes.io/version: &version v0.0.1
spec:
  selector:
    app.kubernetes.io/name: *name
    app.kubernetes.io/instance: *instance
    app.kubernetes.io/version: *version
  ports:
    - port: 80
```
Также необходим создать Headless сервис. Он отличается от обычного тем, что у него значение поля clusterIP установлено в None.
```yaml
apiVersion: apps/v1
kind: Service
metadata:
  name: test-st-headless
  labels:
    app.kubernetes.io/name: &name test-st
    app.kubernetes.io/instance: &instance test-st
    app.kubernetes.io/version: &version v0.0.1
spec:
  clusterIP: None
  selector:
    app.kubernetes.io/name: *name
    app.kubernetes.io/instance: *instance
    app.kubernetes.io/version: *version
  ports:
    - port: 80
```
Именно этот сервис указывается в манифесте `StatefulSet`.  
  
Запустим StatefulSet с помощью команды:
```shell
kubectl -n work apply -f 4_statefullset_headless_service/04-statefullset.yaml
```
```text
service/test-st created
service/test-st-headless created
statefulset.apps/test-st created
```
> Посмотрим содержимое namespace work:
```shell
kubectl -n work get all
```
```text
NAME            READY   STATUS    RESTARTS   AGE
pod/test-st-0   1/1     Running   0          9m52s
pod/test-st-1   1/1     Running   0          9m48s
pod/test-st-2   1/1     Running   0          9m43s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/test-st            ClusterIP   10.103.139.41   <none>        80/TCP    6m46s
service/test-st-headless   ClusterIP   None            <none>        80/TCP    6m45s

NAME                       READY   AGE
statefulset.apps/test-st   3/3     9m53s
```
Первое отличие от `Deployment` - отсутствие `ReplicaSet`. `StatefulSet` самостоятельно управляет подами.  
Второе отличие - имена подов. Как видите их именя содержат суффиксы, 0, 1, 2. Т.е мы точно знаем какое имя будет у пода.  
  
Видим, что поды имеют предсказуемые названия которые состоят из названия StatefulSet (test-st) + номер пода, также
этот номер соответствует порядку запуска, и видно даже по времени, что они запускались один за другим.  

service/test-st - наш сервис, видим что он ClusterIP и ip по которому мы можем к нем обратиться  
service/test-st-headless - наш headless тоже ClusterIP, но по ip не можем к нему обратиться т.к нет этого ip  
поэтому надо делать 2 сервис, один который дает ip и другой который дает днс имя.  
***
Посмотрим как работает обычный сервис. Запустим одноразовый под с контейнером, содержащим `curl`.
```shell
kubectl -n work run curl --rm -ti --image=alpine/curl:8.14.1 -- /bin/sh
```
Параметр `--rm` означает, что под будет удален после выполнения команды, а `-ti` позволяет войти в этот под и выполнять команды
в интерактивном режиме. Команда интерактивного режима указана после `--`.
  
У нас появится командная строка, в которой несколько раз выполним команду `curl test-st`. Таким образом
мы обратимся к сервису `test-st` в том же namespace.  
  
Ответ приложения `StatefulSet` к которому мы обращаемся:
```text
Hostname: test-st-1
IP: 127.0.0.1
IP: ::1
IP: 192.168.125.140
IP: fe80::d02d:d6ff:fe4f:95ac
RemoteAddr: 192.168.125.142:46894
GET / HTTP/1.1
Host: test-st
User-Agent: curl/8.14.1
Accept: */*  
  
Hostname: test-st-2
IP: 127.0.0.1
IP: ::1
IP: 192.168.129.77
IP: fe80::a0dd:a9ff:fede:3a7b
RemoteAddr: 192.168.125.142:56834
GET / HTTP/1.1
Host: test-st
User-Agent: curl/8.14.1
Accept: */*
```
Видим что нам отвечает под: `Hostname: test-st-1` или `Hostname: test-st-2`  
  
Теперь попробуем использовать headless сервис через команду `curl test-st-0.test-st-headless`:
```text
# curl test-st-0.test-st-headless
Hostname: test-st-0
IP: 127.0.0.1
IP: ::1
IP: 192.168.129.76
IP: fe80::9839:baff:fed4:e8af
RemoteAddr: 192.168.125.142:58598
GET / HTTP/1.1
Host: test-st-0.test-st-headless
User-Agent: curl/8.14.1
Accept: */*

# curl test-st-1.test-st-headless
Hostname: test-st-1
IP: 127.0.0.1
IP: ::1
IP: 192.168.125.140
IP: fe80::d02d:d6ff:fe4f:95ac
RemoteAddr: 192.168.125.142:38902
GET / HTTP/1.1
Host: test-st-1.test-st-headless
User-Agent: curl/8.14.1
Accept: */*
```
Наш .test-st-headless отображается как поддомен в системе днс внутри которого есть машина test-st-1 (имя пода).
Это работает только внутри кубера, за это отвечает днс кубера.  
  
### Порядок запуска, обновления и удаления подов
***
Контроллер `StatefulSet` гарантирует определенный порядок запуска подов. Они запускаются в порядке возрастания индекса, начиная с 0.
После того как запущен под с индексом 0, начинается запуска следующего пода. И так по порядку до последнего пода.  
  
Удаление подов происходит в обратном порядке. Сначала удаляется последний под `test-st-2` а затем `test-st-1` и наконец `test-st-0`.  
  
Если в процессе запуска, например перед стартом `test-st-2` по каким то причинам упадет `test-st-0`, то `test-st-2` не будет запущен
до тех пор пока не будет успешно запущен `test-st-0`.  
  
При выполнении `rollout restart` - перезапуска подов просиходнит в обратном порядке. Сначала перезапускается последний под, а затем предыдущий.
  
Посмотрим как реагирует StatefulSet на команду `rollout restart`.
```shell
kubectl -n work rollout restart statefulset test-st
```
Если вы хотите, чтобы порядок старта подов StatefulSet не зависел от времени или других факторов, можно
переопределить `.spec.podManagementPolicy`. Его значение по умолчанию `OrderedReady`. Используйте `Parallel`, когда порядок старта и завершения не важен.  
  
```shell
kubectl -n work delete -f 4_statefullset_headless_service/04-statefullset.yaml
```
```text
service "test-st" deleted
service "test-st-headless" deleted
statefulset.apps "test-st" deleted
```
```shell
kubectl -n work apply -f 4_statefullset_headless_service/05-statefullset-parallel.yaml
```
```text
service/test-st created
service/test-st-headless created
statefulset.apps/test-st created
```
Удалим тестовый `StatefulSet`:
```shell
kubectl -n work delete -f 4_statefullset_headless_service/05-statefullset-parallel.yaml
```
