---
title: "エージェントの記憶喪失を治す：OSS「Hindsight」をClaude APIに繋いで長期記憶を持たせる"
emoji: "🧠"
type: "tech"
topics: ["ai", "llm", "claude", "python", "agent"]
published: true
---

## セッションが終わるたびに、エージェントは全部忘れる

Claude APIでエージェントを組んでいると、必ずぶつかる壁がある。会話が終わってプロセスが落ちれば、そこまでの文脈は綺麗さっぱり消える。ユーザーの好みも、過去に失敗したアプローチも、次のセッションでは「初対面」からやり直しだ。

RAGでドキュメントを検索させるのとは話が別で、これは「エージェント自身の経験」を覚えさせたいという要求になる。自前でPostgres+pgvectorのテーブルを設計してretain/recallのロジックを書いてもいいが、まず既製品で試して足りない部分だけ自作する方が早い。

今回はOSSのエージェント向け長期記憶システム [Hindsight](https://github.com/vectorize-io/hindsight)(vectorize-io製、MITライセンス)を、素のClaude APIエージェントに繋ぎ込むところまで手を動かす。

## Hindsightとは何か

Hindsightは2026年4月にGitHub Star数1万を突破し、直近のリリースは `v0.10.1`(2026年9月21日、50件超のPRと22人以上のコントリビューターがマージされた)。エージェントメモリのベンチマークであるLongMemEvalでSOTA相当のスコアを主張しており、記憶を「world facts(事実)」「experiences(経験)」「observations(観察)」「mental models(メンタルモデル)」に分けて構造化して持つ点が特徴になっている。

APIはシンプルで3つしかない。

- `retain` — 記憶を書き込む
- `recall` — 記憶を検索する
- `reflect` — 蓄積した記憶を分析し、上位の理解(メンタルモデル)を形成する

単純なベクトル検索のラッパーではなく、`reflect` で「経験の積み重ねから学習する」層を持っているのが他のシンプルなメモリストアとの違いだ。

## 1. サーバーを立てる

Docker一発で立ち上がる。ローカル検証ならこれが一番早い。

```bash
docker run -it --pull always --name hindsight \
  --restart unless-stopped \
  -p 8888:8888 -p 9999:9999 \
  -e HINDSIGHT_API_LLM_API_KEY=$OPENAI_API_KEY \
  -v hindsight-data:/home/hindsight/.pg0 \
  ghcr.io/vectorize-io/hindsight:latest
```

`HINDSIGHT_API_LLM_API_KEY` は記憶の抽出・要約に内部で使うLLM用のキーで、エージェント本体がClaudeでもHindsight側は別プロバイダーで構わない。`8888` がREST API、`9999` は管理系ポート。起動したら疎通確認する。

```bash
curl -s http://localhost:8888/health
```

## 2. クライアントを入れて最小構成で試す

```bash
pip install hindsight-client anthropic
```

まず生のretain/recallだけで感覚を掴む。

```python
from hindsight_client import Hindsight

client = Hindsight(base_url="http://localhost:8888")

client.retain(
    bank_id="user-yukito",
    content="ユーザーはDockerのlatestタグを避ける方針。ピン留めしたバージョンを使う。",
)

result = client.recall(bank_id="user-yukito", query="Dockerタグの方針は?")
print(result)
```

`bank_id` が記憶の名前空間になる。マルチユーザー・マルチプロジェクトで運用するなら、ここをユーザーIDやプロジェクトIDで切るのがそのまま権限分離にもなる。

## 3. Claudeエージェントに組み込む

ここからが本題。セッションの開始時に関連する記憶を`recall`してシステムプロンプトに注入し、セッション終了時に得られた新しい事実を`retain`する、という往復を1つのクラスにまとめる。

```python
import anthropic
from hindsight_client import Hindsight

class MemoryAgent:
    def __init__(self, bank_id: str):
        self.bank_id = bank_id
        self.memory = Hindsight(base_url="http://localhost:8888")
        self.claude = anthropic.Anthropic()

    def _recall_context(self, query: str, limit: int = 5) -> str:
        hits = self.memory.recall(bank_id=self.bank_id, query=query, limit=limit)
        if not hits:
            return ""
        facts = "\n".join(f"- {h['content']}" for h in hits)
        return f"過去のやり取りから分かっている事実:\n{facts}"

    def ask(self, user_message: str) -> str:
        context = self._recall_context(user_message)
        system = "あなたは開発アシスタントです。" + (f"\n\n{context}" if context else "")

        response = self.claude.messages.create(
            model="claude-sonnet-5",
            max_tokens=1024,
            system=system,
            messages=[{"role": "user", "content": user_message}],
        )
        answer = response.content[0].text

        # このやり取り自体を経験として書き戻す
        self.memory.retain(
            bank_id=self.bank_id,
            content=f"Q: {user_message}\nA: {answer}",
        )
        return answer
```

動作確認は別プロセス(別セッション)で行うのが重要。プロセスを一度落として、記憶がDBに残っていることを確かめる。

```python
# セッション1(プロセスA)
agent = MemoryAgent(bank_id="user-yukito")
agent.ask("このプロジェクトではpnpmを使う。npmは使わない。")
# ここでプロセス終了
```

```python
# セッション2(別プロセス)
agent = MemoryAgent(bank_id="user-yukito")
print(agent.ask("パッケージマネージャは何を使えばいい?"))
# → recallでpnpmの方針を拾ってからClaudeが回答する
```

これでプロセスをまたいだ記憶が機能する。RAGのドキュメント検索と違い、ここに入るのは「このエージェントが実際にやり取りした内容」なので、事実確認ではなく方針・好み・過去の失敗の再現防止に効く。

## 4. reflectで経験を要約させる

`retain`を積み重ねただけだと、生の会話ログが線形に溜まっていくだけで`recall`のノイズが増える。定期的に`reflect`を呼んで上位の理解を作らせるのが公式の使い方だ。

```python
client.reflect(bank_id="user-yukito")
```

これを1日1回のcronなどで回しておくと、個別のQ&Aペアから「このユーザーはDockerのlatestタグを嫌う」「pnpmを優先する」といったメンタルモデル的な記憶に集約されていく。リリースノート(v0.10.1)でも「コーディングエージェント向けにautomatic reflectの予算制限を修正」という変更が入っており、reflectを回しすぎるとLLM呼び出しコストが積み上がる点は運用上のパラメータとして意識しておく必要がある。

## つまずいたところ

- `recall`は空配列を返すことがある(まだ何も`retain`していない、または`bank_id`の綴りミス)。ガードを外して素通しすると、システムプロンプトに空文字列の見出しだけが残ってClaudeが混乱するので、上のコードのように空なら注入自体をスキップする。
- `bank_id`を適当な固定文字列にすると、複数ユーザーの記憶が混ざる。最初から「誰の記憶か」を`bank_id`の設計に落とし込んでおく。
- `retain`のcontentを生の会話ログそのまま入れ続けると`reflect`前は検索ノイズが増える。要約してから`retain`する運用にした方が精度は安定した。

## まとめ

HindsightはREST経由のretain/recall/reflectという最小限のAPIで、Claude Agentに「セッションをまたぐ記憶」を持たせられる。自前でpgvectorスキーマを設計する前に、まずこの3操作で要件を満たせるか試す価値はある。今回はローカルDocker+Python SDKでの最小構成だったが、本番運用するなら`bank_id`の権限分離とreflectの実行頻度(コスト)の2点を先に設計しておくと手戻りが少ない。

### 参考

- [vectorize-io/hindsight (GitHub)](https://github.com/vectorize-io/hindsight)
- [Hindsight Releases — v0.10.1](https://github.com/vectorize-io/hindsight/releases)
- [Hindsight Reaches 10,000 Stars](https://hindsight.vectorize.io/blog/2026/04/22/hindsight-10k-stars)