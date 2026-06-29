# no-dep-loop — コーディングエージェントのループ定義

再利用可能な **agent 定義**と **skill**として、コーディングエージェントのループ（triage / explorer / implementer / phase-reviewer / verifier を controller が協調させる）を提供するリポジトリです。永続メモリと作業状態は**ローカルファイル（`.docs/memory`）**で完結し、外部サービス（MCP 等）には依存しません。

- 可変な状態（work item・進捗・実行ログ・検証結果）→ `.docs/memory` に置き、標準のファイルツールで読み書きする。
- 静的なルール（ループの動かし方）→ [`CLAUDE.md`](CLAUDE.md) の「ループポリシー」に置き、起動時に全エージェントへ自動読み込みされる。
- 起動・心拍はリポジトリに含めない（トリガー非依存）。controller の手起動や外部スケジューラなど、運用側が任意の手段で起動する。

---

## ループの全体像

**controller**（ループを統括するメインのエージェントセッション）だけがサブエージェントを spawn し、`.docs/memory` への書き込みを適用します。各役は単一の責務を持ちます。

```
controller（統括・spawn・記録の適用）
  ├─ triage          作業の発見・分類・claim 可否の判定        （読み取り専用）
  ├─ explorer        コードパス・依存・既存パターンの調査       （読み取り専用）
  ├─ implementer     claim 済み work item のスコープ内実装       （編集可）
  ├─ phase-reviewer  各工程直後の独立クロスレビュー            （読み取り専用）
  └─ verifier        Done 前の最終検証ゲート                   （読み取り専用）
```

- **書き込み境界**: ループ内でコードを編集するのは `implementer` のみ。`.docs/memory` の更新は controller のみ。
- **オーケストレーション境界**: サブエージェントは `Agent` ツールを持たず、自分で spawn しない。
- **完了の証跡**: 主観的な完了宣言ではなく、`verifier` の APPROVE と検証ログ（テスト / lint / build 等）で Done を判断する。

詳細は [`CLAUDE.md` のループポリシー](CLAUDE.md) を参照してください。

---

## ディレクトリ構成

```
.
├── CLAUDE.md            ループポリシー（全エージェントが project policy として読む）
├── .claude/
│   ├── skills/          再利用可能な手順・規約（SKILL.md 単位）
│   └── agents/          エージェント定義（役割・ツール・権限）
└── .docs/memory/        永続メモリ／作業状態のバックエンド
    ├── index.md         work item 進捗ボード
    ├── issues/          1 ファイル = 1 work item
    └── feedback/        横断フィードバック
        ├── index.md         全エントリの索引
        └── <scope>/         scope 別フォルダ（review / implementation / research / general）
                             1 関心事 = 1 ファイル（F<NN>-<slug>.md）
```

---

## Skills

### ループ中核

| skill | 役割 |
|-------|------|
| `loop-engineering` | ループ全体の設計・実行。作業発見、役割分離、検証、メモリ委譲、次アクション判断を統括する。 |
| `local-memory` | `.docs/memory` を永続メモリ／状態管理バックエンドとして使う。work item 形式・状態機械・書き込み権限を定義。 |
| `tdd-cycle` | t-wada 流 TDD（計画→テスト→Red→Green→リファクタリング→検証記録）をループに適用する。 |
| `cross-review-gates` | 各工程後に、担当とは別エージェントで独立レビューし APPROVE / REJECT / ESCALATE を判定する。 |
| `iterative-retrieval` | 広いクエリから絞り込み、関連ファイルを 3〜5 件に収束させる 4 フェーズ検索パターン。 |

### ループ外の補助（controller が手動起動）

| skill | 役割 |
|-------|------|
| `empirical-prompt-tuning` | 指示（skill / プロンプト）をバイアスのない実行者で評価し反復改善する。新規作成・大幅改訂時の品質検証。 |
| `memory-recall` | `.docs/memory` から意図・実装・レビュー観点・判断/フィードバックを問いに沿って出典引用付きで引き出す。 |
| `skill-authoring` | 繰り返されるユーザー依頼を手順書 = 新しい skill として切り出す軽量 authoring 手順。 |

---

## Agents

### ループ役

