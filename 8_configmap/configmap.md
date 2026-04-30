## Kubernetes volumes
***
### Контейнеры и тома
***
Приложение в контейнере может создавать файлы в файловой системе. При создании такого файла он физически будет находиться 
на машине где был запущен контейнер. Например при использовании containerd они находятся в директории `/var/lib/containerd/`.
  
При удалении контейнера и создании нового, эти файлы будут удаляться.  
  
На самом деле создавать файлы внутри файловой системы контейнеров не самая разумная идея. Поэтому, если вы планируете 
работу с файлами - подключайте внешние тома к контейнеру.  
  
Пример подключения тома к контейнеру при помощи `ctr` (обычно входит в состав пакета containerd):  
```shell
# Создаем директорию на хосте
mkdir -p /host/data

# Запускаем контейнер с подключенным томом
ctr run --rm -t \
  --mount type=bind,src=/host/data,dst=/data,options=rbind:rw \
  docker.io/library/alpine:latest mycontainer sh
```
### Тома в kubernetes
***
При запуске контейнера в kubernetes тоже настоятельно не рекомендуется создавать файлы внутри файловой системы контейнера. 
Если необходимо сохранить файлы - используйте тома, подключаемые к контейнеру в поде.  
  
Если посмотреть документацию по томам в Kubernetes, то вы увидите большое количество типов томов. Их условно можно разделить на категории:
- Ephemeral volume - тома удаляемые после завершения работы пода.
- Persistent volume - тома доступные после удаления пода.
  
#### Ephemeral volume
Ephemeral тома создаются kubelet при запуске контейнеров пода. Если том находится на диске, он будет расположен где то в директории 
`/var/lib/kubelet/`. Поэтому важно знать, какие по размеру тома планирует использовать приложение в контейнере, что бы заранее выделить 
необходимое место на диске.
  
## ConfigMap
***
Тома типа configmap обычно используются для подключения конфигурационных файлов к контейнеру подов. Или для формирования переменных среды 
окружения приложения в контейнере.  
  
### Переменные среды окружения
***
#### env
Предположим, что в контейнер необходимо передать значительное количество переменных среды окружения. Это можно сделать непосредственно в 
манифесте пода, Deployment и т.п в разделе `env` шаблона контейнера.  
  
Файл `8_configmap/17-env.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  namespace: work
  labels:
    app: &app test
    version: &version v1.27.5
spec:
  replicas: 1
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
        - name: main
          image: busybox:1.28
          command:
            - 'sh'
            - '-c'
            - 'tail -f /dev/null'
          resources:
            requests:
              memory: 100Mi
              cpu: 100m
            limits:
              memory: 100Mi
              cpu: 100m
          env:
            - name: TEST_ENV1
              value: "test1"
            - name: TEST_ENV2
              value: "test2"
```
Запустим приложение:
```shell
kubectl -n work apply -f 8_configmap/17-env.yaml
```
```text
deployment.apps/test created
```
Убедимся, что добавленные переменные среды окружения были определены в контейнере
```shell
kubectl -n work get pods
```
```text
NAME                    READY   STATUS    RESTARTS   AGE
test-54f7c88ccd-j8mkk   1/1     Running   0          77m
```
Выполним команду в контейнере
```shell
kubectl -n work exec test-54f7c88ccd-j8mkk -- sh -c 'env | grep TEST'
```
```text
TEST_ENV1=test1
TEST_ENV2=test2
```
Удалим Deployment
```shell
kubectl -n work delete -f 8_configmap/17-env.yaml
```
  
