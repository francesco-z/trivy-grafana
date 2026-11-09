# observability

Observability stack

## Trivy Operator metrics in Grafana

Prometheus + Grafana on the Rancher cluster (`kube-prometheus-stack`, trimmed), scraping the trivy-operator `/metrics` of every cluster; series carry a `cluster` label (`rancher`, `cluster-a`, `cluster-b`, `cluster-c`).

- Rancher cluster: in-cluster service `trivy-operator.trivy-system.svc:80`
- downstream clusters: kube-apiserver service proxy `https://<master-ip>:6443/api/v1/namespaces/trivy-system/services/trivy-operator:80/proxy/metrics`, with a token of the `kube-system/metrics-reader` ServiceAccount (Role `metrics-reader-trivy-metrics`: `services/proxy` get in `trivy-system`)

| cluster | scraped master | TLS `server_name` |
|---|---|---|
| `cluster-a` | `192.0.2.25` | `cluster-a-cluster-api.example.org` |
| `cluster-b` | `192.0.2.161` | `cluster-b-cluster-api.example.org` |
| `cluster-c` | `192.0.2.11` | `cluster-c-cluster-api.example.org` |

The `<cluster>-cluster-api.example.org` names have no load-balanced address yet, so each cluster is scraped through a single master (one target only: more masters would scrape the same trivy-operator and duplicate the series). The certificate is still verified against the `-api` name, which is in each cluster's `tls-san`. When the apiserver VIPs exist, switch the targets back to the hostnames and drop the `toCIDR` rule from `monitoring-egress-NetworkPolicy.yaml`.

### 1. Downstream RBAC

The `metrics-reader` ServiceAccount and its RBAC come from the `monitoring-network` repo (`metrics-api` GitRepos); copy each token to the Rancher cluster:

```bash
kubectl create namespace monitoring  # on Rancher cluster

for c in cluster-a cluster-b cluster-c; do
  kubectl --context "$c" -n kube-system get secret metrics-reader-token -o jsonpath='{.data.token}' | base64 -d > /tmp/token
  kubectl --context "$c" -n kube-system get secret metrics-reader-token -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/ca.crt
  kubectl --context rancher -n monitoring create secret generic "trivy-metrics-$c" \
    --from-file=token=/tmp/token --from-file=ca.crt=/tmp/ca.crt
done
rm /tmp/token /tmp/ca.crt

# check from outside
curl --cacert ca.crt -H "Authorization: Bearer $TOKEN" \
  --resolve cluster-a-cluster-api.example.org:6443:192.0.2.25 \
  https://cluster-a-cluster-api.example.org:6443/api/v1/namespaces/trivy-system/services/trivy-operator:80/proxy/metrics | head
```

### 2. Rancher cluster

```bash
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin --from-literal=admin-password="$(openssl rand -hex 16)"

kubectl apply -f kube-prometheus-stack/kube-prometheus-stack-rancher-cluster.yaml   # GitRepo, fleet-local
```

The GitRepo deploys three bundles from `kube-prometheus-stack/rancher-cluster`: `chart/` (the Helm chart, values inline in its `fleet.yaml`), `manifests/` (HTTPRoute, egress CiliumNetworkPolicy and the ConfigMap-only Role `grafana-sidecar` bound by the chart) and `dashboards/` (kustomize: one ConfigMap per dashboard JSON, loaded by the Grafana sidecar). Change values in git, not in the Rancher console. `kube-prometheus-stack/secrets.yaml` is only a template and stays outside the GitRepo path.

Grafana is published on `https://grafana.example.org` through the Gateway `gateway-system/application` (port 30080 on nodes); the external nginx needs the matching vhost.

### 3. Dashboards (`Trivy` folder)

| dashboard | use |
|---|---|
| Trivy - Security overview | morning report: scorecard per cluster (unique Critical/High CVEs, new CVEs, workloads with critical CVEs, Critical/High misconfigurations, exposed secrets, failed compliance controls, end-of-life OS images, scrape status), new CVEs table, CVEs and failed checks to fix, compliance, 7-day trend |
| Trivy - Vulnerability details | drill-down by cluster/namespace/image/CVE: title, severity, score, package, installed and fixed version, published date, primary link, affected workloads, vulnerable images |
| Trivy - Misconfiguration details | drill-down on failed config audit, RBAC and infra checks: description, remediation link (Aqua AVD), affected resources |
| Trivy Operator - Vulnerabilities | Grafana.com [16337](https://grafana.com/grafana/dashboards/16337) rev 16, vendored with the `cluster`/`namespace` variables patched to trivy-operator metrics (the original reads kube-state-metrics, not scraped here) |

Click a CVE, check, image or scorecard number to open the matching drill-down; CVE cells also link to NVD and OSV. The primary link is rebuilt from the ID (Aqua AVD for CVEs, as in the Trivy report; GitHub for GHSA; OSV otherwise) because trivy-operator does not export it as a metric.

"New CVEs" are Critical/High CVE IDs present now that were never reported for the same cluster in the 7 days before the selected window (24h, 72h on Monday, 7d). A cluster needs at least the window of Prometheus history to be evaluated: Prometheus has no persistent volume, so after a restart the panel stays empty until history builds up again.

The custom dashboards are generated JSON: edit them in Grafana, export, and commit the JSON in `kube-prometheus-stack/rancher-cluster/dashboards/`; a new file also needs an entry in its `kustomization.yaml`.

Manual install:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
yq '.helm.values' kube-prometheus-stack/rancher-cluster/chart/fleet.yaml \
  | helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 92.14.1 \
    -f - --namespace monitoring
kubectl apply -f kube-prometheus-stack/rancher-cluster/manifests/httproute.yaml \
  -f kube-prometheus-stack/rancher-cluster/manifests/grafana-sidecar-Role.yaml \
  -f kube-prometheus-stack/rancher-cluster/manifests/monitoring-egress-NetworkPolicy.yaml
kubectl apply -k kube-prometheus-stack/rancher-cluster/dashboards
```
