# Lecture 20
## Завдання 1: Створення StatefulSet для Redis-кластера
Опис
Redis-кластер використовує StatefulSet для забезпечення постійних даних і стабільних імен для кожного екземпляра 
Redis. Вам потрібно створити StatefulSet для Redis із двома репліками, які взаємодіятимуть між собою.
Кроки:
1. Створіть PersistentVolumeClaim (PVC) для зберігання даних Redis. Кожен под у StatefulSet використовуватиме свій окремий том для зберігання даних.
2. Створіть StatefulSet для Redis із налаштуваннями для запуску двох реплік. Кожна репліка повинна мати  стабільне ім’я та доступ до постійного тому.
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: "redis"
  replicas: 2
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:6.2
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```
3. Створіть Service для Redis: Service для StatefulSet потрібен для доступу до Redis. Використовуйте тип Service ClusterIP для внутрішньої взаємодії між подами.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - port: 6379
```
**Створення конфігураційного файлу kind**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: redis-falco-cluster
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /proc
        containerPath: /host/proc
      - hostPath: /boot
        containerPath: /boot
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
```
4. Перевірка роботи: після створення StatefulSet перевірте, чи запущені поди (kubectl get pods) і чи мають  вони стабільні імена, наприклад, redis-0, redis-1. Застосовуйте команду kubectl exec для підключення до  кожного пода та перевірки збереження даних між перезапусками

**Перевірка робота**
```shell
PS G:\k8s\redis> kubectl get nodes
NAME                                STATUS   ROLES           AGE   VERSION
redis-falco-cluster-control-plane   Ready    control-plane   78s   v1.29.2
PS G:\k8s\redis> kubectl apply -f redis-service.yaml
service/redis created
PS G:\k8s\redis> kubectl apply -f redis-statefulset.yaml
statefulset.apps/redis created
PS G:\k8s\redis> kubectl get pods
NAME      READY   STATUS              RESTARTS   AGE
redis-0   0/1     ContainerCreating   0          8s
PS G:\k8s\redis> kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          59s
redis-1   1/1     Running   0          49s
```
**Перевірка робота постійниї томів**
- Записали дані у два томи, після перезапустив(видалив) поду, автоматично піднялась нова з підлюченням до тома і перевірив наявність даних
```shell
PS G:\k8s\redis> kubectl exec -it redis-0 -- redis-cli
127.0.0.1:6379> set mykey0 "Hello from redis-0"
OK
127.0.0.1:6379> get mykey0
"Hello from redis-0"
127.0.0.1:6379> exit
PS G:\k8s\redis> kubectl exec -it redis-1 -- redis-cli
127.0.0.1:6379> set mykey1 "Hello from redis-1"
OK
127.0.0.1:6379> get mykey1
"Hello from redis-1"
127.0.0.1:6379> exit
PS G:\k8s\redis> kubectl delete pod redis-0
pod "redis-0" deleted
PS G:\k8s\redis> kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          7s
redis-1   1/1     Running   0          5m2s
PS G:\k8s\redis> kubectl exec -it redis-0 -- redis-cli
127.0.0.1:6379> get mykey0
"Hello from redis-0"
127.0.0.1:6379> exit
```
## Завдання 2: Налаштування Falco в Kubernetes за допомогою DaemonSet
**Мета завдання**
Розгорнути інструмент Falco в кластері Kubernetes для моніторингу подій безпеки на кожному вузлі. Falco буде  встановлено через DaemonSet, що забезпечить його запуск на кожному вузлі в кластері для контролю системних  подій у реальному часі.
**Умови завдання**
1. Налаштуйте DaemonSet для Falco:
- Розробіть конфігурацію DaemonSet, яка розгорне Falco на кожному вузлі. Falco повинен працювати з  привілейованим доступом для моніторингу системних викликів.
2. Налаштуйте монтування системних директорій: для того щоб Falco міг правильно збирати системні  події, потрібно змонтувати такі директорії:
- /proc — директорія, що містить інформацію про запущені процеси, це треба для доступу до процесів 
на вузлі
- /boot — може містити дані про конфігурації ядра, що дає змогу Falco краще розпізнавати події
- /lib/modules — тут розташовані модулі ядра, доступ до них дозволяє Falco використовувати eBPF для  збору даних
- /var/run/docker.sock — дає Falco доступ до Docker-сокета для контролю подій, пов’язаних з  контейнерами — важливо, якщо вузол використовує Docker як рантайм
- /usr — дозволяє доступ до системних бібліотек і утиліт, необхідних для розширення функціональності Falco
3. Обмежте використання ресурсів:
- Для Falco потрібно встановити обмеження на застосування ресурсів (CPU і пам’ять), щоб він не  впливав на роботу інших сервісів на вузлі
- Встановіть, наприклад, 100m CPU і 256Mi пам’яті як ліміти, а також 100m CPU і 128Mi пам’яті як  мінімальні запити
4. Створіть YAML-файл конфігурації DaemonSet:
- Створіть YAML-файл, який відповідає вимогам і запускає Falco на кожному вузлі
- Переконайтеся, що всі потрібні директорії змонтовані для коректної роботи Falco
5. Перевірка розгортання та роботи Falco:
Після застосування YAML-файлу за допомогою команди kubectl apply -f falco-daemonset.yaml
- Перевірте, чи всі поди Falco запущені на кожному вузлі: kubectl get pods -l app=falco -n  kube-system
- Переконайтеся, що кожен под Falco працює в статусі Running
6. Виконайте команду для перегляду логів одного з подів Falco:
kubectl logs -l app=falco -n kube-system
- Переконайтеся, що Falco генерує сповіщення про події — логи можуть містити інформацію про такі дії,  як-от доступ до файлів, створення нових процесів або взаємодія з Docker
7. Документація з налаштуванням та аналізом:
- Описати, які кроки виконано для налаштування Falco
- Надати вивід команд kubectl get pods та kubectl logs для підтвердження успішного  розгортання і роботи Falco
**
**Створення конфігураційного файлу з урахуванням обмежень:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: kube-system
  labels:
    app: falco
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        securityContext:
          privileged: true
        resources:
          limits:
            memory: 256Mi
            cpu: 100m
          requests:
            memory: 128Mi
            cpu: 100m
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: boot
          mountPath: /host/boot
          readOnly: true
        - name: lib-modules
          mountPath: /host/lib/modules
          readOnly: true
        - name: docker-socket
          mountPath: /host/var/run/docker.sock
        - name: usr
          mountPath: /host/usr
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: boot
        hostPath:
          path: /boot
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
      - name: usr
        hostPath:
          path: /usr
```
Перевірка роботи:
```shell
PS G:\k8s\redis> kubectl apply -f falco-daemonset.yaml
error: the path "falco-daemonset.yaml" does not exist
PS G:\k8s\redis> 
PS G:\k8s\redis> kubectl apply -f falco-daemonset.yaml
daemonset.apps/falco created
PS G:\k8s\redis> kubectl get pods -l app=falco -n kube-system
NAME          READY   STATUS              RESTARTS   AGE
falco-lqtss   0/1     ContainerCreating   0          11s
PS G:\k8s\redis> kubectl get pods -l app=falco -n kube-system
NAME          READY   STATUS              RESTARTS   AGE
falco-lqtss   0/1     ContainerCreating   0          39s
PS G:\k8s\redis> kubectl get pods -l app=falco -n kube-system
NAME          READY   STATUS    RESTARTS   AGE
falco-lqtss   1/1     Running   0          2m24s
falco-lqtss   1/1     Running   0          2m24s
PS G:\k8s\redis> kubectl logs -l app=falco -n kube-system
2025-05-27T08:51:17+0000: System info: Linux version 5.15.167.4-microsoft-standard-WSL2 (root@f9c826d3017f) (gcc (GCC) 11.2.0, GNU ld (GNU Binutils) 2.37) #1 SMP Tue Nov 5 00:21:55 UTC 2024
2025-05-27T08:51:17+0000: Loading rules from:
2025-05-27T08:51:17+0000:    /etc/falco/falco_rules.yaml | schema validation: ok
2025-05-27T08:51:18+0000:    /etc/falco/falco_rules.local.yaml | schema validation: none
2025-05-27T08:51:18+0000: The chosen syscall buffer dimension is: 8388608 bytes (8 MBs)
2025-05-27T08:51:18+0000: Starting health webserver with threadiness 16, listening on 0.0.0.0:8765
2025-05-27T08:51:18+0000: Loaded event sources: syscall
2025-05-27T08:51:18+0000: Enabled event sources: syscall
2025-05-27T08:51:18+0000: Opening 'syscall' source with modern BPF probe.
2025-05-27T08:51:18+0000: One ring buffer every '2' CPUs.
```
- Falco у статусі Running.
- Логи підтвердили роботу Falco і моніторинг системних викликів.