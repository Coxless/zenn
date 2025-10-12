---
title: "DevcontainerでAtcoderの環境を構築してみた (Rust)"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["競技プログラミング", "atcoder", "rust","devcontainer"]
published: true
---

:::message
この記事は 2025年10月12日時点の内容です。AtCoderが対応するRustのバージョンやcargo-competeの仕様は今後変更される可能性があるため、最新情報は公式を確認してください。
:::

## 概要

Rustの勉強のため競技プログラミングをやってみようと思い、**DevContainer + cargo-compete** を使って構築しました。以下のような工夫を加えています：

- DockerベースのRust開発環境をDevContainerで定義  
- Rustツールチェインと競プロ用ツールの自動インストール（cargo-equip, cargo-compete, cargo-udeps）  
- Rustのアップデートと依存の事前フェッチ  
- キャッシュとビルド成果物の永続化（cargoとtargetディレクトリをvolume化）  

### Github Repository
作成した環境は以下のGithubリポジトリで公開しています。

https://github.com/Coxless/atcoder-starterkit-rust

## 使用ツールと前提
- VS Code + devcontainer 拡張
- Docker (Podman未検証)
- Rust toolchain v1.70.0 (Atcoder対応バージョン)
- cargo-compete（AtCoder提出補助ツール）

## 環境構築手順
### 1. リポジトリをクローン
``` bash
git clone https://github.com/your-repo-link
cd atcoder-starterkit-rust
```

### 2. Devcontainerで開く
- VS Code で「Reopen in Container」で開く。

### 2. rust-analyzer の設定
- コンテナ内で動作するRustに合わせて互換性のある rust-analyzer をインストールする。
``` bash
code --install-extension rust-lang.rust-analyzer@0.3.1402
```

:::message
最新のrust-analyzerは対応していないため注意!
:::

### 3. cargo-competeでlogin
`cargo compete login atcoder`でログインする。

:::message
25/10/12現在、`cargo compete login atcoder`でログインすることができなくなっています。

以下の手順で対応してください。
- ブラウザで AtCoder にログイン
- DevTools で REVEL_SESSION cookie を取得
- `$HOME/.local/share/cargo-compete/cookies.jsonl` に貼り付け
- 動作確認：
``` bash
cargo compete login atcoder
# → "Already Logged in" が表示されればOK
```

参考: [cargo-competeでAtCoderにログインできないときの対処法](https://zenn.dev/stddev/articles/388b4d2acb248e)
:::

### 4. コンテストの開始とtest・submit
cargo competeを使いコンテストに参加します。

簡単な例↓

``` bash
cargo compete new abc054
cd abc054
cargo compete test a
cargo compete submit a
```

- 問題ごとにテンプレートとテストケースが生成されます。
- submit が失敗する場合は手動提出してください。

### その他
- cargo-compete の使い方詳細は [公式README（日本語）](https://github.com/qryxip/cargo-compete/blob/master/README-ja.md) を参照してください。

## おわりに
- DevContainerによりRustでAtcoderに挑戦する環境を構築しました。
- ばりばりAtcoder挑戦してつよつよエンジニアを目指します。
- 不明な点・問題点があればコメントください。
