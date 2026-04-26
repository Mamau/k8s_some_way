## Распределение под по нодам кластера
***
### Требования к кластеру
***
В моем случае:
```shell
kubectl get nodes
```
```text
NAME                        STATUS   ROLES           AGE   VERSION
ubuntu-4gb-hel1-28-node-1   Ready    control-plane   15d   v1.35.3
ubuntu-4gb-hel1-29-node-2   Ready    <none>          15d   v1.35.3
ubuntu-4gb-hel1-30-node-3   Ready    <none>          15d   v1.35.3
```
### nodeName
***
В предыдущих уроках мы уже затронули тему распределения подов по нодам кластера, когда рассматривали `DaemonSet`. 
Там мы посмотрели что такое `Taint` и `Tolerations`. А также `nodeSelector`.

`nodeSelector` достаточно простой механизм подсказки планировщику kubernetes. При помощи него можно определять только метки нод,
на которые планировщик может распределять поды.  
  
Существует еще более простой пример, позволяющий приземлить под на конкретную ноду кластера: `nodeName`.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  namespace: work
spec:
  containers:
    - name: nginx
      image: nginx:1.27.5
      ports:
        - containerPort: 80
  nodeName: ubuntu-4gb-hel1-29-node-2
```
Такой подход удобен например для отладки, когда вы конкретное приложение хотите запустить на конкретной ноде.
### Affinity
***
> В манифестах пода можно использовать более продвинутые инструменты для формирования подсказок планировщику.
> Это Существует `affinity`.
  
- `Node Affinity` - определяет предпочтение размещения подов на нодах кластера.
- `Pod Affinity` - определяет правила размещения подов рядом с другими подами.
- `Pod Anti-Affinity` - определяет правила, не разрешающие размещения подов на нодах, где уже запущены такие поды.
  
#### Запрет размещения подов
Самый распространенный вариант использования это - `Pod Anti-Affinity`. Когда нам необходимо запретить размещение двух подов 
одной управляющей конструкции `Deployment` или `StatefulSet` на одной ноде кластера. Такое размещение необходимо в основном для
отказоустойчивости приложения.  
  
Пример:
```shell
nl 7_affinity_pod_topology_spread/13-podaffinity.yaml
```
```text
1  ---
     2  apiVersion: apps/v1
     3  kind: Deployment
     4  metadata:
     5    name: testdp
     6    labels:
     7      app.kubernetes.io/name: &name testdp
     8      app.kubernetes.io/instance: &instance testdp
     9      app.kubernetes.io/version: &version v0.0.1
    10  spec:
    11    selector:
    12      matchLabels:
    13        app.kubernetes.io/name: *name
    14        app.kubernetes.io/instance: *instance
    15        app.kubernetes.io/version: *version
    16    template:
    17      metadata:
    18        labels:
    19          app.kubernetes.io/name: *name
    20          app.kubernetes.io/instance: *instance
    21          app.kubernetes.io/version: *version
    22      spec:
    23        affinity:
    24          podAntiAffinity:
    25            requiredDuringSchedulingIgnoredDuringExecution:
    26              - labelSelector:
    27                  matchExpressions:
    28                    - key: app.kubernetes.io/name
    29                      operator: In
    30                      values:
    31                        - testdp
    32                    - key: app.kubernetes.io/version
    33                      operator: In
    34                      values:
    35                        - v0.0.1
    36                topologyKey: "kubernetes.io/hostname"
    37        containers:
    38          - name: whoami
    39            image: traefik/whoami
    40            ports:
    41              - containerPort: 80
    ...
```
Перед нами `Deployment`, у контейнера которого установлены три метки (стр. 19-21).  
  
В спецификации пода мы определяем раздел `affinity` (стр. 22,23).  
  
В котором (стр. 24) `podAntiAffinity`.  
  
Дальше (стр. 25) объясняем планировщику насколько "важен" данный вариант `podAntiAffinity`. Возможны два варианта:
- `requiredDuringSchedulingIgnoredDuringExecution` - обязательный к исполнению. Если не удается подобрать ноду, удовлетворяющую данному
соответствию - под создаваться не будет.
- `preferredDuringSchedulingIgnoredDuringExecution` - мягкий вариант. Попытаться найти ноду, подходящую для выполнения соответствия. 
Если нода не будет найдена - все равно запустить под, на какой либо другой ноде.
  
В нашем случае потребуется жесткое соблюдение правил. Чтобы на одной ноде, ни при каких обстоятельствах не было несколько подов
одной управляющей конструкции.  
  
Затем указываем список правил (стр. 26). У нас будет одно правило, определяющее список метод пода, которые учитываютс планировщиком.  
  
`matchExpressions` (стр. 27) задает набор правил соответствия меток и их значений:
```yaml
- key: app.kubernetes.io/name
  operator: In
  values:
    - testdp
