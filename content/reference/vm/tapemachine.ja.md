---
title: "Tapemachine"
date: 2019-10-29T19:50:15+01:00
draft: false
---

`TapeMachine` は一般的にに静的な式を実行するのに役立ちます(つまり計算グラフは変更されない)。
静的な性質があるので `TapeMachine` は一度だけコンパイルされ何度も式を実行するのに向いています(線形回帰や SVM など)。

## 技術詳細

`TapeMachine` はインストラクションのリストに対してグラフを事前コンパイルします。
その後、命令を線形に順次実行します。
主なトレードオフはダイナミズムです。
再コンパイルプロセスが必要になるため、グラフを動的に作成することはできません(コンパイルは比較的高価なため)。
ただし、コード生成段階で多くの最適化が行われるため `TapeMachine` で実行されるグラフははるかに高速に実行されます。