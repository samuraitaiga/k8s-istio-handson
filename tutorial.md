# 事前準備

## プロジェクト名とIDのリストを取得する
```bash
gcloud projects list
```

## 取得したGCPプロジェクトIDを環境変数に設定する

今コマンドのFIXMEを実際のGCPプロジェクトIDに置き換えて実行する。
```bash
export GOOGLE_CLOUD_PROJECT=FIXME
```

## ハンズオンで利用するGCPのAPIを有効化する

```bash
gcloud services enable cloudbuild.googleapis.com sourcerepo.googleapis.com containerregistry.googleapis.com container.googleapis.com cloudtrace.googleapis.com cloudprofiler.googleapis.com logging.googleapis.com compute.googleapis.com
```

## ハンズオンで利用する資材を取得する

Gitレポジトリより資材を取得し、動作確認ができている最新バージョン(2019/4/23時点のlatest)に切り替える

```bash
cd ~
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo/
git reset --hard f2f382f
```

## GKEクラスターの作成

```bash
gcloud container clusters create "microservices-demo"  \
--zone "asia-northeast1-c" \
--enable-autorepair \
--username "admin" \
--machine-type "n1-standard-2" \
--image-type "COS" \
--disk-type "pd-standard" \
--disk-size "100" \
--scopes "https://www.googleapis.com/auth/cloud-platform" \
--num-nodes "5" \
--enable-cloud-logging --enable-cloud-monitoring \
--enable-ip-alias \
--network "projects/$GOOGLE_CLOUD_PROJECT/global/networks/default" \
--subnetwork "projects/$GOOGLE_CLOUD_PROJECT/regions/asia-northeast1/subnetworks/default" \
--addons HorizontalPodAutoscaling,HttpLoadBalancing,Istio --istio-config auth=MTLS_PERMISSIVE
```

# Kubernetesでのアプリケーション開発と運用

## コンテナの作成

```bash
cd ~/microservices-demo
skaffold run -p gcb --default-repo=gcr.io/$GOOGLE_CLOUD_PROJECT
```

## サンプルサービス、Podの作成

```bash
cd ~/microservices-demo
kubectl apply -f release/kubernetes-manifests.yaml
```

## 動作確認

### サービスのエンドポイントを確認する
```bash
kubectl get svc/frontend-external
```

### ブラウザで確認する

```
http://<EXTERNAL-IP>
```

```
http://<EXTERNAL-IP>/product/9SIQT8TOJO
```

## サービスの改善

### ソースコードを修正する

対象ファイル
```
~/microservices-demo/src/adservice/src/main/java/hipstarshop/AdService.java
```

変更内容
```
.put("cycling", bike) # 210行目　変更前
.put("cycling", camera) # 210行目　変更後
```

### コンテナの作成、レジストリへの登録

```bash
cd ~/microservices-demo/src/adservice/
docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2 .
docker push gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2
```

ソースコードレポジトリ、CIサービスを利用し、コミット毎やタグ作成毎にコンテナを自動的に作成する方法がおすすめ

### 新しいバージョンのサービスをリリースする

```bash
kubectl set image deployment/adservice server=gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2
```

マニフェストファイルを修正する方法でも問題無いが、コマンドの方が簡易なので使用。



# サービスメッシュの導入

## 事前準備

### Istioのステータス確認

```bash
kubectl get pods --namespace=istio-system
```

### Istioによる管理の有効化

defaultネームスペースに対して、Istioが管理(sidecar injection)することを有効化する。
```bash
kubectl label namespace default istio-injection=enabled
```

通信を制御するsidecarコンテナ(Envoy)を各Podに追加するために、一度すべてのPodを再作成する。
```bash
kubectl delete --all pods
```

既にプロダクション提供している環境に対してInjectionを行う場合、Podのメタデータや環境変数など影響のないパラメータを変更することによりPodの再作成を行なう、もしくは別のDeploymentを作成することが望ましい。上記の方法は強制的にPodをすべて削除し、再度作成するという方法であるため本番環境で行なうとサービスダウンが発生する。

## Istioを使ったアプリケーションのデプロイ

### ソースコードを修正する

対象ファイル
```
~/microservices-demo/src/adservice/src/main/java/hipstarshop/AdService.java
```

変更内容
```
.put("cycling", bike) # 210行目　変更前
.put("cycling", camera) # 210行目　変更後
```

### コンテナの作成、レジストリへの登録

```bash
cd ~/microservices-demo/src/adservice/
docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v3 .
docker push gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v3
```

### Istioで利用するサービスエンドポイントを定義する

トラフィックの向き先を定義
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: adservice
spec:
  host: adservice
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

トラフィックの向き先を定義の設定確認
```bash
kubectl get destinationrule
kubectl describe destinationrule/adservice
```

トラフィックの配分を設定
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: adservice
spec:
  hosts:
    - adservice
  http:
  - route:
    - destination:
        host: adservice
        subset: v2
      weight: 90
    - destination:
        host: adservice
        subset: v3
      weight: 10
```

トラフィックの配分を設定確認
```bash
kubectl get virtualservices
kubectl describe virtualservices/adservice
```

ブラウザでサービスにアクセスして画面下部の広告が指定された割合で表示されることを確認する
```
http://<EXTERNAL-IP>/product/9SIQT8TOJO
```