---
title: "wip-neovim-go"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim", "plugin", "go", "rpc", "golang"]
published: false
---

# 本記事の内容
この記事ではベーシックな知識のみにはなりますが、Goでneovimのpluginを開発するための基礎ノウハウを書いていきます。
メインの部分では実際に1からneovimの簡単なpluginを作っていきます。

完成したpluginはこちら:
https://github.com/masamerc/wc-demo.nvim

自分の備忘録的な部分もありますが、どなたの役に立てれば幸いです!

# neovimからGoの処理を呼ぶ仕組み 
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

今回作成するpluginは。。。


## 環境
開発・テストした環境は以下の通り:

```
$ go version
go version go1.22.2 linux/amd64

$ nvim -v
NVIM v0.11.2
Build type: RelWithDebInfo
LuaJIT 2.1.1744318430
```

## プロジェクト構成
`wc-demo.nvim`ディレクトリに以下のような構成でファイルを置いていきます。
基本的な構成としては、Goがメインの処理でneovim / luaはそれを呼び出すためのwrapperコード的なもの。

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

:::details main.go

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

:::


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


### Lua・neovim側の実装 (`wc-demo.nvim/lua`)
次にLua / neovim側の実装をしていきます。こちらの構成も非常にsimpleで、`lua`以下にpluginの名前になる`wc-demo`のディレクトリを作成して(neovim pluginシステムの慣習)その中に以下の2つのファイルを作成します。
* `init.lua`: pluginのメインのエントリーポイント
* `rpc.lua`: Goで実装した処理をRPCで呼ぶためのコードで`init.lua`内で使われる

```
wc-demo.nvim
└── lua
    └── wc-demo
        ├── init.lua
        └── rpc.lua
```

#### `init.lua`: pluginの入り口
まずはpluginとしての入り口になる`init.lua`を以下のように用意していきます。

```lua
local M = {}

local rpc = require("wc-demo.rpc")

-- define default options for the plugin
local default_opts = {}

M.setup = function(opts)
    -- override default options if provided via setup call
	opts = vim.tbl_deep_extend("force", default_opts, opts or {})
	rpc.setup(opts)
end

return M

```

この`M`テーブルは、Luaモジュールの標準的なパターンで、プラグインが外部に提供する機能をまとめる役割を果たします。最後に`return M`することで、他のファイルから`require()`でこのモジュールを読み込んだ際に、このテーブルが返されるような仕組みです。

`setup`関数は、ほとんどのneovimプラグインで採用されている初期化パターンです。ユーザーが自分の設定ファイル（通常は`init.vim`や`init.lua`）で以下のように書くことを想定しています：
```lua
require('wc-demo').setup({
  -- ユーザーのオプション設定
})
```

`vim.tbl_deep_extend("force", default_opts, opts or {})`の部分は、デフォルトオプションとユーザーが指定したオプションをマージする処理です。

"force"モードを使うことで、ユーザーのオプションがデフォルトを上書きする形になります。`opts or {}`の部分は、ユーザーがオプションを何も渡さなかった場合に空のテーブルを使うためのLuaイディオムですね。
そして実際の初期化処理は`rpc.setup(opts)`に委譲することで、`init.lua`はシンプルな入り口の役割に徹しています。


#### `rpc.lua`: Goの処理を呼ぶ

`rpc.lua`は実際にGoバイナリとの通信を担当するコードを含みます。以下のように用意をしていきます。

:::details rpc.lua

```lua

local M = {}

---@param opts table options
M.setup = function(opts)
	-- dynamically get the plugin path
	local plugin_path = vim.fn.fnamemodify(debug.getinfo(1, "S").source:sub(2), ":h:h:h")

	local rpc_dir = plugin_path .. "/rpc"
	local binary_path = rpc_dir .. "/bin/server"

    -- [[
    --     Helper Functions
    -- ]]

    -- build the rpc binary if it doesn't exist
	local function ensure_binary()
		local build_cmd = { "go", "build", "-o", binary_path, "." }
		if vim.fn.filereadable(binary_path) == 0 then
            local build_job = vim.fn.jobstart(build_cmd, {
                cwd = rpc_dir,
                on_exit = function(_, exit_code)
                    if exit_code ~= 0 then
                        vim.notify("Failed to build RPC binary", vim.log.levels.ERROR)
                    end
                end,
            })
		vim.fn.jobwait({ build_job })
		end
	end

	-- stores the rpc job id for caching
	local chan

    -- check if the rpc binary exists & if there is an existing job 
    -- if not create a new one
	local function ensure_job()
		if chan then
			return chan
		end
		ensure_binary()
		chan = vim.fn.jobstart({ binary_path }, { rpc = true })
		return chan
	end

    -- [[
    --     Register Commands
    -- ]]

    -- 1: Hello: just greet the caller
	vim.api.nvim_create_user_command("Hello", function(args)
		vim.fn.rpcrequest(ensure_job(), "hello", args.fargs)
	end, { nargs = "*" })

    -- 2: Wc: word count
	vim.api.nvim_create_user_command("Wc", function()
		local text_to_analyze = ""

		-- get visual selection
		local start_pos = vim.fn.getpos("'<")
		local end_pos = vim.fn.getpos("'>")
		local lines = vim.fn.getline(start_pos[2], end_pos[2])

		if #lines == 1 then
			-- single line selection
			text_to_analyze = string.sub(lines[1], start_pos[3], end_pos[3])
		else
			-- multi-line selection
			lines[1] = string.sub(lines[1], start_pos[3])
			lines[#lines] = string.sub(lines[#lines], 1, end_pos[3])
			text_to_analyze = table.concat(lines, "\n")
		end

		vim.fn.rpcrequest(ensure_job(), "wc", { text_to_analyze })
	end, { nargs = "*", range = true })
end

return M

```
:::



