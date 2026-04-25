## Job
***
Job — ресурс Kubernetes для выполнения одноразовых или ограниченных по числу запусков задач.   
Гарантирует, что контейнер выполнится до успешного завершения заданное количество раз.  
  
Манифест выглядит так
```yaml
apiVersion: apps/v1
kind: Job
metadata:
  name: myjob
  labels:
    app.kubernetes.io/name: myjob
    app.kubernetes.io/version: v0.0.1
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: test-job
          image: busybox:1.28
          command:
            - 'sh'
            - '-c'
            - 'echo "Test job running"... && sleep 10 && exit 0'
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```
`spec.selector` как и `spec.template.metadata.labels` в случае `Job` не обязательны. Они будут формироваться автоматически
контроллером. Дальше мы увидим, как это работает.  
  
`restartPolicy: Never` - установлен для того, что бы при неудачном завершении приложения в поде, под завершал свою работу
и контроллер `Job` мог перезапустить под. Если этого не сделать - контейнер будет перезапускаться без участия контроллера `Job`
и может возникнуть недопонимание логики работы `Job`.  
  
В остальном template пода ничем не отличается от такого же шаблона, например, `Deployment`.  
  
Запустим наш Jon:
```shell
kubectl -n work apply -f 6_job_cronjob/09-job.yaml
```
```text
job.batch/myjob created
```
Посмотри информацию о `Job`.
```shell
kubectl -n work get jobs
```
```text
NAME    STATUS     COMPLETIONS   DURATION   AGE
myjob   Complete   1/1           13s        52s
```
Мы получаем список `Jobs` в указанном `namespace`. Видим один заверешенный с состоянием `Complete`.  
  
В том же `namespace` посмотрим список подов:
```shell
kubectl -n work get pods
```
```text
NAME          READY   STATUS      RESTARTS   AGE
myjob-fclgt   0/1     Completed   0          21h
```
Наблюдаем завершивший свою работу под. Под, автоматически не завершается.  
  
Посмотрим подробнее на `myjob`:
```shell
kubectl -n work get job myjob -o yaml | nl
```
```text
     1  apiVersion: batch/v1
     2  kind: Job
     3  metadata:
     4    annotations:
     5      kubectl.kubernetes.io/last-applied-configuration: |
     6        {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"labels":{"app.kubernetes.io/name":"myjob","app.kubernetes.io/version":"v0.0.1"},"name":"myjob","namespace":"work"},"spec":{"template":{"spec":{"containers":[{"command":["sh","-c","echo \"Test job running\"... \u0026\u0026 sleep 10 \u0026\u0026 exit 0"],"image":"busybox:1.28","name":"test-job","resources":{"limits":{"cpu":"500m","memory":"128Mi"},"requests":{"cpu":"250m","memory":"64Mi"}}}],"restartPolicy":"Never"}}}}
     7    creationTimestamp: "2026-04-24T21:50:50Z"
     8    generation: 1
     9    labels:
    10      app.kubernetes.io/name: myjob
    11      app.kubernetes.io/version: v0.0.1
    12    name: myjob
    13    namespace: work
    14    resourceVersion: "1866474"
    15    uid: 86775027-a35b-404a-a264-8d8a36b545ae
    16  spec:
    17    backoffLimit: 6
    18    completionMode: NonIndexed
    19    completions: 1
    20    manualSelector: false
    21    parallelism: 1
    22    podReplacementPolicy: TerminatingOrFailed
    23    selector:
    24      matchLabels:
    25        batch.kubernetes.io/controller-uid: 86775027-a35b-404a-a264-8d8a36b545ae
    26    suspend: false
    27    template:
    28      metadata:
    29        labels:
    30          batch.kubernetes.io/controller-uid: 86775027-a35b-404a-a264-8d8a36b545ae
    31          batch.kubernetes.io/job-name: myjob
    32          controller-uid: 86775027-a35b-404a-a264-8d8a36b545ae
    33          job-name: myjob
    34      spec:
    35        containers:
    36        - command:
    37          - sh
    38          - -c
    39          - echo "Test job running"... && sleep 10 && exit 0
    40          image: busybox:1.28
    41          imagePullPolicy: IfNotPresent
    42          name: test-job
    43          resources:
    44            limits:
    45              cpu: 500m
    46              memory: 128Mi
    47            requests:
    48              cpu: 250m
    49              memory: 64Mi
    50          terminationMessagePath: /dev/termination-log
    51          terminationMessagePolicy: File
    52        dnsPolicy: ClusterFirst
    53        restartPolicy: Never
    54        schedulerName: default-scheduler
    55        securityContext: {}
    56        terminationGracePeriodSeconds: 30
    57  status:
    58    completionTime: "2026-04-24T21:51:03Z"
    59    conditions:
    60    - lastProbeTime: "2026-04-24T21:51:03Z"
    61      lastTransitionTime: "2026-04-24T21:51:03Z"
    62      message: Reached expected number of succeeded pods
    63      reason: CompletionsReached
    64      status: "True"
    65      type: SuccessCriteriaMet
    66    - lastProbeTime: "2026-04-24T21:51:03Z"
    67      lastTransitionTime: "2026-04-24T21:51:03Z"
    68      message: Reached expected number of succeeded pods
    69      reason: CompletionsReached
    70      status: "True"
    71      type: Complete
    72    ready: 0
    73    startTime: "2026-04-24T21:50:50Z"
    74    succeeded: 1
    75    terminating: 0
    76    uncountedTerminatedPods: {}
```
Начнем со строк 23,24,25. Здесь мы видим, что контроллер автоматически сгенерировал selector, для выбора пода. 
А в строках 30-34 автоматически созданные метки.  
  
