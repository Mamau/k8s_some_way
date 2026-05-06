## Downward Api
***
Механизм `Downward Api` позволяет приложения, работающим в контейнерах, получать информацию о самих себе и 
своем окружении без необходимости прямого обращения к API-серверу Kubernetes. Эта информация, относится к поду 
или контейнеру (метаданные, статус), может быть передана двумя способами: через переменные среды окружения 
или через файлы в `volume`.
  
Это бывает полезно, когда приложению необходимо знать свой IP-адрес, имя пода, `namespace`, в котором он запущен, 
или назначенные ему лимиты по ресурсам. 
  
Конечно, этот механизм не позволяет получить все данные из текущего манифеста пода, хранящиеся в базе данных Kubernetes. 
Для получения всего объема необходимо обращаться непосредственно к Kubernetes API. А это требует дополнительных разрешений, 
описанных в правилах RBAC.
  
### Использование Downward Api через переменные окружения (env)
***
Один из самых простых способов использовать Downward Api - это передать информацию в контейнер как переменные среды окружения. 
Это делается с помощью поля `valueFrom.fieldRef` в определении переменной.  
  
В `fieldPath` можно указать различные поля из спецификации пода. Наиболее часто используемые:
- `metadata.name` - имя пода
- `metadata.namespace` - `namespace` в котором запущен под
- `status.podIP` - IP-адрес пода
- `spec.nodeName` - имя узла (node), на котором работает под
- `spec.serviceAccountName` - имя `ServiceAccountName`, используемого подом
- `metadata.labels['<key>']` - значение метки (label) с ключем `<key>`
- `metadata.annotations['<key>']` - значение аннотации с ключем `<key>`
  
Пример
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-env-demo
  labels:
    app.kubernetes.io/name: downward-demo
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  containers:
    - name: main
      image: busybox:1.37
      command:
        - "sh"
        - "-c"
      args:
        - |
          echo "---";
          echo "Pod Name: $POD_NAME";
          echo "Pod NameSpace: $POD_NAMESPACE";
          echo "Pod IP: $POD_IP";
          echo "Node Name: $NODE_NAME";
          echo "App Label: $APP_LABEL";
          sleep infinity
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: APP_LABEL
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['app.kubernetes.io/instance']
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
  restartPolicy: Never
```
Применим
```shell
kubectl -n work apply -f 11_downward_api/31-downward-api-env.yaml
```
Проверим поды
```shell
kubectl -n work get pods
```
```text
NAME                    READY   STATUS    RESTARTS   AGE
downward-api-env-demo   1/1     Running   0          38s
```
Смотрим логи контейнера
```shell
kubectl -n work logs downward-api-env-demo
```
```text
---
Pod Name: downward-api-env-demo
Pod NameSpace: work
Pod IP: 192.168.125.176
Node Name: ubuntu-4gb-hel1-30-node-3
App Label: test
```
Удалим под. Чтобы не ждать (из-за sleep infinity), когда система принудительно удалит под (значение `grace period` по умолчанию 30 сек), 
используем параметр `--force`:
```shell
kubectl -n work delete -f 11_downward_api/31-downward-api-env.yaml --force
```
```text
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "downward-api-env-demo" force deleted
```
### Использование Downward Api через volume
***
Другой способ - смонтировать информацию как файлы в `volume` типа `downwardAPI`. Этот метод более гибкий, так как позволяет передавать 
не только поля (`fieldRef`), но и информацию о ресурсах контейнера (`resourcesFieldRef`), например, лимиты CPU или памяти.
  
Данные смонтированные через `volume`, обновляются автоматически, если они изменяются (например, метки или аннотации). За исключением случая, 
когда вы используете `subPath` при монтировании тома.
  
Рассмотрим пример манифеста `11_downward_api/32-downward-api-volume.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-volume-demo
  labels:
    app.kubernetes.io/name: downward-volume-demo
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
  annotations:
    author: "mamau"
