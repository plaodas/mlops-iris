# MinIO と MLflow

実験のメタデータは MLflow の sqlite、モデル成果物は MinIO に置きます。MinIO は `argo` 名前空間、MLflow は `mlflow` 名前空間です。

## 前提

- [Kind と Ingress](01-cluster.md) が済んでいること
- Helm
- クラスター名 `agritech-mlops`

認証情報はローカル用の `minio` / `minio123` です。

## 手順

### MinIO

チャートは Bitnami MinIO を基に、`MINIO_BROWSER` が二重定義にならないようテンプレートだけ差し替えたものです。詳細は [charts/minio/README.md](../charts/minio/README.md) を見てください。

```bash
kubectl create namespace argo
helm upgrade --install minio ./charts/minio -n argo -f minio-values.yaml
kubectl -n argo get pods
```

[minio-values.yaml](../minio-values.yaml) は API を NodePort `30090` で開き、既定バケット `argo-artifacts` を作ります。MLflow 用の `mlflow-artifacts` は別途作ります。

```bash
kubectl -n argo port-forward svc/minio 9000:9000
```

別ターミナルで:

```bash
mc alias set minio http://localhost:9000 minio minio123 --api s3v4
mc mb minio/mlflow-artifacts
mc ls minio
```

`mc` がホストに無い場合は、MinIO Pod 内の `mc` で同じバケットを作れます。

```bash
POD=$(kubectl -n argo get pods -o name | grep minio | head -n1)
kubectl -n argo exec "$POD" -- mc alias set local http://127.0.0.1:9000 minio minio123 --api s3v4
kubectl -n argo exec "$POD" -- mc mb local/mlflow-artifacts
```

Argo から MinIO を読む Secret は `argo` 名前空間に置きます。`mlflow` 側の Secret は [mlflow-deploy.yaml](../mlflow-deploy.yaml) に含まれます。

```bash
kubectl -n argo create secret generic minio-cred \
  --from-literal=accesskey=minio \
  --from-literal=secretkey=minio123 \
  --dry-run=client -o yaml | kubectl apply -f -
```

ホストの常駐 port-forward が必要なときは [docker/minio-pf/README.md](../docker/minio-pf/README.md) のコンテナを使います。

### MLflow イメージ

公式イメージには boto3 が無いので、[docker/mlflow/Dockerfile](../docker/mlflow/Dockerfile) で追加します。

```bash
bash scripts/build_mlflow_image.sh
kind load docker-image localhost:5001/mlflow:with-boto3 --name agritech-mlops
```

`docker push` が失敗しても、`kind load` してあれば Pod は `imagePullPolicy: IfNotPresent` で起動します。

### MLflow のデプロイ

```bash
kubectl create namespace mlflow
kubectl apply -f mlflow-deploy.yaml
kubectl apply -f mlflow-svc.yaml
kubectl apply -f mlflow-ingress.yaml
kubectl -n mlflow rollout status deployment/mlflow
```

追跡 URI は `sqlite:////mlflow/mlflow.db`、成果物のルートは `s3://mlflow-artifacts`、S3 エンドポイントは `http://minio.argo.svc.cluster.local:9000` です。

## 確認

ブラウザで http://mlflow.local を開きます。

```bash
kubectl -n mlflow get pods,svc,ingress
curl -I http://mlflow.local
```

## 関連ファイル

- [minio-values.yaml](../minio-values.yaml)
- [charts/minio/](../charts/minio)
- [mlflow-deploy.yaml](../mlflow-deploy.yaml)
- [mlflow-svc.yaml](../mlflow-svc.yaml)
- [mlflow-ingress.yaml](../mlflow-ingress.yaml)
- [scripts/build_mlflow_image.sh](../scripts/build_mlflow_image.sh)
