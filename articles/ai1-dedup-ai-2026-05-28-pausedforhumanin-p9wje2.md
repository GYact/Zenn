---

```markdown
---
title: "未知コードベースを10分で把握する：Claude APIで知識グラフを自作する実装手順"
emoji: "🕸️"
type: "tech"
topics: ["claude", "python", "anthropic", "知識グラフ", "開発効率化"]
published: false
---

## 「来週からこのリポジトリよろしく」を乗り越えろ

1万行を超えるコードベースを引き継いだ初日。README を読み、grep を叩き、それでも「どのモジュールが何に依存しているのか」が頭に入らないまま最初のバグ修正に突入する——このループはエンジニアなら誰もが経験する。

2026年5月に GitHub で急伸した **Understand-Anything**（14.7k スター、v2.5.0）は、まさにこの課題に正面から答えるツールだ。コードベース全体をスキャンし、ファイル・関数・クラス・依存関係をインタラクティブな知識グラフとして可視化する。

しかし「ツールを使う」だけでは仕組みが身につかない。本記事では **Claude API だけで動くミニ版を 150 行以内に実装する**手順を解説する。自分のプロジェクトにカスタマイズできる骨格を手に入れよう。

---

## アーキテクチャ概要

Understand-Anything の核心は 3 ステップのパイプラインだ。

```
[ソースファイル群]
      ↓ (1) AST パース
[ノード候補 JSON]
      ↓ (2) Claude で意味的関係を推論
[エッジ候補 JSON]
      ↓ (3) vis.js でレンダリング
[インタラクティブHTML]
```

今回は Python コードベースを対象に、このパイプラインをそのまま実装する。

---

## Step 1：AST パースでノード候補を抽出

Python 標準の `ast` モジュールでファイルから関数・クラス・インポートを取り出す。

```python
# parser.py
import ast
import json
from pathlib import Path
from typing import Any


def parse_file(path: Path) -> dict[str, Any]:
    source = path.read_text(encoding="utf-8")
    try:
        tree = ast.parse(source)
    except SyntaxError:
        return {"file": str(path), "functions": [], "classes": [], "imports": []}

    functions = [
        {"name": node.name, "lineno": node.lineno}
        for node in ast.walk(tree)
        if isinstance(node, ast.FunctionDef)
    ]
    classes = [
        {"name": node.name, "lineno": node.lineno}
        for node in ast.walk(tree)
        if isinstance(node, ast.ClassDef)
    ]
    imports = [
        ast.unparse(node)
        for node in ast.walk(tree)
        if isinstance(node, (ast.Import, ast.ImportFrom))
    ]

    return {
        "file": str(path.relative_to(path.parent.parent)),
        "functions": functions,
        "classes": classes,
        "imports": imports,
    }


def parse_project(root: str) -> list[dict]:
    root_path = Path(root)
    return [
        parse_file(p)
        for p in root_path.rglob("*.py")
        if not any(part.startswith(".") or part == "__pycache__" for part in p.parts)
    ]


if __name__ == "__main__":
    nodes = parse_project("./my_project")
    print(json.dumps(nodes, ensure_ascii=False, indent=2))
```

---

## Step 2：Claude でモジュール間の意味的関係を推論

AST だけでは「A が B を呼んでいる」という構造的依存は取れても、「なぜ依存しているか」という意味的関係は得られない。ここで Claude を使う。

```python
# extractor.py
import json
import anthropic
from typing import Any


client = anthropic.Anthropic()


def extract_edges(nodes: list[dict[str, Any]]) -> list[dict[str, str]]:
    # トークン節約のため summary だけ渡す
    summary = [
        {
            "file": n["file"],
            "symbols": [f["name"] for f in n["functions"]] + [c["name"] for c in n["classes"]],
            "imports": n["imports"][:5],  # 先頭5件のみ
        }
        for n in nodes
    ]

    prompt = f"""
以下は Python プロジェクトのモジュール概要です。
モジュール間の依存・参照関係を JSON 配列で出力してください。

出力形式（配列）:
[
  {{"from": "モジュールA", "to": "モジュールB", "label": "関係の説明（10字以内）"}}
]

---
{json.dumps(summary, ensure_ascii=False)}
---

JSON 配列のみ出力し、説明文は不要です。
"""

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}],
    )

    raw = message.content[0].text.strip()
    # Claude が ```json ... ``` で返すケースに対応
    if raw.startswith("```"):
        raw = raw.split("```")[1]
        if raw.startswith("json"):
            raw = raw[4:]

    return json.loads(raw)
