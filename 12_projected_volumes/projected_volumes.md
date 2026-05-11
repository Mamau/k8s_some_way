## Projected Volumes
***
Механизм projected volume в Kubernetes позволяет монтировать несколько источников данных в одну и ту же директорию внутри контейнера. 
Это инструмент, который позволяет гибко управлять конфигурацией, секретами и метаданными, доступными для приложения.
  
В отличие от монтирования каждого источника как отдельного тома, `projected volume` объединяет их, что упрощает управление файлами 
конфигурации.
  
### Источники данных для projected-томов
***
`Projected` том может включать в себя следующие типы источников:
- `secret` - Позволяет монтировать данные из одного или нескольких Secret.
- `configMap` - Используется для монтирования конфигурационных данных из configMap.
- `downwardAPI` - Подключает DownwardAPI.
- `serviceAccountToken` - Монтирует токен для `ServiceAccount`. Хотя kubernetes по умолчанию автоматически монтирует 
токен в каждый под по пути `/var/run/secrets/kubernetes.io/serviceaccount/token`, использование `projected` тома дает 
дополнительную гибкость. Например, можно настроить аудиторию (`audience`) и срок действия (`expirationSeconds`) токена,
а также смонтировать его в нестандартный путь.  
  
Все источниики в `projected` томе монтируются в одну директорию, указанную в `volumeMounts`. Важно отметить, что данные из 
`ConfigMap`, `Secret` и `downwardAPI` (например метки и аннотации) обновляются автоматически, если они изменяются.
  
### Использование projected-тома
***
Рассмотрим практический пример, в котором мы объединим несколько источников данных в одном `projected` томе. 
Мы создадим `ConfigMap` для конфигурации приложения, `Secret` для учетных данных и используем `downwardAPI` для 
получения метаданных пода.
  
`ConfigMap` с настройками приложения. Файл `12_projected_volumes/34-projected-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    log_level=INFO
    feature_flags=new-ui,dark-mode
```
`Secret` с учетными данными. В манифесте `12_projected_volumes/35-projected-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credential
type: Opaque
stringData:
  username: "admin"
  password: "SuperSecretPassword123"
```
Применим оба манифеста:
```shell
kubectl -n work apply \
  -f 12_projected_volumes/34-projected-configmap.yaml \
  -f 12_projected_volumes/35-projected-secret.yaml
```
```text
configmap/app-config created
secret/db-credential created
```
Создадим под, который будет использовать `projected` том для монтирования данных из `ConfigMap`, `Secret` и `downwardAPI`. 
Файл `12_projected_volumes/36-projected-pod-volume.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-volume-demo
  labels:
    app.kubernetes.io/name: projected-demo
    app.kubernetes.io/instance: test
    app.kubernetes.io/version: v0.0.1
spec:
  volumes:
    - name: all-in-one-volume
      projected:
        sources:
          # Источник 1: DownwardAPI для метаданных пода
          - downwardAPI:
              items:
                - path: "pod_name"
                  fieldRef:
                    fieldPath: metadata.name
                - path: "pod_namespace"
                  fieldRef:
                    fieldPath: metadata.namespace
          # Источник 2: ConfigMap для конфигурации приложения
          - configMap:
              name: app-config # имя configMap
              items:
                - key: app.properties
                  path: app.properties
          # Источник 3: Secret для учетных данных
          - secret:
              name: db-credential # Имя Secret
              items:
                - key: username
                  path: username
                - key: password
                  path: password
  containers:
    - name: main-container
      image: busybox:1.37
      command:
        - "sh"
        - "-c"
      args:
        - |
          echo "--- Reading projected data ---";
          echo "[+] Pod Info:";
          cat /etc/pod-data/pod_name;
          echo "";
          cat /etc/pod-data/pod_namespace;
          echo "";
          echo "[+] App Config:";
          cat /etc/pod-data/app.properties;
          echo "[+] DB Credentials:";
          echo "Username: $(cat /etc/pod-data/username)"
          echo "Password: $(cat /etc/pod-data/password)"
          echo "--- Done ---";
          sleep infinity;
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      volumeMounts:
        - name: all-in-one-volume
          mountPath: /etc/pod-data
          readOnly: true # Рекомендуется для projected томов
  restartPolicy: Never
```
В этом манифесте:  
- Мы определяем том `all-in-one-volume` типа `projected`
- В секции `sources` мы перечисляем все источники данных: `downwardAPI`, `secret` и `configMap`
- Каждый источник указывает, какие данные (`items`) и в какие файлы (`path`) нужно поместить внутри точки монтирования `/etc/pod-data`.
  
Применим манифест
```shell
kubectl -n work apply -f 12_projected_volumes/36-projected-pod-volume.yaml
```
```text
pod/projected-volume-demo created
```
Убедимся что под запущен:
```shell
kubectl -n work get pod projected-volume-demo
```
Посмотрим логи
```shell
kubectl -n work logs projected-volume-demo
```
```text
--- Reading projected data ---
[+] Pod Info:
projected-volume-demo
work
[+] App Config:
log_level=INFO
feature_flags=new-ui,dark-mode
[+] DB Credentials:
Username: admin
Password: SuperSecretPassword123
--- Done ---
```
Удалим:
```shell
kubectl -n work delete pod projected-volume-demo --force
kubectl -n work delete configmap app-config
kubectl -n work delete secret db-credential
```
