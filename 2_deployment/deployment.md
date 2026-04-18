## Deployment
***
Deployment используется для управления подами. Приложения в подах, поддерживаемых Deployment,
обычно являются stateless приложениями (не поддерживающими состояние).
Т.е свое состояние если это необходимо, приложения сохраняют во внешних хранилищах:
базах данных, файловых системах и т.д.  
  
### Манифест
***
Какие значения можно использовать в манифесте, описано в документации по [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/)  
  
Пример манифеста:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-first-deployment
  namespace: work
  labels:
    app: &app nginx
    version: &version 1.27.5
spec:
  replicas: 3
  selector:
    matchLabels:
      app: *app
      version: *version
  template:
    metadata:
      labels:
        app: *app
        version: *version
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```
> & - это якорь в YAML. С его помощью можно пометить значение именем, а потом переиспользовать его ниже через *
> Так не нужно дублировать одно и то же значение, и изменения вносятся только в одном месте.
  
### Запуск Deployment
***
Деплоймент можно применить так:
```shell
kubectl apply -f 2_deployment/02-deployment.yaml
```
Получим ответ
```shell
deployment.apps/my-first-deployment created
```
Посмотрим что получилось
```shell
kubectl -n work get all
```
Используя специальный параметр all получим много много ресурсов:
```shell
NAME                                       READY   STATUS    RESTARTS   AGE
pod/my-first-deployment-66cd4894c6-pnmmc   1/1     Running   0          57s
pod/my-first-deployment-66cd4894c6-qklnj   1/1     Running   0          57s
pod/my-first-deployment-66cd4894c6-qxngz   1/1     Running   0          57s
pod/my-first-pod                           1/1     Running   0          4d23h

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-first-deployment   3/3     3            3           57s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-first-deployment-66cd4894c6   3         3         3       57s
```
Появилась **replicaset**  
 
Мы сделали манифест деплоймента и кинули в k8s api,  
k8s api сохранил манифест деплоймента в БД (etcd), базовый Controller (состоит из много разных контроллеров в том числе и контроллер деплоймента) смотрит в k8s api (переодически).    
И когда контроллер деплоймента обнаруживает что появился новый манифест деплоймента он на него реагирует... и как реагирует - создает манифест ReplicaSet  
и кидает ее через k8s api в БД etcd.  
Далее в базовом контроллере есть контроллер ReplicaSet который в свою очередь следит за манифестами ReplicaSet...  он видит
манифест ReplicaSet в которых указаны поды которые надо запустить и этот контроллер создает 3 (в нашем случа) манифеста подов и кидает их в БД etcd через апи. т.е появились манифесты подов.   
Далее планировщик видит поды, и видит что они не имеют поля описания на каких нодах они назначены и он по своим алгоритмам распределяет на какие ноды кластера разместить поды, добавляет им эти служебные поля и сохраняет в БД.
Далее кублеты видят что есть поды которые назначены на их ноды и берут и запускают у себя эти поды
  
Обрати внимание что имя ReplicaSet: my-first-deployment-<hash>. Они связаны с названием Deployment.  
А имена подов содержат в себе имя ReplicaSet, а также уникальный идентификатор пода. Т.е понять кто кем управляет
можно по именам ресурсов.  
  
Deployment понимает, каким ReplicaSet он управляет, при помощи labels, которые мы указали в манифесте Deployment.
У ReplicaSet тоже есть метки, определяющие какими подами он управляет.  
Можем посмотреть так:
```shell
kubectl -n work get replicaset.apps/my-first-deployment-66cd4894c6 --output=jsonpath={.spec.selector} | jq
```
```shell
{
  "matchLabels": {
    "app": "nginx",
    "pod-template-hash": "66cd4894c6",
    "version": "1.27.5"
  }
}
```
Кроме прямой связи через labels, от Deployment к ReplicaSet и от ReplicaSet к Pod, в свою очередь существует обратная связь от
Pod к ReplicaSet и от ReplicaSet к Deployment. Она осуществляется через поля .metadata.ownerReferences:
```shell
kubectl -n work get replicaset.apps/my-first-deployment-66cd4894c6 --output=jsonpath={.metadata.ownerReferences} | jq
```
```shell
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "Deployment",
    "name": "my-first-deployment",
    "uid": "61d4aeeb-53ce-4c00-9ea0-6c10c1c3ae99"
  }
]
```
### Рестарт приложения.
***
Рестарт приложения можно осуществить несколькими способами:
- Изменив содержимое шаблона пода в Deployment.
- В ручную с помощью команды kubectl rollout restart.
  
В качестве примера можно изменить версию nginx на предыдущую. Это можно сделать при помощи: 
```shell
kubectl set image deployment/my-first-deployment nginx=nginx:1.27.5.
```
Но это не лучший вариант. Мы управляем кубером при помози манифестов. Манифесты храним в git. Предполагается, что в git 
находится то состояние кластера, которое нам необходимо. И если мы будем чтото менять не используя манифесты, в git будут
недостоверные данные. Поэтому лучше один раз написать/изменить манифест в git, чем потом пол дня потерять на поиск ошибки. 
Обычно для управления состоянием кластера используют системы Continue Delivery (CD), такие как ArgoCD, Flux и другие.
Они поддерживают состояние кластера в соответствии с данными в git репозитории.  
  
Лучше применять изменения через новый манифест:
```shell
kubectl apply -f 2_deployment/03-deployment.yaml
```
```shell
deployment.apps/my-first-deployment configured
```
Можно посмотреть через watch (или вашу любимую программу k9s или lens)
```shell
watch kubectl -n work get all
```
Выйти из просмотра - Ctrl+C  
  
Итоговый результат
```shell
kubectl -n work get all
```
При изменении деплоймента будет создаваться новая реплика сет, при этом можно будет откатиться на предыдущую версию реплики сет если надо  
по умолчанию максимальное кол-во реплика сет = 10, если создаете новую - то самая старая будет уже удалятся.
```shell
NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-first-deployment-55746f6b68   0         0         0       5m8s
replicaset.apps/my-first-deployment-66cd4894c6   0         0         0       3h13m
replicaset.apps/my-first-deployment-74d67456ff   4         4         4       24m
```
Можно ограничить максимальное кол-во старых версий параметром: .spec.revisionHistoryLimit:3  
  
Но откат версий - это вещь которой редко пользуются  
поэтому если вы выкатываете новую версию приложения, новый контейнер - вы выкатываете новый деплоймент
  
> Если вам надо по каким либо причинам перезапустить приложение. Например, для того, чтобы оно перечитало свои конфигурационный файлы.
> Воспользуйтесь командой:
```shell
kubectl -n work rollout restart deployment my-first-deployment
```
```shell
deployment.apps/my-first-deployment restarted
```
### Strategy
***
Вы наверное обратили внимание, как контроллер Deployment обновляет приложение.  
По умолчанию он использует стратегию .spec.strategy.type: RollingUpdate. Это означает, что обновление происходит постепенно,  
старые поды заменяются новыми по одному. У этой стратегии есть два конфигурационных параметра:  
- .spec.strategy.rollingUpdate.maxUnavailable - это необязательное поле, в котором указывается максимальное количество подов,  
которые могут быть недоступны в процессе обновления. Значение может быть абсолютным числом (например, 5) или процентом от   
желаемого количества модулей (например, 10%). Абсолютное число рассчитывается из процента путем округления в меньшую сторону.   
Значение не может быть равно 0, если .spec.strategy.rollingUpdate.maxSurge равно 0. Значение по умолчанию - 25%.  
- .spec.strategy.rollingUpdate.maxSurge - это необязательное поле, в котором указывается максимальное количество подов, которые  
могут быть созданы сверх желаемого количества подов. Значение может быть абсолютным числом (например, 5) или процентом от желаемого   
количества модулей (например, 10%). Значение не может быть равно 0, если maxUnavailable равно 0.   
Абсолютное число рассчитывается на основе процента путем округления в большую сторону. Значение по умолчанию - 25%.  

Но в редких случаях может потребоваться полностью удалить все экземпляры приложения, прежде чем запускать новые поды.  
В таких ситуациях можно использовать стратегию Recreate
