# FastAPI

MLflow に登録した `argo-dag-demo` を読み、Iris の推論 API を出します。Deployment は `mlflow` 名前空間です。

## 前提

- [Argo の DAG](03-argo.md) が少なくとも 1 回成功し、Model Registry にバージョンがあること
- クラスター名 `agritech-mlops`

## 何をするか

[api/main.py](../api/main.py) は起動時に `DEFAULT_MODEL_URI`（既定 `models:/argo-dag-demo/15`）を読みます。`mlflow.pyfunc.load_model` が失敗すると、成果物をダウンロードして `MLmodel` または `pkl` / `joblib` から読み直します。それでも失敗したときは、新しいバージョンから順に試します。

| メソッド | パス | 説明 |
| --- | --- | --- |
| GET | `/` | 起動確認 |
| GET | `/health` | `model_loaded` を返す |
| POST | `/predict` | 起動時に読んだモデルで推論 |
| POST | `/predict/{version}` | `models:/argo-dag-demo/{version}` で推論 |

リクエストボディは Iris の 4 特徴量です。

```json
{"sepal length (cm)": 5.1, "sepal width (cm)": 3.5, "petal length (cm)": 1.4, "petal width (cm)": 0.2}
```

応答は `{"prediction": 0}` のようなクラス番号です。

## 手順

`/etc/hosts` の `api.local` は [クラスター手順](01-cluster.md) で追加済みです。

```bash
docker build -t localhost:5001/fastapi-iris:latest -f api/Dockerfile api
kind load docker-image localhost:5001/fastapi-iris:latest --name agritech-mlops

kubectl apply -f api/fastapi-deploy.yaml
kubectl apply -f api/fastapi-svc.yaml
kubectl apply -f api/fastapi-ingress.yaml
kubectl -n mlflow rollout status deployment/fastapi
```

イメージを作り直したあとは、同じ `docker build` と `kind load` のあと `kubectl -n mlflow rollout restart deployment/fastapi` します。

別バージョンを既定にしたいときは、Deployment に `DEFAULT_MODEL_URI` を足します。マニフェストにはこの環境変数は無く、コードの既定値 `models:/argo-dag-demo/15` が使われます。バージョン 15 が無いクラスターでは `/predict` は 503 になり、`/predict/{version}` で存在するバージョンを指定します。

## 確認

```bash
curl http://api.local/health
curl -X POST http://api.local/predict/1 \
  -H "Content-Type: application/json" \
  -d '{"sepal length (cm)":5.1,"sepal width (cm)":3.5,"petal length (cm)":1.4,"petal width (cm)":0.2}'
```

`/health` の `model_loaded` が `false` のときは [つまずき](pitfalls.md) の sqlite とバージョン固定を見てください。Pod が `ImagePullBackOff` のときは、`kind load` した名前と Deployment の `image:`（`localhost:5001/fastapi-iris:latest`）が一致しているかを確認します。

## 関連ファイル

- [api/main.py](../api/main.py)
- [api/Dockerfile](../api/Dockerfile)
- [api/fastapi-deploy.yaml](../api/fastapi-deploy.yaml)
- [api/fastapi-svc.yaml](../api/fastapi-svc.yaml)
- [api/fastapi-ingress.yaml](../api/fastapi-ingress.yaml)
