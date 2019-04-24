KubernetesとIstioを使ったマイクロサービス開発と運用のハンズオン
==

GCPが公開している(マイクロサービスのデモアプリケーション))[https://github.com/GoogleCloudPlatform/microservices-demo]を利用し、KubernetesとIstioを利用したマイクロサービスの開発・運用を試すためのハンズオン。

# 対象
* Kubernetes、Istioをこれから利用しようと思っている人

# 必要な環境
* GCP

# ハンズオンの流れ
* Kubernetesクラスターを作成する
* Kubernetesクラスターにマイクロサービスアプリケーションをデプロイする
* １つのマイクロサービスを改善してKuberentesクラスターにデプロイする
* Istioを導入する
* １つのマイクロサービスを改善してKuberentesクラスターにデプロイする
* Istioを使ってマイクロサービス間の通信制御(トラフィックスプリット)を行なう

# ハンズオンを実行する

## Cloud Shellを使ってハンズオンを実行する
GCPの中にあるCloud Shellを利用して、インタラクティブにハンズオンを勧める方法です。
[![Open in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https%3A%2F%2Fgithub.com%2Fsamuraitaiga%2Fk8s-istio-handson.git&cloudshell_open_in_editor=tutorial.md&cloudshell_tutorial=tutorial.md)

## ドキュメントを読みながら実行する
[tutorial.md](https://github.com/samuraitaiga/k8s-istio-handson/tutorial.md)を参照し、独力でハンズオンを進める方法です。