# Kind と Ingress

ローカルに Kind クラスター `agritech-mlops` を作り、Ingress 経由で `*.local` に届くようにします。

## 前提

- Docker
- Kind（このラボでは v0.31.0、ノードイメージ `kindest/node:v1.26.3`）
- kubectl
- Helm

## 手順

### hosts

WSL と、ブラウザを開く Windows の両方に追加します。

```
127.0.0.1 nginx.local mlflow.local argo.local api.local streamlit.local registry5001
```

Windows では管理者の PowerShell で次を実行します。

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 nginx.local mlflow.local argo.local api.local streamlit.local registry5001"
```

### クラスター

[kind-config.yaml](../kind-config.yaml) はホストの 80 / 443 / 5000 / 30080 をノードへ渡し、`registry5001:5000` を HTTP のローカルレジストリへ向けます。

```bash
kind create cluster --name agritech-mlops --image kindest/node:v1.26.3 --config kind-config.yaml --wait 5m
kubectl get nodes
```

### Ingress と動作確認用の Nginx

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl -n ingress-nginx rollout status deployment/ingress-nginx-controller

kubectl apply -f nginx-deploy.yaml
kubectl apply -f nginx-svc.yaml
kubectl apply -f nginx-ingress.yaml
```

### ローカルレジストリ

MLflow が 5000 を使うので、レジストリはホストの 5001 に出します。

```bash
docker run -d --restart=always -p 5001:5000 --name registry registry:2
docker network connect kind registry
```

## 確認

```bash
curl -I http://nginx.local
curl -I localhost:30080
```

どちらも Nginx の応答が返れば、Ingress と NodePort のポートマッピングは通っています。

## 関連ファイル

- [kind-config.yaml](../kind-config.yaml)
- [nginx-deploy.yaml](../nginx-deploy.yaml)
- [nginx-svc.yaml](../nginx-svc.yaml)
- [nginx-ingress.yaml](../nginx-ingress.yaml)
