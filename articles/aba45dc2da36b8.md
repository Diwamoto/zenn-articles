---
title: "ginをMVCで設計している時にコントローラのテストを書く"
emoji: "🤨"
type: "tech"
topics:
  - "golang"
  - "test"
  - "gin"
published: true
published_at: "2021-03-02 12:48"
---

# ディレクトリ構成
今まで私はlaravel,CakePHP,FuelPHPと、mvc構成になっているPHPのフレームワークを扱ってきたので、
### gin(goのWAF)でもMVC構成で扱いたい！！！

と思っていました。
なので、いろいろゴニョゴニョして、以下のディレクトリ構成に落ち着かせることにしました。

```sh
> tree
.
├── config
│   ├── Config.go
│   └── database
│           └── Database.go
├── controller
│   ├── AppController.go
├── doc
│   └── 仕様書とかその他
├── model
│   ├── AppModel.go
├── server
│   └── Router.go
├── test
│   └── いろんなテスト
├── view
│   └── index.html #今回はapiとしてginを使用するので使わない
│ 
├── go.mod
├── go.sum
└── main.go
```

(中でも一番CakePHPに触れていたのですごく影響を受けていますが)わかりやすいディレクトリ構成になりました。

# モデルのテスト

モデルのテストはinoutがはっきりしているものが多いので、テストは簡単でした。
goのテストで提起されている[Table Driven Test](https://github.com/golang/go/wiki/TableDrivenTests)に簡単に当てはめることができます。

以下はVSCodeのGo拡張から吐き出されたコードを少し書き換えた物です。
```go
package model

import (
	"reflect"
	"testing"
)

var um = NewUserModel("test") //変更点　同じデータベースに二回以上接続しないようにする為

func TestUserModel_Create(t *testing.T) {
	type args struct {
		u User
	}
	tests := []struct {
		name    string
		args    args
		want    User
		wantErr bool
	}{
		// TODO: Add test cases.
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := um.Create(tt.args.u)
			if (err != nil) != tt.wantErr {
				t.Errorf("UserModel.Create() error = %v, wantErr %v", err, tt.wantErr)
				return
			}
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("UserModel.Create() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

単純にメソッドをテストケースで回し、その出力値が想定と間違っているかどうかというテスト形式ですね。

入力値と出力値しか見てないのでブラックボックスなテストになっているのですが、
そのメソッドの中が網羅されているかはカバレッジで見るという思想になっているようです。
カバレッジは以下のような感じで、メソッドのどこの部分が何回通っているかを見ることができます。
赤くなっているところはテストコードでテストできておらず、色が濃くなるにつれて通った回数が多いという表示になっています。

![](https://storage.googleapis.com/zenn-user-upload/dcnqrxw6zl29ynrotbbaigomycg0)


モデルはデータベースのテーブルを構造体にしたものをやりとりしているので、inoutがはっきりしていてテストが楽でした。

問題はコントローラです。。

# コントローラのテスト？

## そもそもこの場合におけるコントローラとは何か

そもそものginの公式ドキュメントのexampleはこのようなものでした。

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}
```

このように１ファイルでやるのもいいですが、どうせならMVC（のようなフォルダ構成）にしたい！と思ったので、
ルートを定義する部分を`router`に渡し、その先の
```go
func(c *gin.Context) {
	c.JSON(200, gin.H{
		"message": "pong",
	}
)
```
の部分をコントローラに渡しました。
その中で保存されたデータに問い合わせる際にはモデルに定義したデータベースとのやりとり処理を呼び出す。という定義でコントローラとしました。

一応コードに書くとこんな感じですかね。
パッケージはわかりやすくする為に分けていますが、
本来は分けないようにしないと、カバレッジが別れて溜まる為面倒です（`goverall`等を使用する際にファイルが複数できる為面倒）


ルータ
```go
package server

import ()//省略

func GetRouter() *gin.Engine {

	router := gin.Default()
	v1 := router.Group("/api/v1")
	{
		v1.POST("/users", controller.CreateUserAction)
	}
	
	return router
}
```

モデル
```go
package model

import ()//省略。　VSCodeのGo拡張を入れていれば自動で挿入されます。

//指定ユーザidの情報を返す
//Idで検索するFind()のラッパー
func (um UserModel) GetById(id int) ([]User, error) {

	var u User
	u.Id = id
	result, err := um.Find(u) //もらったユーザの情報を元にSQLをビルドするメソッドです
	return result, err

}

```

コントローラ
```go
package controller 

import () 省略

func CreateUserAction(c *gin.Context) {

	um := model.NewUserModel("default")

	u := model.User{
		Email:    c.PostForm("Email"),
		Password: string(password), //hashにしたパスワードが入る想定
		Name:     c.PostForm("Name"),
		Phone:    c.PostForm("Phone"),
		Status:   2,
		Profile:  model.UserProfile{},
	}
	u.Created = time.Now()
	u.Modified = time.Now()

	user, err := um.Create(u)
	//エラーじゃなければuserの情報を返す
	if err == nil {

		c.JSON(http.StatusCreated, user)
	} else { 
		//作成できなければエラーメッセージを返す。
		c.JSON(http.StatusConflict, gin.H{"message": err.Error()})

	}
}
```

突っ込みどころは結構あるかと思いますが、一応こんな感じで役割を分けることができました。

**長くなったので話を戻しましょう。**

ここでやりたいのはコントローラのテストです。
さていつものようにVSCodeのテストを生成して見ようとと思ったら、、、ん？

```go
func TestCreateUserAction(t *testing.T) {
	type args struct {
		c *gin.Context
	}
	tests := []struct {
		name string
		args args
	}{
		// TODO: Add test cases.
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			CreateUserAction(tt.args.c)
		})
	}
}
```

ほう。。。これは、、**関数を動かしてるだけで帰ってくるJSONがどうとか全くテストしてないな？？**

てかそもそもcontrollerはboolとか返さないし、どうやってinoutのテストするの？


# 返り値ないならテストできねーじゃん！（某心君感）


先ほどもいいましたが、デフォルトで生成してくれるテスト文ではinoutのテストはできません。
ではどうするか。コントローラの仕事はユーザのリクエストを解析して、正しいデータにしてモデルとやりとりし、その結果をユーザに返すことです。

つまり、正しいテストを行うには**リクエストを生成してレスポンスを解析する**必要があります。

なので、方向性として
1. テストのリクエストを生成する。
2. レスポンスを取得し、それを解析する。
3. 以上の流れを`Table Driven Test`に組み込む

を目指して、ネットにある記事を探し漁りました。

# こうしたらできた

上記の方向性について調べることで、以下のようなテストを書くことができました。


```go 

//テストで使用する帰り値用のオブジェクト
//codeとレスポンスのボディを見て判断する
type preferResponse struct {
	code int                    //httpステータスコード
	body map[string]interface{} //帰ってくる文字列
}

//controller.CreateUserAction()のテスト
func TestCreateUserAction(t *testing.T) {

	tests := []struct {
		user model.User
		want preferResponse
	}{
		{
			//①: テーブルにレコードが何もない状態で作成するユーザ（コンフリクトは起きないはず）
			model.User{
				Email:    "test@example.com",
				Password: "test password",
				Name:     "test name",
			},
			preferResponse{
				code: http.StatusCreated,
				body: map[string]interface{}{
					"Email": "test@example.com",
					//"Password": "test password", //パスワードも調査した方が良いが、データベース内ではhash化されるため
					"Name": "test name",
				},
			}, //ユーザは作成できるはず
		},
		{
			//②: メールアドレスがかぶっているユーザ
			model.User{
				Email:    "test@example.com",
				Password: "test password",
				Name:     "test name",
			},
			preferResponse{
				code: http.StatusConflict,
				body: map[string]interface{}{
					"message": "[\"入力されたメールアドレスは既に登録されています。\"]",
				},
			}, //既に作成されているのでコンフリクトが起きるはず
		},
	}
	for i, tt := range tests {

		//テスト準備
		//リクエストを作成
		requestBody := strings.NewReader("Email=" + tt.user.Email + "&Name=" + tt.user.Name + "&Password=" + tt.user.Password)
		//レスポンス
		//ここに帰ってくる
		response := httptest.NewRecorder()
		//コンテキストを作成
		c, _ := gin.CreateTestContext(response)
		//リクエストを格納
		c.Request, _ = http.NewRequest(
			http.MethodPost,
			"/api/v1/users",
			requestBody,
		)
		//フォーム属性を付与
		c.Request.Header.Set("Content-Type", "application/x-www-form-urlencoded")

		// テストのコンテキストを持って実行
		controller.CreateUserAction(c)

		//検証
		var responseBody map[string]interface{}
		_ = json.Unmarshal(response.Body.Bytes(), &responseBody)

		//ステータスコードがおかしいもしくは帰ってきたメッセージが想定と違えばダメ
		if response.Code != tt.want.code {
			t.Errorf("%d番目のテストが失敗しました。想定返却コード：%d, 実際の返却コード：%d", i+1, tt.want.code, response.Code)
		} else {
			//実際に帰ってきたレスポンスの中に想定された値が入っているかどうか
			for key := range tt.want.body {
				//値の存在チェック
				if _, exist := responseBody[key]; exist {

					//値の内容チェック
					if responseBody[key] != tt.want.body[key] {
						t.Errorf("%d番目のテストが失敗しました。想定されたキー「%s」の値:%s, 実際に返却された値:%s", i+1, key, tt.want.body[key], responseBody[key])
					} // else{
					//クリアはここだけ
					// }

				} else {
					t.Errorf("%d番目のテストが失敗しました。想定された「%s」がレスポンスボディに入っていません。", i+1, key)
				}
			}
		}
	}
}
```


上から順に説明すると、
まず`type preferResponse struct`についてはレスポンスの内、検証が必要な部分(コントローラないで`c.JSON(code, response)`としているところ)を構造体にしています。

そしてテストメソッドを定義し、大部分は`Table Driven Test`のフレームが使われていることがわかるかと思います。

説明したいのはここからです。

以下のコードで、まずテストに入るための準備をしています。


```go
//テスト準備
//リクエストを作成
requestBody := strings.NewReader("Email=" + tt.user.Email + "&Name=" + tt.user.Name + "&Password=" + tt.user.Password)
//レスポンス
//ここに帰ってくる
response := httptest.NewRecorder()
//コンテキストを作成
c, _ := gin.CreateTestContext(response)
//リクエストを格納
c.Request, _ = http.NewRequest(
	http.MethodPost,
	"/api/v1/users",
	requestBody,
)
//フォーム属性を付与
c.Request.Header.Set("Content-Type", "application/x-www-form-urlencoded")

```

順に説明すると、

```go
requestBody := strings.NewReader("Email=" + tt.user.Email + "&Name=" + tt.user.Name + "&Password=" + tt.user.Password)
```

ここではリクエストボディを作っています。フォームからデータが入力された想定ですね。

```go
//レスポンス
//ここに帰ってくる
response := httptest.NewRecorder()
```

次にレスポンスです。
コントローラは返り値を持たないので、`httptest`パッケージにあるレスポンスを取り出してくれる`httptest.NewRecorder()`を利用しました。ここにリクエストのレスポンスが帰ってくるので、中身を解析することでテストとするわけです。

```go
//コンテキストを作成
c, _ := gin.CreateTestContext(response)
//リクエストを格納
c.Request, _ = http.NewRequest(
	http.MethodPost,
	"/api/v1/users",
	requestBody,
)
//フォーム属性を付与
c.Request.Header.Set("Content-Type", "application/x-www-form-urlencoded")
```

ここは実際にメソッドを呼ぶ為に`gin`のテストのコンテキストを作成しています。
「コントローラのテストをしたい時はこれを使え！」と言わんばかりですね。goｽｹﾞｰ
コンテキストの中のリクエストの中にリクエストメソッド、URIとリクエストボディを格納しています。
一番最後のやつは送信するコンテンツのタイプを指定しています。


それではいざ実行！
```go
// テストのコンテキストを持って実行
controller.CreateUserAction(c)
```
これで想定ではリクエストで指定したユーザが作られたはずです。検証してみましょう。

**※「ユーザが作られたかどうか」はモデルのテストが担保するので、ここではモデルを使った結果、コントローラが何を返すかを確認します。**



```go
//検証
var responseBody map[string]interface{}
_ = json.Unmarshal(response.Body.Bytes(), &responseBody)

//ステータスコードがおかしいもしくは帰ってきたメッセージが想定と違えばダメ
if response.Code != tt.want.code {
	t.Errorf("%d番目のテストが失敗しました。想定返却コード：%d, 実際の返却コード：%d", i+1, tt.want.code, response.Code)
} else {
	//実際に帰ってきたレスポンスの中に想定された値が入っているかどうか
	for key := range tt.want.body {
		//値の存在チェック
		if _, exist := responseBody[key]; exist {

			//値の内容チェック
			if responseBody[key] != tt.want.body[key] {
				t.Errorf("%d番目のテストが失敗しました。想定されたキー「%s」の値:%s, 実際に返却された値:%s", i+1, key, tt.want.body[key], responseBody[key])
			} // else{
			//クリアはここだけ
			// }

		} else {
			t.Errorf("%d番目のテストが失敗しました。想定された「%s」がレスポンスボディに入っていません。", i+1, key)
		}
	}
}
```

まず、
```go
//検証
var responseBody map[string]interface{}
_ = json.Unmarshal(response.Body.Bytes(), &responseBody)
```
ここでレスポンスボディを`string`の`map`にしています。
※`map`の値が`interface{}`である理由は`nil`の場合があるからです。

```go
//ステータスコードがおかしいもしくは帰ってきたメッセージが想定と違えばダメ
if response.Code != tt.want.code {
	t.Errorf("%d番目のテストが失敗しました。想定返却コード：%d, 実際の返却コード：%d", i+1, tt.want.code, response.Code)
} else {
```

ここではまずレスポンスコードを見ます。
テストレスポンスに格納されたコード(実際にコントローラから帰ってきたコード)が、想定とあっているかどうかを見ます。
もしあっていなければ中身がおかしいもしくはモデルの処理がおかしいので見直しましょう。


```go
} else {
	//実際に帰ってきたレスポンスの中に想定された値が入っているかどうか
	for key := range tt.want.body {
		//値の存在チェック
		if _, exist := responseBody[key]; exist {

			//値の内容チェック
			if responseBody[key] != tt.want.body[key] {
				t.Errorf("%d番目のテストが失敗しました。想定されたキー「%s」の値:%s, 実際に返却された値:%s", i+1, key, tt.want.body[key], responseBody[key])
			} // else{
			//クリアはここだけ
			// }

		} else {
			t.Errorf("%d番目のテストが失敗しました。想定された「%s」がレスポンスボディに入っていません。", i+1, key)
		}
	}
}
```

次にステータスコードがあっていた場合ですね。
ステータスコードがあっていた場合、次に確認する必要があるのはレスポンスボディです。
先ほど`map[string]interface{}`にした`responseBody`の中身に、想定したレスポンスが入っているかを確認します。

ここの動きをテストケースの一つと合わせて確認しましょう。

```go
{
	//①: テーブルにレコードが何もない状態で作成するユーザ（コンフリクトは起きないはず）
	model.User{
		Email:    "test@example.com",
		Password: "test password",
		Name:     "test name",
	},
	preferResponse{
		code: http.StatusCreated,
		body: map[string]interface{}{
			"Email": "test@example.com",
			//"Password": "test password", //パスワードも調査した方が良いが、データベース内ではhash化されるため
			"Name": "test name",
		},
	}, //ユーザは作成できるはず
},
```

ユーザ構造体を作り、それをメソッドに流すとどのようなレスポンスが帰ってくるかを定義しています。

```go
preferResponse{
	code: http.StatusCreated,
	body: map[string]interface{}{
		"Email": "test@example.com",
		//"Password": "test password", //パスワードも調査した方が良いが、データベース内ではhash化されるため
		"Name": "test name",
	},
}, //ユーザは作成できるはず
```

先ほどの定義から、
このテストケースの場合、**`http.StatusCreated`(201)が帰ってきてレスポンスボディの中には`Email`と`Password`と`Name`の要素が作成したユーザと等しい**かどうかをチェックするということです。

※パスワードについてはhash化されて帰ってくるので、検証はできません。
※帰ってくる構造体の中の`InsertId`(作成に成功した場合Idが自動で割り振られる)の検証もした方がいいかと思いますが、今回はユーザを別のテストでも利用する為にモックを作成するので、IDの順序が固定にならない可能性を考えて今回はIdの検証はしていません。


これでコントローラのテストを定義できました。
それでは実行してみましょう。

![](https://storage.googleapis.com/zenn-user-upload/t1mu8b2m3cky4ngc6m18sab3qekp)


成功していますね。まだ一つのメソッドについてのテストしか書いていないので、カバレッジは低いですが、該当メソッドのカバレッジをみてみましょう。

![](https://storage.googleapis.com/zenn-user-upload/g9dpovtoxkligfxpi1zq58vsxnnq)

全て網羅できていますね。これでテストは完成です。


# 結論

コントローラ（具体的にいうと、*gin.Contextを引数に持つ関数）のテストを書いてみました。

感想としては、goの設計はすごい。goについて一つ新しいことを学ぶ度に
「goの中にある物事には明確に誰かの思想があり、その決まりが考慮している範囲がとても広い」

ということを感じます。今後もgoの能力を高めていけたらいいなと思います。

以上になります。ありがとうございました。



# 参考

[TableDrivenTests · golang/go Wiki · GitHub](https://github.com/golang/go/wiki/TableDrivenTests)
[Go言語ginフレームワークでアプリ開発~Controllerのテスト編~ | セルフノート](https://selfnote.work/20200512/programming/golang-with-gin3/)
[gin.Contextを引数にもつ関数をunitテストする - Qiita](https://qiita.com/takehanKosuke/items/849c732c5892d149b50a)
[GoでHTTP Clientのテストを書く - Qiita](https://qiita.com/sawadashota/items/47d8e990f27372d4c4c8)
[GAE/Go+ginでHTTPリクエストも含めてEnd to Endなテストをする話 - Qiita](https://qiita.com/CST_negi/items/763cb38e139b158ae9b5)
[Goでhttpリクエストを送信する方法 - Qiita](https://qiita.com/taizo/items/c397dbfed7215969b0a5)

