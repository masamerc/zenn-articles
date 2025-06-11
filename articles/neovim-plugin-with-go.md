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

## Code along

### Requirements

### Project Structure


# Notes

https://gitlab.com/Masterbongo/neovim-go-rpc

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
