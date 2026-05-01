## Secret
***
в Kubernetes часто возникает необходимость передавать в приложения конфигурационные данные. Для неконфиденциальной 
информации, такой как URL-адреса баз данных или ключи API для публичных сервисов, обычно используется объект `ConfigMap`. 
Однако, когда речь заходит о паролях, токенах аутентификации или TLS-ключах, `ConfigMap` не обеспечивает должного уровня 
безопасности, так как хранит данные в открытом виде.  
  
Именно для таких случаев предназначен объект `Secret`. Он позволяет хранить и управлять конфиденциальной информацией, 
предоставляя базовые механизмы защиты и контроля доступа. Хотя по умолчанию `Secret` лишь кодирует данные в `base64`, а не 
шифрует их. Его использование является обязательной практикой для безопасного управления чувствительными данными в кластере.  
  
`Secret` - это специальный объект для конфиденциальных данных. **Но важно помнить**: по умолчанию он лишь кодирует данные 
в base64, а не шифрует их. Думайте об этом как о конверте, а не как о сейфе.  
  
### Основные характеристики Secret
***
Secret - это объект Kubernetes, предназначенный для хранения конфиденциальных данных. Основные особенности:
- Данные хранятся в формате base64 (не является шифрованием).
- Максимальный размер - 1Mb.
- Может быть смонтирован в под как том или использован для определения переменных окружения.
- Поддерживает несколько типов: `Opaque`, `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson` и другие.
  
### Создание Secret
***
Существует несколько способов создания Secret.  
  
#### 1. Из литералов
```shell
kubectl -n work create secret generic my-secret --from-literal=username=admin --from-literal=password=secretpassword
```
`generic` создает `Sceret` типа `Opaque`  
После создания Secret можно проверить командой:
```shell
kubectl -n work get secret
```
Результат:
```text
NAME        TYPE     DATA   AGE
my-secret   Opaque   2      73s
```
Подробно
```shell
kubectl -n work describe secret my-secret
```
```text
Name:         my-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  14 bytes
username:  5 bytes
```
Обратите внимание, что команда `describe` не показывает содержимое `Secret`, только размер данных.  
  
Для получения значений данных из Secret можно использовать следующую команду:  
```shell
kubectl -n work get secret my-secret -o jsonpath='{.data}' | jq
```
```text
{
  "password": "c2VjcmV0cGFzc3dvcmQ=",
  "username": "YWRtaW4="
}
```
Для получения значения конкретного ключа можно сделать так
```shell
kubectl -n work get secret my-secret -o jsonpath='{.data.username}' | base64 --decode
```
```text
admin
```
Удалим
```shell
kubectl -n work delete secret my-secret
```

#### 2. Из файлов
Содержимое `Secret` при его создании можно получать из файлов:
```shell
echo -n 'admin' > ./username.txt
echo -n 'secretpassword' > ./password.txt
kubectl -n work create secret generic my-secret --from-file=./username.txt --from-file=./password.txt
```
Проверим
```shell
kubectl -n work get secrets
```
```text
NAME        TYPE     DATA   AGE
my-secret   Opaque   2      86s
```
Подробно
```shell
kubectl -n work describe secret my-secret
```
```text
Name:         my-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  14 bytes
username.txt:  5 bytes
```
Обратите внимание, что ключи соответствуют названиям файлов (password.txt - это ключ)  
  
Удалим
```shell
kubectl -n work delete secret my-secret
rm -f password.txt username.txt
```
#### 3. Из YAML-манифеста
При создании Secret из YAML-манифеста можно использовать два поля для данных:  
- `data` - для уже закодированных в base64 данных.
- `stringData` - для незакодированных данных (удобнее для использования).
  
Пример с data.
файл `9_secret/25-secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  # echo -n admin | base64
  username: YWRtaW4=
  # echo -n secretpassword | base64
  password: c2VjcmV0cGFzc3dvcmQ=
```
Применим
```shell
kubectl -n work apply -f 9_secret/25-secret.yaml
```
Проверим
```shell
kubectl -n work get secret
```
```text
NAME        TYPE     DATA   AGE
my-secret   Opaque   2      3m54s
```
Подробнее
```shell
kubectl -n work describe secret my-secret
```
```text
Name:         my-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  14 bytes
username:  5 bytes
```
Пример со stringData  
файл `9_secret/26-secret-string.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  username: admin
  password: secretpassword
```
Применим
```shell
kubectl -n work apply -f 9_secret/26-secret-string.yaml
```
Проверим
```shell
kubectl -n work get secret
```
```text
NAME        TYPE     DATA   AGE
my-secret   Opaque   2      6m56s
```
Посмотрим как `Secret` хранится в базе Kubernetes:
```shell
kubectl -n work get secret my-secret -o yaml
```
```text
apiVersion: v1
data:
  password: c2VjcmV0cGFzc3dvcmQ=
  username: YWRtaW4=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"my-secret","namespace":"work"},"stringData":{"password":"secretpassword","username":"admin"},"type":"Opaque"}
  creationTimestamp: "2026-04-30T20:48:24Z"
  name: my-secret
  namespace: work
  resourceVersion: "2661413"
  uid: 02eed30b-42e3-4ddb-bea6-cdc342973d4d
type: Opaque
```
Удаляем
```shell
kubectl -n work delete secret my-secret
```
  
