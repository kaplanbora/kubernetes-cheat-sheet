# Kubernetes Cheat Sheet
I made this while learning Kubernetes. It is a very broad topic given all the `.yaml` definitions and `kubectl` commands. This should be helpful when you forget stuff. This is not a comprehensive list but it includes most common options. Consult to the official [Kubernetes Docs](https://kubernetes.io/docs/) whenever you need. Hope you find it useful.

## Topics
- <a href="#kubectl-commands">Kubectl Commands</a>
  - <a href="#get">Get</a>
  - <a href="#describe">Describe</a>
  - <a href="#apply">Apply</a>
  - <a href="#edit">Edit</a>
  - <a href="#logs">Logs</a>
  - <a href="#tops">Tops</a>
  - <a href="#rollout">Rollout</a>
- <a href="#object-definitions">Object Definitions</a>
  - <a href="#pod">Pod</a>
  - <a href="#configmap">ConfigMap</a>
  - <a href="#secret">Secret</a>
  - <a href="#persistentvolume">PersistentVolume</a>
  - <a href="#persistentvolumeclaim">PersistentVolumeClaim</a>
  - <a href="#deployment">Deployment</a>
  - <a href="#job">Job</a>
  - <a href="#cronjob">CronJob</a>
  - <a href="#service">Service</a>
  - <a href="#networkpolicy">NetworkPolicy</a>
  - <a href="#horizontalpodautoscaler">HorizontalPodAutoscaler</a>

## Kubectl Commands

### Get
Get information about all objects of given type or a specific object
```bash
kubectl get $OBJECT_TYPE 
kubectl get $OBJECT_TYPE $OBJECT_NAME
```

You can specify namespace
```bash
kubectl get pods -n prod
kubectl get pods --all-namespaces
```

Get definition as yaml in export mode and save it to a file
```bash
kubectl get pod auth-server -o yaml --export > data.yaml
```

Get resources that match the selector using labels
```bash
kube get -l app=data-ingestion
```

### Describe
Get detailed information about a resource
```bash
kubectl describe $OBJECT_TYPE $OBJECT_NAME
```

### Apply
Create a new object given a definition or update existing one
```bash
kubectl apply -f definition.yaml
```

### Edit
Edit an existing object 
```bash
kubectl edit $OBJECT_TYPE $OBJECT_NAME
```

### Logs
Get logs of a pod
```bash
kubectl logs $POD_NAME
```

Get logs of previous pod
```bash
kubectl logs $POD_NAME --previous
```

### Tops
Monitor resource usage of all objects of given type or a specific object
```bash
kubectl tops $OBJECT_TYPE
kubectl tops $OBJECT_TYPE $OBJECT_NAME
```

### Rollout
Get rollout history of a deployment
```bash
kubectl rollout history deployment/$DEPLOYMENT_NAME
```

Check ongoing rollout status
```bash
kubectl rollout status deployment/$DEPLOYMENT_NAME
```

Undo latest rollout
```bash
kubectl rollout undo deployment/$DEPLOYMENT_NAME
```

Undo to a specific revision
```bash
kubectl rollout undo deployment/$DEPLOYMENT_NAME --to-revision=3
```

Record a command in history
```bash
kubectl --record $COMMAND
```

Update image of a deployment
```bash
kubectl $DEPLOYMENT_NAME set image deployment.v1.apps/nginx-deployment nginx=nginx:1.8.8
```

# Object Definitions

## Pod
Basic building block of a Kubernetes cluster that contains one or more containers.

Some common options without details
```yaml
apiVersion: v1
kind: Pod
metadata:
  name:
  namespace:
  labels:
  annotations:
spec:
  restartPolicy: 
  securityContext:
  volumes: 
  containers:
  - name:
    image:
    command:
    args:
    ports:
    volumeMounts:
    resources:
    serviceAccountName: 
    env:
    livenessProbe:
```

Full example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-collector
  namespace: ingestion
  labels:
    app: data-collector
    environment: dev
  annotations:
    owner: bora@kaplan.dev
spec:
  restartPolicy: OnFailure
  securityContext:
    runAsUser: 100     # Any files created will be owned by this user
    runAsGroup: 200    # Files also will be owned by this group
    fsGroup: 300       # Will own mounted volumes
  volumes: 
  - name: user-data
    hostPath:
      path: /home/user/data
  containers:
  - name: data-collector
    image: data-collector:1.0.0
    command: ['java', '-jar', 'data-collector.jar']
    args: ['--dbHost', '127.0.0.1'] 
    ports:
    - containerPort: 80
    volumeMounts:
    - name: user-data
      mountPath: /data
    resources:
      requests:
        memory: "128Mi"
        cpu: "300m"
      limits:
        memory: "192Mi"
        cpu: "400m"
    serviceAccountName: data-collector-sa
    env:
    - name: DB_PASSWORD
      valueFrom: 
        secretKeyRef: 
          name: db-secrets 
          key: password
    - name: DB_USER
      valueFrom: 
        secretKeyRef: 
          name: db-secrets 
          key: user
    livenessProbe:
      httpGet: 
        path: /health
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 1
```

## ConfigMap
Store your key-value information about an application.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: server-config
data:
  host-address: 127.0.0.1
  host-port: 8008
```

Usage in containers as an environment variable
```yaml
containers:
- ...
  env:
  - name: PORT 
    valueFrom:
      configMapKeyRef:
        name: server-config
        key: host-port
```

Usage in containers as a volume
```yaml
containers:
- ...
  volumeMounts:
  - name: configs
    mount: /etc/configs
volumes:
  - name: configs
    configMap:
      name: server-config
```

## Secret
Create secrets using plain text
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
stringData:
  user: bora
  password: 1337
```

Create secrets from base64 text
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
data:
  user: Ym9yYQ==
  password: MTMzNw==
```

Example usage in a container as an environment variable
```yaml
containers:
- ...
  env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secrets
        key: password
```

Example usage in containers as a volume
```yaml
containers:
- ...
  volumeMounts:
  - name: secrets
    mount: /etc/secrets
    readOnly: true
volumes:
  - name: secrets
    secret:
      name: db-secrets
```

## PersistentVolume
Storage resource that can be attached to pods and is non ephemeral.
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: log-volume
spec: 
  storageClassName: local-storage
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  local:
    path: "/var/log"
```

## PersistentVolumeClaim
Request for the storage resource defined by a PersistentVolume that can be attached to a container.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: log-volume-claim
spec: 
  storageClassName: local-storage
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Usage in a pod definition
```yaml
spec:
  volumes:
  - name: logs
    persistentVolumeClaim:
      claimName: log-volume-claim
  containers:
    volumeMounts:
      - mountPath: "/var/log"
        name: logs
```

## Deployment
Watch pods states and their resources and manage their lifecycles with deployments.

Some common options without details
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:
  namespace:
  labels:
  annotations:
spec:
  replicas: 
  selector:
  strategy:
  template:
    metadata:    # Same as Pod definition
    spec:        # Same as Pod definition
```

Full example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-collector-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      app: data-collector
  strategy:
    rollingUpdate:
      maxSurge: 50%           # Can also be pod number
      maxUnavailable: 25%     # Can also be pod number
  template:
    metadata:
      name: data-collector
      namespace: ingestion
      labels:
        app: data-collector
        environment: dev
      annotations:
        owner: bora@kaplan.dev
    spec:
      restartPolicy: OnFailure
      securityContext:
        runAsUser: 100     # Any files created will be owned by this user
        runAsGroup: 200    # Files also will be owned by this group
        fsGroup: 300       # Will own mounted volumes
      volumes: 
      - name: user-data
        hostPath:
          path: /home/user/data
      containers:
      - name: data-collector
        image: data-collector:1.0.0
        command: ['java', '-jar', 'data-collector.jar']
        args: ['--dbHost', '127.0.0.1'] 
        ports:
        - containerPort: 80
        volumeMounts:
        - name: user-data
          mountPath: /data
        resources:
          requests:
            memory: "128Mi"
            cpu: "300m"
          limits:
            memory: "192Mi"
            cpu: "400m"
        serviceAccountName: data-collector-sa
        env:
        - name: DB_PASSWORD
          valueFrom: 
            secretKeyRef: 
              name: db-secrets 
              key: password
        - name: DB_USER
          valueFrom: 
            secretKeyRef: 
              name: db-secrets 
              key: user
        livenessProbe:
          httpGet: 
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 1
```

## Job
Creates pods to do a specific job and then ensures that they terminate successfully.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: aggregator
spec:
  template:
    spec:
      containers:
      - name: aggregator
        image: data-aggregator:1.0.0
        args: ["--outputLocation", "gs://aggregation-results/"]
  backoffLimit: 3
```

## CronJob
Creates jobs with a given schedule to repeat.
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: aggregator
spec:
  schedule: "*/60 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: aggregator
            image: data-aggregator:1.0.0
            args: ["--outputLocation", "gs://aggregation-results/"]
```

## Service
Services are used to target dynamically changing pods to give client applications an endpoint to use.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  type: ClusterIP
  selector:
    app: auth-server
  ports:
  - protocol: TCP
    port: 80            # Port to reach the service
    targetPort: 8080    # Port that Pods listen

```

## NetworkPolicy
Define rules to specify how pods can communicate with each other.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

## HorizontalPodAutoscaler
Autoscale your deployment when a resource hits the threshold.
```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: event-consumer-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: event-consumer
  minReplicas: 3
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 400Mi
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70
```

