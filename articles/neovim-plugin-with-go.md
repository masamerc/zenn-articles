---
title: "wip-neovim-go"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim", "plugin", "go", "rpc", "golang"]
published: false
---

# 本記事の内容 / Intro
この記事ではベーシックな知識のみにはなりますが、Goでneovimのpluginを開発するための基礎ノウハウを書いていきます。
メインの部分では実際に1からneovimの簡単なpluginを作っていきます。

感性したpluginはこちら:
https://github.com/masamerc/wc-demo.nvim

自分の備忘録的な部分もありますが、どなたの役に立てれば幸いです!

# 仕組み/ How it works 
構成としてはGoのRPCサーバーで提供される処理をneovimのクライアントから呼び出すだけのシンプルなものになります。
neovimの公式Clientがあるのでそちらを使って、Go側の処理を実装しそれをLuaでneovimから呼び出すといった形です。

https://github.com/neovim/go-client

こちらはneovim公式で出しているもので、RPC通信とかNeovimのAPIバインディングとかを良い感じに提供してくれます。

基本的な流れ：

1. **Neovimプラグイン（Lua）側**: `jobstart()`でGoのバイナリをサブプロセスとして起動
2. **Go側**: `nvim.Serve()`でRPCサーバーを立ち上げて呼び出し可能な関数を登録
3. **通信**: stdin/stdoutを使ってRPCでやり取り
4. **実行**: neovimからGoの関数を`rpcrequest()`で呼び出し

重い処理とかGoの豊富なライブラリを使いたい部分はGoで書いて、プラグインのインターフェース部分はLuaで書くという住み分けができる感じですね!

# 実際に作ってみよう / Code Along
ここから実際に一からとりあえず動くpluginを`wc-demo.nvim`というディレクトリを作って開発していきます。

## 環境 / Environment
開発・テストした環境は以下の通り:

```
$ go version
go version go1.22.2 linux/amd64

$ nvim -v
NVIM v0.11.2
Build type: RelWithDebInfo
LuaJIT 2.1.1744318430
```

## プロジェクト構成 / Project Structure
`wc-demo.nvim`ディレクトリに以下のような構成でファイルを置いていきます。

```
wc-demo
│
├── lua              # neovimが使うluaのpluginファイルを置く場所
│   └── wc-demo
│       ├── init.lua
│       └── rpc.lua
│
└── rpc              # Go側の処理を実装する場所
    ├── bin
    │   └── server
    ├── go.mod
    ├── go.sum
    └── main.go
```

## Pluginの作成
まずは今回実装する処理だが、あくまでもDEMOなので、かなりシンプルなものにして以下のような２つ:
- Hello: 単純に指定された名前にGreet
- Wc: Unix wcを模したやつでneovim内でselectしているところのline, word, charカウントを表示するだけ


### Go側の実装 (`wc-demo.nvim/rpc`)
まずはGo側の処理に焦点をあててく。以下の中でも一旦はmain.goだけ見ていけばよい。

```
└── rpc
    ├── bin
    │   └── server
    ├── go.mod
    ├── go.sum
    └── main.go
```

今回の必要な外部packageも一つだけであり、それがIntroでも触れたClientだ。
`wc-demo.nvim/rpc`内で以下を実行しよう

```
$ go mod init wc_demo
$ go get -u github.com/neovim/go-client
```

#### `main.go`
次に以下の内容の`main.go`ファイルを用意します。


まずは、neovim側で利用するロジックを実装しよう
大まかな説明は以下の通り:
- `Hello`, `WordCount`が今回neovimに提供する機能・サービスの一つ一つ
- 上記はそれぞれ`*nvim.Nvim` (neovim instance) structに対するmethodsとして実装してあげる。
  - function signature
