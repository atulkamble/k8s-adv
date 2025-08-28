Awesome—here’s a compact, high-leverage crash set of **advanced Kubernetes concepts** with practical, production-ready **code snippets** you can drop into repos.

# 1) Scheduling controls

### Node affinity + taints/tolerations + topology spread

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 6
  selector: { matchLabels: { app: api } }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 1, maxSurge: 1 }
  template:
    metadata:
      labels: { app: api }
    spec:
      nodeSelector: { "node.kubernetes.io/instance-type": "m5.large" }
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["ap-south-1a","ap-south-1b"]
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels: { app: api }
      tolerations:
      - key: "workload"
        operator: "Equal"
        value: "critical"
        effect: "NoSchedule"
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector: { matchLabels: { app: api } }
      containers:
      - name: api
        image: ghcr.io/org/api:1.2.3
        resources:
          requests: { cpu: "250m", memory: "256Mi" }
          limits:   { cpu: "1",    memory: "512Mi" }
```

# 2) Resilience & rolling safety

### Pod Disruption Budget (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 5
  selector:
    matchLabels: { app: api }
```

### Readiness, liveness & startup probes

```yaml
        readinessProbe:
          httpGet: { path: /healthz/ready, port: 8080 }
          periodSeconds: 5
          failureThreshold: 2
        livenessProbe:
          httpGet: { path: /healthz/live, port: 8080 }
          initialDelaySeconds: 30
        startupProbe:
          httpGet: { path: /healthz/startup, port: 8080 }
          failureThreshold: 30
          periodSeconds: 2
```

# 3) Config & secrets with safe rollouts

### ConfigMap + Secret + checksum annotation (forces rollout on change)

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: api-config }
data:
  APP_LOG_LEVEL: "info"
  APP_FEATURE_X: "true"
---
apiVersion: v1
kind: Secret
metadata: { name: api-secret }
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:pass@db:5432/app"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  selector: { matchLabels: { app: api } }
  replicas: 3
  template:
    metadata:
      labels: { app: api }
      annotations:
        checksum/config: "{{CONFIGMAP_SHA256}}"
        checksum/secret: "{{SECRET_SHA256}}"
    spec:
      containers:
      - name: api
        image: ghcr.io/org/api:1.2.3
        envFrom:
        - configMapRef: { name: api-config }
        - secretRef:    { name: api-secret }
        volumeMounts:
        - name: cfg
          mountPath: /etc/app
        volumes:
        - name: cfg
          projected:
            sources:
            - configMap: { name: api-config }
            - secret:    { name: api-secret }
```

> Compute the `{{..._SHA256}}` in your CI and inject via Helm/Kustomize to trigger a rolling update.

# 4) Sidecars & init containers

```yaml
spec:
  initContainers:
  - name: migrate-db
    image: ghcr.io/org/api-migrate:1.2.3
    args: ["up"]
  containers:
  - name: api
    image: ghcr.io/org/api:1.2.3
  - name: oauth-proxy
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
    args: ["--http-address=0.0.0.0:4180", "--upstream=http://localhost:8080"]
```

# 5) Advanced autoscaling

### HPA v2 (CPU + custom metric)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api-hpa }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 60 }
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second   # needs metrics adapter
      target:
        type: AverageValue
        averageValue: "50"
```

# 6) Network security

### Default-deny then allow egress/ingress (Calico/Cilium/IPTables)

```yaml
# Namespace default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: prod }
spec:
  podSelector: {}
  policyTypes: ["Ingress","Egress"]
---
# Allow api -> db only on 5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-to-db, namespace: prod }
spec:
  podSelector:
    matchLabels: { app: db }
  ingress:
  - from:
    - podSelector: { matchLabels: { app: api } }
    ports:
    - protocol: TCP
      port: 5432
```

# 7) Ingress & Gateway API (future-proof)

### Ingress (Nginx example)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "16m"
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["api.example.com"]
    secretName: tls-api
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: api-svc, port: { number: 8080 } }
```

> If you’re adopting **Gateway API**, map the same via `HTTPRoute` with `GatewayClass`.

# 8) Stateful workloads & storage

### StorageClass with volume expansion + StatefulSet with orderedReady

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: gp3-expand }
allowVolumeExpansion: true
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: postgres }
spec:
  serviceName: postgres-h
  podManagementPolicy: OrderedReady
  replicas: 3
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
      - name: db
        image: postgres:16
        ports: [{containerPort: 5432, name: db}]
        volumeMounts: [{name: data, mountPath: /var/lib/postgresql/data}]
  volumeClaimTemplates:
  - metadata: { name: data }
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3-expand
      resources: { requests: { storage: 50Gi } }
```

# 9) Jobs, retries & cleanup

### Job with `PodFailurePolicy` + TTL cleanup

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: nightly-etl }
spec:
  backoffLimit: 4
  ttlSecondsAfterFinished: 3600
  podFailurePolicy:
    rules:
    - action: Ignore
      onExitCodes:
        containerName: etl
        operator: In
        values: [2]  # known transient error, let controller retry
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: etl
        image: ghcr.io/org/etl:2.0
        args: ["--import"]
```

# 10) RBAC with least privilege

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: reader, namespace: prod }
rules:
- apiGroups: [""]
  resources: ["pods","services","endpoints"]
  verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: reader-bind, namespace: prod }
subjects:
- kind: ServiceAccount
  name: ci
  namespace: tooling
roleRef:
  kind: Role
  name: reader
  apiGroup: rbac.authorization.k8s.io
```

# 11) Pod Security Admission (PSA) – restricted

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
```

# 12) Traffic strategies

### Blue/Green with two Deployments, one Service selector

```yaml
apiVersion: v1
kind: Service
metadata: { name: api-svc }
spec:
  selector: { app: api, color: blue }   # flip to green during cutover
  ports: [{ port: 8080, targetPort: 8080 }]
```

### Canary by splitting via two Services + Ingress canary annotations (nginx)

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
```

# 13) Observability hooks

### Prometheus scrape hints (annotations)

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

> For Prometheus Operator, prefer `ServiceMonitor`/`PodMonitor` CRDs.

# 14) Packaging & GitOps

### Kustomize overlay example

```yaml
# base/kustomization.yaml
resources: [deployment.yaml, service.yaml]
configMapGenerator:
- name: api-config
  literals: ["APP_LOG_LEVEL=info"]
generatorOptions: { disableNameSuffixHash: true }
---
# overlays/prod/kustomization.yaml
resources: ["../../base"]
patches:
- target: { kind: Deployment, name: api }
  patch: |
    - op: replace
      path: /spec/replicas
      value: 6
```

# 15) `kubectl` power tips (quick refs)

```bash
# Server-side apply & diff
kubectl diff -f deploy.yaml --server-side
kubectl apply -f deploy.yaml --server-side --force-conflicts

# JSONPath
kubectl get po -o jsonpath='{.items[*].status.containerStatuses[*].restartCount}'

# On-the-fly debug container
kubectl debug -it deploy/api --image=busybox:stable --target=api -- bash

# Port-forward to a pod behind a Service
kubectl -n prod port-forward svc/api-svc 8080:8080
```

---

If you want, I can turn this into a ready-to-run **Helm chart** (with values for HPA, PDB, probes, NetworkPolicies, PSA labels, and Kustomize overlays) or tailor these snippets to your stack (Ingress controller, CNI, cloud, and CI/CD).
