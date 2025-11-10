# Mind Evolution / PageIndex 調査メモ（Phase 0）

作成日: 2025-11-10  
作成者: GPT-5 Codex

---

## 1. Mind Evolution 実装概要
- 参照ファイル: `_project/reference/mind_evolution.py`（lucidrains/mind-evolution）
- 現状は**ほぼ骨組みのみ**。遺伝的アルゴリズムの主要関数（`init_islands`, `migration`, `refinement`, `mutation`, `fitness`, `recombination` など）がすべて `NotImplementedError`。
- `MindEvolution` クラスも `__call__` が恒等返しで、実質的な探索処理は未着手。
- `RefinementCriticalConversation` など概念的コンポーネントは定義されているが、実装無し。
- トランスフォーマモデルをインジェクトする構造だが、型注釈の `Module` や `critic_transformers` が未定義のまま。
- → 実際に利用するには、オリジナル論文（Sec. 3.2）に基づき**全ロジックを埋める必要がある**。  
  - 特に「会話による改善（RCC）」や「島モデル」「移民」「トーナメント選択」などを Kimi CLI 用に設計し直す前提。

### Reading Steiner への示唆
- Mind Evolution は「探索フレームワーク」のみ提供している状態。それを世界線個体の進化探索に転用するには以下が課題:
  1. 世界線状態（チェックポイント・メモリ参照状況）を遺伝子表現へ落とし込む。
  2. 適応度関数（Temporal Consistency, Divergence など）を定義し、LLM/Review Agent と連携。
  3. RCC を Kimi のマルチエージェント会話へマッピング。
- 事実上、新規実装に近い作業規模と想定する。

## 2. PageIndex 実装概要
- 参照ファイル: `_project/reference/page_index.py`, `_project/reference/page_index_md.py`
- 主用途: PDF/Markdown などの文書から目次・ページ対応を抽出し、構造化インデックス（PageIndex）を生成・活用する。
- 必要なユーティリティ (`ChatGPT_API`, `extract_json`, `count_tokens` など) は `utils.py` 依存。OpenAI API を前提としたプロンプト駆動型。
- 主な機能:
  - TOC 検出 (`toc_detector_single_page`, `find_toc_pages` など)
  - 目次 JSON 変換 (`toc_transformer`, `toc_index_extractor`)
  - ページ番号マッチングとオフセット計算 (`extract_matching_page_pairs`, `calculate_page_offset`)
  - Markdown 構造解析 (`md_to_tree`) とトークン数ベースの要約生成
- 非同期処理（`asyncio`）と `ThreadPoolExecutor` を併用し、大量の LLM 呼び出しをさばく想定。
- → Kimi CLI へ取り込む場合は、自前の LLM 呼び出しレイヤ（`kimi_cli.llm`）へ置き換え＆依存部分を整理する必要がある。

### Reading Steiner への示唆
- PageIndex は「文書構造を索引化し、セクション単位の参照を容易にする」ための仕組み。世界線メモリに対して:
  - 各世界線の履歴ログや要約を Markdown/JSON として蓄積 → PageIndex でメタデータ化 → 参照候補を高速抽出。
  - `md_to_tree` によるノード要約機能を、世界線ごとのトレース「ハイライト生成」に転用できる。
  - 物理ページ番号の概念は世界線チェックポイント ID へマップし直す必要あり。
  - LLM 呼び出し部分は Kimi CLI の `kosong` API (`kimi_cli.llm`) を利用するラッパ層を用意し、外部 API 依存を排除する。

## 3. ブログ記事より（pageindex.ai）
- `pageindex_intro.html`: PageIndex のコンセプト紹介。目次抽出→ハイライト→セクション単位検索の流れを説明。
- `pageindex_ocr.html`: OCR 結果のノイズを処理しつつ PageIndex 構造へ変換する課題と対策。
- Reading Steiner で応用する場合:
  - OCR 文書の扱いよりも「世界線ログを如何に構造化するか」にフォーカス。
  - 抽出結果を「1行要約＋ハイライト」で提示する UI 仕様に合致。
  - ノイズ抑制・正規化処理の知見（ドット列 → コロン置換など）は、世界線ログのフォーマット変換にも応用可能。

## 4. 差分・注意点
1. Mind Evolution はアルゴリズム実装が無いため、**再構築が前提**。もしくは別実装を探索する必要あり。  
2. PageIndex は ChatGPT API 前提。`kimi_cli` の LLM インターフェースに合わせて抽象化・置換が必要。  
3. 非同期呼び出しの大量使用に伴うリソース管理（`asyncio`, `ThreadPoolExecutor`）を Kimi CLI 内でどう扱うか設計が必要。  
4. PageIndex の Markdown 処理パイプラインは Kimi CLI の世界線ログフォーマットに適合させるカスタマイズが必要。  
5. ライセンスはいずれも MIT。README 等でのクレジット追記タイミングを Phase 4 に設定。

## 5. 次のアクション
- DeepWiki (Time travel, Subagents, Meta Commands, etc.) の該当セクションを読み、既存仕様との整合をまとめる。
- Mind Evolution 論文/実装を補完する資料を調査し、要素分解（島モデル、適応度設計、RCC）を具体化。
- PageIndex の `utils.py` を確認し、必要な補助関数（LLM 呼び出しや JSON 正規化）をリストアップ。
- 世界線メモリへの適用イメージ（データ構造、PageIndex インポートフロー）を図式化し、設計フェーズで詳細化する。

---

以上。Phase 0 調査の基礎資料として保管。
