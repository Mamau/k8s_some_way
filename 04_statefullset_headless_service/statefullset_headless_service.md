## StatefulSet
***
> В отличии от Deployment, StatefulSet гарантирует уникальность имени каждого пода в кластере kubernetes.
> И строго определяет порядок запуска, обновления и удаления подов.
  
### Пример конфигурации StatefulSet
***
Пример манифеста StatefulSet:  
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: test-st
  labels:
    app.kubernetes.io/name: &name test-st
    app.kubernetes.io/instance: &instance test-st
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 3
  serviceName: "test-st-headless"
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
      containers:
        - name: whoami
          image: traefik/whoami
```