spec:
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          # Pod fields
          - path: "pod_name"
            fieldRef:
              fieldPath: metadata.name
          - path: "pod_namespace"
            fieldRef:
              fieldPath: metadata.namespace
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
          # Container Fields
          - path: "cpu_limit"
            resourceFieldRef:
              containerName: main-container
              resource: limits.cpu
              divisor: "1"
          - path: "memory_limit"
            resourceFieldRef:
              containerName: main-container
              resource: limits.memory
              divisor: 1Mi
  containers:
    - name: main-container
      image: busybox:1.37
      command:
        - "sh"
        - "-c"
      args:
        - |
          echo "---";
          echo "Pod Name: $POD_NAME";
          echo "Pod NameSpace: $POD_NAMESPACE";
          echo "Labels:
          $(cat /etc/podinfo/labels)
          ";
          echo "Annotations:
          $(cat /etc/podinfo/annotations)
          ";
          echo "CPU Limit: $(cat /etc/podinfo/cpu_limit)";
          echo "Memory Limit: $(cat /etc/podinfo/memory_limit)";
          sleep infinity
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  restartPolicy: Never
```
В этом манифесте мы создаем том `podinfo`, который использует `downwardAPI`. Внутри `item` мы определяем, какие данные 
и какие файлы нужно поместить.  
- `path` - этом имя файла, который будет создан в точке монтирования.
- `fieldRef` - используется для метаданных пода (имя, метки, аннотации).
- `resourceFieldRef` - используется для получения информации о ресурсах контейнера (CPU, memory). Обратите внимание на `containerName`,
которое должно соответствовать имени контейнера (из которого хотим получить значение ресурсов), и `divisor` для форматирования значения.
  
Применим
```shell
kubectl -n work apply -f 11_downward_api/32-downward-api-volume.yaml
```
Убедимся, что под работает
```shell
kubectl -n work get pods
```
```text
NAME                       READY   STATUS    RESTARTS   AGE
downward-api-volume-demo   1/1     Running   0          29s
```
Смотрим логи контейнера
```shell
kubectl -n work logs downward-api-volume-demo
```
```text
Pod Name: 
Pod NameSpace: 
Labels:
app.kubernetes.io/instance="test"
app.kubernetes.io/name="downward-volume-demo"
app.kubernetes.io/version="v0.0.1"
topology.kubernetes.io/zone="Australia"

Annotations:
author="mamau"
kubectl.kubernetes.io/last-applied-configuration="{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{\"author\":\"mamau\"},\"labels\":{\"app.kubernetes.io/instance\":\"test\",\"app.kubernetes.io/name\":\"downward-volume-demo\",\"app.kubernetes.io/version\":\"v0.0.1\"},\"name\":\"downward-api-volume-demo\",\"namespace\":\"work\"},\"spec\":{\"containers\":[{\"args\":[\"echo \\\"---\\\";\\necho \\\"Pod Name: $POD_NAME\\\";\\necho \\\"Pod NameSpace: $POD_NAMESPACE\\\";\\necho \\\"Labels:\\n$(cat /etc/podinfo/labels)\\n\\\";\\necho \\\"Annotations:\\n$(cat /etc/podinfo/annotations)\\n\\\";\\necho \\\"CPU Limit: $(cat /etc/podinfo/cpu_limit)\\\";\\necho \\\"Memory Limit: $(cat /etc/podinfo/memory_limit)\\\";\\nsleep infinity\\n\"],\"command\":[\"sh\",\"-c\"],\"image\":\"busybox:1.37\",\"name\":\"main-container\",\"resources\":{\"limits\":{\"cpu\":\"500m\",\"memory\":\"128Mi\"},\"requests\":{\"cpu\":\"250m\",\"memory\":\"64Mi\"}},\"volumeMounts\":[{\"mountPath\":\"/etc/podinfo\",\"name\":\"podinfo\"}]}],\"restartPolicy\":\"Never\",\"volumes\":[{\"downwardAPI\":{\"items\":[{\"fieldRef\":{\"fieldPath\":\"metadata.name\"},\"path\":\"pod_name\"},{\"fieldRef\":{\"fieldPath\":\"metadata.namespace\"},\"path\":\"pod_namespace\"},{\"fieldRef\":{\"fieldPath\":\"metadata.labels\"},\"path\":\"labels\"},{\"fieldRef\":{\"fieldPath\":\"metadata.annotations\"},\"path\":\"annotations\"},{\"path\":\"cpu_limit\",\"resourceFieldRef\":{\"containerName\":\"main-container\",\"divisor\":\"1\",\"resource\":\"limits.cpu\"}},{\"path\":\"memory_limit\",\"resourceFieldRef\":{\"containerName\":\"main-container\",\"divisor\":\"1Mi\",\"resource\":\"limits.memory\"}}]},\"name\":\"podinfo\"}]}}\n"
kubernetes.io/config.seen="2026-05-05T21:12:59.106221755Z"
kubernetes.io/config.source="api"

