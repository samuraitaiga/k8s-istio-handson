# KubernetesとIstioを使ったマイクロサービス開発と運用のハンズオン

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

## GCPのデフォルトプロジェクトを設定する

```bash
gcloud config set project $GOOGLE_CLOUD_PROJECT
```

## ハンズオンで利用するディレクトリを環境変数に設定する

```bash
export HANDSON_WORKSPACE=$PWD
```

## ハンズオンで利用するGCPのAPIを有効化する

```bash
gcloud services enable cloudbuild.googleapis.com sourcerepo.googleapis.com containerregistry.googleapis.com container.googleapis.com cloudtrace.googleapis.com cloudprofiler.googleapis.com logging.googleapis.com compute.googleapis.com
```

## ハンズオンで利用する資材を取得する

Gitレポジトリより資材を取得し、動作確認ができている最新バージョン(2019/4/23時点のlatest)に切り替える

```bash
cd $HANDSON_WORKSPACE
```

Gitで資材を取得する。
```bash
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
```

取得した資材が入っているディレクトリに移動する
```bash
cd microservices-demo/
```

動作確認ができているバージョンに切り替える
```bash
git reset --hard f2f382f
```

## GKEクラスターの作成

```bash
gcloud beta container clusters create "microservices-demo"  \
--zone "asia-northeast1-c" \
--enable-autorepair \
--username "admin" \
--machine-type "n1-standard-2" \
--image-type "COS" \
--disk-type "pd-standard" \
--disk-size "100" \
--scopes "https://www.googleapis.com/auth/cloud-platform" \
--num-nodes "4" \
--enable-cloud-logging --enable-cloud-monitoring \
--enable-ip-alias \
--network "projects/$GOOGLE_CLOUD_PROJECT/global/networks/default" \
--subnetwork "projects/$GOOGLE_CLOUD_PROJECT/regions/asia-northeast1/subnetworks/default" \
--addons HorizontalPodAutoscaling,HttpLoadBalancing,Istio --istio-config auth=MTLS_PERMISSIVE
```

## GKEクラスターへ接続するための認証情報取得

```bash
gcloud container clusters get-credentials microservices-demo --zone asia-northeast1-c --project $GOOGLE_CLOUD_PROJECT
```

# Kubernetesでのアプリケーション開発と運用

## コンテナの作成、Kubernetesクラスターへのデプロイ

```bash
cd $HANDSON_WORKSPACE/microservices-demo
```

skaffoldを利用してコンテナのビルドとレジストリへの登録を行なう。
```bash
skaffold run -p gcb --default-repo=gcr.io/$GOOGLE_CLOUD_PROJECT
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
$HANDSON_WORKSPACE/microservices-demo/src/adservice/src/main/java/hipstershop/AdService.java
```

変更内容
```
.put("cycling", bike) # 210行目　変更前
.put("cycling", camera) # 210行目　変更後
```

### コンテナの作成、レジストリへの登録

```bash
cd $HANDSON_WORKSPACE/microservices-demo/src/adservice/
```

コンテナのビルドを行なう
```bash
docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2 .
```

コンテナをレジストリへ登録する
```bash
docker push gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2
```

ソースコードレポジトリ、CIサービスを利用し、コミット毎やタグ作成毎にコンテナを自動的に作成する方法がおすすめ

### 新しいバージョンのサービスをリリースする

```bash
kubectl set image deployment/adservice server=gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v2
```

マニフェストファイルを修正する方法でも問題無いが、コマンドの方が簡易なので使用。

### 動作確認

以下にある自転車の商品ページにアクセスし、カメラの広告(Advertisement)が表示されることを確認する。
```
http://<EXTERNAL-IP>/product/9SIQT8TOJO
```


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

## Istioを使うためのアプリケーションのデプロイ

### ソースコードを修正する

対象ファイル
```
$HANDSON_WORKSPACE/microservices-demo/src/adservice/src/main/java/hipstershop/AdService.java
```

