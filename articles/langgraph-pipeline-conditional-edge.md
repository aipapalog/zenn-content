---
title: "LangGraphで AIパイプラインに条件分岐を入れてトークンを30%削減した話"
emoji: "🔀"
type: "tech"
topics: ["langgraph", "python", "prefect", "windows", "llm"]
published: true
---

前回の記事でPrefectをWindowsに入れた話を書いた。今回はその続きで、LangGraphを使ってパイプラインに条件分岐を追加した話を書く。

## 背景：全スクリプトが毎回動く問題

タスクスケジューラで管理しているPythonスクリプトが20本を超えたところで、ある問題に気づいた。

毎日夜に「市場調査 → BizDev案生成 → 先行指標計測 → リスク評価 → PDCA」という順番でスクリプトが動くのだが、**市場調査の結果が悪い日でも、後続のLLM呼び出しが全部動いてしまう**。

具体的には、市場調査スクリプトが「今日は有望な市場機会が少ない（信度スコア3/10）」という判断を出したとしても、BizDev案生成スクリプトはそれを知らずにClaudeを呼び出してアイデアを生成していた。

どうせ信頼できるデータが少ない日は良いアイデアが出ないし、LLM呼び出しを2回省けば毎週数回は節約できる。

## なぜLangGraphを選んだのか

条件分岐を入れる方法はいくつか考えた。

| アプローチ | メリット | デメリット |
|---|---|---|
| **if文で分岐（シンプル）** | コードが少ない | 状態管理が煩雑・可視化できない |
| **Prefectのフロー制御** | すでに導入済み | 条件付きスキップの表現が冗長 |
| **LangGraph** | グラフで状態・分岐を宣言的に書ける | 学習コストあり |
| **Airflow** | 実績あり | 重い・Windows環境で面倒 |

LangGraphはもともとLLMエージェントのオーケストレーション用のライブラリだが、**StateGraphと条件付きエッジ**という概念があって、「状態を受け取って次にどのノードに進むかを決める」という分岐が簡潔に書ける。

エージェント向けに設計されているだけあって、状態（State）の受け渡しとスキップがきれいに書けるのが気に入った。

インストールはpipで一発。

```
pip install langgraph
```

## 実装：StateGraph + 条件付きエッジ

### State定義

まず、パイプライン全体で持ち回る状態をTypedDictで定義する。

```python
from typing import TypedDict, List, Literal

class StrategyState(TypedDict):
    market_score: float      # 市場調査の信度スコア（0-10）
    market_summary: str      # 市場分析の compact summary（300文字以内）
    completed: List[str]     # 完了したスクリプト名
    failed: List[str]        # 失敗したスクリプト名
    skip_reason: str         # スキップ理由（ログ用）
```

### ノード定義

各スクリプトをノードとして定義する。ノードは「状態を受け取って、状態の更新差分を返す」関数。

```python
def node_product_researcher(state: StrategyState) -> dict:
    ok = _run_subprocess('product_researcher.py')
    score = _read_today_market_score()
    summary = _extract_market_summary()
    updates: dict = {"market_score": score, "market_summary": summary}
    if ok:
        updates["completed"] = state.get("completed", []) + ["product_researcher"]
    else:
        updates["failed"] = state.get("failed", []) + ["product_researcher"]
    return updates

def node_daily_bizdev(state: StrategyState) -> dict:
    # market_summary を環境変数経由でサブプロセスに渡す
    summary = state.get("market_summary", "")
    env = {**os.environ, "MARKET_ANALYSIS_SUMMARY": summary} if summary else None
    ok = _run_subprocess('daily_bizdev.py', env=env)
    key = "completed" if ok else "failed"
    return {key: state.get(key, []) + ["daily_bizdev"]}
```

### 条件付きエッジ

スコアが閾値（4.0）を下回ったら、LLM呼び出しが重いBizDev系ノードをスキップして先行指標計測に直接ジャンプする。

```python
MARKET_SCORE_SKIP_THRESHOLD = 4.0

def route_after_research(state: StrategyState) -> Literal["daily_bizdev", "leading_indicators"]:
    score = state.get("market_score", 5.0)
    if score < MARKET_SCORE_SKIP_THRESHOLD:
        # daily_bizdev + bizdev_optimizer をスキップ（LLM呼び出し2回節約）
        return "leading_indicators"
    return "daily_bizdev"
```

