# DeepWiki 調査サマリ（Reading Steiner 関連）

作成日: 2025-11-10  
作成者: GPT-5 Codex

---

## 1. 4.2 Meta Commands
- `/`プレフィックスのメタコマンドは `ShellApp` がユーザー入力を解析する段階で処理される。`get_meta_command()` が `_meta_commands` / `_meta_command_aliases` の二重レジストリを参照し、`MetaCommand` タプル（名前・説明・実行関数・エイリアス・KimiSoul専用フラグ）を返す。
- 登録は `@meta_command` デコレータで行い、同期/非同期関数どちらも登録可能。`kimi_soul_only=True` のコマンドは `ShellApp` 側でアクティブな Soul が `KimiSoul` か検証される。
- `/help`, `/version`, `/release-notes`, `/feedback`, `/exit` など汎用コマンドの構造・出力例が整理されている。
- 設計示唆: Reading Steiner もメタコマンドとして登録し、既存レジストリと同一フローに乗せる。エイリアス、説明文、KimiSoul限定フラグなどを正しく定義する必要がある。

## 2. 4.1 Interactive Shell Mode
- `ShellApp` が対話型モードの中核。入力ループでは `exit/quit` 系、`/` コマンド、シェルコマンド、エージェント入力を振り分ける。
- `CustomPromptSession` が思考モード切替・補完などを管理。`visualize()` がリアルタイム UI（Richベース）を提供。
- 設計示唆: 世界線ビューを新しいタブとして `visualize` に追加する場合、キーボードイベントとレンダリング更新ロジックに統合する必要がある。

## 3. 5.1 Agent Specification Format / 5.2 Agent Loading and Initialization
- YAML仕様で `version`, `agent` セクションを持ち、`name`, `system_prompt_path`, `tools` が必須。`extend` により継承可能で、`system_prompt_args` と組み合わせてテンプレート置換を行う。
- `load_agent()` が YAML → `AgentSpec` → `Agent` の生成までを担う。`BuiltinSystemPromptArgs`（現在時刻、作業ディレクトリ、`AGENTS.md` 内容等）と、仕様で定義された引数をマージしてプロンプトを構築。
- 設計示唆: Reading Steiner に関わるツールや Review Agent を導入する場合、YAML構造での宣言／継承に対応するための仕様拡張や例示が必要。

## 4. 5.3 Subagents and Task Delegation / 6.4 Task Management Tools
- `Task` ツールは指定サブエージェントをロードし、独立した履歴ファイル (`session_sub_*.jsonl`) とコンテキストで実行する。
- `_SubWire` がサブエージェントからの `ApprovalRequest` を親にバブルアップし、それ以外のイベントを遮断する。サブエージェントは SendDMail 等の時間操作ツールを持たず、再帰的委譲も抑制。
- レスポンスが短すぎる場合、自動で追記プロンプトを送り詳細を書かせる継続フローが存在。
- 設計示唆: 並列世界線上で複数サブエージェントを走らせる場合、各世界線IDを付与した履歴ストレージと `_SubWire` の拡張（世界線識別情報、レビュー要求転送など）が必要。

## 5. 6.5 Time Travel and Context Tools
- `SendDMail` ツール → `DenwaRenji` → `BackToTheFuture` 例外 → `Context.revert_to()` の流れで過去チェックポイントへ巻き戻す。
- 各ステップで自動チェックポイント (`Context.checkpoint()`) を作成。DenwaRenji がチェックポイント数を同期し、D-Mail はステップ完了後に処理される。ツール呼び出しが拒否された場合は D-Mail 無効化。
- コンテキスト操作は `asyncio.shield` でキャンセル耐性を確保。コンパクション時はチェックポイント0へ戻って再構築。
- 設計示唆: Reading Steiner の世界線分岐は既存チェックポイント機構を拡張利用するが、複数枝の管理・可視化・記憶分離を追加で設計する必要がある。

## 6. 7.2 Approval System
- `Approval` クラスがキューイング・YOLOモード・セッション単位承認などを管理。`ApprovalRequest`（UUID, `tool_call_id`, `sender`, `action`, `description`）が `Wire` 経由で UI に届く。
- UI 側は Rich パネルでリクエスト表示、上下キーで選択、Enter で承認／拒否／セッション承認。`APPROVE_FOR_SESSION` 選択時には同種アクションを一括承認する仕組みがある。
- 設計示唆: Review Agent が生成するレビュー要求はこの Approval パイプラインに流し、半自動で「承認が必要な状態」を生成するのが自然。必要であれば専用 `action` 名や `description` 生成規約を定義。

---

### Reading Steiner への全体的インプリケーション
1. **新メタコマンド設計**  
   - `/reading-steiner` は既存 `meta_command` デコレータで登録し、KimiSoul専用フラグを立てる。タブ表示・世界線の状態遷移などの説明を `description` に明記。
2. **世界線ビュー UI**  
   - `visualize()` のタブ切り替え (`KeyEvent`) を拡張。Rich Table/Tree を用いて「1行要約＋ハイライト」をノードとして描画。承認要求や世界線マージ操作は追加キーバインド/パネルで表現。
3. **サブエージェント連携**  
   - `Task` ツールのコンテキスト分離を活用しつつ、世界線IDとメモリストアを紐付ける。`_SubWire` に世界線メタデータを付加し、Review Agent/Approval システムと連携。
4. **タイムトラベル基盤の再利用**  
   - 既存チェックポイントと `DenwaRenji` ロジックを踏まえ、世界線グラフ管理（複数枝）とバタフライ効果再評価を追加する。過程ログは PageIndex で索引化し、必要時に `kosong` LLM で要約抽出。
5. **Approval 拡張**  
   - Review Agent が `Approval.request()` を通じてレビュー依頼を生成し、ユーザー承認後に世界線マージ／再評価を実行。`APPROVE_FOR_SESSION` や YOLO モードとの整合性を考慮した設計が必要。

以上を `/_project/planning/reading_steiner_plan.md` の設計検討・詳細化に活用する。
