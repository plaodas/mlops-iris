# つまずき

このラボを組んだときに繰り返した失敗と、マニフェストに残っている対処です。

## Kind ではイメージ名を揃えて load する

ノードは Docker Hub 以外を pull できません。Deployment と Workflow の `image:` と同じ名前で `kind load` し、`imagePullPolicy: IfNotPresent` にします。DAG、FastAPI、Streamlit、MLflow、MLflow exporter のマニフェストには既に `IfNotPresent` が入っています。

```bash
kind load docker-image localhost:5001/mlflow-dag:latest --name agritech-mlops
kind load docker-image localhost:5001/fastapi-iris:latest --name agritech-mlops
kind load docker-image localhost:5001/streamlit-iris:latest --name agritech-mlops
kind load docker-image localhost:5001/mlflow:with-boto3 --name agritech-mlops
```

`ImagePullBackOff` のときは、Pod のイメージ名と `kind load` に渡した名前が一致しているかを先に見ます。`registry5001:5000/...` と `localhost:5001/...` は別の名前です。単発学習ワークフローは前者、DAG と API は後者を参照します。

## レジストリは 5001、MLflow は 5000

MLflow の Service と [kind-config.yaml](../kind-config.yaml) の extraPortMappings が 5000 を使います。ローカルレジストリはホストの 5001 に publish し、コンテナ内の 5000 へ渡します。

```bash
docker run -d --restart=always -p 5001:5000 --name registry registry:2
docker network connect kind registry
```

## workflowtaskresults の RBAC

Workflow が次で落ちることがあります。

`workflowtaskresults.argoproj.io is forbidden: User "system:serviceaccount:argo:default" cannot create resource "workflowtaskresults"`

`argo` 名前空間の `default` ServiceAccount に権限を足します。

```bash
kubectl apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argo-workflowtaskresults-role
  namespace: argo
rules:
- apiGroups: ["argoproj.io"]
  resources: ["workflowtaskresults"]
  verbs: ["create", "get", "list", "watch", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argo-workflowtaskresults-binding
  namespace: argo
subjects:
- kind: ServiceAccount
  name: default
  namespace: argo
roleRef:
  kind: Role
  name: argo-workflowtaskresults-role
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl auth can-i create workflowtaskresults --as=system:serviceaccount:argo:default -n argo
```

`yes` が返れば再実行できます。

## MinIO chart の MINIO_BROWSER

Bitnami chart の values で `MINIO_BROWSER` を足すと、テンプレート側の定義と重なってインストールが失敗します。このリポジトリは [charts/minio](../charts/minio) でその env を一か所にまとめています。インストールは次です。

```bash
helm upgrade --install minio ./charts/minio -n argo -f minio-values.yaml
```

上流 chart の全ファイルは含めていません。詳細は [charts/minio/README.md](../charts/minio/README.md) です。

## sqlite だとモデル読み込みが不安定

MLflow の `--backend-store-uri` は `sqlite:////mlflow/mlflow.db` です。この状態で `mlflow.pyfunc.load_model("models:/argo-dag-demo/<version>")` が失敗し、特定バージョンだけ読めることがありました。FastAPI の既定 URI はそのため `models:/argo-dag-demo/15` です。

[api/main.py](../api/main.py) は pyfunc が失敗すると成果物を下ろし、`MLmodel` か pickle から読み直します。バージョン 15 が無いクラスターでは `/predict` が 503 になるので、`/predict/{version}` で存在するバージョンを指定するか、Deployment に `DEFAULT_MODEL_URI` を設定します。追跡 DB の PostgreSQL 移行は未実装です。

## Pushgateway の Service 名

DAG が送る先は `pushgateway-prometheus-pushgateway.monitoring.svc.cluster.local:9091` です。Helm のリリース名が `pushgateway` のときの Service 名です。`pushgateway.monitoring.svc` は DNS にありません。

```bash
helm install pushgateway prometheus-community/prometheus-pushgateway -n monitoring
```

## Argo の認証無効化はローカル専用

`argo-server` の `--auth-mode=server` は、このノート PC の UI 確認用です。手順と戻し方は [local-auth.md](local-auth.md) にあります。