Пример data + stringData  
Также можно комбинировать `data` и `stringData` в одном манифесте. В этом случае значения из `stringData` будут иметь приоритет.  
файл `9_secret/27-secret-sd.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  username: admin
  password: secretpassword
data:
  # echo -n administrator | base64
  password: YWRtaW5pc3RyYXRvcg==
```
Применим
```shell
kubectl -n work apply -f 9_secret/27-secret-sd.yaml
```
Посмотрим
```shell
kubectl -n work get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```
```text
secretpassword
```
Удаляем
```shell
kubectl -n work delete secret my-secret
```
  
#### 4. Тип kubernetes.io/tls
`Secret` типа `kubernetes.io/tls` используется для сохранения TLS-сертификатов и приватных ключей. Для создания такого Secret 
сначала нужно сгенерировать сертификат и ключ с помощью OpenSSl.  
  
Генерация приватного ключа:
```shell
openssl genrsa -out tls.key 2048
```
Генерация самоподписанного сертификата:
```shell
openssl req -new -x509 -key tls.key -out tls.crt -days 365 -subj "/CN=myapp.mamau.local"
```
После генерации сертификата и ключа можно создать `Secret` типа `tls`.
```shell
kubectl -n work create secret tls my-tls-secret --cert=tls.crt --key=tls.key
```
```shell
kubectl -n work get secrets
```
```text
NAME            TYPE                DATA   AGE
my-tls-secret   kubernetes.io/tls   2      48s
```
```shell
kubectl -n work describe secret my-tls-secret
```
```text
Name:         my-tls-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  997 bytes
tls.key:  1679 bytes
```
Для получения серты и его вывода можно сделать так:
```shell
kubectl -n work get secret my-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d
```
```text
-----BEGIN CERTIFICATE-----
MIICtDCCAZwCCQDU0+jXQQ2L2jANBgkqhkiG9w0BAQsFADAcMRowGAYDVQQDDBFt
eWFwcC5tYW1hdS5sb2NhbDAeFw0yNjA0MzAyMTExMTBaFw0yNzA0MzAyMTExMTBa
MBwxGjAYBgNVBAMMEW15YXBwLm1hbWF1LmxvY2FsMIIBIjANBgkqhkiG9w0BAQEF
AAOCAQ8AMIIBCgKCAQEAsh7qu/lpZHYTDVlpWm54g5K89SdcWdjPsyU38h5Ia0tX
5aKb8D9RpZaTtc7aD+l662Qb1o3Wb1Go1sLhuGZc9bRgTncEKVAeB+m3Ahlry/mF
7PThB7oOG5HmDly3e/QTJvABclvUjQDpyfRdkAl8OhfGAXvWB0aa0vlpEFEzOg8G
CDuVKfTk4tRalbtJ944q5LrnfLArCWY0ueiTS33iNyN9QHH7kvm4XZ6zpD2X5Isg
Ez9ZVl1E0wlNtNclRQPeRDyY5jNe5KfkxX6TqDhy4YpJBUAZZjaP4AOMZ2NfkCiU
X05LuVO3CUdeWztrxcw8F4wbvZiLNa84TMYWP1ubbwIDAQABMA0GCSqGSIb3DQEB
CwUAA4IBAQAH9ZqjptPtmPl+ZMhxyvMXoJo5fM7rcwODNRr1eZSfqKbXn0Hz5PcM
RBnpPdMEVBDNoqKjwo+ScPcKJ4ienCW+Mnrbw91jrgcMCWUV4iBXmC4i6eLRMK05
nWddc9RQxMhLaYUT6LFJQiEhueP8JP1ocnCu2cE3ZDlsr48sB/6qVbfN7FFNuHxL
KI13mBN1obDyAN5jHiIJ4KtARjoD5lxVeQcEBVlBaAv9TFWp0f0kj+g6HNWjQ/i5
iB3WA6kuhLyGrK3IWGCrourXzNJErh+Se0OpBuijD2tT5oR/iCgSkPF/05NSVTtb
lBWwh6wY+C2RG/xIN25MwpSxdJKADEyY
-----END CERTIFICATE-----
```
Удаляем
```shell
rm -f tls.*
kubectl -n work delete secret my-tls-secret
```
#### 5. Тип kubernetes.io/dockerconfigjson
`Secret` или `kubernetes.io/dockerconfigjson` используется для хранения учетных данных Docker registry. Его можно создать 
двумя способами:
1. Используя файл конфигурации Docker:
```shell
kubectl create secret generic my-registry-secret \
  --from-file=.dockerconfigjson=/path/to/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```
2. Используя команду `kubectl create secret docker-registry`:
```shell
kubectl -n work create secret docker-registry my-registry-secret \
  --docker-server=https://my-registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@exampl.com
```
Вызовим вторую команду  
посомтрим список
```shell
kubectl -n work get secrets
```
```text
NAME                 TYPE                             DATA   AGE
my-registry-secret   kubernetes.io/dockerconfigjson   1      35s
```
Подробно
```shell
kubectl -n work describe secret my-registry-secret
```
```text
Name:         my-registry-secret
Namespace:    work
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  154 bytes
```
Удалим
```shell
kubectl -n work delete secret my-registry-secret
```
  
### Использование Secret
По аналогии с `ConfigMap`, `Secret` можно использовать для хранения переменных среды окружения или содержимого файлов.  
  
Переменные среды окружения
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  USERNAME: admin
  PASSWORD: secretpassword
---
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
          envFrom:
            - secretRef:
                name: my-secret
```