# REST API Helm Chart

Helm chart for deploying a stateless REST API on Kubernetes.

The chart covers multi-environment deployment, security hardening, health checks, autoscaling, monitoring with alerting, and zero-downtime updates.

All features are configurable via `values.yaml`. Optional resources (Ingress, HPA, PDB, ServiceMonitor, PrometheusRule) are disabled by default and enabled per environment.

## Demo application

The chart uses [podinfo](https://github.com/stefanprodan/podinfo) as a demo app. It was chosen because it's a real Go microservice that exposes `/healthz`, `/readyz`, `/metrics` out of the box - so all probes and monitoring work without extra setup.

### Using your own application

Replace the image and adjust ports:

```yaml
image:
  repository: your-registry/your-app
  tag: "1.0.0"                    # pinned tag, not "latest"
app:
  listenPort: 8080
```

If your app needs writable filesystem:

```yaml
securityContext:
  readOnlyRootFilesystem: false
```


Health check paths - update if your app uses different endpoints:

```yaml
probes:
  readiness:
    path: /health
  liveness:
    path: /health
```

## Project Structure

```
├── rest-api/                          # Helm chart
│   ├── Chart.yaml                     # chart metadata (name, version)
│   ├── values.yaml                    # default values for all environments
│   ├── templates/
│   │   ├── _helpers.tpl               # reusable template helpers (labels, names)
│   │   ├── deployment.yaml            # pods, containers, probes, security
│   │   ├── service.yaml               # ClusterIP network service
│   │   ├── configmap.yaml             # env vars (LISTEN_HOST, LISTEN_PORT)
│   │   ├── serviceaccount.yaml        # dedicated SA with no K8s API access
│   │   ├── ingress.yaml               # external access (conditional)
│   │   ├── hpa.yaml                   # horizontal pod autoscaler (conditional)
│   │   ├── pdb.yaml                   # pod disruption budget (conditional)
│   │   ├── servicemonitor.yaml        # Prometheus scraping config (conditional)
│   │   ├── prometheusrule.yaml        # alerting rules (conditional)
│   │   ├── NOTES.txt                  # post-install instructions
│   │   └── tests/
│   │       └── test-connection.yaml   # helm test - checks /healthz endpoint
│   └── .helmignore
├── values-staging.yaml                # staging overrides
├── values-production.yaml             # production overrides
└── README.md
```

All chart parameters are documented with comments in [values.yaml](rest-api/values.yaml).

## Prerequisites

- Docker
- Kubernetes cluster (tested by author on Minikube)
- Helm
- Ingress Controller (e.g. nginx) - if `ingress.enabled: true`
- Prometheus Operator - if `serviceMonitor.enabled: true` or `prometheusRule.enabled: true`

## Usage

### Validate

```bash
helm lint rest-api/
helm template rest-api ./rest-api
helm install rest-api ./rest-api --dry-run
```

### Deploy

```bash
# staging
helm upgrade --install rest-api ./rest-api -n staging --create-namespace -f values-staging.yaml

# production
helm upgrade --install rest-api ./rest-api -n production --create-namespace -f values-production.yaml
```

### Upgrade

```bash
helm upgrade rest-api ./rest-api -n staging -f values-staging.yaml
helm upgrade rest-api ./rest-api -n production -f values-production.yaml
```

### Rollback

```bash
# rollback to previous revision
helm rollback rest-api -n production
```

### Verify

```bash
kubectl get all -n production
helm list -A
helm test rest-api -n production

# application logs
kubectl logs -n production -l app.kubernetes.io/name=rest-api
```

### Access locally

```bash
kubectl port-forward svc/rest-api 8080:80 -n production
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl http://localhost:8080/metrics
```

## Multi-Environment Setup

The chart uses one set of templates and different values files per environment.
Environment-specific files live in the project root, not inside the chart - this keeps the chart portable and reusable.

```
├── rest-api/              # chart (generic)
│   └── values.yaml        # defaults
├── values-staging.yaml    # staging overrides
└── values-production.yaml # production overrides
```

### What changes between environments


|                   | Default  | Staging                      | Production                           |
| ----------------- | -------- | ---------------------------- | ------------------------------------ |
| Replicas          | 1        | 1 (default)                  | 3                                    |
| Resources         | 50m/64Mi | 50m/64Mi (default)           | 100m/128Mi                           |
| HPA               | off      | off                          | on (3–20 pods, CPU 70%, Memory 80%)  |
| PDB               | off      | off                          | on (minAvailable: 30%)               |
| Ingress           | off      | on                           | on (+rate limiting)|
| Pod anti-affinity | off      | off                          | on (spread across nodes)             |
| Topology spread   | off      | off                          | on (spread across AZs)               |
| ServiceMonitor    | off      | on                           | on                                   |
| PrometheusRule    | off      | off                          | on                                   |


### Why these choices

**Staging** mirrors defaults, but adds Ingress and ServiceMonitor.
The goal is to catch monitoring and routing issues before they reach production, without the cost of running extra replicas or HPA.

**Production** enables what's needed for reliability:

- 3+ replicas with HPA so the app scales under load (now 20 max - should be adjusted for real prod needs)
- PDB uses `minAvailable: 30%` instead of a fixed number. A percentage scales automatically with replicas - whether HPA runs 3 or 20 pods, Kubernetes always knows how many it can evict safely. With a fixed value like `minAvailable: 2` and only 2 replicas, Kubernetes can't evict any pod at all - node drains and cluster upgrades get stuck
- Pod anti-affinity spreads pods across different nodes - if one node dies, not all replicas go down
- Topology spread distributes pods across availability zones - if an entire AZ goes down, the service keeps running
- Rate limiting on Ingress protects from traffic spikes
- PrometheusRule fires alerts when something goes wrong
- Resource requests (100m/128Mi) look small for production, but this depends on the actual app - start low and add pods as needed

**All environments** share these deploy settings:

- RollingUpdate with `maxUnavailable: 0` - new pods start before old ones stop, so there's no downtime during deploys
- `terminationGracePeriodSeconds: 30` - needed during deploys, scale-downs, etc. 30s is enough for a typical REST API; increase for long-running operations (report generation, file uploads)
- Pods auto-restart when ConfigMap changes (via checksum annotation on the Deployment)
- CPU limits are set 4x higher than requests (e.g. request 50m, limit 200m). If the limit is too close to the request, Kubernetes slows down the container and response times spike. Some teams remove CPU limits entirely and rely only on requests to avoid this

## Security

### What is implemented

**Pod and container hardening:**

- Containers run as non-root (UID 1000, `runAsNonRoot: true`) - if the container is compromised, the attacker doesn't get root on the host
- All Linux capabilities are dropped (`capabilities.drop: [ALL]`) - the process can't mount filesystems, change network config, or load kernel modules
- Privilege escalation is blocked (`allowPrivilegeEscalation: false`) - a process can't gain more privileges than its parent
- Seccomp profile is set to `RuntimeDefault` - blocks dangerous syscalls at kernel level

**ServiceAccount isolation:**

- Dedicated ServiceAccount with no RBAC bindings
- `automountServiceAccountToken: false` - the pod has no access to the Kubernetes API
- The SA also works as a hook for cloud IAM (e.g. GCP Workload Identity, AWS IRSA)

**Secrets:**

- DB passwords, API keys, tokens should never go in ConfigMap or values files
- In production, the ideal setup: secrets are stored in an external vault (HashiCorp Vault, AWS SSM, GCP Secret Manager), and syncs into Kubernetes Secrets automatically. The app reads them as env vars or mounted files - same as ConfigMap, but the values never appear in git or Helm values

**Network:**

- Service type is ClusterIP - not exposed externally unless Ingress is enabled
- Production Ingress includes rate limiting annotations to protect from traffic spikes

**TLS:**

Ingress TLS is not configured by default. In most cloud setups (GCP, AWS, Azure), the external load balancer handles SSL termination, so the Ingress works over HTTP behind it. Can be configured by `tls` param

### Not implemented (and why)

- `readOnlyRootFilesystem` - not set because many real apps need writable /tmp. Can be enabled per app in values
- NetworkPolicy - firewall rules between pods (e.g. "only the API can talk to DB on port 5432"). Depend on cluster CNI plugin, better managed at platform level
- Image digest pinning - the chart uses a pinned tag (`6.10.2`). Digest pinning (`image@sha256:...`) is more strict and recommended for regulated environments, but is usually handled by CI/CD, not the chart
- Image scanning - use Trivy or Snyk in the CI/CD pipeline, not in the Helm chart
- Secrets management - the chart has no Kubernetes Secret templates. Secrets should come from an external source (Vault, AWS SSM, GCP Secret Manager) via external-secrets operator, not from Helm. This keeps secrets out of git and values files

## Monitoring & Alerting

The chart is monitoring-ready. It works with any Prometheus setup, from basic annotations to full Prometheus Operator stack.

### How it works

The monitoring has three layers:

**1. Prometheus annotations (always on)**

Pods get `prometheus.io/scrape`, `/port`, `/path` annotations. Any Prometheus instance that does annotation-based discovery will pick them up automatically. Zero config needed.

**2. ServiceMonitor (optional, needs Prometheus Operator)**

A more reliable way to configure scraping. Instead of Prometheus scanning all pods for annotations, ServiceMonitor explicitly tells it: scrape this Service, on this port, every 30s. Enable with `serviceMonitor.enabled: true`.

**3. PrometheusRule (optional, needs Prometheus Operator)**

Alerting rules that Prometheus evaluates continuously. When a condition is true for a specified duration, it fires an alert to Alertmanager. Enable with `prometheusRule.enabled: true`.

### Alerts

11 alerts grouped by logic. Thresholds are parameterized in `values.yaml` - override per environment in `values-production.yaml`.

Default thresholds are industry-standard starting points: alert at 80% resource usage so you have time to react before the container hits the limit and gets killed (memory) or slowed down (CPU). 5% error rate catches real problems without false alarms. 500ms/1s latency targets match typical REST API SLAs.

**Availability (pod health):**


| Alert            | Severity | Condition                                             |
| ---------------- | -------- | ----------------------------------------------------- |
| PodDown          | critical | All replicas are down for 5 min                       |
| PodNotReady      | warning  | Pod is running but failing readiness checks for 5 min |
| PodPending       | warning  | Pod stuck in Pending for 10 min (scheduling issues)   |
| HighRestartCount | warning  | More than 3 restarts in 15 min (crash loop)           |


**Resources - for infrastructure health:**

| Alert           | Severity | Condition                           |
| --------------- | -------- | ----------------------------------- |
| HighCPUUsage    | warning  | CPU above 80% of limit for 5 min    |
| HighMemoryUsage | warning  | Memory above 80% of limit for 5 min |


**Traffic (RED - Rate, Errors, Duration):**


| Alert           | Severity | Condition                                              |
| --------------- | -------- | ------------------------------------------------------ |
| HighErrorRate   | critical | More than 5% of requests return 5xx for 5 min          |
| HighLatencyP99  | critical | 99th percentile latency above 1s for 5 min             |
| HighLatencyP95  | warning  | 95th percentile latency above 500ms for 5 min          |
| HighTrafficRate | warning  | Request rate above 1000 rps for 5 min (possible DDoS)  |
| LowTrafficRate  | warning  | Request rate below 0.1 rps for 30 min (upstream issue) |

### Override thresholds per environment

Staging and production have different traffic patterns. In staging, 5% error rate is fine - you're testing. In production, you want to catch problems earlier, so you lower the threshold:

```yaml
# values-production.yaml
prometheusRule:
  enabled: true
  thresholds:
    errorRate: 0.01    # 1% instead of default 5%
    latencyP99: 0.5    # 500ms instead of default 1s
```

### Installing Prometheus Operator

ServiceMonitor and PrometheusRule require Prometheus Operator on the cluster. The easiest way to get it:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

This installs Prometheus, Alertmanager, Grafana, kube-state-metrics, and node-exporter.

### What the chart does NOT include

The chart provides metrics endpoint, scraping config, and alerting rules. The rest of the observability stack is infrastructure-level - install it separately:

- **Prometheus + Alertmanager** - scrapes metrics, evaluates alert rules, routes notifications to Slack/PagerDuty/email. Install via kube-prometheus-stack (see above)
- **Grafana dashboards** - visualize request rate, latency percentiles, error rate, CPU/memory per pod. Recommended panels: RPS by status code, latency, pod restarts over time, uptime and other SLO/SLA metrics
- **Log aggregation** - containers write logs to stdout, Kubernetes stores them on the node but only until the pod is deleted. To keep logs longer and search across pods, use Promtail/Fluentbit + Loki/ELK/Cloud Logging
- **Distributed tracing** - for request flows across microservices, use OpenTelemetry + Jaeger or Tempo. Requires instrumentation in application code

## Deploying at Scale

### What the chart already supports

- **HPA** scales pods horizontally based on CPU and memory. In production config: 3 to 20 replicas (can be easily adjusted manually or through Cluster Autoscaler)
- **PDB** protects availability during node maintenance and cluster upgrades
- **Pod anti-affinity** spreads replicas across nodes so a single node failure doesn't take down the service
- **Topology spread** distributes pods across availability zones for AZ-level fault tolerance
- **Zero-downtime deploys** via RollingUpdate with `maxUnavailable: 0`
- **nodeSelector** and **tolerations** for pinning pods to specific node pools (e.g. GPU, high-memory) or dedicated nodes

### Resource calibration

Default resource values in the chart are starting points. In production, we should calibrate them:

1. Deploy with defaults
2. Run VPA in read-only mode - it observes real usage and suggests requests/limits
3. Monitor `container_cpu_cfs_throttled_periods_total` in Grafana - if throttling > 5-10%, CPU limits are too tight
4. Adjust after 1-2 weeks of real traffic

### Multi-environment strategy

One chart, multiple values files, namespace-per-environment:

```bash
helm upgrade --install rest-api ./rest-api -n staging -f values-staging.yaml
helm upgrade --install rest-api ./rest-api -n production -f values-production.yaml
```

Each environment is isolated by namespace. The same chart deploys everywhere - only configuration differs. If needed override `image.tag` per environment in values files or pass it via CI/CD (`--set image.tag=1.2.3`).

### What changes when the app grows

**Adding a database:**

- The app is no longer fully stateless - need to think about migrations, backups, failover
- Connection pooling (PgBouncer, ProxySQL) between app and DB - without it, each pod opens its own connections and the DB runs out of slots under HPA scaling
- DB credentials go through external-secrets, not ConfigMap
- Health checks should reflect DB connectivity (readiness probe checks DB connection)
- New alerts needed: connection pool exhaustion, slow queries, disk usage, etc.
- Consider a managed service (RDS, Cloud SQL) over self-hosted - less operational overhead
- Caching layer (Redis) between app and DB to reduce read load
- CDN (Cloudflare, CloudFront) in front of the API reduces backend load for cacheable responses - useful even with a single service

**Microservices:**

- Each service gets its own Helm chart (or use a shared library chart)
- GitOps (ArgoCD/Flux) - store values files in git, ArgoCD syncs them to clusters automatically. No manual `helm upgrade` across dozens of services. One change in git deploys to all environments. Monolith is possible without GitOps, but for microservices it is strongly recommended
- Service mesh (Istio/Linkerd) - encrypts traffic between services automatically (mTLS) and shows which service calls which, how often, and how slow. Without it, inter-service communication is unencrypted and hard to debug
- Shared Ingress with path-based or host-based routing, or API gateway (Kong, Ambassador) when you need per-user rate limits, authentication (JWT, API keys), and usage quotas. For a single API, Ingress is enough
- Message queues (RabbitMQ, Kafka) - this decouples services, absorbs traffic spikes, and makes the system easier to scale independently

### Scaling even more - to millions of users

For higher-scale production:

**Infrastructure:**
- **Cluster autoscaling** - HPA scales pods, but nodes also need to scale. Cluster Autoscaler or Karpenter automatically add/remove nodes when pods can't be scheduled
- **Multi-cluster** - run more clusters across regions (e.g. us-east, eu-west). Use DNS-based routing (Route53, Cloud DNS) or a global load balancer (GCP GLB, Cloudflare) to send users to the nearest cluster

**Deploy strategy:**
- **Canary / Blue-Green deploys** - to gradually shift traffic to new versions. If error rate spikes, it rolls back automatically without showing bad application to all users

**Data:**
- **Read replicas** for the database to separate read and write traffic
- **Database sharding** - when one master can't handle all writes, split data across multiple databases (e.g. by user ID or region). Adds complexity but removes the single-writer bottleneck
- Horizontal scaling has limits - at some point, optimize the app itself (caching, connection pooling, async processing)

**Observability:**
- **Centralized metrics** - with multiple clusters you need a single place to see metrics from all of them. Use Thanos or Grafana Mimir on top of per-cluster Prometheus
- **Centralized logging** - you can't `kubectl logs` across 20 services in 5 clusters. Aggregate all logs into one place (Loki, ELK, CloudWatch) so you can search and correlate across everything
- Observability becomes critical at scale - dashboards per service, per dependency, end-to-end latency breakdown. Without it, debugging production issues in a distributed system is a nightmare