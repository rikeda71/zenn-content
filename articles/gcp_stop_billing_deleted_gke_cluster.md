---
title: "GKEのクラスタを削除してもネットワークサービスで課金が止まらない場合の対処法"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp", "kubernetes"]
published: true
---

[アプリを GKE クラスタにデプロイする](https://cloud.google.com/kubernetes-engine/docs/deploy-app-cluster?hl=ja) などを参考に GKE で遊び終わったあと、GKE クラスタを削除しても GCP からの軽微な請求が続いていました。

おそらく私の GKE クラスタの削除方法が悪かったのですが、請求を止めることができたので備忘録を残しておきます。

## 前提

- GKE クラスタを作成し、全て削除した後もネットワークサービスでの課金が止まらない場合
- ネットワークサービスを利用している覚えがない場合

## 請求先の確認

「料金明細」より、請求先のサービスを確認したところ、 `Network` -> `Networking Service Directory Registered Resource` で課金が発生していることが確認できました。

![請求先の確認](/images/gcp_stop_billing_deleted_gke_cluster/1.png)

次に、[Network -> Service Directory](https://console.cloud.google.com/net-services/service-directory/namespaces/list) を確認したところ、名前空間「goog-psc-default」が作成されていることがわかりました。  
課金が発生しているプロジェクト内で他に Service Directory の名前空間は存在しないため、この名前空間で課金が発生していることがわかりました。

[エンドポイントを通じて公開サービスにアクセスする](https://cloud.google.com/vpc/docs/configure-private-service-connect-services?hl=ja) などによると名前空間「goog-psc-default」は GCP 上でエンドポイントを作成すると、名前空間を選択しなかった場合に利用される名前空間みたいなので、存在しない場合は自動で作成されるみたいです。
(GKE でロードバランサを作成したタイミングなどで作成されたかもしれません)

## Service Directory の名前空間の削除

早速、Google Cloud Consule の UI 上から「goog-psc-default」を削除を試みましたが、エラーが出て削除ができないようでした。  
以下のように`gcloud`コマンドを実行することで、Service Directory の名前空間「goog-psc-default」を削除でき、課金も停止しました。

```shell
# location は名前空間が作成されているRegion
$ gcloud service-directory namespaces delete goog-psc-default --location=us-west1
```

2023/03 時点で 14円/月 程度の金額ですが、同じ事象で発生している課金を停止したい場合はお試しいただけると幸いです。
