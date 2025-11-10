# Mind Evolution 仕様再設計ノート（世界線探索向け）

作成日: 2025-11-10  
作成者: GPT-5 Codex

---

## 1. 元論文および既存実装の要点
- 論文「Evolving Deeper LLM Thinking」（DeepMind, 2025）は、LLM に複数の思考モードを生成させ、進化的に改善するフレームワーク（Mind Evolution, ME）を提案。
- コア要素:
  - **Islands Model**: 複数の独立した人口（アイランド）で探索し、周期的に移民（migration）を行い多様性を確保。
  - **Refinement Critical Conversation (RCC)**: Author/critic の会話で解を洗練するプロセス。
  - **Tournament Selection / Recombination / Mutation**: 適応度に基づき親の選択と交叉・突然変異を行う進化的操作。
- lucidrains/mind-evolution 実装は骨組みのみ（関数がほぼ未実装）。論文実装を参考にしつつ、Kimi CLI 世界線探索向けに再設計が必要。

## 2. 世界線探索へのマッピング方針
### 2.1 個体（Solution）の定義
- 個体 = ある時点の世界線状態（チェックポイント、コンテキスト、記憶リンク、ツール履歴、評価メトリクス）。
- 染色体表現:
  - `worldline_id`（親の継承先）
  - `decision_trace`: 主要なツール呼び出しシーケンス（Task, WorldlineFork, Think 等）
  - `memory_refs`: 参照した記憶/ハイライト ID
  - `agent_prompts`: RCC 用の Author 生成テキスト
  - `metrics_snapshot`: 過去評価値

### 2.2 適応度（Fitness）関数
- 世界線の有用性・安定性を複合評価する指標群を設計:
  1. **Temporal Consistency**: 未来チェックポイントとの矛盾度。低いほど良い。
  2. **Divergence Score**: ベース世界線との差分量（コード差分、メモリ差分、アクション差分）。一定範囲内で正規化し、多様性確保。
  3. **Task Progress**: サブタスク達成度。外部シグナル（テスト結果、ユーザ指示達成度）と連携。
  4. **Review Feedback**: Review Agent からの承認履歴（approved=+、rejected=-）。
  5. **Resource Cost**: 消費トークン・時間・ツール数。低いほど良い。
- 複合スコア例:
  ```
  fitness = w1 * temporal_consistency
          + w2 * (1 - divergence_normalized)
          + w3 * task_progress
          + w4 * review_feedback
          - w5 * resource_cost
  ```
  - 重み `w1..w5` はメタ設定（ユーザー調整可能）。Review Agent 承認前後で更新。

### 2.3 RCC（Refinement Critical Conversation）の適用
- Author: 世界線候補（ノード）の説明を生成するエージェント。`summary + highlight + metrics` を踏まえ、改善案を言語化。
- Critic: 批評役。Author の提案に対し矛盾や欠落を指摘し、改善方向を提示。
- RCC を `kosong` ベースで実装し、以下パターンを組み合わせる:
  1. **Self-Conversation**: 同一エージェントが author/critic を交互にシミュレート。
  2. **Subagent-Driven**: Specialist sub-agent (e.g., Review Agent) が critic を担当。
- 会話ターン数 (`turns`) は 4～6 を想定。RCC 結果から新世界線策定:
  - Critic が指摘した改善点を `WorldlineFork` ツールに落とし込み、分岐世界線を生成。
  - Author の応答から `memory_links` を更新し、PageIndex に論考を記録。

### 2.4 島モデル（Islands）再設計
- `num_islands`: 3～5 の探索プールを生成。各島は異なる探索戦略／重みを持つ。
  - Island A: Temporal Consistency 重視
  - Island B: Divergence（創造性）重視
  - Island C: Resource Cost 抑制
  - Island D (optional): Review Feedback 重視
- 初期個体生成:
  - 基本世界線（W0）から Task ツールや Think ツールをトリガして多様なスタート地点を作成。
  - サブエージェントを利用した parallel investigation を実行し、各島に成果を割り当て。
- 進化ループ:
  1. 各島で RCC + Mutation + Recombination を適用して新世界線を提案。
  2. `generations` ごとに最下位島をリセット（論文の reset 概念）。
  3. `migration` 間隔で高スコア世界線を隣島へ移住させる（メトリクスリセット or 加重平均）。
- Mutation / Recombination の具体例:
  - **Mutation**: 直近のツール呼び出し/サブタスクのパラメータを微調整。例: 参照する記憶を変更、別サブエージェントに切替。
  - **Recombination**: 2 つの世界線の決定トレースを交差し、新しい Fork Plan を生成。
  - Kimi CLI 側では Worldline Manager がこれら操作を提供する API を実装。

## 3. 実行フロー（擬似コード）
```
initialize_worldline_graph()
seed_islands = initialize_islands(worldline_graph, num_islands)

for generation in range(max_generations):
    for island in seed_islands:
        parents = tournament_selection(island.population, k)
        child_plan = recombination(parents)
        mutated_plan = mutation(child_plan)
        child_worldline = run_worldline_plan(mutated_plan)  # 世界線分岐実行

        # RCC refinement
        refined_worldline = rcc_refinement(island.author, island.critic, child_worldline)

        fitness_score = evaluate_worldline(refined_worldline)
        island.update_population(refined_worldline, fitness_score)

    if generation % migration_interval == 0:
        migrate(seed_islands)

    if generation % reset_interval == 0:
        reset_lowest_islands(seed_islands)

best_worldlines = select_top_worldlines(seed_islands)
```

- `run_worldline_plan()` が Kimi CLI 側の実装責務。`WorldlineFork`, `Task`, `Think` 等を駆使して新世界線を生成し、PageIndex へメモリ登録。
- `rcc_refinement()` は `kosong` LLM を利用して Author/Critic 会話を実施。結果世界線の `summary`, `highlight`, `metrics` を更新。
- `evaluate_worldline()` は適応度関数を呼び出し、Review Agent の承認結果も加味。

## 4. 追加検討事項
1. **並列実行**: 島毎にサブエージェントを走らせるため、Kimi CLI のイベントループ設計と整合を取る（Task ツール＋WorldlineFork が並行実行可能か）。
2. **リソース制約**: トークン上限/時間制約を surpass しないよう、各島に Resource Budget を設定。
3. **収束判定**: 一定基準（fitness 不変、Review approved, divergence < ε）で世代ループを停止。
4. **ログ追跡**: 島ごとの進捗、RCC 対話ログ、評価スコアを PageIndex に格納し、UI から分析できるようにする。
5. **安全措置**: 破壊的操作（ファイル書き換え等）は Review Agent 承認後に適用。進化実験中は一時ワークスペース（sandbox）を使用し、承認後に main branch へ merge。

---

以上を基に、実装フェーズで `WorldlineManager`, `ReviewAgent`, `MindEvolutionEngine` を詳細設計する。*** End Patch
