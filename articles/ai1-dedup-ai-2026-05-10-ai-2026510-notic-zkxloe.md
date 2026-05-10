---
title: "12日間で4モデルが激突：中国系オープンウェイトLLM、どれを選ぶか？実装コード付き選定ガイド"
emoji: "⚔️"
type: "tech"
topics: ["llm", "ai", "openai", "deepseek", "ポエム"]
published: true
---

2026年4月末から5月頭にかけての約12日間で、中国系の主要オープンウェイトLLMが一気に4本リリースされた。GLM-5.1、MiniMax M2.7、Kimi K2.6、DeepSeek V4——どれも「フロンティア級」を謳っており、正直どれを選べばいいか迷う。

ベンチマークを並べるだけの記事はもう十分あるので、この記事では **「何をしたいか」で使い分けるための比較軸** を整理し、実際に試せるコードを添えて解説する。

## 背景：なぜ今これほど激しいのか

GPT-5.5やClaude Opus 4.6が有償モデルの頂点を走る中、オープンウェイト陣営は「コスト＋コントロール」の面で独自のポジションを確立しつつある。特に Kimi K2.6 は SWE-Bench Pro で 58.6% を記録し、GPT-5.4（57.7%）やClaude Opus 4.6（53.4%）を抜いてオープンウェイト初の首位に立った（出典：Artificial Analysis, 2026-04）。

ただし「最高スコア＝最適選択」ではない。コスト・コンテキスト長・ライセンス・レイテンシなど、現場で効いてくる軸はベンチマークとは別にある。

## 4モデルの特性早見表

| | Kimi K2.6 | DeepSeek V4 Pro | GLM-5.1 | MiniMax M2.7 |
|---|---|---|---|---|
| リリース | 2026-04-20 | 2026-04-24 preview | 2026-04末 | 2026-04末 |
| アーキテクチャ | 1T/32B active MoE | 非公開 | 非公開 | 10B active MoE |
| コンテキスト | 256K | **1M** | 128K | 128K |
| SWE-Bench Pro | **58.6%** | 〜58% | 58.4% | 56.2% |
| LiveCodeBench | 89.6% | **93.5%** | - | - |
| 入力単価(官製) | $0.55/1M | $0.44/1M† | 未公表 | 安価 |
| 出力単価(官製) | $2.65/1M | $0.87/1M† | 未公表 | 安価 |
| ライセンス | Modified MIT | 商用OK | 商用OK | 商用OK |
| マルチエージェント | **300 sub-agents** | - | - | - |

†DeepSeek V4 Proは2026/05/31まで75%オフのプロモ価格。通常は$1.74/$3.48。

## 選定基準別：どれを使うか

### 1. 長時間・自律エージェントタスク → Kimi K2.6

Kimi K2.6 の最大の強みは「持久力」だ。公式ブログによると、13時間の継続実行で1,000回以上のツールコール、4,000行超のコード修正を自律的に行ったという。サブエージェントを最大300体まで水平スケールできるアーキテクチャは、現時点でオープンウェイトでは他に並ぶものがない。

CI/CDに組み込んだ自動リファクタリングや、複数リポジトリを横断するバグ修正自動化など、**「ひとつのタスクに長く張り付く」用途**に向いている。

```python
from openai import OpenAI

client = OpenAI(
    api_key="YOUR_MOONSHOT_API_KEY",
    base_url="https://api.moonshot.ai/v1",
)

response = client.chat.completions.create(
    model="kimi-k2.6",
    messages=[
        {
            "role": "system",
            "content": "You are an expert software engineer. Work autonomously until the task is complete.",
        },
        {
            "role": "user",
            "content": "Refactor the authentication module in the provided codebase to use JWT RS256 instead of HS256.",
        },
    ],
    max_tokens=8192,
)
print(response.choices[0].message.content)
```

### 2. 長文書処理・RAG・大規模コードベース解析 → DeepSeek V4 Pro

1Mトークンのコンテキストウィンドウは、現行オープンウェイトで最大クラスだ。モノレポ全体を一度に読ませたり、長大なログファイルを丸ごと渡したりするユースケースでは、チャンクして再結合するより遥かに精度が高い。

さらに5月31日まで75%オフのプロモ価格が続いており、入力 $0.44/1M・出力 $0.87/1M は同性能帯で最安水準。コスパ重視なら今のうちに試す価値がある。