このファイルに関しては以下のような3部構成になっているので、それぞれセクション毎に内容を見ていきます:
1. GoバイナリのPath設定
2. Helper関数の用意
3. RPC処理を実際のneovimコマンドとして登録


**GoバイナリのPath設定**

まずは、関数の上部にある部分で実際に利用するGoバイナリのPathを設定しています。

```lua
-- dynamically get the plugin path
local plugin_path = vim.fn.fnamemodify(debug.getinfo(1, "S").source:sub(2), ":h:h:h")

local rpc_dir = plugin_path .. "/rpc"
local binary_path = rpc_dir .. "/bin/server"

```

一番上の行の処理は少しトリッキーですが、現在実行中のLuaファイルのパスから動的にプラグインのルートディレクトリを取得しています。`debug.getinfo(1, "S").source`で現在のファイルパスを取得し、`:h:h:h`で3階層上のディレクトリ（つまりプラグインのルート）を指すようになっています。

最終的に`binary_path`は上記の`plugin_path`に`/rpc/bin/server`をつけたPathに解決されるため、Goのセクションであった`rpc/bin/server`にバイナリとしてはあることを想定しているということになります。

```
wc-demo.nvim
└── rpc
    ├── bin
    │   └── server  # rpc.lua側で設定したバイナリの場所
    ├── go.mod
    ├── go.sum
    └── main.go
```

**Helper関数の用意**

次に、Helper関数として2つの重要な関数が定義されています。

**`ensure_binary()`関数**
```lua
local function ensure_binary()
  local build_cmd = { "go", "build", "-o", binary_path, "." }
  if vim.fn.filereadable(binary_path) == 0 then
    local build_job = vim.fn.jobstart(build_cmd, {
      cwd = rpc_dir,
      on_exit = function(_, exit_code)
        if exit_code ~= 0 then
          vim.notify("Failed to build RPC binary", vim.log.levels.ERROR)
        end
      end,
    })
    vim.fn.jobwait({ build_job })
  end
end
```

こちらはRPCバイナリが存在しない場合に自動的にビルドを実行する役割を担っていて、`vim.fn.filereadable(binary_path) == 0`でバイナリファイルの存在確認を行い、存在しない場合は`go build`コマンドを非同期Jobとしてneovimから実行するというような形です。ビルドが失敗した場合は`vim.notify()`でエラーメッセージを表示してくれます。

**`ensure_job()`関数**
```lua
local chan
local function ensure_job()
  if chan then
    return chan
  end
  ensure_binary()
  chan = vim.fn.jobstart({ binary_path }, { rpc = true })
  return chan
end
```

この関数は、RPCジョブの生成と管理を行います。`chan`変数でジョブIDをキャッシュしており、既存の接続がある場合はそれを再利用する仕組みになっています。

もしまだ接続がなければ、まず`ensure_binary()`を呼び出してバイナリの存在を確認・ビルドした後、`vim.fn.jobstart({ binary_path }, { rpc = true })`でRPCサーバーを起動します。

**RPC処理を実際のneovimコマンドとして登録**

最後のセクションでは、2つのユーザーコマンドを定義してRPC機能をNeovimから利用できるようにしています。

**Helloコマンド**
```lua
vim.api.nvim_create_user_command("Hello", function(args)
  vim.fn.rpcrequest(ensure_job(), "hello", args.fargs)
end, { nargs = "*" })
```

シンプルな動作確認用のコマンドです。`vim.fn.rpcrequest()`を使用してGoサーバーの`hello`メソッドを呼び出し、引数として`args.fargs`（コマンドライン引数）を渡しています。

**Wcコマンド**
```lua
vim.api.nvim_create_user_command("Wc", function()
  local text_to_analyze = ""
  -- get visual selection
  local start_pos = vim.fn.getpos("'<")
  local end_pos = vim.fn.getpos("'>")
  local lines = vim.fn.getline(start_pos[2], end_pos[2])
  if #lines == 1 then
    -- single line selection
    text_to_analyze = string.sub(lines[1], start_pos[3], end_pos[3])
  else
    -- multi-line selection
    lines[1] = string.sub(lines[1], start_pos[3])
    lines[#lines] = string.sub(lines[#lines], 1, end_pos[3])
    text_to_analyze = table.concat(lines, "\n")
  end
  vim.fn.rpcrequest(ensure_job(), "wc", { text_to_analyze })
end, { nargs = "*", range = true })
```

こちらが今回メインの機能となり、ビジュアル選択された範囲のテキストを取得し、文字数をカウントするコマンドを作成しています。

ざっくりと処理の中身を覗くと以下のような感じになっています:
* `vim.fn.getpos("'<")`と`vim.fn.getpos("'>")`でビジュアル選択の開始位置と終了位置を取得
* `vim.fn.getline()`で該当する行を取得
* 単一行の選択の場合は`string.sub()`で部分文字列を切り出す
* 複数行の場合は最初の行と最後の行をトリミングしてから`table.concat()`で結合

最終的に`vim.fn.rpcrequest(ensure_job(), "wc", { text_to_analyze })`でGoサーバーの`wc`メソッドを呼び出して解析対象のテキストを渡している形です。
このコマンド登録をすることでユーザーはneovimから`:Hello`や`:Wc`コマンドを打ってRPC機能を利用できるようになります。

### 動作確認