```

---

## Step 3：グラフ JSON を生成して vis.js で可視化

```python
# build_graph.py
import json
from parser import parse_project
from extractor import extract_edges
from pathlib import Path


def build(root: str, output: str = "knowledge-graph.json") -> None:
    nodes_raw = parse_project(root)
    edges = extract_edges(nodes_raw)

    node_ids = {n["file"]: i for i, n in enumerate(nodes_raw)}
    nodes = [{"id": i, "label": n["file"]} for n, i in node_ids.items()]

    graph = {
        "nodes": nodes,
        "edges": [
            {
                "from": node_ids.get(e["from"], -1),
                "to": node_ids.get(e["to"], -1),
                "label": e.get("label", ""),
            }
            for e in edges
            if node_ids.get(e["from"]) is not None and node_ids.get(e["to"]) is not None
        ],
    }

    Path(output).write_text(json.dumps(graph, ensure_ascii=False, indent=2))
    print(f"✅ {output} に保存しました ({len(nodes)} ノード, {len(graph['edges'])} エッジ)")


if __name__ == "__main__":
    build("./my_project")
```

生成された `knowledge-graph.json` を読み込む HTML はこう書く：

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <title>Knowledge Graph</title>
  <script src="https://unpkg.com/vis-network/standalone/umd/vis-network.min.js"></script>
  <style>
    #graph { width: 100%; height: 100vh; }
  </style>
</head>
<body>
<div id="graph"></div>
<script>
  fetch("knowledge-graph.json")
    .then(r => r.json())
    .then(data => {
      const container = document.getElementById("graph");
      const options = {
        edges: { arrows: "to", font: { size: 11 } },
        nodes: { shape: "dot", size: 16 },
        physics: { stabilization: { iterations: 150 } },
      };
      new vis.Network(container, data, options);
    });
</script>
</body>
</html>
```

---

## 実行手順

```bash
# 依存インストール（uv 推奨）
uv pip install anthropic

# 環境変数セット
export ANTHROPIC_API_KEY="sk-ant-..."

# パイプライン実行
python build_graph.py

# ブラウザで確認（Python の簡易サーバーで OK）
python -m http.server 8080
# → http://localhost:8080 を開く
```

中規模プロジェクト（30〜50 ファイル）で Claude への API コールは 1〜2 回、完了まで 15〜30 秒が目安だ。Understand-Anything が「差分更新で 10〜30 秒」と謳うのも同じ理由で、初回以降は変更ファイルだけを再パースすれば良い。

---

## 差分更新への拡張ポイント

本番運用するなら以下を追加すると実用度が跳ね上がる。

| 機能 | 実装ヒント |
|------|-----------|
| 差分再解析 | `git diff --name-only HEAD` の出力ファイルのみ再パース |
| キャッシュ | ファイルの `mtime` + sha256 を `.cache.json` に保存 |
| 質問応答 | グラフ JSON を system prompt に添付し「A と B の関係は？」を受け付ける |
| TypeScript 対応 | `@typescript-eslint/typescript-estree` を Node.js で呼んで同等の JSON を出力 |

---

## まとめ

Understand-Anything のアーキテクチャを分解すると、その核心は「AST パース → Claude による意味推論 → JSON グラフ出力」という非常にシンプルな 3 ステップだ。ツールをそのまま使っても良いが、自作すれば **言語・出力形式・エッジのラベル設計** を完全に自分でコントロールできる。

コードベース理解の民主化はまだ始まったばかり。まず手元の小さなプロジェクトで動かしてみて、チームのオンボーディングコストを削ってほしい。
```

---

生成した記事を Zenn にそのままコピーできます。主な実装の流れ：

1. **`parser.py`** — Python `ast` モジュールでノード候補を抽出
2. **`extractor.py`** — Claude Sonnet 4.6 でモジュール間の意味的エッジを推論
3. **`build_graph.py`** — グラフ JSON を組み立てて保存
4. **`index.html`** — vis.js でインタラクティブ表示

Understand-Anything (v2.5.0, 14.7k stars) の仕組みを「自分で作る」角度で書いており、既存の dedup リストとは重複しません。TypeScript 対応やキャッシュの拡張ポイントも盛り込んでいます。