---
template: post
title: GolangのORM「ent」使ってみた
slug: go_ent
socialImage: /media/42-line-bible.jpg
draft: false
date: 2020-11-02T12:14:45.505Z
description: 最近耳にするfacebook製のORMであるentを使用して簡単なAPIサーバーを構築してみた
category: Go
tags:
  - Go
---
## はじめに

Goには様々なORMがあります。メジャーなところでいうと[GORM](https://github.com/go-gorm/gorm)でしょうか。
最近よく聞く「[ent](https://github.com/facebook/ent)」というFacebookが開発したORMを使って軽く遊んでみます。

## ent

entが開発された動機は以下の通りです。

* Goコードとしてのスキーマ
* 型の安全性
  コードジェネレータもあり、これからどんどんリッチになるであろうライブラリです。

## 準備

GoとMySQLの環境はローカルにあるものとします。
entを使うためにはCLIをインストールする必要があります。

```
$ go get github.com/facebook/ent/cmd/entc
```

また、空のプロジェクトディレクトリを作成し、Go modulesで初期化します。

```
$ go  mod init
```

次にスキーマを作成します。今回はとりあえずユーザー情報（名前と年齢）を登録・取得するAPIを作成します。`User`を作成します。

```
$ entc init User
```

すると以下のようにファイルが作成され、`ent/schema/user.go`にスキーマコードが生成されます。

```
├── ent
│   ├── generate.go
│   └── schema
│       └── user.go
├── go.mod
└── go.sum
```

## フィールドの作成

* `ent/schema/user.go`

```
package schema

import "github.com/facebook/ent"

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return nil
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return nil
}
```

このままでは`User`フィールドは空のままなので、`name`と`age`のフィールドを作成します。

```
package schema

import (
	"github.com/facebook/ent"
	"github.com/facebook/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").Default("unknown"),
		field.Int("age").Positive(),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return nil
}
```

ここまでできたらモデルを生成します。

```
$ go generate ./ent
```

すると以下のファイルが生成されます。

```
├── ent
│   ├── client.go
│   ├── config.go
│   ├── context.go
│   ├── ent.go
│   ├── enttest
│   │   └── enttest.go
│   ├── generate.go
│   ├── hook
│   │   └── hook.go
│   ├── migrate
│   │   ├── migrate.go
│   │   └── schema.go
│   ├── mutation.go
│   ├── predicate
│   │   └── predicate.go
│   ├── privacy
│   │   └── privacy.go
│   ├── runtime
│   │   └── runtime.go
│   ├── runtime.go
│   ├── schema
│   │   └── user.go
│   ├── tx.go
│   ├── user
│   │   ├── user.go
│   │   └── where.go
│   ├── user.go
│   ├── user_create.go
│   ├── user_delete.go
│   ├── user_query.go
│   └── user_update.go
├── go.mod
└── go.sum
```

## 実装

### マイグレーション

main.goを作成し、マイグレーション用のコードを書いていきます。このようにしておくことで、`ent/schema/user.go`を修正・追記してもテーブルをよしなにしてくれます。

* main.go

```
package main

import (
	"context"
	"log"
	"os"

	"github.com/go-sql-driver/mysql"
	"github.com/joho/godotenv"
	"github.com/shuto/ent-api/ent"
)

func main() {
	entOptions := []ent.Option{}

	entOptions = append(entOptions, ent.Debug())

	// .envファイルを読み込む設定
	err := godotenv.Load()
	if err != nil {
		log.Fatal("Error loading .env file")
	}

	// MySQLの設定
	mc := mysql.Config{
		User:                 os.Getenv("USER"),
		Passwd:               os.Getenv("PASSWORD"),
		Net:                  "tcp",
		Addr:                 "localhost" + ":" + "3306",
		DBName:               os.Getenv("DBNAME"),
		AllowNativePasswords: true,
		ParseTime:            true,
	}

	client, err := ent.Open("mysql", mc.FormatDSN(), entOptions...)
	if err != nil {
		log.Fatalf("Error open mysql ent client: %v\n", err)
	}

	defer client.Close()

	// マイグレーションツールの実行
	if err := client.Schema.Create(context.Background()); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
}
```

今回は、コードをGitHubなどで管理することも考慮して、MySQLの設定を.envファイルに記述して読み込むことにします。.envは.gitignoreに追記することを忘れないようにしましょう。

* .env 記述例

```
USER=username
PASSWORD=password
DBNAME=ent_api
```

#### マイグレーションの確認

ここまですると、MySQLにカラムが作成されているはずです。
MySQLに入り、

```
mysql> use {データベース名};
mysql> show columns from users
```

↑を実行すると、カラムが作成されていることが確認できます。
![https://gyazo.com/388300134dbe60dcc24a7a5242fd230e](https://gyazo.com/388300134dbe60dcc24a7a5242fd230e/raw)

### ハンドラの実装

ユーザー情報を全件取得するGET、id指定で取得するGET、ユーザー情報を登録するPOSTの３種類のハンドラを作成します。
それぞれのハンドラをmain.goのmainに追記していきます。
ウェブアプリケーションは今回はechoを使用します。

```
	r := echo.New()

	// 全件取得
	r.GET("/users", func(c echo.Context) error {
		eq := client.User.Query()
		u := eq.AllX(context.Background())
		return c.JSON(http.StatusOK, u)
	})

	// １件のみ取得
	r.GET("/user/:id", func(c echo.Context) error {
		id, _ := strconv.ParseInt(c.Param("id"), 10, 0)
		eq := client.User.
			Query().
			Where(user.IDEQ(int(id)))
		u := eq.OnlyX(context.Background())
		return c.JSON(http.StatusOK, u)
	})

	//　ユーザー情報の登録
	r.POST("/user/add", func(c echo.Context) error {
		e := client.User.
			Create().
			SetName("hoge").
			SetAge(5)
		if _, err := e.Save(context.Background()); err != nil {
			return c.JSON(http.StatusInternalServerError, err)
		}
		return c.JSON(http.StatusOK, "OK")
	})

	r.Start(":8080")
```

#### 各エンティティの操作

データの選択の場合は`Query()`を使用します。
これは条件の指定がないのでとてもシンプルですね。

```
eq := client.User.Query()
u := eq.AllX(context.Background())
```

idを指定して1件のみ取得したい場合は、生のSQLと同じように`Where()`句を使うことで条件を指定できます。１件のみの取得の場合は`OnlyX()`を使用します。

```
eq := client.User.
	  Query().
	  Where(user.IDEQ(int(id)))
u := eq.OnlyX(context.Background())
```

最後にデータの追加です。
`Create()`を使い、登録したいNameとAgeを与えて上げれば登録できます。
リクエストボディから値を受け取れるようにすればもっと良い感じになりそうです。

```
e := client.User.
	 Create().
     SetName("hoge").
	 SetAge(5)
e.Save(context.Background())
```

## おわりに

今回は簡単なGETリクエストとPOSTリクエストのみを実装しましたが、とてもわかりやすかったです。マイグレーション機能も他のORMより良さげな雰囲気を感じたので、今後採用される機会は増えていくのではないでしょうか。
今回作成したリポジトリを置いておくので、ぜひ参考にしてください。
間違い等ございましたら、twitterかgithubから教えて下さい！

* [SHOOOOUT/ent_api](https://github.com/SHOOOOUT/ent_api)