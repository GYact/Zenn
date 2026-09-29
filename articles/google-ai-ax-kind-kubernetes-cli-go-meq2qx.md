---
title: "GoogleのAIエージェント基盤「ax」をkindクラスタで動かす実装手順"
emoji: "🧭"
type: "tech"
topics: ["kubernetes", "ai", "cli", "go", "devops"]
published: true
---

## エージェントを「kubectl」で動かす、という発想

GitHubのトレンドを眺めていたら、1日で1,500スター以上を集めているリポジトリが目に留まった。[google/ax](https://github.com/google/ax) — Googleが公開した、AIエージェントのワークロードをKubernetesの上で宣言的に動かすオーケストレーションランタイムだ。

インターフェースは意図的に `kubectl` に寄せてあり、`ax apply` / `ax get` / `ax watch` / `ax ssh` といった見慣れた動詞でエージェントタスクを操作する。ライセンスはApache 2.0、記事執筆時点でスター数は9,000超。中身は「Task / Workspace / Gateway / Model」という4つのマニフェストに分かれていて、Pod・ConfigMap・Service・Secretの感覚がそのままエージェント運用に持ち込まれている構成だ。

「57%の企業がエージェントをproductionに載せ、governanceが新しい課題になっている」という話をよく見かける今、Kubernetesの運用資産をそのまま流用できるこの設計は理にかなっている。今回は実際に手元のkindクラスタでaxを動かすところまでを、コマンドベースで追ってみる。

## 前提条件

axはコントロールプレーンをKubernetesクラスタ上にデプロイする方式なので、ローカルでも `kind` で作ったクラスタで構わない。必要なものは以下の4つ。

- Go 1.27以上
- `ko`(コンテナイメージのビルド・デプロイ用)
- Docker(タスクランナーイメージのビルド用)
- kubeconfigが通ったKubernetesクラスタ(kindでOK)