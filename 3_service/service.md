## Service
***
> Доступ к приложению в Kubernetes осуществляется через сервисы, которые предоставляют единую точку доступа к группе Pod`ов

[Kubernetes API: Service](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/)  
  
Пример конфигурации сервиса:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-first-nginx
  namespace: work
  labels:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.27.5
spec:
  selector:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.27.5
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      name: http
```
Традиционно два раздела: metadata и spec. Существует [Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/). Поэтому добавили немного стандарта.  
Кстати, если будете использовать helm, он будет автоматически добавлять эти метки.  
  
Ссылка на другие объекты в kubernetes осуществляется при помощи меток. Имена объектов используются крайне редко. **В spec определен
раздел selector, который содержит условия выбора Pods для обслуживания сервисом.**  
  
В ports мы говорим на каком порту (port) будет доступен наш сервис и куда он будет пересылать запросы (targetPort). Если явно
не определить protocol, то по умолчанию используется TCP. Имя протокола не обязательно. Но его рекомендуют указывать.  
  
Существует несколько типов сервисов, определяемых при помощи типа **type**. Если его явно не указывать, то по умолчанию используется тип
ClusterIP.  
  
Согласно синтаксиса yaml, в одном файле можно определить несколько ресурсов, резделяя их тремя дефисами `---`. 
Точнее говоря, каждый ресурс должен начинаться с `---`, а заканчиваться `...`. Но завершающие точки для разграничения ресурсов не обязательны.  
  
Применим манифест:  
```shell
kubectl apply -f 3_service/03-service.yaml
```
```text
service/my-first-nginx created
deployment.apps/my-first-deployment created
```
Посмотрим что получилось:
```yaml
kubectl -n work get all
```
```text
NAME                                       READY   STATUS    RESTARTS   AGE
pod/my-first-deployment-74b66d6d9d-c2cw7   1/1     Running   0          2m47s
pod/my-first-deployment-74b66d6d9d-c4cnf   1/1     Running   0          2m47s
pod/my-first-deployment-74b66d6d9d-qftmx   1/1     Running   0          2m47s
pod/my-first-pod                           1/1     Running   0          6d5h

NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/my-first-nginx   ClusterIP   10.102.191.218   <none>        80/TCP    4m47s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-first-deployment   3/3     3            3           2m47s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-first-deployment-74b66d6d9d   3         3         3       2m47s
```
Посмотрим подробнее на сервис:
```yaml
kubectl -n work describe service my-first-nginx
```
```text
Name:                     my-first-nginx
Namespace:                work
Labels:                   app.kubernetes.io/name=nginx
                          app.kubernetes.io/version=1.27.5
Annotations:              <none>
Selector:                 app.kubernetes.io/name=nginx,app.kubernetes.io/version=1.27.5
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.102.191.218
IPs:                      10.102.191.218
Port:                     http  80/TCP
TargetPort:               80/TCP
Endpoints:                192.168.125.138:80,192.168.125.139:80,192.168.129.75:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```
Видим IP - это ip самого сервиса. Также видим Endpoints - это ip подов к которым он прикреплен, и если внутри кластер сходить на IP сервиса
10.102.191.218 - то попадем на один из адресов подов. Можно внутри ноды прям через `curl 10.102.191.218` это проверить - получим ответ nginx.  
  
Посмотрим endpoints:
```shell
kubectl -n work get ep my-first-nginx
```
```text
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME             ENDPOINTS                                                 AGE
my-first-nginx   192.168.125.138:80,192.168.125.139:80,192.168.129.75:80   20m
```
Видим те же эндпоинты, но такая команда устарела и предлагает заюзать EndpointSlice
```shell
kubectl -n work get EndpointSlice
```
```text
NAME                   ADDRESSTYPE   PORTS   ENDPOINTS                                        AGE
my-first-nginx-ld4bf   IPv4          80      192.168.125.138,192.168.125.139,192.168.129.75   22m
```
Можем посмотреть более подробно весь манифест
```shell
kubectl -n work get EndpointSlice my-first-nginx-ld4bf -o yaml | less
```
Вырезка оттуда
```text
endpoints:
- addresses:
  - 192.168.125.138
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: ubuntu-4gb-hel1-30-node-3
  targetRef:
    kind: Pod
    name: my-first-deployment-74b66d6d9d-qftmx
    namespace: work
    uid: 1280461e-0bb2-4f2f-9389-baa411700e8b
- addresses:
  - 192.168.125.139
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: ubuntu-4gb-hel1-30-node-3
  targetRef:
    kind: Pod
    name: my-first-deployment-74b66d6d9d-c2cw7
    namespace: work
    uid: 982b845c-c288-45fa-aafb-5322a6a95f48
```
Видим подробную инфу о каждом ендпоинте.  
  
Сейчас этот сервис живет внутри кластера и не выводится за его пределы. Чтобы получить доступ к сервису извен, надо 
создавать ресурсы типа `Ingress` или воспользоваться механизмами, предоставляемыми `Gateway API`.  
  
Также доступ к сервису со своей локальной машины можно получить через `kubectl port-forward`. Этот способ позволяет нам
перенаправлять порты из контейнера на локальную машину. Пример:
```shell
kubectl -n work port-forward svc/my-first-nginx 8080:80 
```
Откройте новое терминальное окно и выполните команду:
```shell
curl localhost:8080
```