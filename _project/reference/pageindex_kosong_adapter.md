# PageIndex × Kosong 統合設計メモ

作成日: 2025-11-10  
作成者: GPT-5 Codex

---

## 1. 目的
PageIndex 既存実装は OpenAI ChatGPT API と `tiktoken` に依存しており、そのままでは Kimi CLI（`kosong` ベース）と整合しない。  
本ノートでは、世界線メモリ索引化に PageIndex の処理パイプラインを再利用するためのアダプタ設計をまとめる。

## 2. 既存依存関係（`utils.py` 抜粋）
- `openai.OpenAI` / `AsyncOpenAI` を直接呼び出し、プロンプト単位でチャット補完を実行。
- `tiktoken.encoding_for_model()` によりトークン数を算出。
- PDF/Markdown 処理に `PyPDF2`, `pymupdf`, `yaml` 等を使用。

## 3. Kimi CLI 側の制約
- LLM 呼び出しは `kosong` の `ChatProvider` 経由（`kimi_cli.llm`）。  
  - `kosong` の thinking/off モードやツール呼び出し連携を維持する必要あり。  
  - 既存 `Think` 等との併用を考慮し、同期/非同期インタフェースをラップする。
- トークンカウントは `kosong` 由来の `LLM` から取得可能な `count_tokens` 等が無いため、別途 `tiktoken` 互換 API か `kosong` 側の `max_context_size` を用いた概算が必要。
- PageIndex の処理は長時間（大量ページ処理）になることがあるため、非同期化とキャンセル制御（`asyncio`）が重要。

## 4. アダプタアーキテクチャ案
```
PageIndexKosongAdapter
├── LLMClient (wraps kimi_cli.llm.LLM)
│    ├── chat(prompt, history) -> str
│    └── chat_async(prompt, history) -> str
├── TokenCounter
│    ├── count(text) -> int (fallback: heuristics)
│    └── set_encoder(encoder)
├── PromptTemplates (reuse PageIndex defaults)
└── IOHelpers (PDF/Markdown処理はそのまま活用)
```

### 4.1 LLMClient 実装方針
- `kimi_cli.llm.LLM` を受け取り、`with_thinking` 等の設定を明示。  
- 同期版 (`chat`): `asyncio.run()` は避ける。Kimi CLI 内から呼ばれる場合は既存イベントループがあるため、`async` バージョンを基本とし、どうしても同期 API が必要なら `asyncio.run_coroutine_threadsafe` 等を検討。
- 非同期版 (`chat_async`): `kosong.step()` をラップせず、`ChatProvider.chat()` に直接アクセスする軽量関数を用意するか、専用 minimal provider を構築。  
  - 大規模履歴を送らないようプロンプトテンプレートを最適化。

### 4.2 TokenCounter
- 選択肢:
  1. `tiktoken` をそのまま利用（ライブラリを依存に追加）。  
  2. 空間節約のため `len(text) // 4` の近似など簡易推定を使用。  
- PageIndex のノード選択・間引き (`tree_thinning_for_index`) に正確なトークン数が必要な 場合は tiktoken を導入する方向かなり濃厚。

### 4.3 Prompt 調整
- 既存プロンプトは ChatGPT 前提（JSON 返却など）。  
- Kimi CLI への適用時は:
  - `system` メッセージに再利用可能なテンプレートを組み込む。
  - Review Agent や世界線情報を prompt に含め、コンテキストに沿った目次抽出/要約が行えるよう拡張。
  - 出力 JSON の schema を定義し、`extract_json` ロジックを堅牢化（`ToolResult` との一貫性を持たせる）。

## 5. 世界線メモリ統合フロー（案）
1. 世界線ノードの `highlight` / `memory_links` に対応する Markdown / JSON ログを生成。
2. `PageIndexKosongAdapter` を通じて:
   - `md_to_tree` → セクション構造＋要約を抽出。
   - `generate_doc_description` でドキュメント要約を得る。
3. 結果を PageIndex 用ストレージ（例: `.kimi/worldlines/pageindex/*.json`）に保存。
4. 世界線ビューで該当ノードを選択すると、PageIndex エントリを読み込み「1行サマリ＋ハイライト」を表示。

## 6. 実装 TODO
- [ ] `PageIndexKosongAdapter` インタフェースを設計し、既存 PageIndex 関数の呼び出し差し替えポイントを洗い出す。
- [ ] `kosong` LLM への minimal chat ラッパ（同期/非同期）を作成。
- [ ] トークンカウント戦略を決定（tiktoken 導入可否含む）。
- [ ] PageIndex 出力を世界線データモデルに統合するマッピング仕様を記述。
- [ ] 例外処理・タイムアウト・キャッシュ戦略（同一ページ再処理の回避）を定義。

---

上記を踏まえ、PageIndex 機能を世界線メモリと UI に統合する詳細設計へ進める。*** End Patch
