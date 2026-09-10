# Monitoring with Prometheus

This repo showcases

- installing Prometheus Stack in Kubernetes
- configuring alerting
- configuring monitoring for a 3rd-party application
- configuring monitoring for an own application

## Technologies used

Prometheus, Kubernetes, Helm, T Cloud Public (OpenTelekomCloud) CCE, Terraform, Grafana, Redis, Node.js, Podman, DockerHub, Linux

## Installing Prometheus Stack in Kubernetes

1. Provisioned a CCE cluster (managed K8s) on T Cloud Public using Terraform. The infrastructure code lives in the sibling [12-terraform-exercises repo](https://github.com/MonikaJassova/12-terraform-exercises/tree/prometheus) under `environments/prometheus/`.
   - initialised and applied it: `mise exec -- terraform -chdir=environments/prometheus init` then `mise exec -- terraform -chdir=environments/prometheus apply --auto-approve`
   - generated a kubeconfig pointing at the cluster's public EIP: `mise exec -- bash generate-kubeconfig.sh prometheus` (writes `environments/prometheus/kubeconfig.yaml`)
   - `mise.toml` sets `KUBECONFIG` to point to the generated file, so `mise exec -- kubectl` and `mise exec -- helm` commands target the CCE cluster
1. Deployed the microservices-demo app to the cluster: `kubectl apply -f microservices-config.yaml`
1. Deployed Prometheus stack using Prometheus Operator Helm chart
   - added the Helm repo: `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
   - updated the index: `helm repo update`
   - created a separate namespace: `kubectl create ns monitoring`
   - installed the Helm chart: `helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring`
   - verified the pods are running: `kubectl -n monitoring get pods -l "release=monitoring"`
   - verified the stack end-to-end: all components Ready, 24/24 Prometheus targets up, and the frontend LoadBalancer public EIP returns HTTP 200

## Configuring Alerting

1. Created [alert-rules.yaml](./alert-rules.yaml) as a PrometheusRule CRD with 2 custom rules
   - when CPU usage exceeds 50 % on a Kubernetes node
   - when a Pod cannot start (in a crash loop)
1. Deployed the custom Kubernetes resource for Prometheus Operator to pick up and orchestrate adding it to the rule file and reloading Prometheus Server: `kubectl apply -f alert-rules.yaml`
   - verified with `kubectl get PrometheusRule -n monitoring` and viewing Alerts in Prometheus Web UI (port forwarded to localhost with `kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090 &`)
   - confirmed the rules actually loaded via the API: `curl http://127.0.0.1:9090/api/v1/rules` shows the `main.rules` group with both rules (`state: inactive`, `health: ok`); the config-reloader log shows "Reload triggered" at the moment of the apply
   - tested both rules firing end-to-end (see [Testing the Alert Rules](#testing-the-alert-rules) below)
1. In my Gmail account (2FA-enabled), created an app-specific password at <https://myaccount.google.com/apppasswords>
1. Created an email secret with the app password encoded and Alertmanager configuration with an email address configured using AlertmanagerConfig CRD from Prometheus Operator which is merged with default config: `kubectl apply -f email-secret.yaml` and `kubectl apply -f alert-manager-configuration.yaml`
   - verified with `kubectl get alertmanagerconfig -n monitoring` and viewing config in Status tab of Alertmanager UI (port forwarded to localhost with `kubectl port-forward service/monitoring-kube-prometheus-alertmanager -n monitoring 9093:9093 &`)
   - debugging logs with `kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c config-reloader` and `kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c alertmanager`
   - confirmed the effective (merged) config via the API: `curl http://127.0.0.1:9093/api/v2/status` shows the `email` receiver wired to `smtp.gmail.com:587` with the routed sub-receiver; the config-reloader log shows "Reload triggered" on apply
   - verified email delivery end-to-end: after making an alert fire, the notification arrived in the Gmail inbox

## Testing the Alert Rules

To prove the two custom rules actually fire (and email), I forced each condition on a real node and watched the alert transition `inactive -> (pending) -> firing -> resolved`, checking both the Prometheus UI (`/alerts`) and the Alertmanager UI/API.

The chaos workloads used in this section are committed in `test/chaos-crashloop.yaml` (crash-looping pod) and `test/chaos-cpu.yaml` (CPU stressor — set its `nodeSelector` to a non-monitoring node). Apply with `mise exec -- kubectl apply -f test/chaos-crashloop.yaml` / `mise exec -- kubectl apply -f test/chaos-cpu.yaml`; clean up with `mise exec -- kubectl delete pod crash-test` and `mise exec -- kubectl delete pod cpu-stress`.

1. `KubernetesPodCrashLooping` (Pod cannot start / crash loop)
    - created a deliberately crash-looping pod - one that runs fine but has an always-failing liveness probe, so the kubelet restarts it on every probe failure rather than only after the container exits. This is *still* backoff-throttled: after each failed liveness probe the kubelet waits with an exponentially growing delay (10 s, 20 s, 40 s, ... capped at 5 min), so in practice the pod crossed 5 restarts in ~7 min:
     ```yaml
     containers:
     - name: crash-test
       image: busybox
       command: ["sh", "-c", "sleep 3600"]
       livenessProbe:
         exec: { command: ["false"] }
         initialDelaySeconds: 1
         periodSeconds: 2
         timeoutSeconds: 1
         failureThreshold: 1
     ```
   - verified `kubectl get pod crash-test` shows `RESTARTS` climbing past 5, then the rule flips to `firing`; the email arrives in Gmail; deleting the pod resolves it
1. `HostHighCpuLoad` (node CPU > 50%)
   - ran a busy-loop stressor pinned to a single node with `nodeSelector: kubernetes.io/hostname: <node>` (three `while :; do :; done` loops, ~100% of a 2-vCPU node):
     ```yaml
     containers:
     - name: cpu-stress
       image: busybox
       command: ["sh", "-c", "for i in 1 2 3; do (while :; do :; done) & done; wait"]
     ```
   - the rule has `for: 2m`, so it stays pending for ~2 min of sustained load, then flips to `firing`; verified the per-instance CPU expression returns ~100% on the target node and ~5–8% on the rest; deleting the pod brings it back to `inactive`
   - side effect: the chart's default `NodeCPUHighUsage` rule (node CPU > 75%) also fired during the stress and sent its own email notification
1. Confirmed both alerts reach the `email` receiver in Alertmanager (not `null`) and both emails were received
1. Verified the `repeatInterval: 10m` re-send for both rules: kept each alert firing for >10 min and confirmed a second email per rule. Observed inter-email delay was ~15 min — Alertmanager sends at the 10 m deadline at the next `group_interval` batch flush (default 5 m), so effective worst-case cadence is ~15 min

### Verification summary

| Alert | Forced condition | Fired | Email | Repeat (10 m) | Resolved |
|---|---|---|---|---|---|
| `KubernetesPodCrashLooping` | liveness-failing pod, RESTARTS > 5 | ✅ | ✅ | ✅ (2nd email) | ✅ after pod delete |
| `HostHighCpuLoad` | CPU stressor pinned to non-monitoring node | ✅ | ✅ | ✅ (2nd email) | ✅ after pod delete |
| `RedisDown` (branch 1, `redis_up == 0`) | `redis-cart` scaled to 0 | ✅ | ✅ | n/a | ✅ after scale back to 2 |
| `RedisDown` (branch 2, `absent(redis_up)`) | `redis-exporter` scaled to 0 | ✅ | ✅ | n/a | ✅ after exporter restored |
| `RedisTooManyConnections` | 14 persistent connections + 5 background = 19/20 clients (95%), `redis-cart` at 1 replica | ✅ | ✅ | n/a | ✅ after stressor deleted, ratio back to 25% |

No third (repeat) email arrived after the chaos pods were deleted, confirming the repeat loop ends with the alert.

## Configuring Monitoring for a 3rd-party Application (Redis instance)

1. Deployed Redis Exporter using its [Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-redis-exporter): `helm install redis-exporter oci://ghcr.io/prometheus-community/charts/prometheus-redis-exporter -f redis-values.yaml`
   - verified with `helm ls`, `kubectl get servicemonitor`, seeing new target in Prometheus Web UI and `redis_` metrics available on the main page (`redis_up = 1`, 195 `redis_*` series)
   - the exporter scrapes the existing `redis-cart` instance via `redisAddress: redis://redis-cart:6379` (values in `redis-values.yaml`), with the ServiceMonitor labelled `release: monitoring` so the operator picks it up
1. Configured alert rules for Redis down and Redis has too many connections and applied them for Prometheus Operator to pick up: `kubectl apply -f redis-rules.yaml`
    - added to Alerts in Prometheus Web UI
    - tested `RedisDown` for real, branch 1 (`redis_up == 0` — exporter up, Redis unreachable): `kubectl scale deployment redis-cart --replicas=0` -> rule flips to `firing` (it has `for: 0m`) -> restore with `--replicas=2` -> back to `inactive` and `redis_up = 1` again (full fire/resolve cycle confirmed, email received)
    - tested `RedisDown` for real, branch 2 (`absent(redis_up)` — exporter gone, Redis fine): with `redis-cart` left at 2, `kubectl scale deployment redis-exporter-prometheus-redis-exporter --replicas=0` -> `absent(redis_up) = 1` -> rule `firing` in ~74 s (target deregistration writes a staleness marker immediately, so no 5-min lookback wait) -> restore exporter -> `redis_up = 1`, alert resolved (email received)
    - note: the redis-exporter ServiceMonitor targets the `redis-cart` **Service** (ClusterIP virtual IP), so Prometheus effectively probes the VIP only — `RedisDown` means "VIP unreachable / exporter down", not "all Redis replicas down". With 2 replicas, one healthy replica keeps the VIP up, so the alert fires only when the VIP itself is unreachable or the exporter dies.
    - tested `RedisTooManyConnections` for real: set `redis-cart` to `--maxclients 20` (see [microservices-config.yaml](./microservices-config.yaml)), scaled it to **1 replica** (with 2 replicas the VIP round-robins both the stress connections and the exporter's per-scrape connection between instances, so `redis_connected_clients` alternates and the 2-min `for` timer keeps resetting), then ran the stressor in `test/chaos-redis-conn.yaml` — 14 persistent `nc` connections, landing the instance at 19/20 clients (95%, just under the 20 limit so Redis stays healthy and `RedisDown` doesn't co-fire). Rule fired in ~3 min (2-min `for` + scrape interval), email received; deleting the stressor dropped clients to 5 and resolved it. Cleanup: `mise exec -- kubectl delete -f test/chaos-redis-conn.yaml` and scale `redis-cart` back to 2.
1. To access Grafana: `kubectl port-forward service/monitoring-grafana -n monitoring 8080:80 &` and used decoded credentials from `kubectl get secret monitoring-grafana -n monitoring -o yaml`
1. Imported [Grafana dashboard for Redis](https://grafana.com/grafana/dashboards/763-redis-dashboard-for-prometheus-redis-exporter-1-x/) to visualize Redis metrics
   - in Dashboards -> New -> Import dashboard - pasted in Redis dashboard ID (763) -> Load, selected Prometheus as data source -> Import

## Configuring Monitoring for an Own (Node.js) Application

1. An example Node.js app with Prometheus client library integrated and 2 metrics defined and tracked: number of requests (`http_request_operations_total` counter) and request duration (`http_request_duration_seconds` histogram)
1. Built the container image with Podman and pushed it to a private DockerHub repo (from `nodejs-app-monitoring/`):
   - `podman build -t docker.io/monikajassova/demo-app:nodeapp .`
   - `podman login -u monikajassova docker.io`
   - `podman push docker.io/monikajassova/demo-app:nodeapp`
1. Created a Secret in K8s cluster for DockerHub registry: `kubectl create secret docker-registry my-registry-key --docker-server=https://index.docker.io/v1/ --docker-username=monikajassova --docker-password=xxx`
1. Deployed Node.js app together with its ServiceMonitor (to tell Prometheus to scrape a new endpoint) to K8s cluster: `kubectl apply -f nodejs-app-monitoring/k8s-config.yaml`
   - verified with `kubectl port-forward svc/nodeapp 3000:3000` and visiting `127.0.0.1:3000/metrics` — both `http_request_operations_total` and the `http_request_duration_seconds` histogram are exposed
   - a new target (`job=nodeapp`) appeared and went `up` in Prometheus; both metrics became queryable, e.g. `http_request_operations_total`
   - to generate traffic for the dashboard, sent a ~1 min burst of requests (`curl` in a loop against `/`); the counter climbed (3 -> 63) and the duration histogram filled (63 observations, ~5.5s average from the app's simulated 0.5–10s sleep), which the Grafana panels then visualized
1. Created a dashboard for metrics of our Node.js app with Requests per second visualization (`rate(http_request_operations_total[2m])`) and average request duration (`rate(http_request_duration_seconds_sum[2m]) / rate(http_request_duration_seconds_count[2m])` — the ratio of time-rate to request-rate gives seconds per request; `rate(..._sum[2m])` alone is the rate of accumulated time, not a per-request duration)
    - the app's own metrics appear in Prometheus UI under Status -> Targets, Explore, and the main Graph page

## Gotchas & Adjustments

- **Crash-loop rule fires forever on a dead pod.** `kube_pod_container_status_restarts_total > 5` is evaluated against retained TSDB samples, so once a pod crosses 5 restarts the alert keeps firing for the pod's whole lifetime and never auto-resolves (and re-sends email every 10 min) even after the pod is deleted. Fixed by AND-ing with pod existence so it only matches live pods (the whole expr must be quoted in YAML, `>` starts a block scalar):
  ```yaml
  expr: "kube_pod_container_status_restarts_total > 5 and on(namespace, pod) kube_pod_info > 0"
  ```
   - the number stays until the pod is recreated (counter reset). A rate-based variant such as `increase(kube_pod_container_status_restarts_total[10m]) > 5` would match "actively crash-looping" more precisely, at the cost of a shorter detection window. 
   - the chart's own default `KubePodCrashLooping` rule (in `k8s.rules`) has the same cumulative-counter behavior — after deleting the crash-test pod it kept firing on the dead pod until the retained TSDB samples aged out. It routes to `null`, so it's only noise in `/alerts`, but it auto-clears without action.
- **CPU stressor must be pinned away from the monitoring node.** The first stressor landed on the same node as the Prometheus and Alertmanager pods; 100% CPU there made Prometheus's liveness/startup probes time out, so the kubelet restarted the Prometheus container (1/2 Ready, flapping UI/port-forward). Pinned the stressor to a node running only node-exporter instead.
- **`KubeProxyDown` / `KubeControllerManagerDown` / `KubeSchedulerDown` are false positives.** The chart's default rules use `absent(up{job="kube-proxy|..."})`, but on managed CCE those scrape jobs don't exist (the control plane and kube-proxy are managed by TCP), so `absent()` is permanently true. They route to `null` so they're harmless, just noise in `/alerts`.
- **Resolve notifications are not emailed.** The chart's default email receiver has `send_resolved: false`, so you only get an email when an alert *fires* (and per `repeatInterval` while it stays firing). Resolves are visible in Prometheus `/alerts` (state `resolved`) and the Alertmanager UI, not in the inbox.
- **Alertmanager "status: disabled" is expected.** The label refers to Alertmanager's HA **cluster** (gossip) mode, which is off for a single-replica install. Alerting itself works fine — confirmed by received emails.
- **`Watchdog` is always firing by design** — it's a synthetic self-check alert from the default stack, explicitly routed to `null`.
- **Prometheus has no persistent storage by default.** The kube-prometheus-stack chart does not attach a PVC to the Prometheus server by default, so all metrics are lost if the Prometheus pod is rescheduled or its node is drained/restarted. This is also why a Prometheus restart would clear the retained TSDB samples the original crash-loop rule kept firing on. (To persist: set `prometheus.prometheusSpec.storageSpec` to a `volumeClaimTemplate` on a `csi-disk` PVC.)
- **The `namespace: monitoring` label on the custom rules is load-bearing — do not remove it.** At one point I removed the static `namespace: monitoring` label from `HostHighCpuLoad` and `KubernetesPodCrashLooping` (it looked misleading: meaningless for a node alert, and it overrode the pod's real namespace in labels). That silently broke email delivery: the operator merges AlertmanagerConfig routes *under* the default per-namespace route, so the effective route tree is `root (null) → match namespace="monitoring" → email → alertname=... sub-routes`. Without the label, alerts fell through to the `null` receiver (the only mail that arrived was the default `InfoInhibitor`, which naturally has `namespace: monitoring`). Re-added `namespace: monitoring` to all 4 custom rules (both `alert-rules.yaml` and `redis-rules.yaml`); verified email delivery again for each rule. The trade-off stays: for the pod rule the label overrides the pod's real namespace in the alert labels, but the annotation names the pod and routing is what matters.

## Tearing Down

Tear down in the reverse order of provisioning. Cost stops accruing only after the Terraform destroy.

1. **Custom PrometheusRule / AlertmanagerConfig / secrets**:
   ```
   kubectl delete prometheusrule main-rules -n monitoring
   kubectl delete prometheusrule redis-rules -n default        # redis-rules.yaml omits namespace, so it landed in default
   kubectl delete alertmanagerconfig main-rules-alert-config -n monitoring
   kubectl delete secret gmail-auth -n monitoring
   kubectl delete secret my-registry-key -n default
   ```
2. **Helm releases** (removes the whole Prometheus stack + Redis exporter and their workloads):
   ```
   helm uninstall monitoring -n monitoring
   helm uninstall redis-exporter -n default
   ```
3. **Own application** (Node.js app + its ServiceMonitor, deployed from `nodejs-app-monitoring/k8s-config.yaml`):
   ```
   kubectl delete -f nodejs-app-monitoring/k8s-config.yaml
   ```
4. **microservices-demo** (deployed from `microservices-config.yaml`):
   ```
   kubectl delete -f microservices-config.yaml
   ```
5. **Cluster + TCP resources** (Terraform in the sibling `12-terraform-exercises` repo — removes the CCE cluster, ELB, EIP, and any Everest CSI volumes):
   ```
   mise exec -- terraform -chdir=environments/prometheus destroy --auto-approve
   ```
6. **Verify** — `mise exec -- terraform -chdir=environments/prometheus state list` returns no resources.
