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
      name: my-first-pod
      namespace: work
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