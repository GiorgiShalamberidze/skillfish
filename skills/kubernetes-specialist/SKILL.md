---
name: kubernetes-specialist
description: Kubernetes operations: deployments, services, ingress, Helm charts, operators, RBAC, resource limits, debugging pods, and cluster management for production.
---

# Kubernetes Specialist

Production-grade Kubernetes expertise for deploying, securing, scaling, and operating containerized workloads. Covers the full lifecycle from resource definitions through Helm packaging, RBAC hardening, observability, and GitOps-driven delivery across multiple environments.

## Table of Contents

1. [Core Resources](#1-core-resources)
2. [Services and Networking](#2-services-and-networking)
3. [Helm Charts](#3-helm-charts)
4. [RBAC and Security](#4-rbac-and-security)
5. [Resource Management](#5-resource-management)
6. [Storage](#6-storage)
7. [Debugging and Observability](#7-debugging-and-observability)
8. [Production Patterns](#8-production-patterns)

---

## 1. Core Resources

### Deployments

Manage stateless workloads with declarative updates and rollback support.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
  labels:
    app.kubernetes.io/name: api-server
    app.kubernetes.io/version: "2.4.0"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: api-server
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: api-server
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: api-server
          image: registry.example.com/api-server:2.4.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 15
            periodSeconds: 20
          startupProbe:
            httpGet: { path: /healthz, port: 8080 }
            failureThreshold: 30
            periodSeconds: 2
```

- **RollingUpdate** (default) -- gradually replaces pods. **Recreate** -- kills all pods first; use only when two versions cannot coexist.

```shell
kubectl rollout status deployment/api-server -n production
kubectl rollout history deployment/api-server -n production
kubectl rollout undo deployment/api-server -n production --to-revision=3
```

### StatefulSets

Use when pods need stable network identities and persistent storage (databases, brokers).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  selector:
    matchLabels: { app: postgres }
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports: [{ containerPort: 5432 }]
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef: { name: postgres-credentials, key: password }
          volumeMounts:
            - { name: data, mountPath: /var/lib/postgresql/data }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources: { requests: { storage: 50Gi } }
```

### DaemonSets

One pod per node -- typical for log collectors, monitoring agents, and CNI plugins.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentbit
  namespace: logging
spec:
  selector:
    matchLabels: { app: fluentbit }
  template:
    spec:
      tolerations: [{ operator: Exists }]
      containers:
        - name: fluentbit
          image: fluent/fluent-bit:3.0
          volumeMounts:
            - { name: varlog, mountPath: /var/log, readOnly: true }
      volumes:
        - name: varlog
          hostPath: { path: /var/log }
```

### Jobs and CronJobs

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 3600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: registry.example.com/db-backup:1.2.0
```

### Anti-patterns

- **Using `latest` tag in production.** Pin image versions. Mutable tags cause silent drift and break rollback.
- **Missing probes.** Without readiness probes traffic reaches unready pods; without liveness probes crashed processes stay running.
- **Setting `revisionHistoryLimit: 0`.** Prevents rollback. Keep at least 3-5 revisions.
- **StatefulSets for stateless workloads.** Unnecessary ordering constraints and complexity.

---

## 2. Services and Networking

### Service Types

```yaml
# ClusterIP -- internal only (default)
apiVersion: v1
kind: Service
metadata:
  name: api-server
spec:
  type: ClusterIP
  selector: { app.kubernetes.io/name: api-server }
  ports: [{ port: 80, targetPort: 8080 }]
---
# LoadBalancer -- provisions cloud LB (prefer Ingress for HTTP)
apiVersion: v1
kind: Service
metadata:
  name: api-server-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector: { app.kubernetes.io/name: api-server }
  ports: [{ port: 443, targetPort: 8080 }]
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: api-server, port: { number: 80 } }
```

### NetworkPolicies

Default-deny all traffic, then whitelist explicitly.

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
# Allow from ingress controller only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-api
spec:
  podSelector:
    matchLabels: { app.kubernetes.io/name: api-server }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: ingress-nginx }
      ports: [{ protocol: TCP, port: 8080 }]
```

Kubernetes DNS: `<service>.<namespace>.svc.cluster.local`. Short names resolve within the same namespace. Always use DNS names, never hardcoded ClusterIPs.

### Anti-patterns

- **One LoadBalancer per HTTP service.** Each provisions a cloud LB. Use a single Ingress controller for HTTP fan-out.
- **No NetworkPolicies.** Any pod can reach any other pod. Start with default-deny.
- **Hardcoding ClusterIP addresses.** IPs change on service recreation.

---

## 3. Helm Charts

### Chart Structure

```
my-chart/
  Chart.yaml              values.yaml
  values-staging.yaml     values-production.yaml
  templates/
    _helpers.tpl          deployment.yaml
    service.yaml          ingress.yaml
    hpa.yaml              NOTES.txt
  charts/                 tests/
```

```yaml
# Chart.yaml
apiVersion: v2
name: api-server
version: 1.3.0            # Chart version -- bump on any chart change
appVersion: "2.4.0"       # App version being deployed
dependencies:
  - name: postgresql
    version: "15.x"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: postgresql.enabled
```

### values.yaml

```yaml
replicaCount: 2
image:
  repository: registry.example.com/api-server
  tag: ""                  # Defaults to appVersion
  pullPolicy: IfNotPresent
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits: { cpu: 500m, memory: 512Mi }
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### Hooks and Commands

```yaml
# Pre-upgrade database migration hook
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          command: ["./migrate", "up"]
```

```shell
helm upgrade --install api-server ./my-chart \
  -n production --create-namespace \
  -f values-production.yaml --wait --timeout 5m

helm diff upgrade api-server ./my-chart -f values-production.yaml
helm test api-server -n production
helm push my-chart-1.3.0.tgz oci://registry.example.com/charts
```

### Anti-patterns

- **Secrets in values.yaml.** Use ExternalSecrets, SealedSecrets, or vault integration.
- **Not versioning chart separately from app.** Bump chart version when templates change, even if appVersion stays the same.
- **Skipping `helm diff` before production upgrades.** Unexpected diffs indicate drift.

---

## 4. RBAC and Security

### Roles and Bindings

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-deploy
  namespace: production
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-server
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/api-server-role
automountServiceAccountToken: false
```

Always create dedicated ServiceAccounts per workload. Never rely on the `default` account.

### Pod Security Standards

Enforce at the namespace level via labels (replaces deprecated PodSecurityPolicy):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
```

Restricted profile requires in each container:

```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }
  seccompProfile: { type: RuntimeDefault }
```

### External Secrets and Kyverno

```yaml
# External Secrets Operator -- pull from AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-credentials
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-secretsmanager, kind: ClusterSecretStore }
  target: { name: api-credentials, creationPolicy: Owner }
  data:
    - secretKey: database-url
      remoteRef: { key: production/api-server, property: database_url }
---
# Kyverno policy -- enforce resource limits on all pods
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-limits
      match:
        any: [{ resources: { kinds: [Pod] } }]
      validate:
        message: "CPU and memory limits are required."
        pattern:
          spec:
            containers:
              - resources:
                  limits: { memory: "?*", cpu: "?*" }
```

### Anti-patterns

- **ClusterRoleBindings for namespace-scoped needs.** Use RoleBindings to limit blast radius.
- **Storing raw secrets in Git.** Use SealedSecrets, ExternalSecrets, or SOPS. Never commit base64-encoded secrets.
- **Running containers as root.** Enforce `runAsNonRoot` and `readOnlyRootFilesystem` via PSS or Kyverno.

---

## 5. Resource Management

### Requests and Limits

```yaml
resources:
  requests: { cpu: 100m, memory: 128Mi }   # Scheduling guarantee
  limits: { cpu: 500m, memory: 512Mi }     # Throttled / OOMKilled
```

- Set requests to observed steady-state (p50). Set memory limits to 1.5-2x requests.
- CPU limits are controversial -- omitting them avoids throttling but risks noisy-neighbor issues.
- Always set memory limits. A leaking pod without them can destabilize an entire node.

### LimitRanges and ResourceQuotas

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default: { cpu: 200m, memory: 256Mi }
      defaultRequest: { cpu: 100m, memory: 128Mi }
      max: { cpu: "2", memory: 4Gi }
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
```

### HPA and VPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies: [{ type: Percent, value: 50, periodSeconds: 60 }]
    scaleDown:
      stabilizationWindowSeconds: 300
      policies: [{ type: Percent, value: 25, periodSeconds: 120 }]
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

Start VPA in `updateMode: "Off"` (recommendation only), review with `kubectl describe vpa`, then switch to `Auto`.

### PDB and Affinity

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels: { app.kubernetes.io/name: api-server }
```

```yaml
# Spread across AZs, prefer specific instance types
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - { key: node.kubernetes.io/instance-type, operator: In, values: ["m5.xlarge", "m5.2xlarge"] }
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels: { app.kubernetes.io/name: api-server }
          topologyKey: topology.kubernetes.io/zone
```

### Anti-patterns

- **No resource requests.** Scheduler cannot make informed decisions; leads to overcommitted nodes.
- **HPA and VPA on the same metric.** They fight. Use HPA for scale-out and VPA in recommendation mode for right-sizing.
- **Missing PDBs.** Node drains evict all pods simultaneously without a PDB.

---

## 6. Storage

### PVCs and StorageClasses

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources: { requests: { storage: 20Gi } }
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters: { type: gp3, iops: "5000", encrypted: "true" }
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

- **Delete** reclaim -- volume destroyed with PVC. Use for ephemeral data.
- **Retain** reclaim -- volume persists. Use for databases.

### Volume Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snap
spec:
  volumeSnapshotClassName: csi-aws-snapclass
  source:
    persistentVolumeClaimName: data-postgres-0
---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restored
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources: { requests: { storage: 50Gi } }
  dataSource:
    name: postgres-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### Backup with Velero

```shell
velero install --provider aws --bucket velero-backups --secret-file ./creds

velero schedule create daily-backup \
  --schedule="0 3 * * *" --ttl 720h \
  --include-namespaces production,data

velero restore create --from-backup daily-backup-20260405030000
```

### Anti-patterns

- **`Delete` reclaim for database volumes.** A mistaken `kubectl delete pvc` destroys your data.
- **`ReadWriteMany` when `ReadWriteOnce` suffices.** RWX requires shared-filesystem CSI drivers (EFS, NFS) -- slower and more expensive.
- **No backup strategy.** Snapshots alone are not backups. Back up to a separate region/account.

---

## 7. Debugging and Observability

### Essential kubectl Commands

```shell
# Cluster health
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running

# Pod inspection
kubectl describe pod <pod> -n <ns>
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Logs
kubectl logs <pod> -n <ns> --tail=100 -f
kubectl logs <pod> -n <ns> -c <container>      # specific container
kubectl logs <pod> -n <ns> --previous           # previous crash

# Resource usage
kubectl top nodes
kubectl top pods -n <ns> --sort-by=memory
```

### Ephemeral Debug Containers

```shell
# Attach debug container to running pod (for distroless images)
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot --target=<container>

# Copy pod with debug sidecar
kubectl debug <pod> -n <ns> --copy-to=debug-pod --image=busybox:1.36 --share-processes

# Debug a node
kubectl debug node/<node> -it --image=ubuntu:22.04
```

### Common Scenarios

**CrashLoopBackOff:** Check `kubectl logs --previous`, look for OOMKilled in `kubectl describe pod`, verify config/secret mounts.

**ImagePullBackOff:** Verify image tag exists, check `imagePullSecrets`, validate registry credentials.

**Pending:** Check events for scheduling failures -- insufficient resources, unschedulable nodes, unbound PVCs.

### Prometheus Stack

```shell
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f monitoring-values.yaml
```

```yaml
# ServiceMonitor for app metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-server-metrics
spec:
  selector:
    matchLabels: { app.kubernetes.io/name: api-server }
  endpoints: [{ port: metrics, interval: 15s, path: /metrics }]
---
# Alert on high error rate
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-alerts
spec:
  groups:
    - name: api-server
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{job="api-server",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{job="api-server"}[5m])) > 0.05
          for: 5m
          labels: { severity: critical }
        - alert: PodRestarting
          expr: increase(kube_pod_container_status_restarts_total{namespace="production"}[1h]) > 3
          for: 10m
          labels: { severity: warning }
```

### Anti-patterns

- **Logging to local files.** Logs are lost on restart. Write to stdout/stderr; let the collector aggregate.
- **No alerting on restarts.** CrashLoopBackOff goes unnoticed until users report outages.
- **`kubectl exec` as standard ops practice.** Automate operational tasks with Jobs or CronJobs.
- **Missing ServiceMonitors.** Prometheus will not scrape your app even if it exposes `/metrics`.

---

## 8. Production Patterns

### Multi-Environment with Kustomize

```
clusters/
  staging/   { kustomization.yaml, patches/ }
  production/{ kustomization.yaml, patches/ }
base/
  deployment.yaml  service.yaml  kustomization.yaml
```

```yaml
# clusters/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: [../../base]
namespace: production
patches:
  - path: patches/increase-replicas.yaml
  - path: patches/production-resources.yaml
images:
  - name: registry.example.com/api-server
    newTag: "2.4.0"
```

### GitOps with ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-server-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: main
    path: clusters/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
    retry: { limit: 3, backoff: { duration: 5s, factor: 2, maxDuration: 3m } }
```

### GitOps with Flux

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: k8s-manifests
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/k8s-manifests.git
  ref: { branch: main }
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: production
  namespace: flux-system
spec:
  interval: 5m
  sourceRef: { kind: GitRepository, name: k8s-manifests }
  path: ./clusters/production
  prune: true
  timeout: 3m
```

### Blue-Green Deployments

Maintain two full deployments; switch traffic by updating the Service selector.

```shell
# Deploy green version, verify health, then switch
kubectl apply -f deployment-green.yaml
kubectl rollout status deployment/api-server-green -n production

kubectl patch service api-server -n production \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback: switch back to blue
kubectl patch service api-server -n production \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Canary with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-server
spec:
  replicas: 10
  selector:
    matchLabels: { app.kubernetes.io/name: api-server }
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 30
        - pause: { duration: 5m }
        - setWeight: 60
        - pause: { duration: 5m }
      canaryService: api-server-canary
      stableService: api-server-stable
      analysis:
        templates: [{ templateName: success-rate }]
        startingStep: 1
```

### Cluster Upgrades

```shell
# EKS: upgrade control plane, then node groups, then add-ons
eksctl upgrade cluster --name=production --version=1.30 --approve
eksctl upgrade nodegroup --cluster=production --name=workers --kubernetes-version=1.30
```

**Checklist:** (1) Run pluto/kubent for deprecated APIs. (2) Test in staging first. (3) Ensure PDBs exist. (4) Upgrade one minor version at a time. (5) Update Helm charts and CRDs.

### Disaster Recovery

```shell
velero backup create pre-upgrade-backup --include-namespaces production,data --wait
```

- **RTO < 1 hour:** Active-passive cluster with Velero cross-region restore.
- **RTO < 15 minutes:** Active-active multi-cluster with global load balancing.
- **RPO = 0:** Synchronous cross-region database replication.

### Anti-patterns

- **Manual `kubectl apply` in production.** Use GitOps. Every change through a reviewed PR.
- **Upgrading multiple minor versions at once.** Skips deprecation warnings and breaks workloads on removed APIs.
- **No progressive delivery.** Big-bang deployments risk full outages.
- **GitOps without drift detection.** Disable `selfHeal` in ArgoCD and manual changes accumulate silently.
- **No DR testing.** A backup you have never restored is not a backup.
