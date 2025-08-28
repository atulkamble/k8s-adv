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


Perfect—here’s a **production-grade Helm chart** you can drop into a repo and install. It bakes in HPA, PDB, probes, NetworkPolicy, PSA labels, Prometheus scrape hints, config/secret checksums, and canary/blue-green hooks via values.

---

# Repo layout

```
k8s-app-chart/
├─ Chart.yaml
├─ values.yaml
├─ values-prod.yaml         # example prod overrides
├─ templates/
│  ├─ _helpers.tpl
│  ├─ deployment.yaml
│  ├─ service.yaml
│  ├─ ingress.yaml
│  ├─ hpa.yaml
│  ├─ pdb.yaml
│  ├─ configmap.yaml
│  ├─ secret.yaml
│  ├─ networkpolicy.yaml
│  ├─ serviceaccount.yaml
│  ├─ rbac.yaml
│  ├─ servicemonitor.yaml
│  └─ NOTES.txt
└─ .helmignore
```

---

# Chart.yaml

```yaml
apiVersion: v2
name: k8s-app-chart
description: Production-ready app chart with HPA, PDB, NetworkPolicy, and observability
type: application
version: 0.1.0
appVersion: "1.2.3"
kubeVersion: ">=1.26.0-0"
```

# .helmignore

```gitignore
.git/
.gitignore
README.md
*.swp
*.bak
.DS_Store
```

---

# values.yaml (editable defaults)

```yaml
nameOverride: ""
fullnameOverride: ""

replicaCount: 3

image:
  repository: ghcr.io/org/api
  tag: "1.2.3"
  pullPolicy: IfNotPresent

imagePullSecrets: []     # e.g., [{ name: regcred }]

serviceAccount:
  create: true
  name: ""
  annotations: {}
  automountServiceAccountToken: false

rbac:
  create: true
  rulesReadOnly:
    - apiGroups: [""]
      resources: ["pods","services","endpoints","configmaps"]
      verbs: ["get","list","watch"]

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"

podSecurityContext:
  runAsNonRoot: true
  seccompProfile: { type: RuntimeDefault }

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080
  annotations: {}

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts: ["api.example.com"]
      secretName: tls-api

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"

nodeSelector: {}
tolerations: []
affinity: {}
topologySpreadConstraints: []
# Example:
# topologySpreadConstraints:
# - maxSkew: 1
#   topologyKey: topology.kubernetes.io/zone
#   whenUnsatisfiable: ScheduleAnyway
#   labelSelector: {}

env:            # key=val pairs -> env vars
  APP_LOG_LEVEL: info
  APP_FEATURE_X: "true"

envFromSecret: []  # e.g., ["db-secret"] if you already have existing secrets

config:
  enabled: true
  data:
    application.yaml: |
      server:
        port: 8080
      featureX: true

secret:
  enabled: true
  stringData:
    DATABASE_URL: "postgres://user:pass@postgres:5432/app"

probes:
  readiness:
    httpGet: { path: /healthz/ready, port: 8080 }
    periodSeconds: 5
    failureThreshold: 2
  liveness:
    httpGet: { path: /healthz/live, port: 8080 }
    initialDelaySeconds: 30
  startup:
    httpGet: { path: /healthz/startup, port: 8080 }
    failureThreshold: 30
    periodSeconds: 2

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    # - type: Pods
    #   pods:
    #     metric: { name: http_requests_per_second }
    #     target: { type: AverageValue, averageValue: "50" }

pdb:
  enabled: true
  minAvailable: 2

networkPolicy:
  enabled: true
  defaultDeny: true
  allowNamespaceSelectors: []  # list of labelSelectors to allow ingress from
  allowFromPods: []            # list of { matchLabels: {...} }
  allowToPorts:
    - port: 5432
      protocol: TCP
  egressCIDRs: []              # e.g., ["0.0.0.0/0"] if you need outbound internet

serviceMonitor:
  enabled: false
  interval: 30s
  scrapeTimeout: 10s
  labels: {}

podSecurityAdmission:
  labels:
    enforce: "restricted"
    version: "latest"

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1

extraInitContainers: []   # full container specs
extraContainers: []       # sidecars
extraVolumes: []
extraVolumeMounts: []

deploymentAnnotations: {}
deploymentLabels: {}

# Canary / blue-green hints (used in labels/selectors)
color: "blue"   # switch to "green" during cutover
```

