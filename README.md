# kcc-config-sync

`PROJECT_ID=anthosday`

1. Google Cloud Console を開く
2. Config Connector を適用したクラスタの作成
```shell
gcloud container clusters create infra-admin-cluster \
    --release-channel stable \
    --addons ConfigConnector \
    --workload-pool=anthosday.svc.id.goog \
    --enable-stackdriver-kubernetes \
    --enable-autoscaling \
    --num-nodes 2 \
    --min-nodes 1 \
    --max-nodes 5 \
    --region asia-northeast1
```
cnrm-system namespaces にインストールされていることを確認  
```shell
kubectl get all -n cnrm-system
```
3. Config Connector Service Account 設定
```shell
gcloud iam service-accounts create config-connector
```
```shell
gcloud projects add-iam-policy-binding anthosday \
--member="serviceAccount:config-connector@anthosday.iam.gserviceaccount.com" \
--role="roles/owner"
```
```shell
gcloud iam service-accounts add-iam-policy-binding \
config-connector@anthosday.iam.gserviceaccount.com \
--member="serviceAccount:anthosday.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
--role="roles/iam.workloadIdentityUser"
```

4. configconnector.yaml の作成
```yaml
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnector
metadata:
  # the name is restricted to ensure that there is only one
  # ConfigConnector resource installed in your cluster
  name: configconnector.core.cnrm.cloud.google.com
spec:
  mode: cluster
  googleServiceAccount: "config-connector@anthosday.iam.gserviceaccount.com"
```
```shell
kubectl apply -f configconnector.yaml
```

5. Namespace の作成、設定
```shell
kubectl create namespace kcc-project-anthosday
```
```shell
kubectl annotate namespace \
kcc-project-anthosday cnrm.cloud.google.com/project-id=anthosday
```

6. nomos init してディレクトリ作成し、Google Cloud リソースを namespaces 以下に作成  
https://github.com/tyorikan/kcc-config-sync/tree/main/sync-root
   
7. Config Sync Operator の CRD 適用
```shell
gsutil cp gs://config-management-release/released/latest/config-sync-operator.yaml config-sync-operator.yaml
```
```shell
kubectl apply -f config-sync-operator.yaml
```

8. SSH 認証鍵ペアの作成
```shell
kubectl create secret generic git-creds \
--namespace=config-management-system \
--from-file=ssh=.ssh/git-creds
```

9. config-management.yaml の作成
```yaml
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  # clusterName is required and must be unique among all managed clusters
  clusterName: infra-admin-cluster
  # Enable multi-repo mode to use additional features
  enableMultiRepo: true
```
```shell
kubectl apply -f config-management.yaml
```

10. root-sync.yaml の作成
```yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata:
  name: root-sync
  namespace: config-management-system
spec:
  sourceFormat: hierarchy
  git:
    repo: git@github.com:tyorikan/kcc-config-sync.git
    branch: main
    dir: "sync-root"
    auth: ssh
    secretRef:
      name: git-creds
```
```shell
kubectl apply -f root-sync.yaml
```
Watch sync status
```shell
nomos status --contexts gke_anthosday_asia-northeast1_infra-admin-cluster
```