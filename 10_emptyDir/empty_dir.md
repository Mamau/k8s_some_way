## EmptyDir
***
Тома типа `emptyDir` используются для предоставления временного хранилища поду. Они создаются при запуске пода 
и удаляются при его завершении. Данные в таких томах доступны всем контейнерам подам могут использоваться для обмена 
информацией между ними.  
  
### Основные характеристики EmptyDir.
***
`EmptyDir` - это том Kubernetes, который:
- Создается при запуске пода и удаляется при его завершении.
- Изначально пустой (отсюда и название).
- Доступен всем контейнерам в поде.
- Может использовать различные типы носителей (диск, SSD, память).
- Не сохраняет данные после удаления пода.
  
Физически файлы томов типа emptyDir хранятся на диске узла, где запущен под. Путь к этим файлам обычно находится 
в директории `/var/lib/kubelet/pods/<pod_uid>/volumes/kubernetes.io~empty-dir/`. Эти тома создаются `kubelet`.  
  
### Типичные сценарии использования (Use Cases)
***
`emptyDir` идеально подходит для следующих задач:
- **Обмен данными между контейнерами:** Как показано в примере ниже, `emptyDir` является простым и эффективным способом 
для организации обмена файлами между несколькими контейнерами в одном поде.
- **Временное хранилище:** Для приложений, которым нужно временное пространство для кеширования, хранения промежуточных результатов 
вычислений или логов, которые не требуется сохранять надолго.
- **Подготовка данных с помощью Init Containers:** `InitContainer` может сгенерировать или загрузить конфигурационные файлы, 
скрипты или другие данные в `emptyDir`, которые затем будут использоваться основным контейнером.
  
### Создание EmptyDir
***
EmptyDir создается при определении пода в спецификации `volumes`. Пример простого использования:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: nginx
      volumeMounts:
        - name: temp-storage
          mountPath: /tmp/data
  volumes:
    - name: temp-storage
      emptyDir: {}
```
### Использование EmptyDir с различными типами носителей
***
EmptyDir может использовать различные типы носителей, которые определяются в поле `medium`:  
1. **По умолчанию (пустое значение)** - используется диск узла. Это наиболее распространенный вариант.
2. `Memory` - использует `tmpfs` (файловая система в памяти). Это значительно ускоряет операции с файлами, но следует помнить,
что потребляет оперативную память узла. Используйте этот вариант для небольших объемов чувствительных к скорости доступа данных, 
например, для кеша.
  
Пример использования `tmpfs`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-memory-pod
spec:
  containers:
    - name: my-container
      image: nginx
      volumeMounts:
        - name: memory-storage
          mountPath: /tmp/memory-data
  volumes:
    - name: memory-storage
      emptyDir:
        medium: Memory
```
> **Внимание:** при использовании `medium: Memory` том будет потреблять оперативную память узла. Если лимиты контейнера на память 
> будут превышены, это может привести к его завершению (OOMKilled).

### Управление размером EmptyDir
***
Начиная с Kubernetes 1.22, можно ограничить размер `emptyDir` с помощью поля `sizeLimit`. Это помогает предотвратить переполнение диска 
узла из-за неконтролируемого роста временных файлов.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-sized-pod
spec:
  containers:
    - name: my-container
      image: busybox
      command:
        - /bin/sh
        - -c
        - dd if=/dev/zero of=/data/largefile bs=1M count=100; sleep 3600
      volumeMounts:
        - name: data-storage
          mountPath: /data
  volumes:
    - name: data-storage
      emptyDir:
        sizeLimit: 50Mi
```
В этом примере если контейнер попытается записать в `/data` больше 50 мебибайт, под будет выселен (evicted) с узла.

### Пример использования EmptyDir для обмена данными между контейнерами
***
`emptyDir` идеально подходит для обмена данными между несколькими контейнерами в одном поде. В этом примере один контейнер
(producer) будет записывать данные в `emptyDir`, а другой (consumer) - читать их.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-share-pod
  labels:
    app.kubernetes.io/name: emptydir-share-pod
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  volumes:
    - name: shared-volume
      emptyDir: {}
  containers:
    - name: producer
      image: busybox
      command:
        - "/bin/sh"
        - "-c"
      args:
      - |
        while true; do
          echo "Data generated at $(date)" >> /shared-data/data.txt;
          sleep 5;
        done
      volumeMounts:
        - name: shared-volume
          mountPath: /shared-data
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
    - name: consumer
      image: busybox
      command:
        - "/bin/sh"
        - "-c"
      args:
        - |
          tail -f /shared-data/data.txt
      volumeMounts:
        - name: shared-volume
          mountPath: /shared-data
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```
Теперь применим манифест:
```shell
kubectl -n work apply -f 10_emptyDir/28-emptydit-share.yaml
```
Проверим, что под запустился:
```shell
kubectl -n work get pods
```
```text
NAME                 READY   STATUS    RESTARTS   AGE
emptydir-share-pod   2/2     Running   0          40s
```
Теперь можно посмотреть логи контейнера-потребителя, чтобы убедиться, что он читает данные, которые генерирует контейнер-производитель:
```shell
kubectl -n work logs -f emptydir-share-pod -c consumer
```
Результат будет выглядеть примерно так, с новыми строками, появляющимися каждые 5 секунд:
```text
Data generated at Sat May  2 19:32:47 UTC 2026
Data generated at Sat May  2 19:32:52 UTC 2026
Data generated at Sat May  2 19:32:57 UTC 2026
Data generated at Sat May  2 19:33:02 UTC 2026
Data generated at Sat May  2 19:33:07 UTC 2026
```
### Как тома emptyDir отображаются на файловую систему сервера
Посмотрим на "физическую" реализацию `emptyDir` в файловой системе дерева. Для этого нам необходимо узнать:
- Сервер на котором запущен под.
- UID пода.
```shell
kubectl -n work get pod emptydir-share-pod -o custom-columns=NAME:.metadata.name,UID:.metadata.uid,NODE:.spec.nodeName
```
```text
NAME                 UID                                    NODE
emptydir-share-pod   73a8f5d0-9a91-47e1-a39e-6ecd33cf8561   ubuntu-4gb-hel1-30-node-3
```
Как узнать какие параметры и по какому пути находятся в манифесте? Ответ простой: смотрим манифест непосредственно в базе 
kubernetes:
```shell
kubectl -n work get pod emptydir-share-pod -o yaml
```
или так
```shell
kubectl -n work get pod emptydir-share-pod -o yaml | egrep 'uid|nodeName'
```
Заходим на сервер и смотрим содержимое директории:
```shell
ls -l /var/lib/kubelet/pods/73a8f5d0-9a91-47e1-a39e-6ecd33cf8561/volumes/kubernetes.io~empty-dir/shared-volume/
```
```text
total 44
-rw-r--r-- 1 root root 40091 May  2 20:28 data.txt
```
Смотрим содержимое файла data.txt
```shell
tail -2 /var/lib/kubelet/pods/73a8f5d0-9a91-47e1-a39e-6ecd33cf8561/volumes/kubernetes.io~empty-dir/shared-volume/data.txt
```
```text
Data generated at Sat May  2 20:30:39 UTC 2026
Data generated at Sat May  2 20:30:44 UTC 2026
```
Удалим под (удаляться будет какоето время потомучто файловые операции идут)
```shell
kubectl -n work delete pod emptydir-share-pod
```
```text
pod "emptydir-share-pod" deleted
```
### Пример использования EmptyDir с InitContainer
***
Следующий пример демонстрирует, как `InitContainer` может подготовить конфигурационные файлы для основного контейнера, 
используя `emptyDir` в качестве общего хранилища.  
  