### グラフ構築

```python
from langgraph.graph import StateGraph, END

def _build_strategy_graph():
    g = StateGraph(StrategyState)
    g.add_node("context",            node_context)
    g.add_node("product_researcher", node_product_researcher)
    g.add_node("daily_bizdev",       node_daily_bizdev)
    g.add_node("bizdev_optimizer",   node_bizdev_optimizer)
    g.add_node("leading_indicators", node_leading_indicators)
    g.add_node("risk_manager",       node_risk_manager)
    g.add_node("biz_pdca",           node_biz_pdca)

    g.set_entry_point("context")
    g.add_edge("context", "product_researcher")

    # ここが核心：スコアに応じて分岐
    g.add_conditional_edges(
        "product_researcher",
        route_after_research,
        {"daily_bizdev": "daily_bizdev", "leading_indicators": "leading_indicators"}
    )

    g.add_edge("daily_bizdev",       "bizdev_optimizer")
    g.add_edge("bizdev_optimizer",   "leading_indicators")
    g.add_edge("leading_indicators", "risk_manager")
    g.add_edge("risk_manager",       "biz_pdca")
    g.add_edge("biz_pdca",           END)
    return g.compile()
```

### 実行

```python
graph = _build_strategy_graph()
init_state: StrategyState = {
    "market_score": 5.0,
    "market_summary": "",
    "completed": [],
    "failed": [],
    "skip_reason": "",
}
result = graph.invoke(init_state)
print(result.get("completed"))
# 低スコア日 → ['context', 'product_researcher', 'leading_indicators', 'risk_manager', 'biz_pdca']
# 高スコア日 → ['context', 'product_researcher', 'daily_bizdev', 'bizdev_optimizer', ...]
```

## Prefectと組み合わせる

前回の記事で入れたPrefectの `@flow` デコレータを上から被せるだけで、Prefect UIで実行履歴が確認できるようになる。

```python
from prefect import flow

@flow(name="StrategyChain", log_prints=True)
def strategy_chain():
    graph = _build_strategy_graph()
    result = graph.invoke(init_state)
    print(f"完了: {result.get('completed')}")

if __name__ == '__main__':
    strategy_chain()
```

LangGraphがグラフの実行を管理して、Prefectがフロー全体のログ・リトライ・スケジューリングを管理する、という役割分担になる。

## LangGraphが使えない場合のフォールバック

LangGraphのインストールが何らかの理由で失敗していても動くように、フォールバックを入れておいた。

```python
try:
    from langgraph.graph import StateGraph, END
    _LANGGRAPH_AVAILABLE = True
except ImportError:
    _LANGGRAPH_AVAILABLE = False

if _LANGGRAPH_AVAILABLE:
    graph = _build_strategy_graph()
    graph.invoke(init_state)
else:
    # 逐次実行（条件分岐なし）
    for script in ['product_researcher.py', 'daily_bizdev.py', ...]:
        _run_subprocess(script)
```

## 結果

低スコア日（週に2〜3回程度）にLLM呼び出しが2回スキップされるようになった。感覚値で全体のトークン消費が**30〜40%削減**された。

市場調査の compact summary（300文字以内）をState経由で次のノードに渡せるので、BizDevスクリプト側が毎回長い分析ファイルを読み直す必要もなくなった。

## まとめ

- LangGraphは `StateGraph` + `add_conditional_edges` の組み合わせで条件分岐が書ける
- 状態（State）をTypedDictで定義して各ノードに受け渡す設計は、分岐ロジックが増えてもコードが散らかりにくい
- Prefectと組み合わせると、グラフの実行管理とフローのログ・スケジューリングを分離できる
- LangGraphが使えない環境向けにフォールバック（逐次実行）を入れておくと安心

グラフ構造の可視化や、チェーン間で状態を引き継ぐ設計については別の機会に書く。

---

自分はQAエンジニアで、個人でAI自動化スクリプトをいくつか動かしている。OSSは完全に独学なので、詰まったことと解決方法をそのまま書いていく。