```python
from openai import OpenAI

client = OpenAI(
    api_key="YOUR_DEEPSEEK_API_KEY",
    base_url="https://api.deepseek.com",
)

# 大規模コードベースを丸ごと渡す例
with open("entire_codebase.txt") as f:
    codebase_content = f.read()

response = client.chat.completions.create(
    model="deepseek-v4-pro",
    messages=[
        {
            "role": "user",
            "content": f"以下のコードベース全体を解析し、N+1クエリが発生している箇所をすべて列挙してください。\n\n{codebase_content}",
        }
    ],
    max_tokens=4096,
)
print(response.choices[0].message.content)
```

DeepSeek V4 には Flash（軽量・安価）と Pro（高性能）の2バリアントがある。ルーティングの参考に：

```python
def select_deepseek_variant(task_complexity: str) -> str:
    """タスク複雑度でモデルを振り分ける"""
    if task_complexity == "simple":
        return "deepseek-v4-flash"   # $0.14/$0.28 per 1M
    return "deepseek-v4-pro"         # $0.44/$0.87 per 1M (promo)
```

### 3. コスト最優先・スループット重視 → MiniMax M2.7

MiniMax M2.7 はアクティブパラメータ10Bという軽量MoEアーキテクチャで、SWE-Bench Proスコア56.2%をたたき出す。GLM-5.1（58.4%）との差はわずか2.2%なのに、コストは概ね5分の1程度という試算が出ている。

大量のバッチ推論（コードレビューのCIチェック、ドキュメント自動生成など）を走らせるなら、精度の微差よりコストの差が効いてくる。

### 4. 完全な自社ホスティング・ライセンスの透明性 → Kimi K2.6

Modified MIT ライセンスで公開されているため、商用利用・派生モデル作成ともに比較的制約が少ない。HuggingFace（`moonshotai/Kimi-K2.6`）からウェイトをダウンロードして社内に閉じて運用できる点は、金融・医療・法務系のコンプライアンス要件がある現場で刺さる。

GLM-5.1 や DeepSeek V4 も商用利用は可能だが、ライセンス条件の細部は必ず公式を確認してほしい。

## 実際に複数モデルをA/Bテストするコード

どれを選ぶか迷うなら、同じプロンプトで並列実行して比較するのが早い。

```python
import asyncio
from openai import AsyncOpenAI

PROVIDERS = {
    "kimi-k2.6": {
        "client": AsyncOpenAI(api_key="MOONSHOT_KEY", base_url="https://api.moonshot.ai/v1"),
        "model": "kimi-k2.6",
    },
    "deepseek-v4-pro": {
        "client": AsyncOpenAI(api_key="DEEPSEEK_KEY", base_url="https://api.deepseek.com"),
        "model": "deepseek-v4-pro",
    },
}

PROMPT = "以下のPythonコードを最適化し、計算量を改善してください:\n\ndef find_duplicates(lst):\n    return [x for x in lst if lst.count(x) > 1]"


async def run_single(name: str, config: dict) -> tuple[str, str]:
    response = await config["client"].chat.completions.create(
        model=config["model"],
        messages=[{"role": "user", "content": PROMPT}],
        max_tokens=1024,
    )
    return name, response.choices[0].message.content


async def compare_models():
    tasks = [run_single(name, cfg) for name, cfg in PROVIDERS.items()]
    results = await asyncio.gather(*tasks)
    for name, output in results:
        print(f"\n=== {name} ===\n{output}")


asyncio.run(compare_models())
```

## 選定フローチャート（まとめ）

```
タスクは何時間も続く自律エージェントか？
  → YES: Kimi K2.6（300 sub-agents、13h 実績）

コンテキストが 256K を超える？
  → YES: DeepSeek V4 Pro（1M context）

バッチ量が多くてコスト最優先か？
  → YES: MiniMax M2.7（GLM-5.1の1/5コスト）

上記すべてに当てはまらない標準的なコーディングタスク？
  → DeepSeek V4 Flash（速い・安い・十分な性能）か
     Kimi K2.6（SWE-Bench Pro 首位、オープンウェイト）
```

## おわりに

「最強モデルを選べばいい」という時代は終わりつつある。1Mコンテキストが欲しいならDeepSeek V4、エージェントを長く走らせたいならKimi K2.6、コストを削りたいならMiniMax M2.7——用途ごとに最適解が変わる。

個人的には、まず DeepSeek V4 Flash でコスト感を把握してから、重いタスクだけ Kimi K2.6 に切り替えるというハイブリッド構成が今の「現実解」だと感じている。5月中は DeepSeek V4 Pro のプロモ価格が続くので、この期間に自分のユースケースで比較するのがおすすめだ。