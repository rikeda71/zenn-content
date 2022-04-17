---
title: "ECS Fargate上で公開しているPrometheus形式のメトリクスをDatadogのカスタムメトリクスとして登録するまで"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["datadog", "ECS", "fargate"]
published: false
---

# 概要

Datadog にカスタムメトリクスを送信する方法は、[Datadog 公式のカスタムメトリクスのページ](https://docs.datadoghq.com/metrics/custom_metrics/) に記載されているようにいくつかあります。

本記事では、datadog-agentを介して、ECS Fargate上のアプリケーションがPrometheus形式（以下、OpenMetrics形式）で公開しているメトリクスをDatadogのカスタムメトリクスとして登録する方法について解説します。

# TL;DR

- datadog-agentをECSタスク上にサイドカーとして登録し動作させる
- ECSタスク内のメトリクスを計測したいタスクに対して、以下の設定をコンテナ定義のDockerラベルに含める
  - `openmetrics_endpoint` 内の port_number はアプリケーションがprometheus形式のメトリクスを公開しているポート番号
  - `namespace` は `***` から任意の文字列に変更する

```json
"com.datadoghq.ad.check_names": ["openmetrics"],
"com.datadoghq.ad.init_configs": [{}],
"com.datadoghq.ad.instances": [{"openmetrics_endpoint": "http://%%host%%:port_number/metrics", "namespace": "***", "metrics": [".*"]}]"
```

# カスタムメトリクスを登録するまでの手順

以下を前提に手順を記載しています

- ECS Fargate上にアプリケーションを公開している
- アプリケーションがPrometheusのクライアントライブラリなどを利用して、OpenMetrics形式のメトリクスを公開するエンドポイントが存在する（本記事では `/metrics` エンドポイントでメトリクスを公開しているものとします）
- Datadog に登録済み
  - Amazon Web Services, OpenMetrics インテグレーションを有効化。Amazon Web Services インテグレーション上でカスタムメトリクスを取得するか否かのチェックボックスがあるのでチェックを入れておく
    - `Other options` の `Collect custom metrics` チェックボックス

## アプリケーション側のECSタスクの設定修正

[Datadog 公式のOpenMetricsインテグレーションのページ](https://docs.datadoghq.com/integrations/openmetrics/) を参考にECSタスクのDockerラベルに以下のパラメータをそれぞれ設定します。

```json
"com.datadoghq.ad.check_names": ["openmetrics"],
"com.datadoghq.ad.init_configs": [{}],
"com.datadoghq.ad.instances": [{"openmetrics_endpoint": "http://%%host%%:port_number/metrics", "namespace": "***", "metrics": [".*"]}]"
```

### ポイント

- `openmetrics_endpoint` 内の port_number はアプリケーションがOpenMetrics形式のメトリクスを公開しているポート番号
  - [Datadog の Autodiscovery Template Variables](https://docs.datadoghq.com/ja/agent/guide/template_variables/) には `%%port%%` というポート番号を自動で埋めてくれるパラメータが存在しますが、ECS Fargate ではサポートされていないため、明示的に設定する必要があります
- `namespace` は `***` から任意の文字列に変更してください
- メトリクスを公開しているエンドポイントが `/metrics` でない場合は変更してください
- `metrics` は取得したいメトリクスを絞るための正規表現を指定してください。datadog-agent では [DataDog/integrations-core](https://github.com/DataDog/integrations-core/tree/master/openmetrics)が使われており、pythonでメトリクスの取得処理が描かれているため、正規表現もpythonを想定して書くと良いと思います。
  - 全てのメトリクスを送信したい場合は `.*` を指定してください。
  - インターネット上の記事では、カスタムメトリクスの取得のために、Prometheusインテグレーションを使っている場合に `*` を指定している場合がいくつかありますが、python の正規表現では解釈できないみたいです
    - 参考： https://github.com/DataDog/integrations-core/pull/10978
- [Prometheusインテグレーション](https://docs.datadoghq.com/ja/integrations/prometheus/) を使うこともできますが、メトリクスのエンドポイントがテキスト形式をサポートしている場合はOpenMetricsインテグレーションを利用するのが良いみたいです
  - Prometheusの2系からはProtobuf形式をサポートしていなさそうであるため、基本的にはOpenMetricsインテグレーションを利用するで良さそうです
  - ref. https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format

## datadog-agent の導入

アプリケーションをECS Fargate上で公開しているECSタスクに[datadog-agent](https://docs.datadoghq.com/agent/)を新たにサイドカーとして動作させます。

手順は[Datadog のドキュメント上で公開](https://docs.datadoghq.com/integrations/ecs_fargate)されているため、こちらを流れに沿って進めると良いと思います。
REPLICA サービスとしてタスクを実行するところまで進めてください。

また、datadog-agent にいくつか環境変数を付与する必要があります。
[Docker Agent](https://docs.datadoghq.com/agent/docker/?tab=standard)が参考になります。カスタムメトリクスを取得する場合は `DD_DOGSTATSD_NON_LOCAL_TRAFFIC` に `true` をセットする必要があります。

datadog-agent にメモリ・CPUを割り振ることになるため、必要な分だけアプリケーションを動作させているタスクからメモリ・CPUの値を小さくさせる必要があることに注意してください。

## Terraform や AWS CDK で ECS のタスク定義をしている場合

ECSのタスク定義は以下のようになるかと思います。

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

OpenMetrics 形式のメトリクスを Datadog のカスタムメトリクスとして登録することによる利点は以下があると思います。（筆者はDatadogを使い始めたばかりで、業務でDatadogは使ったことがないため、他にもあるかもしれません）

- [Prometheusのクライアントライブラリ](https://prometheus.io/docs/instrumenting/clientlibs/)が利用できる
  - 既にPrometheusやGrafanaを利用してメトリクスを監視している場合でもDatadogに移行がしやすい
  - クライアントライブラリがデフォルトで計測するメトリクスも監視できる
- （SpringBootを使っている場合）Spring Boot Actuator で公開している `actuator/metrics` エンドポイントで公開しているメトリクスをDatadogで監視できる
- (カスタムメトリクスそのものの利点ですが、) アプリケーションへの負荷だけでなく、ビジネス指標の可視化や柔軟性の高いメトリクスの作成が可能
- etc...

# おわりに

ECS Fargate上で動作するアプリケーションのOpenMetrics形式のメトリクスをDatadogのカスタムメトリクスとして登録する方法についてまとめました。

必要な設定や注意事項は全て公式ドキュメントに記載されていたもの、網羅的にまとめている記事が存在しなかったり、公式の日本語ドキュメントが更新されていなかったりで設定に苦戦したため、自身の備忘録としてこちらの記事を書きました。

よければ参考にしていただけたらと思います。
