# k8s-infra
このレポジトリはおうちKubernetesを支えるGitOpsリポジトリです。  

## 初期構築
### k3s
k3sをインストール
```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```
  
Helmをインストール
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
  
#### Optional
必要があれば、外部からのアクセスを許可するために、自己証明書にIPを追加する
```bash
sudo kubectl -n kube-system edit secrets/k3s-serving
```
```
    listener.cattle.io/cn-10.43.0.1: 10.43.0.1
    listener.cattle.io/cn-127.0.0.1: 127.0.0.1
    listener.cattle.io/cn-100.99.125.62: 100.99.125.62  // この行追加
```
`100.99.125.62`はtailscaleのv4アドレスに置き換える。  
  
証明書を作り直すので一旦k3sを止める
```bash
sudo systemctl stop k3s
```
証明書作り直し
```bash
sudo k3s certificate rotate
```
k3s 開始
```bash
sudo systemctl start k3s
```
  
  
### SealedSecret
```
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/${VERSION}/controller.yaml
```
${VERSION} は [SealedSecretのリポジトリ](https://github.com/bitnami-labs/sealed-secrets/releases/)の最新を使う。  
マスターキーのリストア  
```bash
# 鍵のSecretを生成
kubectl apply -f sealed-secrets-key.yaml

# Secretを生成しただけでは反映されないので、ControllerのPodを再生成する
# このPodはDeploymentによって生成されているので、Podを削除するとすぐに生成される
kubectl delete pods -n kube-system -l name=sealed-secrets-controller

# ControllerのPodが作り直されたことを確認
kubectl get pods -n kube-system -l name=sealed-secrets-controller
#NAME                                         READY   STATUS    #RESTARTS   AGE
#sealed-secrets-controller-867447b788-zhrg4   1/1     Running   #0          18s
```
  
> [!NOTE]
> バックアップ時は
> ```bash
> kubectl get secret -n kube-system \
> -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
> -o yaml > sealed-secrets-key.yaml
> ```
> で保存しておく。
  

#### ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm -rf argocd-linux-amd64
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
Ingressの作成
```bash
nano argocd-ingress.yaml
```
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: PathPrefix(`/`)
      priority: 10
      services:
        - name: argocd-server
          port: 80
    - kind: Rule
      match: PathPrefix(`/`) && Headers(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: default
```
```bash
kubectl apply -f argocd-ingress.yaml
```