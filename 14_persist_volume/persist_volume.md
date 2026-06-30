## PersistentVolume
***

- PersistentVolume
    - Основные характеристики PersistentVolume
        - Жизненный цикл
        - Политика возврата (Reclaim Policy)
        - Режимы доступа (Access Modes)
    - Ручное создание PersistentVolume

`PersistentVolume (PV)` - это абстракция над физическим хранилищем в кластере (NFS, iSCSI, облачные диски и т.д.), которую администратор подготавливает для использования. PV является ресурсом кластера, так же как и узлы (nodes).

# Основные характеристики PersistentVolume

## Жизненный цикл

`PersistentVolume` может находиться в одном из следующих состояний:

- **Available** (Доступен): PV создан и готов к использованию, но еще не связан ни с одним запросом (`PersistentVolumeClaim`).
- **Bound** (Связан): PV связан с PVC, и, следовательно, используется подом.
- **Released** (Освобожден): PVC, с которым был связан PV, удален, но сам ресурс еще не освобожден для повторного использования. Что произойдет с данными и самим PV, определяет политика возврата.
- **Failed** (Сбой): Произошла ошибка, и PV недоступен.

## Политика возврата (Reclaim Policy)

Политика возврата определяет, что произойдет с `PersistentVolume` после того, как `PersistentVolumeClaim`, который его использовал, будет удален.

- **Retain** (Сохранить): После удаления PVC сам PV и данные на нем сохраняются. PV переходит в статус `Released` и требует ручного вмешательства администратора для очистки и повторного использования. Это самая безопасная политика, предотвращающая случайную потерю данных.
- **Delete** (Удалить): После удаления PVC сам PV и связанное с ним физическое хранилище (например, диск в облаке) удаляются. Эта политика используется в основном при динамическом выделении ресурсов.
- **Recycle** (Переиспользовать): *Устарела.* Ранее эта политика позволяла автоматически очищать содержимое тома (`rm -rf /thevolume/*`) и делать его снова доступным. Сейчас рекомендуется использовать динамическое выделение с политикой `Delete`.

## Режимы доступа (Access Modes)

Режимы доступа определяют, как поды могут обращаться к тому. Один `PV` может поддерживать несколько режимов доступа.

- **ReadWriteOnce (RWO)**: Том может быть смонтирован для чтения и записи только одним узлом. Это не значит, что его может использовать только один под: несколько подов на одном узле могут использовать этот том.
- **ReadOnlyMany (ROX)**: Том может быть смонтирован в режиме "только для чтения" многими узлами одновременно.
- **ReadWriteMany (RWX)**: Том может быть смонтирован для чтения и записи многими узлами одновременно. Этот режим поддерживается не всеми типами хранилищ (например, его поддерживает NFS).
- **ReadWriteOncePod (RWOP)**: Том может быть смонтирован для чтения и записи только одним подом в кластере.

# Ручное создание PersistentVolume
***
Чаще всего `PV` создаются динамически с помощью `StorageClass`, но для понимания концепции полезно рассмотреть процесс ручного создания. 
Администратор кластера может заранее подготовить несколько `PV`, которые разработчики затем смогут запрашивать через `PVC`.

Создадим `PersistentVolume` размером 1Gi с режимом доступа `ReadWriteMany`.

Файл `14_persist_volume/40-nfs-pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-01
  labels:
    # Метки используются для более гибкого связывания с PVC
    app.kubernetes.io/name: nfs-storage
    app.kubernetes.io/instance: manual

spec:
  # Объем хранилища, доступный в этом томе.
  capacity:
    storage: 1Gi

  # Режимы доступа, которые поддерживает этот том.
  accessModes:
    - ReadWriteMany

  # Политика возврата.
  persistentVolumeReclaimPolicy: Retain

  # Тип тома. В данном случае - NFS.
  nfs:
    # Путь к директории на NFS-сервере.
    path: /volume1/nfs

    # Адрес NFS-сервера.
    server: 192.168.218.13
```
> Обратите внимание на то, что PV - ресурс не зависящий от namespace. Он доступен приложениям из любых namespaces.

В примере используется ключ `spec.nfs`. В определении `PersistentVolume` поддерживается большое количество файловых систем. 
Подробнее про поддерживаемые типы и их параметры можно посмотреть в документации Kubernetes API.

Применим манифест:

```bash
kubectl apply -f 14_persist_volume/40-nfs-pv.yaml
```
Проверим статус созданного `PV`:
```shell
kubectl get pv
```
```text
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
nfs-pv-01   1Gi        RWX            Retain           Available                          <unset>                          40s
```

# PersistentVolumeClaim
***
- PersistentVolumeClaim
    - Механизм связывания PV и PVC
    - Создание PVC и его использование в поде
        - Использование PVC в поде

> Если `PersistentVolume` - это ресурс в кластере, то `PersistentVolumeClaim` (PVC) - это запрос на использование этого ресурса со стороны пользователя или приложения. `PVC` позволяет запрашивать хранилище с определенными характеристиками (размер, режим доступа), не вникая в детали его реализации.

