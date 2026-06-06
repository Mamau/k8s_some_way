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
### Пример 2: Обмен данными между подами
***
Поскольку `hostPath` использует общую для всех подов на одном узле файловую систему, его можно использовать для обмена
данными между ними.
  
Рассмотрим манифест 13_host_path_persist_volume/38-hostpath-share.yaml который запускает "писателя" и "читателя"
```yaml
# 1. Под-писатель (writer)
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-writer
  labels:
    app: hostpath-share
spec:
  containers:
    - name: main
      image: busybox:1.37
      command:
        - "sh"
        - "-c"
      args:
        - |
          while true; do
            echo "Writer say hello at $(date)" >> /data/shared.log;
            sleep 5;
          done
      volumeMounts:
        - name: shared-data
          mountPath: /data
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
  volumes:
    - name: shared-data
      hostPath:
        path: /tmp/k8s-book-shared
        type: DirectoryOrCreate
---
# 2. Под-читатель (reader)
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-reader
spec:
  # Гарантируем запуск на том же узле, что и writer
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - hostpath-share
          topologyKey: kubernetes.io/hostname
  containers:
    - name: reader
      image: busybox:1.37
      command:
        - "sh"
        - "-c"
      args:
        - |
          while [ ! -f /data/shared.log ]; do sleep 2; done;
          tail -f /data/shared.log
      volumeMounts:
        - name: shared-data
          mountPath: /data
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
  volumes:
    - name: shared-data
      hostPath:
        path: /tmp/k8s-book-shared
        type: DirectoryOrCreate
```
Здесь мы используем `podAfinity`, чтобы планировщик Kubernetes разместил под `hostpath-reader` на том же узле, где
уже работает под с меткой `app: hostpath-share`.
  
Применим манифест:
```shell
kubectl -n work apply -f 13_host_path_persist_volume/38-hostpath-share.yaml
```
Запустим просмотр логов "читателя":
```shell
kubectl -n work logs -f hostpath-reader
```
Вывод будет примерно такой
```text
Writer say hello at Sat Jun  6 22:14:49 UTC 2026
Writer say hello at Sat Jun  6 22:14:54 UTC 2026
Writer say hello at Sat Jun  6 22:14:59 UTC 2026
```
Это подтверждает, что "читатель" видит данные, которые записывает "писатель" в общую директорию на хосте.
  
Удалим поды и очистим данные:
```shell
# Сначала узнаем, на каком узле работал под-писатель
NODE_NAME=$(kubectl -n work get pod hostpath-writer -o jsonpath='{.spec.nodeName}')

# Удаляем поды
kubectl -n work delete -f 13_host_path_persist_volume/38-hostpath-share.yaml --force

# Удаляем директорию с данными на узле
ssh "root@${NODE_NAME}" -- sudo rm -rf /tmp/k8s-book-shared && \
  echo "Cleaned up data on $NODE_NAME"
```
### Риски и ограничения hostPath
***
Использование `hostPath` сопряжено со значительными рисками, которые необходимо понимать.
  
##### Потеря данных при перезапуске пода на другом узле
Данные, сохраненные через `hostPath`, привязаны к конкретному узлу. Если узел выходит из строя или по какой-либо причине
переезжает на другой узел, данные будут утеряны.
  
##### Проблемы безопасности
`hostPath` предоставляет контейнеру прямой доступ к файловой системе узла. Это **серьезная угроза безопасности**, и его
использование должно быть строго ограничено.
1. **Доступ к системным файлам:** Вредоносное или скомпрометированное приложение может получить доступ к чувствительным данным 
на хосте, например, к `/etc/shadow`.
2. **Изменение конфигурации узла:** Если смонтировать системные директории в режиме `read-write`, приложение сможет изменить
конфигурацию узла, что повлияет на все остальные поды.
3. **Доступ к Docker-сокету:** Монтирование `/var/run/docker.sock` позволяет контейнеру управлять Docker-демоном на хосте, 
что равносильно получению root-доступа ко всему узлу.
  
**Пример: Побег из контейнера через /proc**
  