#### envFrom
Создадим `ConfigMap` содержащий набор переменных среды окружения (файл `8_configmap/18-env-configmap.yaml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  TEST_DATABASE_HOST: "postgresql.example.com"
  TEST_DATABASE_PORT: "5432"
  TEST_APP_ENV: "production"
  TEST_LOG_LEVEL: "info"
```
Как видите, определение переменных в `ConfigMap` достаточно простое.
  
Добавим `ConfigMap` в Kubernetes:
```shell
kubectl -n work apply -f 8_configmap/18-env-configmap.yaml
```
Посмотри, какие `ConfigMap` есть в `namespace` work:
```shell
kubectl -n work get cm
```
```text
NAME               DATA   AGE
app-config         4      57s
kube-root-ca.crt   1      15d
```
`ConfigMap` `kube-root-ca.crt` добавляется автоматически при создании `namespace`. Нас интересует `app-config`.  
Столбец `DATA` показывает, сколько элементов находится в `ConfigMap`.  
  
Модифицируем манифест `Deployment`, добавив в него `envFrom` (файл `8_configmap/19-env-configmap-dp1.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  namespace: work
  labels:
    app: &app test
    version: &version v1.27.5
spec:
  replicas: 1
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
        - name: main
          image: busybox:1.28
          command:
            - 'sh'
            - '-c'
            - 'tail -f /dev/null'
          resources:
            requests:
              memory: 100Mi
              cpu: 100m
            limits:
              memory: 100Mi
              cpu: 100m
          env:
            - name: TEST_ENV1
              value: "test1"
            - name: TEST_ENV2
              value: "test2"
          envFrom:
            - configMapRef:
                name: app-config
```
Параметр `envFrom` позволяет получить **все** переменные среды окружения из `ConfigMap`.
  
Применим манифест:
```shell
kubectl -n work apply -f 8_configmap/19-env-configmap-dp1.yaml
```
```shell
kubectl -n work get pods
```
```text
NAME                    READY   STATUS    RESTARTS   AGE
test-5747d86f8c-rtfrk   1/1     Running   0          45s
```
Посмотрим, какие переменные среды окружения в контейнере:
```shell
kubectl -n work exec test-5747d86f8c-rtfrk -- sh -c 'env | grep TEST'
```
```text
TEST_DATABASE_HOST=postgresql.example.com
TEST_APP_ENV=production
TEST_ENV1=test1
TEST_ENV2=test2
TEST_DATABASE_PORT=5432
TEST_LOG_LEVEL=info
```
Удалим `Deployment`
```shell
kubectl -n work delete -f 8_configmap/19-env-configmap-dp1.yaml
```
  
#### env и valueFrom
Бывают ситуации когда в контейнер необходимо добавить только несколько переменных среды окружения из `ConfigMap`. 
В следующем примере мы добавим только две переменные, находящиеся в `ConfigMap` (файл `8_configmap/20-env-configmap-dp2.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  namespace: work
  labels:
    app: &app test
    version: &version v1.27.5
spec:
  replicas: 1
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
        - name: main
          image: busybox:1.28
          command:
            - 'sh'
            - '-c'
            - 'tail -f /dev/null'
          resources:
            requests:
              memory: 100Mi
              cpu: 100m
            limits:
              memory: 100Mi
              cpu: 100m
          env:
            - name: TEST_ENV1
              value: "test1"
            - name: TEST_ENV2
              value: "test2"
            - name: TEST_APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: TEST_APP_ENV
            - name: TEST_LOG_LEVEL_CURRENT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: TEST_LOG_LEVEL
```
В этот раз мы используем стандартное `env`. Но вместо `value` используем `valueFrom` в котором определяем из какого `ConfigMap` 
выбрать конкретное содержимое.  
  
В случае переменной TEST_LOG_LEVEL_CURRENT мы видим, что ее имя не обязательно должно совпадать с именем переменной в `ConfigMap`.
  
Применим манифест:
```shell
kubectl -n work apply -f 8_configmap/20-env-configmap-dp2.yaml
```
```shell
kubectl -n work get pods
```
```text
NAME                    READY   STATUS    RESTARTS   AGE
test-5d5bf5f9b4-t8b6m   1/1     Running   0          6s
```
Посмотрим, какие переменные среды окружения в контейнере:
```shell
kubectl -n work exec test-5d5bf5f9b4-t8b6m -- sh -c 'env | grep TEST'
```
```text
TEST_APP_ENV=production
TEST_ENV1=test1
TEST_ENV2=test2
TEST_LOG_LEVEL_CURRENT=info
```
Удалим `Deployment` и `ConfigMap`
```shell
kubectl -n work delete -f 8_configmap/20-env-configmap-dp2.yaml -f 8_configmap/18-env-configmap.yaml
```
```text
deployment.apps "test" deleted
configmap "app-config" deleted
```
#### Пример ConfigMap с текстовыми файлами
  
Создадим `ConfigMap` с двумя текстовыми файлами (файл `8_configmap/21-file-configmap.yaml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true
data:
  file1.txt: |
    Текстовый файл 1
    и его
    содержимое
  file2.txt: |
    Текстовый файл 2
    и его
    содержимое
```
`immutable` - поле которое запрещает менять `ConfigMap` на лету. Нельзя будет изменить его содержимое.  
Чтобы его поменять надо будет сперва удалить `ConfigMap` и заново накатить измененный.
  
Применим манифест:
```shell
kubectl -n work apply -f 8_configmap/21-file-configmap.yaml
```
```shell
kubectl -n work get cm
```
```text
NAME               DATA   AGE
app-config         2      21s
kube-root-ca.crt   1      16d
```
Посмотрим содержимое `ConfigMap`:
```shell
kubectl -n work get cm app-config -o jsonpath='{.data}'
```
```text
{"file1.txt":"Текстовый файл 1\nи его\nсодержимое\n","file2.txt":"Текстовый файл 2\nи его\nсодержимое\n"}
```
#### Подключение ConfigMap как volume
  
В отличии от переменных среды окружения, подключение `ConfigMap` как том в контейнер трейбует другой подход. 
В поде добавляют определение тома, в котором в качестве источника данных указывается `ConfigMap`. В контейнерах этот том 
монтируется уже в файловую систему контейнера.  
  
Пример Deployment (файл `8_configmap/22-file-configmap-dp1.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  namespace: work
  labels:
    app: &app test
    version: &version v1.27.5
spec:
  replicas: 1
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
      volumes:
        - name: config-volume
          configMap:
            name: app-config
      containers:
        - name: main
          image: busybox:1.28
          command:
            - 'sh'
            - '-c'
            - 'tail -f /dev/null'
          resources:
            requests:
              memory: 100Mi
              cpu: 100m
            limits:
              memory: 100Mi
              cpu: 100m
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
```
В шаблоне пода мы определяем том. Данные для тома берутся из `ConfigMap`.
```yaml
spec:
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```
Затем, непосредственно в определении контейнера мы этот том монтируем в файловую систему контейнера к директории `/etc/config`:
```yaml
containers:
  - name: main
    volumeMounts:
      - name: config-volume
        mountPath: /etc/config
```
Директория `/etc/config` создается автоматически. Все файлы, находящиеся в `ConfigMap` будут помещены в эту директорию.  
  
Применим манифест:
```shell
kubectl -n work apply -f 8_configmap/22-file-configmap-dp1.yaml
```
```shell
kubectl -n work get pods
```
```text
NAME                    READY   STATUS    RESTARTS   AGE
test-6d8d5d87b7-45ghw   1/1     Running   0          24s
```
Посмотрим содержимое директории `/etc/config` в контейнере:
```shell
kubectl -n work exec test-6d8d5d87b7-45ghw -- sh -c 'ls -la /etc/config'
```
```text
drwxrwxrwx    3 root     root          4096 Apr 29 21:05 .
drwxr-xr-x    1 root     root          4096 Apr 29 21:05 ..
drwxr-xr-x    2 root     root          4096 Apr 29 21:05 ..2026_04_29_21_05_52.3659578175
lrwxrwxrwx    1 root     root            32 Apr 29 21:05 ..data -> ..2026_04_29_21_05_52.3659578175
lrwxrwxrwx    1 root     root            16 Apr 29 21:05 file1.txt -> ..data/file1.txt
lrwxrwxrwx    1 root     root            16 Apr 29 21:05 file2.txt -> ..data/file2.txt
```
Видим, маленькие хитрости реализации монтирования файлов из `ConfigMap`. Много символьных ссылок. 
Посмотрим содержимое реальной директории:
```shell
kubectl -n work exec test-6d8d5d87b7-45ghw -- sh -c 'ls -la /etc/config/..2026_04_29_21_05_52.3659578175'
```
```text
drwxr-xr-x    2 root     root          4096 Apr 29 21:05 .
drwxrwxrwx    3 root     root          4096 Apr 29 21:05 ..
-rw-r--r--    1 root     root            61 Apr 29 21:05 file1.txt
-rw-r--r--    1 root     root            61 Apr 29 21:05 file2.txt
```
#### Изменение содержимого ConfigMap
Система позволяет изменить содержимое `ConfigMap`. Что будет с данными в контейнере с уже подключенным `ConfigMap`? 
Посмотрим содержимое файла `file1.txt`:
```shell
kubectl -n work exec test-6d8d5d87b7-45ghw -- sh -c 'cat /etc/config/file1.txt'
```
```text
Текстовый файл 1
и его
содержимое
```
Измени содержимое `ConfigMap` (файл 8_configmap/23-file-configmap.yaml)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
#immutable: true
data:
  file1.txt: |
    Текстовый файл 1
    и его
    новое содержимое
  file2.txt: |
    Текстовый файл 2
    и его
    новое содержимое
```
Изменено содержимое файла `file1.txt`  
  
Обновим наш манифест
```shell
kubectl -n work apply -f 8_configmap/23-file-configmap.yaml
```
Убедимся, что содержимое файла `file1.txt` изменилось.
```shell
kubectl -n work get cm app-config -o jsonpath='{.data.file1\.txt}'
```
```text
Текстовый файл 1
и его
новое содержимое
```
Через некоторое время посмотрим содержимое файла в контейнере:
```shell
kubectl -n work exec test-6d8d5d87b7-45ghw -- sh -c 'cat /etc/config/file1.txt'
```
```text
Текстовый файл 1
и его
новое содержимое
```
Удалим `Deployment`:
```shell
kubectl -n work delete -f 8_configmap/22-file-configmap-dp1.yaml
```
```text
deployment.apps "test" deleted
```
  
#### Подключение одного файла
Рассмотрим ситуацию когда нужно подключить только один файл из `ConfigMap` в директорию в которой уже есть другие файлы. 
Пример манифеста `Deployment` (файл 8_configmap/24-file-configmap-dp2.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  namespace: work
  labels:
    app: &app test
    version: &version v1.27.5
spec:
  replicas: 1
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
      volumes:
        - name: config-volume
          configMap:
            name: app-config
      containers:
        - name: main
          image: busybox:1.28
          command:
            - 'sh'
            - '-c'
            - 'tail -f /dev/null'
          resources:
            requests:
              memory: 100Mi
              cpu: 100m
            limits:
              memory: 100Mi
              cpu: 100m
          volumeMounts:
            - name: config-volume
              mountPath: /etc/file1.txt
              subPath: file1.txt
```
Для подключения только одного файла, без создании директории точки монтирования, будем ипользовать параметр `subPath`. 
В котором мы указываем файл в `ConfigMap`, а в `mountPath` пишем имя файла, на которое он будет отображен.
```yaml
containers:
  - name: main
    volumeMounts:
      - name: config-volume
        mountPath: /etc/subfile.txt
        subPath: file1.txt
```
Примени манифест:
```shell
kubectl -n work apply -f 8_configmap/24-file-configmap-dp2.yaml
```
```shell
kubectl -n work get pods
```
```text
NAME                   READY   STATUS    RESTARTS   AGE
test-8cbc984f4-kcthb   1/1     Running   0          20s
```
Посмотри содержимое директории `/etc` в контейнере. Посколько файлов там много, будем выбирать только файлы начинающиеся на `s`:
```shell
kubectl -n work exec test-8cbc984f4-kcthb -- sh -c 'ls -l /etc/s*'
```
```text
-rw-------    1 root     root           243 Dec 31  2017 /etc/shadow
-rw-r--r--    1 root     root            72 Apr 29 21:29 /etc/subfile.txt
```
Содержимое файла `/etc/subfile.txt`:
```shell
kubectl -n work exec test-8cbc984f4-kcthb -- sh -c 'cat /etc/subfile.txt'
```
```text
Текстовый файл 1
и его
новое содержимое
```
В режиме `subpath` существует дополнительное ограничение: монтирование таких файлов доступно только в режиме `ro`. 
С другой стороны, редактировать конфигурационные файлы в контейнере - это странное желание. Измененные файлы обратно в `ConfigMap` не отображаются. 
Т.е `ConfigMap` не изменяется и после перезапуска контейнера файлы будут содержать то, что было написано в `ConfigMap`.  
  
Удалим `Deployment` и `ConfigMap`:
```shell
kubectl -n work delete cm app-config && kubectl -n work delete deployment test
```
```text
configmap "app-config" deleted
deployment.apps "test" deleted
```