Строка 17 `backoffLimit: 6` - определяет количество попыток перезапуска пода после неудачного завершения. В нашем случае будет
максимум 6 попыток перезапуска.  
  
Посмотри как `Job` будет себя вести в случае неудачного завершения приложения в контейнере. Я добавил `exit 1` в `command`, чтобы эмулировать ошибку.  
```shell
kubectl -n work delete job myjob
```
```shell
kubectl -n work apply -f 6_job_cronjob/10-fail-job.yaml
```
```shell
kubectl -n work get pods -w
```
```text
NAME          READY   STATUS    RESTARTS   AGE
myjob-7nhnq   0/1     Error     0          55s
myjob-h6nph   0/1     Error     0          35s
myjob-qmpcq   1/1     Running   0          5s
myjob-qmpcq   0/1     Error     0          10s
myjob-qmpcq   0/1     Error     0          12s
myjob-qmpcq   0/1     Error     0          12s
myjob-qmpcq   0/1     Error     0          13s
```
Каждая следующая попытка будет по времени все дальше и дальше, дойдет до минут 13...  
```shell
kubectl -n work describe job myjob | nl
```
```text
1  Name:             myjob
     2  Namespace:        work
     3  Selector:         batch.kubernetes.io/controller-uid=72cebb5f-c5ce-42a0-b5ee-a34dc996f809
     4  Labels:           app.kubernetes.io/name=myjob
     5                    app.kubernetes.io/version=v0.0.1
     6  Annotations:      <none>
     7  Parallelism:      1
     8  Completions:      1
     9  Completion Mode:  NonIndexed
    10  Suspend:          false
    11  Backoff Limit:    6
    12  Start Time:       Sat, 25 Apr 2026 22:28:08 +0300
    13  Pods Statuses:    0 Active (0 Ready) / 0 Succeeded / 5 Failed
    14  Pod Template:
    15    Labels:  batch.kubernetes.io/controller-uid=72cebb5f-c5ce-42a0-b5ee-a34dc996f809
    16             batch.kubernetes.io/job-name=myjob
    17             controller-uid=72cebb5f-c5ce-42a0-b5ee-a34dc996f809
    18             job-name=myjob
    19    Containers:
    20     test-job:
    21      Image:      busybox:1.28
    22      Port:       <none>
    23      Host Port:  <none>
    24      Command:
    25        sh
    26        -c
    27        echo "Test job running"... && sleep 10 && exit 1
    28      Limits:
    29        cpu:     500m
    30        memory:  128Mi
    31      Requests:
    32        cpu:         250m
    33        memory:      64Mi
    34      Environment:   <none>
    35      Mounts:        <none>
    36    Volumes:         <none>
    37    Node-Selectors:  <none>
    38    Tolerations:     <none>
    39  Events:
    40    Type    Reason            Age    From            Message
    41    ----    ------            ----   ----            -------
    42    Normal   SuccessfulCreate      11m    job-controller  Created pod: myjob-7nhnq
    43    Normal   SuccessfulCreate      11m    job-controller  Created pod: myjob-h6nph
    44    Normal   SuccessfulCreate      11m    job-controller  Created pod: myjob-qmpcq
    45    Normal   SuccessfulCreate      10m    job-controller  Created pod: myjob-tt8qj
    46    Normal   SuccessfulCreate      8m49s  job-controller  Created pod: myjob-qx64m
    47    Normal   SuccessfulCreate      5m59s  job-controller  Created pod: myjob-hj5gv
    48    Normal   SuccessfulCreate      29s    job-controller  Created pod: myjob-q7rp5
    49    Warning  BackoffLimitExceeded  16s    job-controller  Job has reached the specified backoff limit

```
В 39 строке в Events мы видим что достигнут лимит перезапуска пода. 1 раз запустили + 6 перезапусков = всего 7 попыток.  
  