変更内容
```
.put("cycling", camera) # 210行目　変更前
.put("cycling", airPlant) # 210行目　変更後
```

### コンテナの作成、レジストリへの登録

```bash
cd $HANDSON_WORKSPACE/microservices-demo/src/adservice/
```

コンテナのビルドを行なう
```bash
docker build -t gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v3 .
```

コンテナをレジストリへ登録する
```bash
docker push gcr.io/$GOOGLE_CLOUD_PROJECT/adservice:v3
```

### Istioで管理するための情報を追加したDeployementをデプロイする

adservice v2のためのDeploymentを作成する
```bash
cd $HANDSON_WORKSPACE
```

以下の内容のファイルをk8s-adservice-v2.yamlという名前で作成する。作成時FIXMEをGCPプロジェクトIDに置き換える。

プロジェクトIDの確認方法
```bash
echo $GOOGLE_CLOUD_PROJECT
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adservice
spec:
  selector:
    matchLabels:
      app: adservice
  template:
    metadata:
      labels:
        app: adservice
        version: v2
    spec:
      terminationGracePeriodSeconds: 5
      containers:
      - name: server
        image: gcr.io/FIXME/adservice:v2
        ports:
        - containerPort: 9555
        env:
        - name: PORT
          value: "9555"
        #- name: JAEGER_SERVICE_ADDR
        #  value: "jaeger-collector:14268"
        resources:
          requests:
            cpu: 200m
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 300Mi
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
        livenessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
```

adservice v2のためのDeploymentをデプロイする
```bash
kubectl apply -f k8s-adservice-v2.yaml
```

adservice v3のためのDeploymentを作成する
```bash
cd $HANDSON_WORKSPACE
```

以下の内容のファイルをk8s-adservice-v3.yamlという名前で作成する。
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adservice-v3
spec:
  selector:
    matchLabels:
      app: adservice
  template:
    metadata:
      labels:
        app: adservice
        version: v3
    spec:
      terminationGracePeriodSeconds: 5
      containers:
      - name: server
        image: gcr.io/FIXME/adservice:v3
        ports:
        - containerPort: 9555
        env:
        - name: PORT
          value: "9555"
        #- name: JAEGER_SERVICE_ADDR
        #  value: "jaeger-collector:14268"
        resources:
          requests:
            cpu: 200m
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 300Mi
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
        livenessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
```

adservice v3のためのDeploymentをデプロイする
```bash
kubectl apply -f k8s-adservice-v3.yaml
```

## Istioを使ったトラフィック制御

### Istioで利用するサービスエンドポイントを定義する

トラフィックの向き先を定義したファイルを作成する。
```bash
cd $HANDSON_WORKSPACE
```

以下の内容のファイルをistio-destinationrule-adservice.yamlという名前で作成する。
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

トラフィックの向き先定義(destination rule)をデプロイする。
```bash
kubectl apply -f istio-destinationrule-adservice.yaml
```

トラフィックの向き先を定義の設定確認
```bash
kubectl describe destinationrule/adservice
```

トラフィックの配分を設定
```bash
cd $HANDSON_WORKSPACE
```

以下の内容のファイルをistio-virtualservice-adservice.yamlという名前で作成する。
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

トラフィックの配分定義(virtual service)をデプロイする。
```bash
kubectl apply -f istio-virtualservice-adservice.yaml
```

トラフィックの配分を設定確認
```bash
kubectl describe virtualservices/adservice
```

ブラウザでサービスにアクセスして画面下部の広告が指定された割合(camera 9:airplant 1)で表示されることを確認する
```
http://<EXTERNAL-IP>/product/9SIQT8TOJO
```


# Congratulations !!

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

これにてkubernetesとistioを使ったマイクロサービスの開発と運用のハンズオンは終了です。

より詳細はKubernetesおよびIstioの設定値などはこちらから確認ください。
[Kubernetes](https://kubernetes.io)
[Istio](https://istio.io)