Это разделение ролей - ключевая идея персистентного хранения в Kubernetes:

- **Администратор** настраивает физические хранилища и предоставляет их кластеру в виде `PV`.
- **Клиент (разработчик)** запрашивает необходимые объемы хранения для своих приложений с помощью `PVC`.

# Механизм связывания PV и PVC
***
Когда разработчик создает `PVC`, Kubernetes ищет подходящий `PV`, который может удовлетворить этот запрос. Для того чтобы `PV` и `PVC` были связаны, должны совпасть следующие параметры:

1. `accessModes`: `PV` должен поддерживать режим доступа, запрошенный в `PVC`.
2. `storage`: Размер `PV` должен быть не меньше, чем размер, запрошенный в `PVC`.
3. `storageClassName`: `PVC` и `PV` должны принадлежать одному и тому же `StorageClass`. Если `storageClassName` не указан, то используются `PV` без `StorageClass`.
4. `selector`: `PVC` может использовать `matchLabels` для выбора `PV` с определенными метками.

Как только Kubernetes находит подходящий `PV`, он связывает их, и оба объекта переходят в статус `Bound`. После этого `PVC` можно использовать в поде как обычный том.

# Создание PVC и его использование в поде
***
Продолжим наш предыдущий пример. У нас есть `PV` `nfs-pv-01` в статусе `Available`. 
Теперь создадим `PVC`, чтобы запросить это хранилище.

Файл `14_persist_volume/41-nfs-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-01
  labels:
    app.kubernetes.io/name: app-config
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1

spec:
  # Запрашиваем те же режимы доступа, что и у нашего PV.
  accessModes:
    - ReadWriteMany

  # Запрашиваем 500 мегабайт. Это меньше, чем 1Gi, доступный в PV, поэтому он подходит.
  resources:
    requests:
      storage: 500Mi

  # Мы не указываем storageClassName, чтобы Kubernetes искал среди PV без класса.
  # Также можно использовать selector для выбора PV по меткам:
  selector:
    matchLabels:
      app.kubernetes.io/instance: manual
```

Применим манифест:

```bash
kubectl -n work apply -f 14_persist_volume/41-nfs-pvc.yaml
```
Теперь посмотрим на статусы `PV` и `PVC`
```shell
kubectl -n work get pvc,pv
```
```text
NAME                               STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/nfs-pvc-01   Bound    nfs-pv-01   1Gi        RWX                           <unset>                 48s

NAME                         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/nfs-pv-01   1Gi        RWX            Retain           Bound    work/nfs-pvc-01                  <unset>                          14m
```

## Использование PVC в поде

Чтобы использовать хранилище, в манифесте пода нужно определить том, указав в качестве его источника `persistentVolumeClaim`.

Файл `14_persist_volume/42-pod-with-pvc.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-writer
  labels:
    app.kubernetes.io/name: app-writer
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1

spec:
  volumes:
    # Определяем том и указываем, что он должен использовать наш PVC.
    - name: nfs-storage
      persistentVolumeClaim:
        claimName: nfs-pvc-01

  containers:
    - name: main
      image: busybox:1.37
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            echo "Log entry at $(date)" >> /data/app.log;
            sleep 5;
          done

      volumeMounts:
        # Монтируем том в файловую систему контейнера.
        - name: nfs-storage
          mountPath: /data

      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```
Запустим под:  
```shell
kubectl -n work apply -f 14_persist_volume/42-pod-with-pvc.yaml
```
Убелимся, что под работает, и проверим, что данные были записаны в файл:
```shell
kubectl -n work exec app-writer -- cat /data/app.log
```
Данные успешно записаны в персистентное хранилище. Даже если мы удалим под `app-writer` и создадим новый, который будет читать из этого же `PVC`, он увидит этот файл. Это и есть основное преимущество `PersistentVolume`.

Не забудьте удалить созданные ресурсы, чтобы они не мешали в следующих примерах:

```bash
kubectl -n work delete pod app-writer --force
kubectl -n work delete pvc nfs-pvc-01
```
Посмотрим состояние `PV`:
```shell
kubectl get pv
```
```text
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
nfs-pv-01   1Gi        RWX            Retain           Released   work/nfs-pvc-01                  <unset>                          28m
```
Поле `STATUS: Released`. Несмотря на то, что этот PV продолжает находиться в списках (он не удален), пользоваться им нельзя. Посмотрим что будет, если повторно использовать манифест `PVC`, который использовали до этого:

```shell
kubectl -n work apply -f 14_persist_volume/41-nfs-pvc.yaml
```
```shell
kubectl -n work get pvc
```
```text
NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
nfs-pvc-01   Pending                                                     <unset>                 29s
```
PVC находится в состоянии `Pending`. Ждет, когда в кластере появится `PV`, который будет подходить под его требования.

Удалим не нужные `PVC` и `PV`:

```bash
kubectl -n work delete pvc nfs-pvc-01
kubectl delete pv nfs-pv-01
```