Поды, контролируемы `Job` будут находится в списке подов до тех пор пока вы их не удалите или не удалите job, управляющий этими подами.  
  
Удалим job и вместе с ним удалятся поды
```shell
kubectl -n work delete job myjob
```
  
### Автоматическое удаление завершенных Jon`ов
  
Если вам нужно, чтобы завершенные задачи автоматически удалялись достаточно добавить `.spec.ttlSecondsAfterFinished` в манифесте `Job`.
Таймер запускается, когда статус задания меняется на `Complete` или `Failed`.
```yaml
spec:
  ttlSecondsAfterFinished: 180
```
Запустим `Job`, который автоматически удаляет свои поды через 3 минуты после завершения их работы:
```shell
kubectl -n work apply -f 6_job_cronjob/11-job-autoclean.yaml
```
```shell
kubectl -n work get pods -w
```
Приблизительно через 4 минуты `Job` и `Pod` будут удалены
  
Можно посмотреть по `events`
```shell
kubectl -n work events
```
```text
LAST SEEN   TYPE     REASON             OBJECT                      MESSAGE
2m17s       Normal   Scheduled          Pod/myjob-autoclean-kdxcq   Successfully assigned work/myjob-autoclean-kdxcq to ubuntu-4gb-hel1-30-node-3
2m17s       Normal   Pulled             Pod/myjob-autoclean-kdxcq   Container image "busybox:1.28" already present on machine and can be accessed by the pod
2m17s       Normal   Created            Pod/myjob-autoclean-kdxcq   Container created
2m17s       Normal   Started            Pod/myjob-autoclean-kdxcq   Container started
2m17s       Normal   SuccessfulCreate   Job/myjob-autoclean         Created pod: myjob-autoclean-kdxcq
2m4s        Normal   Completed          Job/myjob-autoclean         Job completed
```
  
### CronJob
`Cronjob` - позволяет запускать `Job` по расписанию.  
  
Пример манифеста `Cronjob`:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/2 * * * *"
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: test-job
              image: busybox:1.28
              command:
                - 'sh'
                - '-c'
                - 'echo "Test job running"... && sleep 10 && exit 0'
              resources:
                requests:
                  memory: "64Mi"
                  cpu: "250m"
                limits:
                  memory: "128Mi"
                  cpu: "500m"
```
`schedule` - строка в формате CRON, определяющая расписание запуска  
`successfulJobsHistoryLimit` и `failedJobHistoryLimit` - количество успешных и неудачных заданий для хранения в истории. 
Значения по умолчанию 5 и 1 соответственно.  
  
Применим манифест:
```shell
kubectl -n work apply -f 6_job_cronjob/12-cronjob.yaml
```
Посмотреть
```shell
kubectl -n work get cronjobs
```
```text
NAME         SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
my-cronjob   */2 * * * *   <none>     False     0        <none>          7s
```
Теперь раз в 2 минуты будут запускаться `Job` и соответствующие ему поды.
```shell
watch kubectl -n work get jobs
```
```text
Every 2.0s: kubectl --kubeconfig=/Users/mamau/.kube/my-kube.conf -n work get jobs                                          
                                                                                                                            
NAME                  STATUS     COMPLETIONS   DURATION   AGE
my-cronjob-29619258   Complete   1/1           13s        4m27s
my-cronjob-29619260   Complete   1/1           13s        2m27s
my-cronjob-29619262   Complete   1/1           13s        27s
```
Понаблюдаем и потом удалим
```shell
kubectl -n work delete -f 6_job_cronjob/12-cronjob.yaml
```
```text
cronjob.batch "my-cronjob" deleted
```