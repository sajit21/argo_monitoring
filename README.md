<!-- AGENT_CONTEXT: repo_name=monitoring-gitops | purpose=Argo CD GitOps repo for observability stack (kube-prometheus-stack,loki,alloy,tempo) | stack=helm+argocd | pattern=direct-upstream-charts+multi-source($values)+ApplicationSet-list | apps=kube-prometheus-stack,loki,alloy,tempo | ns=monitoring argocd_ns=argocd | sync=auto,selfHeal=true,prune=false,CreateNamespace,ServerSideApply | psa=privileged-on-monitoring | repo=https://github.com/sajit21/argo_monitoring.git | values=all-4-present(loki/alloy/tempo-minimal) | versions=* (pin-after-sync) | no-Chart.yaml/templates/helpers | storage=ephemeral(no-pvc) | grafana-ds=Loki/Tempo-wired | notes=node-exporter-privileged,SSApply-for-CRDs -->
# monitoring-gitops

Everything needed to deploy **kube-prometheus-stack** (Prometheus, Alertmanager, Grafana, node-exporter,
kube-state-metrics) with Argo CD. Argo CD is already installed in the cluster.

```
argocd/appproject.yaml       what Argo CD may pull from / deploy to
argocd/applicationset.yaml   source (chart), version, destination (cluster + namespace), sync policy
values/kube-prometheus-stack.yaml   the Helm values (all settings you change live here)
```

Status: unvalidated draft. It has not been rendered with Helm or deployed. Step 4 shows how to check.

## Steps

**1. Put this in your Git repository.** Create an empty private repo, then:
```bash
cd monitoring-gitops
sed -i 's|__REPO_URL__|<YOUR_GIT_URL>|g' argocd/*.yaml      # your real repository URL
git remote add origin <YOUR_GIT_URL> && git push -u origin main
```
If your Argo CD is not in the `argocd` namespace, change `namespace: argocd` in both files in `argocd/`.

**2. Add the repository to Argo CD** (Settings -> Repositories -> Connect Repo, HTTPS, username + read-only token). Status must be **Successful**.
A public repository needs no credentials but still must be added.

**3. Apply the two Argo CD files once** (the only things applied by hand; after this, Git is the source of truth):
```bash
kubectl apply -f argocd/appproject.yaml -f argocd/applicationset.yaml
```

**4. Watch it deploy** (3-10 minutes; `Synced` = cluster matches Git, `Healthy` = pods ready, they are different):
```bash
kubectl -n argocd get applicationset,application
kubectl -n monitoring get pods
```
If the Application shows an error: `kubectl -n argocd describe application kube-prometheus-stack`.

**5. Pin the chart version** (replace the temporary `"*"`):
```bash
kubectl -n argocd get application kube-prometheus-stack -o jsonpath='{.status.sync.revisions[0]}'; echo
```
Put that number in `argocd/applicationset.yaml` (`version:`), commit, push, then `kubectl apply -f argocd/applicationset.yaml` once more
(the ApplicationSet itself is not auto-managed in this minimal setup).

**6. Open Grafana:**
```bash
kubectl -n monitoring get secret kube-prometheus-stack-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```
Browse `http://localhost:3000`, user `admin`. Open Dashboards -> "Kubernetes / Compute Resources / Cluster": graphs must show data.

## Changing things later
Edit `values/kube-prometheus-stack.yaml` (retention, resources, storage), commit, push. Argo CD applies it automatically.
Change the destination cluster or namespace in `argocd/applicationset.yaml`.

## Optional: RKE2 control-plane metrics
Add on each server node in `/etc/rancher/rke2/config.yaml`, then `systemctl restart rke2-server`:
```yaml
etcd-expose-metrics: true
kube-scheduler-arg: ["bind-address=0.0.0.0"]
kube-controller-manager-arg: ["bind-address=0.0.0.0"]
```
Then set `kubeEtcd`, `kubeScheduler`, `kubeControllerManager` to `enabled: true` in the values file.

## Notes
- No persistent disks yet: Prometheus data and Grafana changes are lost on pod restart until a StorageClass exists (next stage: Longhorn).
- `prune: false`: deleting something from Git does not delete it from the cluster automatically.
- Nothing is exposed outside the cluster; use port-forward until an authenticated, TLS ingress is added.