```
`topologyKey` - определяем НОДЫ на которых будет работать данное `affinity`. Значение - это метка ноды (label), значение label не важно главное чтобы сам label был.
  
Запустим `Deployment`
```shell
kubectl -n work apply -f 7_affinity_pod_topology_spread/13-podaffinity.yaml
```
```shell
kubectl -n work get pods
```
```text
NAME                      READY   STATUS    RESTARTS   AGE
testdp-6b6b4c966b-4jzkg   1/1     Running   0          38s
```
Увеличим кол-во подов
```shell
kubectl -n work get deployments
```
```text
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
testdp   1/1     1            1           2m21s
```
```shell
kubectl -n work scale --replicas=5 deployment testdp
```
```text
NAME                      READY   STATUS    RESTARTS   AGE
testdp-7d948bd89d-cj55j   1/1     Running   0          23s
testdp-7d948bd89d-lq49l   0/1     Pending   0          23s
testdp-7d948bd89d-pkb6l   0/1     Pending   0          23s
testdp-7d948bd89d-vc9b9   0/1     Pending   0          23s
testdp-7d948bd89d-zdwvr   1/1     Running   0          52s
```
Видим что поды запустили только 2 из 5, т.е у меня всего 3 ноды, то на контрол плейн она не запустится (зараза - taint стоит) 
на ноде 2 и 3 запустились поды а остальные из-за афинити не могут быть нигде запущены, потомучто мы запретили одинаковым 
подам быть на одной ноде, но когда появится новая нода - то там сразу запустится одна из этих pending.  
  
Посмотрим подробнее на один из pending подов.
```shell
kubectl -n work events --for pod/testdp-7d948bd89d-vc9b9
```
```text
LAST SEEN   TYPE      REASON             OBJECT                        MESSAGE
3m11s       Warning   FailedScheduling   Pod/testdp-7d948bd89d-vc9b9   0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) didn't match pod anti-affinity rules. no new claims to deallocate, preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
```
0 из 3 нод доступны. 1 нода заражена, 2 ноды не подходят из-за правила анти афинити.  
  
Удалим `Deployment`:
```shell
kubectl -n work delete deployments testdp
```
`podAntiAffinity` правило работает также как анти, но наоборот, он будет наоборот поды загонять вместе.  
  
#### NodeAffinity
При помощи `nodeAffinity` мы можем определять на какие ноды кластера будет размещаться под.
  
Для эксперимента поделим наш кластер на две зоны: `Africa` и `Australia`. Для этого поставим соответствующие метки на ноды кластера.
```shell
kubectl label nodes ubuntu-4gb-hel1-29-node-2 topology.kubernetes.io/zone=Africa
kubectl label nodes ubuntu-4gb-hel1-30-node-3 topology.kubernetes.io/zone=Australia
```
```text
node/ubuntu-4gb-hel1-29-node-2 labeled
node/ubuntu-4gb-hel1-30-node-3 labeled
```
В манифесте в разделе `afinity` добавим `nodeAfinity`:
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - Africa
```
Создадим `Deployment`:
```shell
kubectl -n work apply -f 7_affinity_pod_topology_spread/14-nodeaffinity.yaml 
```
```text
deployment.apps/testdp created
```
Посмотрим, куда приземлился под
```shell
kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'
```
```text
Name                      Node
testdp-7c4d8d5689-d7t46   ubuntu-4gb-hel1-29-node-2
```
Сделаем scale
```shell
kubectl -n work scale --replicas=3 deployment testdp
```
Проверим
```shell
kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'
```
```text
Name                      Node
testdp-7c4d8d5689-9sxg7   <none>
testdp-7c4d8d5689-d7t46   ubuntu-4gb-hel1-29-node-2
testdp-7c4d8d5689-qqmq8   <none>
```
Видим что остался один под на ноде которая Африканская, и два пода разместить не куда, потомучто еще одна нода у нас Австралийская.  
  
Удаляем:
```shell
kubectl -n work delete deployments testdp
```
  
### Вес
При использовании правил afinity можно указывать какое из правил имеет большее преимущество. Для этого правилам можно указывать
их `weight`.  
  
Посмотрим
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/os
              operator: In
              values:
                - linux
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - Africa
      - weight: 50
        preference:
          matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - Australia
