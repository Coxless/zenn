# Articles Directory Agent Instructions

このディレクトリはZennに公開する技術記事を管理しています。

## 記事のフォーマット

記事はMarkdown形式で作成し、以下のフロントマターを含める必要があります：

```yaml
---
title: "記事タイトル"
emoji: "絵文字"
type: "tech"  # tech: 技術記事 / idea: アイデア
topics: ["topic1", "topic2"]
published: false  # 下書きはfalse、公開時はtrue
---
```

## 記事作成コマンド

新しい記事を作成する場合：
```bash
pnpm create-article
```

## 画像の参照方法

記事内で画像を使用する場合は、`/images/{記事のslug}/` ディレクトリに配置し、以下の形式で参照：
```markdown
![alt text](/images/{記事のslug}/filename.png)
```

## 記事のスタイルガイド

- 見出しは`##`から開始（`#`は使用しない）
- コードブロックには言語を指定する
- 補足情報は`:::message`ブロックを使用
- 手順は番号付きリストを使用

## プレビュー

記事のプレビューを確認する場合：
```bash
pnpm preview
```
