---
title: "wip-neovim-go"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim", "plugin", "go", "rpc", "golang"]
published: false
---

# Neovim Plugin with Go

## Intro
- めちゃ簡単にGoでNeovimのRPCプラグインが作れることを知ったので、試しに作ってみた。
- GoのRPCサーバーをNeovimのRPCクライアントから呼び出すだけのシンプルなもの。
- かなりシンプルな構成で、Goのコードも少ない。

## Concept / How it works
簡単にまとめると、Goで実装した処理をRPCとして用意し、NeovimからRPCを使ってその処理を呼ぶ形になります。

NeovimのAPIをGoで呼び出す際には公式のClientを利用し、RPCという仕組みを活用する形になります。
https://github.com/neovim/go-client

こちらはneovim公式で出しているもので、RPC通信とかNeovimのAPIバインディングとかを良い感じに提供してくれます。

基本的な流れはこんな感じ：

1. **Neovimプラグイン（Lua）側**: `jobstart()`でGoのバイナリをサブプロセスとして起動
2. **Go側**: `nvim.Serve()`でRPCサーバーを立ち上げて、呼び出し可能な関数を登録
3. **通信**: stdin/stdoutを使ってRPCでやり取り
4. **実行**: NeovimからGoの関数を`rpcrequest()`で呼び出し

要するに、重い処理とかGoの豊富なライブラリを使いたい部分はGoで書いて、プラグインのインターフェース部分はLuaで書くという住み分けができる感じですね。

NeovimのPlugin自体はLuaで書いて、その中でGoのプロセスを起動して連携させる形になります。

## Code along

### Requirements

### Project Structure


# Notes

https://gitlab.com/Masterbongo/neovim-go-rpc

https://github.com/masamerc/neovim-go-rpc

This project consists of two parts working together via Neovim's RPC mechanism:

---

### Lua Plugin:
- Defines a `setup()` function to initialize the plugin.
- Launches the Go binary (`.ngr-rpc`) using `vim.fn.jobstart()` with `{ rpc = true }`.
- Caches the RPC channel so the Go binary is only started once.
- Creates a Neovim user command `:Hello` that sends an `rpcrequest()` to the Go RPC process with arguments.

---

### Go App:
- Registers a function called `Hello` that returns a custom message with a random number.
- Sets up a long-running RPC server on `stdin`/`stdout` using `nvim.Serve()`.
- Redirects `os.Stdout` to `os.Stderr` to avoid garbling the RPC stream.
- Meant to be launched as a subprocess from Neovim using Lua (`jobstart` with `rpc = true`).