```
`requiredDuringSchedulingIgnoredDuringExecution` - заставляет планировщик распределять поды только на linux машинах  
`preferredDuringSchedulingIgnoredDuringExecution` - добавляет более мягкие правила, с указанием веса каждого правила. Чем выше вес тем важнее правило и первее будет выполнено. 
В нашем случае если в Австралии не будет места то так и быть разместим в Африке.
  
Создадим:
```shell
kubectl -n work apply -f 7_affinity_pod_topology_spread/15-nodeaffinityweight.yaml 
```
Проверим
```shell
kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'
```
```text
Name                      Node
testdp-6796b688d7-fmdgt   ubuntu-4gb-hel1-30-node-3
```
ubuntu-4gb-hel1-30-node-3 - Австралия  
  
Добавим еще
```shell
kubectl -n work scale --replicas=2 deployment testdp
```
```shell
kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'
```
```text
Name                      Node
testdp-6796b688d7-9n2d4   ubuntu-4gb-hel1-29-node-2
testdp-6796b688d7-fmdgt   ubuntu-4gb-hel1-30-node-3
```
Видим, что под теперь приземлился и на Африканскую ноду, т.к в Австралии уже все закончилось.
  
Почистим
```shell
kubectl -n work delete deployment testdp
```
```text
deployment.apps "testdp" deleted
```
  
### Pod Topology Spread Constraints
  
В предыдущем примере мы распределяли поды по зонам. Но, как вы наверное заметили поды сначала добавлялись в одной зона.
Потом когда ноды в зоне заканчивались, поды начали создаваться во второй зоне.  
  
Как сделать, чтобы поды равномерно распределялись между нодами зоны? Причем, чтобы не было перекосов, когда в одной зоне
подов значительно больше, чем в другой, при одинаковом количестве нод в каждой зоне.  

> Pod Topology Spread Constraints - определяет набор правил управления распределения подов между доменами сбоя, такими как регионы,
> зоны, узлы и другие пользовательские домены топологии.
  
Изменим наш манифест. Удалим из него раздел `affinity`. В спецификации пода добавим `topologySpreadConstraints`.
```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: *name
          app.kubernetes.io/instance: *instance
          app.kubernetes.io/version: *version
      nodeAffinityPolicy: Ignore
      nodeTaintsPolicy: Honor
```
Параметры `topologySpreadConstraints`:
- `maxSkew` - максимальная разница количество подов между доменами топологии
- `topologyKey` - метка на ноде кластера, которая используется для определения доменов топологии.
- `whenUnsatisfiable` - что делать с подом, если он не соответствует ограничению.
  - `DoNotSchedule` - (по умолчанию) запрещает планировщику запускать поды на ноде.
  - `ScheduleAnyway` - разрешает запускать под на ноде.
- `labelSelector` - определяет список меток подов, попадающих под это правило.
- `nodeAffinityPolicy` - определяет будут ли учитываться `nodeAffinity`/`nodeSelector` пода при расчете неравномерности распределения пода.
  - `Honor` - (по умолчанию) в расчет включаются только ноды, соответствующие `nodeAffinity`/`nodeSelector`.
  - `Ignore` - в расчеты включены все ноды.
- `nodeTaintsPolicy` - аналогично `nodeAffinityPolicy`, только учитываются `taints`.
  - `Honor` - включаются ноды без установленных `Taints`, а также ноды для которых у пода есть `Toleration`.
  - `Ignore` - (по умолчанию) в расчеты включены все ноды.
  
Создадим Deployment
```shell
kubectl -n work apply -f 7_affinity_pod_topology_spread/16-ptsc.yaml
```
```shell
kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'
```
```text
Name                      Node
testdp-5668f9f5f5-kxg42   ubuntu-4gb-hel1-30-node-3
```
Добавим еще
```shell
kubectl -n work scale --replicas=2 deployment testdp
```
```shell
kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'
```
```text
Name                      Node
testdp-5668f9f5f5-kxg42   ubuntu-4gb-hel1-30-node-3
testdp-5668f9f5f5-xgtbn   ubuntu-4gb-hel1-29-node-2
```
```shell
kubectl -n work scale --replicas=8 deployment testdp
```
```shell
kubectl -n work get pods -o='custom-columns=Name:.metadata.name,Node:.spec.nodeName'
```
Распределено равномерно
```text
Name                      Node
testdp-5668f9f5f5-657vs   ubuntu-4gb-hel1-29-node-2
testdp-5668f9f5f5-6td2w   ubuntu-4gb-hel1-30-node-3
testdp-5668f9f5f5-bswc6   ubuntu-4gb-hel1-30-node-3
testdp-5668f9f5f5-ddd4f   ubuntu-4gb-hel1-29-node-2
testdp-5668f9f5f5-g628x   ubuntu-4gb-hel1-29-node-2
testdp-5668f9f5f5-kxg42   ubuntu-4gb-hel1-30-node-3
testdp-5668f9f5f5-nq6hg   ubuntu-4gb-hel1-30-node-3
testdp-5668f9f5f5-xgtbn   ubuntu-4gb-hel1-29-node-2
```
Почистим
```shell
kubectl -n work delete deployment testdp
```
```text
deployment.apps "testdp" deleted
```