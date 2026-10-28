# observability

Observability stack

## Trivy Operator metrics in Grafana

Prometheus + Grafana on the Rancher cluster (`kube-prometheus-stack`, trimmed), scraping the trivy-operator `/metrics` of every cluster; series carry a `cluster` label (`rancher`, `cluster-a`, `cluster-b`, `cluster-c`).

- Rancher cluster: in-cluster service `trivy-operator.trivy-system.svc:80`
- downstream clusters: kube-apiserver service proxy `https://<cluster>-cluster-api.example.org:6443/api/v1/namespaces/trivy-system/services/trivy-operator:80/proxy/metrics`, with a token of the `kube-system/metrics-reader` ServiceAccount (Role `metrics-reader-trivy-metrics`: `services/proxy` get in `trivy-system`)

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
  https://cluster-a-cluster-api.example.org:6443/api/v1/namespaces/trivy-system/services/trivy-operator:80/proxy/metrics | head
```

### 2. Rancher cluster

```bash
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin --from-literal=admin-password="$(openssl rand -hex 16)"

kubectl apply -f kube-prometheus-stack/kube-prometheus-stack-rancher-cluster.yaml   # GitRepo, fleet-local
```

The GitRepo deploys two bundles from `kube-prometheus-stack/rancher-cluster`: `chart/` (the Helm chart, values inline in its `fleet.yaml`) and `manifests/` (HTTPRoute and egress CiliumNetworkPolicy). Change values in git, not in the Rancher console. `kube-prometheus-stack/secrets.yaml` is only a template and stays outside the GitRepo path.

Grafana is published on `https://grafana.example.org` through the Gateway `gateway-system/application` (port 30080 on nodes); the external nginx needs the matching vhost. The Trivy Operator dashboard ([17813](https://grafana.com/grafana/dashboards/17813-trivy-operator-dashboard/)) is provisioned in the `Trivy` folder.

Manual install:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
yq '.helm.values' kube-prometheus-stack/rancher-cluster/chart/fleet.yaml \
  | helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 92.14.1 \
    -f - --namespace monitoring
kubectl apply -f kube-prometheus-stack/rancher-cluster/manifests/httproute.yaml \
  -f kube-prometheus-stack/rancher-cluster/manifests/monitoring-egress-NetworkPolicy.yaml
```
