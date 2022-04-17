---
title: "ECS Fargate上で公開しているPrometheus形式のメトリクスをDatadogのカスタムメトリクスとして登録するまで"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["datadog", "ECS", "fargate"]
published: true
---

# 概要

Datadog にカスタムメトリクスを送信する方法は、[Datadog 公式のカスタムメトリクスのページ](https://docs.datadoghq.com/metrics/custom_metrics/) に記載されているようにいくつかあります。

本記事では、datadog-agent を介して、ECS Fargate 上のアプリケーションが Prometheus 形式（OpenMetrics 形式）で公開しているメトリクスを Datadog のカスタムメトリクスとして登録する方法について解説します。

# TL;DR

- datadog-agent を ECS タスク上にサイドカーとして登録し動作させる
- ECS タスク内のメトリクスを計測したいタスクに対して、以下の設定をコンテナ定義の Docker ラベルに含める
  - `openmetrics_endpoint` 内の port_number はアプリケーションが OpenMetrics 形式のメトリクスを公開しているポート番号
  - `namespace` は `***` から任意の文字列に変更する

```json
"com.datadoghq.ad.check_names": ["openmetrics"],
"com.datadoghq.ad.init_configs": [{}],
"com.datadoghq.ad.instances": [{"openmetrics_endpoint": "http://%%host%%:port_number/metrics", "namespace": "***", "metrics": [".*"]}]"
```

# カスタムメトリクスを登録するまでの手順

以下を前提に手順を記載しています

- ECS Fargate 上にアプリケーションを公開している
- アプリケーションが Prometheus のクライアントライブラリなどを利用して、OpenMetrics 形式のメトリクスを公開するエンドポイントが存在する（本記事では `/metrics` エンドポイントでメトリクスを公開しているものとします）
- Datadog に登録済み
  - Amazon Web Services, OpenMetrics インテグレーションを有効化。Amazon Web Services インテグレーション上でカスタムメトリクスを取得するか否かのチェックボックスがあるのでチェックを入れておく
    - `Other options` の `Collect custom metrics` チェックボックス

## アプリケーション側のECSタスクの設定修正

[Datadog 公式のOpenMetricsインテグレーションのページ](https://docs.datadoghq.com/integrations/openmetrics/) を参考に ECS タスクの Docker ラベルに以下のパラメータをそれぞれ設定します。

```json
"com.datadoghq.ad.check_names": ["openmetrics"],
"com.datadoghq.ad.init_configs": [{}],
"com.datadoghq.ad.instances": [{"openmetrics_endpoint": "http://%%host%%:port_number/metrics", "namespace": "***", "metrics": [".*"]}]"
```

### ポイント

- `openmetrics_endpoint` 内の port_number はアプリケーションが OpenMetrics 形式のメトリクスを公開しているポート番号
  - [Datadog の Autodiscovery Template Variables](https://docs.datadoghq.com/ja/agent/guide/template_variables/) には `%%port%%` というポート番号を自動で埋めてくれるパラメータが存在しますが、ECS Fargate ではサポートされていないため、明示的に設定する必要があります
- `namespace` は `***` から任意の文字列に変更してください
- メトリクスを公開しているエンドポイントが `/metrics` でない場合は変更してください
- `metrics` は取得したいメトリクスを絞るための正規表現を指定してください。datadog-agent では [DataDog/integrations-core](https://github.com/DataDog/integrations-core/tree/master/openmetrics)が使われており、Python でメトリクスの取得処理が描かれているため、正規表現もPythonを想定して書くと良いと思います。
  - 全てのメトリクスを送信したい場合は `.*` を指定してください。
  - インターネット上の記事では、カスタムメトリクスの取得のために、Prometheus インテグレーションを使っている場合に `*` を指定している場合がいくつかありますが、Python の正規表現では解釈できないみたいです
    - 参考： https://github.com/DataDog/integrations-core/pull/10978
- [Prometheusインテグレーション](https://docs.datadoghq.com/ja/integrations/prometheus/) を使うこともできますが、メトリクスのエンドポイントがテキスト形式をサポートしている場合はOpenMetricsインテグレーションを利用するのが良いみたいです
  - Prometheus の2系からは Protobuf 形式をサポートしていなさそうであるため、基本的にはOpenMetrics インテグレーションを利用するで良さそうです
  - 参考： https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format

## datadog-agent の導入

アプリケーションを ECS Fargate 上で公開している ECS タスクに[datadog-agent](https://docs.datadoghq.com/agent/)を新たにサイドカーとして動作させます。

手順は[Datadog のドキュメント上で公開](https://docs.datadoghq.com/integrations/ecs_fargate)されているため、こちらを流れに沿って進めると良いと思います。
REPLICA サービスとしてタスクを実行するところまで進めてください。

また、datadog-agent にいくつか環境変数を付与する必要があります。
[Docker Agent](https://docs.datadoghq.com/agent/docker/?tab=standard)が参考になります。カスタムメトリクスを取得する場合は `DD_DOGSTATSD_NON_LOCAL_TRAFFIC` に `true` をセットする必要があります。

datadog-agent にメモリ・CPUを割り振ることになるため、必要な分だけアプリケーションを動作させているタスクからメモリ・CPUの値を小さくさせる必要があることに注意してください。

## Terraform や CloudFormation で ECS のタスク定義をしている場合

IaC で ECS のタスク定義をする場合、以下のような設定になるかと思います。

```javascript
{
  ...
  "containerDefinitions": [
    // アプリケーション
    {
      ...
      "dockerLabels": {
        "com.datadoghq.ad.check_names": "[\"openmetrics\"]",
        "com.datadoghq.ad.init_configs": "[{}]",
        "com.datadoghq.ad.instances": "[{\"openmetrics_endpoint\": \"http://%%host%%:port_number/metrics\", \"namespace\": \"***\", \"metrics\": [\".*\"]}]"
      },
      ...
    },
    // datadog-agent
    {
      "name": "datadog-agent",
      "image": "gcr.io/datadoghq/agent:latest",
      "cpu": 100,
      "memory": 256,
      "portMappings": [
        {
          "hostPort": 8126,
          "protocol": "tcp",
          "containerPort": 8126
        },
        // カスタムメトリクスの送信のため、DogStatsD パケットをリスニング
        {
          "hostPort": 8125,
          "protocol": "udp",
          "containerPort": 8125
        }
      ],
      "environment": [
        {
          "name": "DD_DOGSTATSD_NON_LOCAL_TRAFFIC",
          "value": "true"
        },
        {
          "name": "ECS_FARGATE",
          "value": "true"
        },
        ...
      ]
      ...
    }
  ],
  ...
}
```

# モチベーション

OpenMetrics 形式のメトリクスを Datadog のカスタムメトリクスとして登録することによる利点は以下があると思います。（筆者は Datadog を使い始めたばかりで、業務で Datadog は使ったことがないため、他にもあるかもしれません）

- [Prometheusのクライアントライブラリ](https://prometheus.io/docs/instrumenting/clientlibs/)が利用できる
  - 既に Prometheus や Grafana を利用してメトリクスを監視している場合でも Datadog に移行がしやすい
  - クライアントライブラリがデフォルトで計測するメトリクスも監視できる
- （SpringBootを使っている場合）SpringBoot Actuator で公開している `actuator/metrics` エンドポイントで公開しているメトリクスを Datadog で監視できる
- (カスタムメトリクスそのものの利点ですが、) アプリケーションへの負荷だけでなく、ビジネス指標の可視化や柔軟性の高いメトリクスの作成が可能
- etc...

# おわりに

ECS Fargate 上で動作するアプリケーションの OpenMetrics 形式のメトリクスを Datadog のカスタムメトリクスとして登録する方法についてまとめました。

必要な設定や注意事項は全て公式ドキュメントに記載されていたもの、設定に必要な情報が網羅的にまとめている記事が存在しなかったり、公式の日本語ドキュメントが更新されていなかったりで設定に苦戦したため、自身の備忘録としてこちらの記事を書きました。

よければ参考にしていただけたらと思います。