---

# values-prod.yaml (example overrides for prod)

```yaml
replicaCount: 6
image:
  tag: "1.2.4"
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "2",    memory: "1Gi" }
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts: ["api.yourdomain.com"]
      secretName: tls-api
networkPolicy:
  allowFromPods:
    - matchLabels: { app: gateway }
  allowToPorts:
    - port: 5432
      protocol: TCP
serviceMonitor:
  enabled: true
autoscaling:
  minReplicas: 6
  maxReplicas: 60
```

---

# templates/\_helpers.tpl

```yaml
{{- define "k8s-app-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{- define "k8s-app-chart.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name (include "k8s-app-chart.name" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end }}

{{- define "k8s-app-chart.labels" -}}
app.kubernetes.io/name: {{ include "k8s-app-chart.name" . }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- if .Values.color }} app: api
color: {{ .Values.color | quote }} {{- end }}
{{- end }}

{{/* Used to trigger rollout when config/secret change */}}
{{- define "k8s-app-chart.configChecksum" -}}
{{- if .Values.config.enabled -}}
{{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
{{- end -}}
{{- end }}

{{- define "k8s-app-chart.secretChecksum" -}}
{{- if .Values.secret.enabled -}}
{{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
{{- end -}}
{{- end }}
```

---

# templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}
  labels:
    {{- include "k8s-app-chart.labels" . | nindent 4 }}
  annotations:
    {{- toYaml .Values.deploymentAnnotations | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "k8s-app-chart.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
      {{- if .Values.color }} color: {{ .Values.color | quote }} {{- end }}
  strategy:
    {{- toYaml .Values.strategy | nindent 4 }}
  template:
    metadata:
      labels:
        {{- include "k8s-app-chart.labels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include "k8s-app-chart.configChecksum" . | quote }}
        checksum/secret: {{ include "k8s-app-chart.secretChecksum" . | quote }}
        {{- toYaml .Values.podAnnotations | nindent 8 }}
    spec:
      {{- if .Values.serviceAccount.create }}
      serviceAccountName: {{ include "k8s-app-chart.fullname" . }}
      {{- else if .Values.serviceAccount.name }}
      serviceAccountName: {{ .Values.serviceAccount.name }}
      {{- end }}
      {{- if .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml .Values.imagePullSecrets | nindent 8 }}
      {{- end }}
      securityContext: {{- toYaml .Values.podSecurityContext | nindent 8 }}
      {{- with .Values.affinity }}
      affinity: {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector: {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations: {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.topologySpreadConstraints }}
      topologySpreadConstraints: {{- toYaml . | nindent 8 }}
      {{- end }}
      initContainers:
        {{- if .Values.extraInitContainers }}
        {{- toYaml .Values.extraInitContainers | nindent 8 }}
        {{- end }}
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports: [{ containerPort: {{ .Values.service.targetPort }}, name: http }]
          env:
            {{- range $k, $v := .Values.env }}
            - name: {{ $k }}
              value: {{ $v | quote }}
            {{- end }}
          envFrom:
            {{- range .Values.envFromSecret }}
            - secretRef: { name: {{ . | quote }} }
            {{- end }}
          volumeMounts:
            {{- if .Values.config.enabled }}
            - name: app-config
              mountPath: /etc/app
            {{- end }}
            {{- if .Values.extraVolumeMounts }}
            {{- toYaml .Values.extraVolumeMounts | nindent 12 }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.probes.readiness | nindent 12 }}
          livenessProbe:
            {{- toYaml .Values.probes.liveness | nindent 12 }}
          startupProbe:
            {{- toYaml .Values.probes.startup | nindent 12 }}
        {{- if .Values.extraContainers }}
        {{- toYaml .Values.extraContainers | nindent 8 }}
        {{- end }}
      volumes:
        {{- if .Values.config.enabled }}
        - name: app-config
          projected:
            sources:
              - configMap:
                  name: {{ include "k8s-app-chart.fullname" . }}
              {{- if .Values.secret.enabled }}
              - secret:
                  name: {{ include "k8s-app-chart.fullname" . }}
              {{- end }}
        {{- end }}
        {{- if .Values.extraVolumes }}
        {{- toYaml .Values.extraVolumes | nindent 8 }}
        {{- end }}
```

---

# templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}-svc
  labels:
    {{- include "k8s-app-chart.labels" . | nindent 4 }}
  annotations:
    {{- toYaml .Values.service.annotations | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app.kubernetes.io/name: {{ include "k8s-app-chart.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    {{- if .Values.color }} color: {{ .Values.color | quote }} {{- end }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
```

---

# templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}
  labels:
    {{- include "k8s-app-chart.labels" . | nindent 4 }}
  annotations:
    {{- toYaml .Values.ingress.annotations | nindent 4 }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  tls:
    {{- toYaml .Values.ingress.tls | nindent 4 }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType | default "Prefix" }}
            backend:
              service:
                name: {{ include "k8s-app-chart.fullname" $ }}-svc
                port: { number: {{ $.Values.service.port }} }
          {{- end }}
    {{- end }}
{{- end }}
```

---

# templates/hpa.yaml

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}
  labels:
    {{- include "k8s-app-chart.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "k8s-app-chart.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- toYaml .Values.autoscaling.metrics | nindent 4 }}
{{- end }}
```

---

# templates/pdb.yaml

```yaml
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}
  labels:
    {{- include "k8s-app-chart.labels" . | nindent 4 }}
spec:
  {{- if hasKey .Values.pdb "minAvailable" }}
  minAvailable: {{ .Values.pdb.minAvailable }}
  {{- else if hasKey .Values.pdb "maxUnavailable" }}
  maxUnavailable: {{ .Values.pdb.maxUnavailable }}
  {{- end }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "k8s-app-chart.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

---

# templates/configmap.yaml

```yaml
{{- if .Values.config.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}
  labels:
    {{- include "k8s-app-chart.labels" . | nindent 4 }}
data:
  {{- range $k, $v := .Values.config.data }}
  {{ $k }}: |
{{ $v | indent 4 }}
  {{- end }}
{{- end }}
```

# templates/secret.yaml

```yaml
{{- if .Values.secret.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}
  labels:
    {{- include "k8s-app-chart.labels" . | nindent 4 }}
type: Opaque
stringData:
  {{- toYaml .Values.secret.stringData | nindent 2 }}
{{- end }}
```

---

# templates/networkpolicy.yaml

```yaml
{{- if .Values.networkPolicy.enabled }}
{{- if .Values.networkPolicy.defaultDeny }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}-default-deny
spec:
  podSelector: {}
  policyTypes: ["Ingress","Egress"]
---
{{- end }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}-rules
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: {{ include "k8s-app-chart.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  policyTypes: ["Ingress","Egress"]
  ingress:
    - from:
        {{- range .Values.networkPolicy.allowNamespaceSelectors }}
        - namespaceSelector: {{- toYaml . | nindent 12 }}
        {{- end }}
        {{- range .Values.networkPolicy.allowFromPods }}
        - podSelector: {{- toYaml . | nindent 12 }}
        {{- end }}
  egress:
    - to:
        {{- range .Values.networkPolicy.egressCIDRs }}
        - ipBlock: { cidr: {{ . | quote }} }
        {{- end }}
      ports:
        {{- range .Values.networkPolicy.allowToPorts }}
        - protocol: {{ .protocol | default "TCP" }}
          port: {{ .port }}
        {{- end }}
{{- end }}
```

---

# templates/serviceaccount.yaml

```yaml
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}
  annotations:
    {{- toYaml .Values.serviceAccount.annotations | nindent 4 }}
automountServiceAccountToken: {{ .Values.serviceAccount.automountServiceAccountToken | default false }}
{{- end }}
```

# templates/rbac.yaml

```yaml
{{- if and .Values.rbac.create .Values.serviceAccount.create }}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}-reader
rules:
  {{- toYaml .Values.rbac.rulesReadOnly | nindent 2 }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}-reader
subjects:
  - kind: ServiceAccount
    name: {{ include "k8s-app-chart.fullname" . }}
roleRef:
  kind: Role
  name: {{ include "k8s-app-chart.fullname" . }}-reader
  apiGroup: rbac.authorization.k8s.io
{{- end }}
```

---

# templates/servicemonitor.yaml

```yaml
{{- if .Values.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "k8s-app-chart.fullname" . }}
  labels:
    {{- include "k8s-app-chart.labels" . | nindent 4 }}
    {{- toYaml .Values.serviceMonitor.labels | nindent 4 }}
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "k8s-app-chart.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  endpoints:
    - port: http
      interval: {{ .Values.serviceMonitor.interval }}
      scrapeTimeout: {{ .Values.serviceMonitor.scrapeTimeout }}
      path: /metrics
{{- end }}
```

---

# templates/NOTES.txt

```txt
1) Service:
   kubectl -n {{ .Release.Namespace }} get svc {{ include "k8s-app-chart.fullname" . }}-svc

2) Ingress (if enabled):
   kubectl -n {{ .Release.Namespace }} get ingress {{ include "k8s-app-chart.fullname" . }}

3) HPA:
   kubectl -n {{ .Release.Namespace }} get hpa {{ include "k8s-app-chart.fullname" . }}

4) Rollout status:
   kubectl -n {{ .Release.Namespace }} rollout status deploy/{{ include "k8s-app-chart.fullname" . }}
```

---

## Install & use

```bash
# From chart root
helm lint .

# Dry-run / render
helm template myapp ./k8s-app-chart \
  --namespace prod --create-namespace \
  -f values.yaml

# Install (dev defaults)
helm upgrade --install myapp ./k8s-app-chart -n dev

# Install with prod overrides
helm upgrade --install myapp ./k8s-app-chart -n prod -f values-prod.yaml

# Check rollout
kubectl -n prod rollout status deploy/myapp-k8s-app-chart

# Scale via values or temporarily
kubectl -n prod scale deploy/myapp-k8s-app-chart --replicas=10
```

---

## Blue/Green & Canary

* **Blue/Green**: flip `.Values.color` from `blue` → `green`. Your `Service` selector includes `color`, so cutover is instant.
* **Canary with NGINX Ingress**: add to `ingress.annotations`:

```yaml
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-weight: "20"
```

---

## Notes & prerequisites

* **HPA custom metrics** require a metrics adapter (e.g., Prometheus Adapter) if you enable the Pods metric example.
* **ServiceMonitor** requires the Prometheus Operator CRDs installed.
* **NetworkPolicy** enforcement needs a CNI that supports it (Calico/Cilium/Weave, etc.).
* **PSA**: Add namespace labels when you create the `prod` namespace:

  ```bash
  kubectl create ns prod
  kubectl label ns prod pod-security.kubernetes.io/enforce=restricted \
      pod-security.kubernetes.io/enforce-version=latest
  ```

---

If you want, I can tailor this chart to **EKS (gp3 StorageClass, IRSA), AKS (Azure Workload Identity), or GKE (Workload Identity)** and wire in CI/CD (GitHub Actions/Jenkins) to push checksumed releases automatically.