Сначала создадим `ConfigMap`, который содержит шаблон конфигурации. Файл `10_emptyDir/29-emptydir-initc-cf.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    app.kubernetes.io/name: app-config
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
data:
  config-template.txt: |
    APP_NAME={{APP_NAME}}
    APP_VERSION={{APP_VERSION}}
    DATABASE_URL={{DATABASE_URL}}
```
Применим манифест для создания `ConfigMap` в кластере
```shell
kubectl -n work apply -f 10_emptyDir/29-emptydir-initc-cf.yaml
```
Теперь создадим `Deployment`. `InitContainer` (`config-init`) прочитает шаблон из `ConfigMap`, заменить переменные-плейсхолдеры на 
реальные значения из переменных окружения и запишет итоговый файл в `emptyDir` том. Основной контейнер (`app-container`) затем 
сможет использовать этот готовый файл.  
  
Манифест файл (10_emptyDir/30-emptydir-initc-dep.yaml)
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app.kubernetes.io/name: &name app-deployment
    app.kubernetes.io/instance: &instance app-deployment
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 1
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
      volumes:
        - name: config-template
          configMap:
            name: app-config
        - name: app-config
          emptyDir: {}
      initContainers:
        - name: config-init
          image: busybox
          command:
            - sh
            - -c
            - |
              sed -e "s|{{APP_NAME}}|$APP_NAME|g" \
                  -e "s|{{APP_VERSION}}|$APP_VERSION|g" \
                  -e "s|{{DATABASE_URL}}|$DATABASE_URL|g" \
              /config-template/config-template.txt > /app-config/app-config.txt
              echo "Done"
          env:
            - name: APP_NAME
              value: "MyApplication"
            - name: APP_VERSION
              value: "1.0.0"
            - name: DATABASE_URL
              value: "postgresql://user:password@db:5432/mydb"
          volumeMounts:
            - name: config-template
              mountPath: /config-template
            - name: app-config
              mountPath: /app-config
      containers:
        - name: app-container
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
            - name: app-config
              mountPath: /etc/app-config
              readOnly: true
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```
Применим манифест
```shell
kubectl -n work apply -f 10_emptyDir/30-emptydir-initc-dep.yaml
```
```text
deployment.apps/app-deployment created
```
Проверим
```shell
kubectl -n work get pods -l app.kubernetes.io/name=app-deployment
```
Когда под перейдет в статус `Running`, это будет означать, что `InitContainer` успешно завершил свою работу. Теперь можно проверить 
содержимое сгенерированного файла внутри основного контейнера.
```shell
kubectl -n work exec -ti \
  $(kubectl -n work get pods -l app.kubernetes.io/name=app-deployment -o jsonpath='{.items[0].metadata.name}') \
  -c app-container -- cat /etc/app-config/app-config.txt
```
Результат - видим переменные которые записаны в app-config.txt
```text
APP_NAME=MyApplication
APP_VERSION=1.0.0
DATABASE_URL=postgresql://user:password@db:5432/mydb
```
Удаляем созданные ресурсы
```shell
kubectl -n work delete deployment app-deployment
kubectl -n work delete configmap app-config
```
#### Ограничения
Несмотря на свою простоту и удобство, `emptyDir` имеют важные ограничения:
- **Данные не сохраняются:** Это самое главное ограничение. При удалении, перезапуске или сбое пода все данные в `emptyDir` 
будут безвозвратно утеряны.
- **Привязка к узлу:** Том `emptyDir` существует только на том узле, где запущен под. Если под будет перенесен на другой узел, 
данные не переместятся вместе с ним.