Рассмотрим самый опасный сценарий. Создадим pod который запускает от имени `root` и монтирует директорию `/proc` с хост-машины,
как описано в манифесте `13_host_path_persist_volume/39-hostpath-security-risk.yaml`. Файловая система `/proc` в Linux 
предоставляет интерфейс к структурам данных ядра, и ее монтирование дает огромные возможности для атаки.
```yaml
# ВНИМАНИЕ: этот манифест демонстрирует КРАЙНЕ небезопасную конфигурацию
# НЕ ИСПОЛЬЗУЙТЕ его в проде
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-security-risk
spec:
  containers:
    - name: main
      image: busybox:1.37
      command: ["/bin/sh", "-c", "sleep 1h"]
      securityContext:
        # Запускаем контейнер от имени root
        runAsUser: 0
      volumeMounts:
        - name: host-proc
          mountPath: /host-proc
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
  volumes:
    - name: host-proc
      hostPath:
        path: /proc
        type: Directory
```
Применим манифест:
```shell
kubectl -n work apply -f 13_host_path_persist_volume/39-hostpath-security-risk.yaml
```
После запуска такого пода злоумышленник, получивший доступ к контейнеру, может выполнить следующие действия:
```shell
# Заходим в контейнер
kubectl -n work exec -ti hostpath-security-risk -- /bin/sh

# Внутри контейнера
# Получаем доступ к процессам хоста
ls -l /host-proc

# Это даст нам список все процессов, запущенных на УЗЛЕ, а не внутри контейнера.
# Зная это, можно, например, попытаться внедрить код в другой процесс
# или прочитать из памяти чувствительные данные.
```
Этот пример наглядно показывает, как `hostPath` в сочетании с высокими привилегиями (`runAsUser: 0`) полностью нарушает изоляцию
контейнера и создает критическую уязвимость в безопасности кластера.
  
Удаляем под:
```shell
kubectl -n work delete -f 13_host_path_persist_volume/39-hostpath-security-risk.yaml --force
```
## Защита с помощью Admission Controllers

Чтобы предотвратить развертывание опасных конфигураций, администраторы кластера могут использовать 
**Admission Controllers** - специальные компоненты Kubernetes, которые перехватывают запросы к API-серверу до того, 
как объект будет сохранён в etcd.
  
Современные инструменты, такие как **Kyverno** или **OPA Gatekeeper**, 
позволяют определять гибкие политики безопасности. Например, можно создать политику, которая полностью 
запрещает использование `hostPath` или разрешает его только для заранее одобренных путей.
  
Пример политики Kyverno, которая запрещает любые `hostPath`-тома, кроме тех, что указывают на `/var/log/my-app`:
  
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-hostpath-volumes
spec:
  validationFailureAction: Enforce
  rules:
    - name: restrict-hostpath
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Использование hostPath запрещено, кроме разрешенного пути /var/log/my-app."
        pattern:
          spec:
            =(volumes):
              - =(hostPath):
                  path: "/var/log/my-app"
```
Такая политика автоматически отклонит любой Pod, который пытается смонтировать, например, `/proc` или `/etc`, 
тем самым защищая кластер на уровне API.

## Рекомендации по использованию hostPath
***
Несмотря на все недостатки, `hostPath` может быть полезен в строго определённых сценариях:
- **Системные агенты** - для приложений, которым по своей природе нужен доступ к файлам хоста. Например:
    - агенты мониторинга (`node-exporter`);
    - сборщики логов (`Fluentd`, `Logstash`);
    - агенты безопасности.

- **Одноузловые кластеры** - в средах разработки и тестирования (`Minikube`, `kind`), где есть только один узел, `hostPath` может быть простым способом хранения данных.
- **Инициализация узла** - для задач, которые должны выполнить одноразовую настройку непосредственно на узле.
  
### Рекомендации по безопасности
При использовании `hostPath` всегда придерживайтесь следующих правил:
- Используйте `nodeSelector` или `nodeAffinity` - жёстко привязывайте Pod к конкретному узлу, чтобы избежать проблем с доступностью данных.
- Используйте `readOnly: true` - если приложению нужен только доступ на чтение, явно указывайте это.
- Ограничивайте доступ - никогда не монтируйте корневую директорию (`/`) или критически важные системные каталоги. Используйте максимально узкие и конкретные пути.
  
### Когда hostPath не подходит
Для большинства задач, требующих постоянного хранения данных, рекомендуется использовать более надёжные и гибкие решения:  
- `PersistentVolume` (PV)  
- `PersistentVolumeClaim` (PVC)  
в связке с сетевыми хранилищами:
- NFS
- Ceph
- облачные диски (AWS EBS, GCE PD, Azure Disk и др.)  
Такие решения обеспечивают переносимость данных между узлами, отказоустойчивость и более безопасное управление хранилищем.