- 今回は基本的にはneovim上で何かしらの文字出力がOutputなので、どちらもv.WriteOutを使って終了
- Helloの説明
- WordCountの説明
```go
package main

import (
	"fmt"
	"log"
	"os"
	"strings"

	"github.com/neovim/go-client/nvim"
)

// function / handler exposed to neovim via RPC
func Hello(v *nvim.Nvim, args []string) error {
	return v.WriteOut(fmt.Sprintf("Hello %s!!\n", strings.Join(args, " ")))
}

// another function / handler for neovim
func WordCount(v *nvim.Nvim, args []string) error {
	if len(args) == 0 {
		return v.WriteOut("Usage: WordCount <text>\n")
	}

	text := args[0]

	// count words (split by whitespace and filter empty strings)
	words := strings.Fields(text)
	wordCount := len(words)

	// count characters (excluding spaces)
	charCount := len(strings.ReplaceAll(text, " ", ""))

	// count lines
	lineCount := strings.Count(text, "\n") + 1
	if text == "" {
		lineCount = 0
	}

	result := fmt.Sprintf(`
Lines: %d
Words: %d
Characters: %d
`, lineCount, wordCount, charCount)

	return v.WriteOut(result)
}

// binary entry point
func main() {
    // ----- snip -----
}
```

次にバイナリのEntrypointとなるmain関数似ついて以下のようなものを用意します。
細かい説明はコメントで記載している通りですが、重要な部分をPick upすると以下のような流れになります。
1. `v := nvim.New()`でneovimインスタンスを取得して`v.RegisterHandler()`にて上記で定義したそれぞれの処理を登録する
2. `v.Serve()`呼んでRPC message loopを開始する

APIとしては非常にGoでHTTP Serverを作る時に似ているので、馴染みやすいと感じるケースもあるような気がします。

TODO: stdin stdoutあたりのわかりやすい説明を書く。

```go
// binary entry point
func main() {
	// turn off timestamps in output.
	log.SetFlags(0)

	// direct writes by the application to stdout garble the rpc stream.
	// redirect the application's direct use of stdout to stderr.
	stdout := os.Stdout
	os.Stdout = os.Stderr

	// create a client connected to stdio. the
	// sconfigure the client to use tandard log package for logging.
	v, err := nvim.New(os.Stdin, stdout, stdout, log.Printf)
	if err != nil {
		log.Fatal(err)
	}

	// register functions with the client.
	v.RegisterHandler("hello", Hello)
	v.RegisterHandler("wc", WordCount)

	// run the RPC message loop.
	// the Serve function returns when nvim closes.
	if err := v.Serve(); err != nil {
		log.Fatal(err)
	}
}

```

<details><summary>Full Code</summary>

```go
package main

import (
	"fmt"
	"log"
	"os"
	"strings"

	"github.com/neovim/go-client/nvim"
)

// function / handler exposed to neovim via RPC
func Hello(v *nvim.Nvim, args []string) error {
	return v.WriteOut(fmt.Sprintf("Hello %s!!\n", strings.Join(args, " ")))
}

// another function / handler for neovim
func WordCount(v *nvim.Nvim, args []string) error {
	if len(args) == 0 {
		return v.WriteOut("Usage: WordCount <text>\n")
	}

	text := args[0]

	// count words (split by whitespace and filter empty strings)
	words := strings.Fields(text)
	wordCount := len(words)

	// count characters (excluding spaces)
	charCount := len(strings.ReplaceAll(text, " ", ""))

	// count lines
	lineCount := strings.Count(text, "\n") + 1
	if text == "" {
		lineCount = 0
	}

	result := fmt.Sprintf(`
Lines: %d
Words: %d
Characters: %d
`, lineCount, wordCount, charCount)

	return v.WriteOut(result)
}

// binary entry point
func main() {
	// turn off timestamps in output.
	log.SetFlags(0)

	// direct writes by the application to stdout garble the rpc stream.
	// redirect the application's direct use of stdout to stderr.
	stdout := os.Stdout
	os.Stdout = os.Stderr

	// create a client connected to stdio. the
	// sconfigure the client to use tandard log package for logging.
	v, err := nvim.New(os.Stdin, stdout, stdout, log.Printf)
	if err != nil {
		log.Fatal(err)
	}

	// register functions with the client.
	v.RegisterHandler("hello", Hello)
	v.RegisterHandler("wc", WordCount)

	// run the RPC message loop.
	// the Serve function returns when nvim closes.
	if err := v.Serve(); err != nil {
		log.Fatal(err)
	}
}

```

</details>


### Lua・Neovim側の実装 (`wc-demo.nvim/lua`)