CPU Limit: 1
Memory Limit: 128
```
Удалим
```shell
kubectl -n work delete -f 11_downward_api/32-downward-api-volume.yaml --force
```
```text
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "downward-api-volume-demo" force deleted
```
### Динамическое обновление данных
***
Ключевое преимущество `downwardAPI` при использовании `volume` заключается в том, что данные в файлах обновляются автоматически 
при изменении метаданных пода.  
В отличии от переменных окружения, которые устанавливаются один раз при запуске контейнера, файлы в `downwardAPI volume` 
отражают актуальное состояние ресурса.
  
Продемонстрируем это на примере. Создадим под, который будет периодически считывать свои метки и выводить их в лог. Затем мы изменим 
одну из меток и убедимся, что приложение внутри контейнера увидит это изменение.
  
Рассмотрим манифест 11_downward_api/33-downward-api-volume-dynamic.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-dynamic-demo
  labels:
    app.kubernetes.io/name: downward-api-dynamic-demo
    status: pending
spec:
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
  containers:
    - name: main-container
      image: busybox:1.37
      command:
        - "sh"
        - "-c"
      args:
        - |
          while true; do
            echo "---";
            echo "Reading labels at $(date):";
            cat /etc/podinfo/labels;
            sleep 10;
          done
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  restartPolicy: Never
```
Применим
```shell
kubectl -n work apply -f 11_downward_api/33-downward-api-volume-dynamic.yaml
```
Теперь посмотрим логи в режими слежения (`-f`), чтобы увидеть обновления в реальном времени.
```shell
kubectl -n work logs -f downward-api-dynamic-demo
```
```text
status="pending"
topology.kubernetes.io/zone="Australia"---
Reading labels at Tue May  5 21:34:05 UTC 2026:
app.kubernetes.io/name="downward-api-dynamic-demo"
status="pending"
topology.kubernetes.io/zone="Australia"---
Reading labels at Tue May  5 21:34:15 UTC 2026:
app.kubernetes.io/name="downward-api-dynamic-demo"
status="pending"
```
Не прерывая выполнение предыдущей команды, откройте новый терминал и измените метку `status` с `pending ` на `running`.
Флаг `--overwrite` необходим для изменения существующей метки.
```shell
kubectl -n work label pod downward-api-dynamic-demo status=running --overwrite
```
```text
pod/downward-api-dynamic-demo labeled
```
Вернувшись в первый терминал, вы увидите, что содержимое файла `/etc/podinfo/labels` изменилось, и приложение теперь видит новое значение метки
```text
topology.kubernetes.io/zone="Australia"---
Reading labels at Tue May  5 21:41:46 UTC 2026:
app.kubernetes.io/name="downward-api-dynamic-demo"
status="running"
topology.kubernetes.io/zone="Australia"---
Reading labels at Tue May  5 21:41:56 UTC 2026:
app.kubernetes.io/name="downward-api-dynamic-demo"
status="running"
```  
Этот механизм позволяет создавать приложения, которые могут динамически реагировать на изменения в своей конфигурации 
или состоянии Kubernetes без необходимости прямого обращения к API-серверу.
  
Посмотрим внутри пода на директорию, куда монтируются данные:
```shell
kubectl -n work exec downward-api-dynamic-demo -ti -- ls -la /etc/podinfo
```
```text
total 4
drwxrwxrwt    3 root     root           100 May  5 21:39 .
drwxr-xr-x    1 root     root          4096 May  5 21:33 ..
drwxr-xr-x    2 root     root            60 May  5 21:39 ..2026_05_05_21_39_51.2222314020
lrwxrwxrwx    1 root     root            32 May  5 21:39 ..data -> ..2026_05_05_21_39_51.2222314020
lrwxrwxrwx    1 root     root            13 May  5 21:33 labels -> ..data/labels
```
Удалим под
```shell
kubectl -n work delete -f 11_downward_api/33-downward-api-volume-dynamic.yaml --force
```
```text
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "downward-api-dynamic-demo" force deleted
```