| agent | 役割 | 権限 |
|-------|------|------|
| `triage` | work item の発見・分類・claim 可否の判定 | 読み取り専用 |
| `explorer` | コードパス・依存・既存パターン・リスクの調査 | 読み取り専用 |
| `implementer` | claim 済み work item のスコープ内実装（TDD） | 編集可 |
| `phase-reviewer` | 各工程直後の独立クロスレビューゲート | 読み取り専用 |
| `verifier` | Done 前の最終検証ゲート | 読み取り専用 |

### ループ外の補助（controller が必要に応じて手動起動）

| agent | 役割 | 権限 |
|-------|------|------|
| `architect` | 構造設計・技術選定・ADR | 読み取り専用 |
| `planner` | 設計確定後の実装ステップ分解 | 読み取り専用 |
| `tech-docs-searcher` | 最新ドキュメント調査（Web 取得） | 読み取り専用 |
| `memory-searcher` | `.docs/memory` を検索し意図・実装・レビュー観点を出典付きで要約 | 読み取り専用 |
| `refactor-cleaner` | デッドコード整理（JS/TS 向け） | 編集可 |

読み取り専用エージェントは `permissionMode: plan` で動き、`.docs/memory` を読めますが書き込みはしません（更新案を controller に返す）。

---

## メモリモデル（`.docs/memory`）

作業は **work item**（`.docs/memory/issues/<id>-<slug>.md`）として記録します。各ファイルは frontmatter（`type: issues` / `id` / `title` / `status` / `labels` / …）と本文セクションを持ちます。

- `## 作業概要` — 何の作業か（2〜3 行）
- `## 目的` / `## 背景` — なぜやるか
- `## 受け入れ条件` / `## レビュー観点` — 達成条件と確認基準（チェックボックスが細粒度の進捗）
- `## 人間の判断・フィードバック` — 対話で決めたこと（`[判断]` / `[フィードバック]`、append-only）
- `## 検証` / `## エスカレーション` / `## 実行ログ` — 検証手段・行き詰まり・時系列の記録（実行ログは append-only）

**状態機械**: `Backlog → Todo → In Progress → In Review → Done`（＋ `Canceled` / `Duplicate`）。一覧は [`index.md`](.docs/memory/index.md) が進捗ボードになります。

横断的に効くフィードバックは `.docs/memory/feedback/` に昇格させ、継続的改善につなげます。scope（review / implementation / research / general）をフォルダで表し、1 関心事を 1 ファイル（`feedback/<scope>/F<NN>-<slug>.md`）として置き、全エントリを `feedback/index.md` で索引します。各エントリは frontmatter で `種別`（`Guardrail` = 必ず守る制約 / `Knowledge` = 必要に応じ参照する知見）と `status`（active / applied / retired）を管理します。各エージェントは「自分の scope ＋ general」だけを読みます。

詳細（frontmatter・状態機械・書き込み権限・横断フィードバックの運用）は `local-memory` skill（[`.claude/skills/local-memory/SKILL.md`](.claude/skills/local-memory/SKILL.md)）を参照してください。

---

## 使い方

1. **起動**: controller（ループを統括するメインのエージェントセッション）を起動する。`CLAUDE.md` のループポリシーが project policy として読み込まれる。起動レシピは `loop-engineering` の automation prompt 雛形を使う。
2. **対象指定**: 対象とする work item の範囲を依頼または automation prompt で指定する（`.docs/memory/issues/` の `ready-for-agent` ラベル付きなど）。
3. **ループ**: controller が work item を claim し、explorer / implementer / phase-reviewer を協調させ、各工程の証跡を `.docs/memory` に記録する。
4. **完了**: `verifier` が証跡付きで APPROVE し、各工程の `cross-review-gates` が通った場合のみ Done にする。

> バージョン管理（VCS 操作）はループの責務に含めません。運用側がループ外で扱います。

---

## 関連ドキュメント

- [`CLAUDE.md`](CLAUDE.md) — ループポリシー（claim / 書き込み境界 / 完了の証跡 / 補助エージェント・skill の位置づけ）
- [`.docs/memory/index.md`](.docs/memory/index.md) — work item 進捗ボード（一覧）／[`.docs/memory/feedback/index.md`](.docs/memory/feedback/index.md) — 横断フィードバック索引
- `.claude/skills/<name>/SKILL.md` — 各 skill の手順定義
- `.claude/agents/<name>.md` — 各 agent の責務・ツール・権限
