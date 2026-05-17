## Persistent volume
***
### Что такое hostPath
***
Том (`volume`) типа hostPath позволяет монтировать файл или директорию из файловой системы хост-узла (node) непосредственно в под.
Это один из самых простых способов обеспечить постоянное хранение данных, однако он имеет серьезные ограничения, которые 
делают его непригодным для большинства реальных приложений.  
  
В отличии от тома `emptyDir`, который пуст при создании пода и уничтожается вместе с ним, данные в `hostPath` volume 
сохраняются после перезапуска контейнера или даже всего пода. Однако, эти данные жестко привязаны к конкретному узлу кластера. 
Если под будет пересоздан на другом узле, он потеряет доступ к своим данным.  
  
### Пример 1: Запись данных на узле кластера
***
Рассмотрим, как под может записать данные в файловую систему узла, на котором он запущен.
  
Создадим манифест `13_host_path_persist_volume/37-hostpath-write.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-write-demo
spec:
  volumes:
    - name: host-data
      hostPath:
        # Путь на хост-машине
        path: /tmp/k8s-book-data
        # Kubernetes создаст директорию, если ее нет
        type: DirectoryOrCreate
  restartPolicy: Never
  containers:
    - name: main
      image: busybox:1.37
      command:
        - "sh"
        - "-c"
      args:
        - |
          echo "Hello from pod $(POD_NAME)!" > /data/hello.txt
          echo "Data written to /data/hello.txt"
          echo "---"
          sleep infinity
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      volumeMounts:
        - name: host-data
          mountPath: /data
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```
В этом манифесте мы определяем `volume` с именем `host-data` и типом `hostPath`. Он указывает на путь `/tmp/k8s-book-data` 
на хост-машине. Поле `type`: `DirectoryOrCreate` гарантирует, что Kubernetes создаст эту директорию на узле, если она отсутствует. 
Затем мы монтируем этот том в контейнер по пути `/data`.  
  
Применим манифест
```shell
kubectl -n work apply -f 13_host_path_persist_volume/37-hostpath-write.yaml
```
```text
pod/hostpath-write-demo created
```
Посмотрим на логи пода
```shell
kubectl -n work logs hostpath-write-demo
```
```text
Data written to /data/hello.txt
---
```
Нам нужно узнать, на каком узле запущен под, и проверить содержимое файла на самом узле.
```shell
# Узнаем имя узла
NODE_NAME=$(kubectl -n work get pod hostpath-write-demo -o jsonpath='{.spec.nodeName}') && \
echo "Pod is running on node: $NODE_NAME"

# Выполним команду на узле для чтения файла
# (Эта команда предполагает наличие ssh-доступа к узлам)
ssh $NODE_NAME -- cat /tmp/k8s-book-data/hello.txt 
```
```text
Pod is running on node: ubuntu-4gb-hel1-30-node-3
```
```text
Hello from pod hostpath-write-demo!
```
Как мы видим, файл, созданный внутри контейнера, успешно сохранился в файловой системе узла.  
  
Не забудьте удалить под и созданные им данные на узле:
```shell
# Удаляем под
kubectl -n work delete -f 13_host_path_persist_volume/37-hostpath-write.yaml --force

# Удаляем директорию с данными на узле
ssh root@k8s_worker_2 -- rm -rf /tmp/k8s-book-data && \
echo "Cleaned up data"
```
```text
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "hostpath-write-demo" force deleted

Cleaned up data
```