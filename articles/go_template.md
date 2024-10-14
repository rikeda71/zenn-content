---
title: "Go + gqlgen + sqlc で開発を始めれる GitHub Template"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "graphql", "sqlc"]
published: false
---

タイトルの通り、Go + [gqlgen](https://github.com/99designs/gqlgen) + [sqlc](https://github.com/sqlc-dev/sqlc) を使った GitHub Template を公開しました。

https://github.com/rikeda71/go-gql-sqlc-template

Web・モバイルフロントエンド向けの小 ~ 中規模の Web API を開発することを想定しています。

誰でも利用できるようになっています。よければご利用ください！

宣伝も兼ねて、Template のポイントをいくつか紹介します。

## 技術選定

ライブラリ特有の独自構文を覚えずに、Go が得意とするコードの自動生成に頼った開発ができる技術選定となっています。

### gqlgen

[gqlgen](https://github.com/99designs/gqlgen) は Schema First アプローチによって GraphQL サーバーを開発するためのライブラリです。

Schema First で開発することによって以下の利点があると考えています。
- ライブラリ独自の構文を覚えずに GraphQL Schema に関する知識があればスキーマを定義できる
- スキーマを定義すればフロントエンドの開発も進めていける

### sqlc

[sqlc](https://github.com/sqlc-dev/sqlc) は SQL を書くだけで、データベースとのアクセスと得られた結果を型安全で返すコードを自動生成できる O/R Mapper です。

Java の O/R Mapper である [MyBatis](https://mybatis.org/mybatis-3/) と近しい使い心地です。

SQL からデータベース通信のコードを自動生成することで、以下の利点があると考えています。
- O/R Mapper 特有の構文を覚える必要がない
- クエリチューニングがしやすい

テスト用のクエリを書く必要があったり、動的にクエリを生成することができなかったりする点など他の O/R Mapper と比べて劣る点もありますが、上記2つの利点より採用しました。

### Others

その他、Template では以下の技術を利用しています。

- https://github.com/go-task/task : Task Runner
- https://github.com/amacneil/dbmate : Table Migration Tool
- etc...

## Setup

[README.md](https://github.com/rikeda71/go-gql-sqlc-template/blob/main/README.md) に書かれている通りですが、ここでも簡単に記載します。

### Preparing

以下をセットアップすれば利用できるようになっています。

- Go 1.23
- Docker が扱える環境 (Docker Desktop など)

なお、Docker が扱える環境なのであれば、VSCode の Dev Container を使って自動で開発環境を構築可能です。

![Dev Container での動作イメージ](/images/go_template/1.gif)

### Installation

Task Runner である [task](https://github.com/go-task/task) をインストールすれば 1 コマンドで環境を構築可能です。

```bash
$ go install github.com/go-task/task/v3/cmd/task@latest
$ task setup
```

### Start Development

docker compose を使ってローカルで Web API が立ち上がるようになっています。

```bash
$ task up
$ task migrate # table migration

# table migration を rollback したい場合、以下のコマンドを使う
$ task rollback
```

GraphQL Schema や各種 DDL, DML を更新したら、コードの自動生成して開発を進めていけます。

```bash
# 自動生成も task でコマンド化しています
$ task sqlc-gen
$ task gql-gen
```

## Other Techniques

### CI

GitHub Actions による Lint, Test をデフォルトでセットアップしています。

### API Based Test

GraphQL I/F に通信するテストもデフォルトで備えています。
このテストでは DB との通信も mock せず実行します。

Go で [Test Size](https://testing.googleblog.com/2010/12/test-sizes.html) の大きいテストを書くには少し工夫が必要ですが、設定も Template の中に用意しています。
テストの設定は https://github.com/rikeda71/go-gql-sqlc-template/tree/main/test/api に配置しています。
[dockertest](https://github.com/ory/dockertest) を使って DB との通信も行うテストが実行できるようになっています。
この設定は以下の記事に参考させていただきました。

https://zenn.dev/shiguredo/articles/go-test-dockertest

以下のように GraphQL のクエリを書いて期待するレスポンスが返ることを検証するテストを書くことができます。

```go
// go:build api
package api_test

import (
	"encoding/json"
	"fmt"
	"testing"

	"github.com/google/go-cmp/cmp"
	"github.com/<user or org name>/<project name>/internal/generated/graph"
	api "github.com/<user or org name>/<project name>/test/api/helper"
)

// GraphQL の定義に合わせた Response の json 構造
type createUserMutationResponse struct {
	Data struct {
		CreateUserOutput graph.CreateUserOutput `json:"createUser"`
	} `json:"data"`
}

func TestXxx(t *testing.T) {
...
	// given
	createUserMutation := api.NewQuery(`
	mutation CreateUser {
		createUser(input: {name: "test", email: "test@example.com"}) {
			status
			errorMessage
			metadata {
				user {
					id
				}
			}
		}
	}
	`)

	// when
  /// Server is global scope variable for api test.
	resBytes, err := api.PostGraphQLRequest(createUserMutation, Server)

	// then
	var actual
	err = json.Unmarshal(resBytes, &actual)
	expected := createUserMutationResponse{
		Data: struct {
			CreateUserOutput graph.CreateUserOutput `json:"createUser"`
		}{
			CreateUserOutput: graph.CreateUserOutput{
				Status: graph.MutationStatusSuccess,
				Metadata: &graph.CreateUserOutputMetadata{
					User: &graph.User{
						...
					},
				},
				ErrorMessage: nil,
			},
		},
	}
	if got := cmp.Diff(actual, expected); got != "" {
		t.Errorf("unexpected response: %v", got)
  }
}
```

## まとめ

本記事では Go + [gqlgen](https://github.com/99designs/gqlgen) + [sqlc](https://github.com/sqlc-dev/sqlc) を使った GitHub Template について紹介しました。

環境構築やライブラリの勉強時間を短縮し、Web API の開発をサクッと始めていけると思うのでぜひご利用ください！
