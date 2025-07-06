# Claude Code 活用ベストプラクティス・ガイド (Table Reference)

Claude Codeを自律的なパートナーとして運用するための戦略マップです。

## 1. 開発の地図を置く (Setup & Context Management)
| 機能 / アクション | 活用シーン・詳細 | ベストプラクティス | Geminiでの代替・相当機能 |
| :--- | :--- | :--- | :--- |
| **`/init` (CLAUDE.md)** | プロジェクト構造、ビルド・テスト方法、規約の明文化 | プロジェクト開始時に必ず実行し、地図を置く | **`GEMINI.md`** (プロジェクト固有の指示書) |
| **CLAUDE.md 更新** | Claudeが同じ勘違いをした際のルール化 | 「同じミスを2回したら追記」し、不要な長文は削る | **`GEMINI.md` の継続的更新** |
| **Memory 分離** | 個人設定(User)とチーム規約(Project)の使い分け | `/memory` で現在読まれている記憶を常に監視する | **`save_memory`** (global / project scope) |
| **`/permissions`** | `npm test` や `git status` 等の安全なコマンド | 頻出コマンドを事前承認し、確認の往復を排除する | **Policies** (コマンド実行前の承認フロー) |
| **Settings 管理** | `.claude/settings.json` による共有設定 | チーム共通設定と `settings.local.json` を分ける | **Configファイル** (`~/.gemini/config.yaml`等) |
| **`/statusline`** | ブランチ、モデル、コンテキスト使用率、コストの表示 | 常に表示をONにし、現在地とコストを可視化する | **ステータスバー** (トークン・コスト表示) |

## 2. 精度を上げる依頼と検証 (Planning & Quality)
| 戦略 / 手法 | 具体的な運用 | 成功のポイント | Geminiでの代替・相当機能 |
| :--- | :--- | :--- | :--- |
| **Plan Mode 使い分け** | 大きい変更はPlanから開始、1行の修正には使わない | 影響範囲が不明な時のみPlan Modeで探索させる | **`enter_plan_mode`** |
| **成功条件の指定** | 「このテストが通ること」「期待出力との差分なし」 | 実装前に「何を以て完了とするか」を先に渡す | 標準的なプロンプト手法 |
| **自己検証ループ** | テスト、Lint、TypecheckをClaude自身に回させる | スクリーンショットやブラウザ確認までセットにする | **Core Mandate** (Research-Strategy-Execution-Validate) |
| **ログの扱い** | エラーメッセージは要約せず「生」のまま渡す | ファイルに保存して読ませ、原因特定を高速化する | `read_file`, `run_shell_command` の活用 |
| **既存パターン継承** | 「既存のWidget実装を読み、同じパターンで作れ」 | コードベース内の慣習を明示的に参照させる | **Engineering Standards** (Conventions & Style) |
| **曖昧さの解消** | 仕様が不明確な場合、実装前にClaudeに質問させる | 大きな機能ほど実装前にインタビューを挟む | **`ask_user`** / 曖昧さの解消義務 |
| **`/rewind`** | 破壊的変更や試行錯誤の巻き戻し | チェックポイントを活用し、戻せる前提で攻める | **`/rewind`** (ファイル変更のRevertも可能) |

## 3. 文脈を外部化する (Skills & Subagents)
| ツール | 用途・役割 | 運用のコツ | Geminiでの代替・相当機能 |
| :--- | :--- | :--- | :--- |
| **Skills** | `.claude/skills/` に再利用可能な手順を定義 | 毎回読まないドメイン知識や手順はSkillsへ分離 | **Agent Skills** (`SKILL.md`, `activate_skill`) |
| **Slash Commands** | `/fix-issue 123` のような引数付きの定型依頼 | 頻繁に行う独自の定型作業をMarkdownで定義 | **Custom Commands** (`.gemini/commands/*.toml`) |
| **Subagents** | 調査、レビュー、デバッグ等の特定タスクの委任 | 役割を絞り、使えるツールを必要最小限にする | **Sub-agents** (`codebase_investigator` 等) |
| **分業戦略** | 調査はSubagent、意思決定はメインセッション | メインの文脈を汚さず、重い調査は裏で回す | **Strategic Orchestration & Delegation** |
| **学びの還元** | PRコメントでの指摘をCLAUDE.mdに反映 | レビューの結果を未来のClaudeの行動指針に変える | `GEMINI.md` へのフィードバック反映 |

## 4. 自動化と仕組み化 (Hooks, MCP & Non-interactive)
| 仕組み | 機能と効果 | 活用例 | Geminiでの代替・相当機能 |
| :--- | :--- | :--- | :--- |
| **Hooks (Lifecycle)** | `PreToolUse`, `PostToolUse` 等での自動処理 | 編集後に formatter を自動実行しスタイルを整える | **Hooks** (`BeforeTool`, `AfterTool` 等) |
| **禁止事項の強制** | プロンプトではなくHookによる書き込み阻止 | 「migrationsは触らせない」をHookで物理的に止める | **Hooks (BeforeTool)** による検証・ブロック |
| **MCP (Protocol)** | GitHub, Jira, DB等の外部ツールとの統合 | Issueからの自動実装や、監視データに基づく分析 | **MCP (Model Context Protocol)** 対応 |
| **MCP トークン管理** | 出力サイズの制限とページング設計 | 必要な範囲だけを取得し、コンテキスト圧迫を防ぐ | 標準的なMCP機能 |
| **`claude -p`** | 非対話モードでのバッチ処理・CI連携 | ログ解析や、複数ファイルへの機械的一括変換 | **Headless mode** (`-p` / `--prompt`) |
| **Fan-out 実行** | 大量ファイルを1セッションに抱えず並列処理 | 1ファイルずつ `claude -p` を回して成功率を上げる | **Generalist agent** による並列・バッチ処理 |
| **GitHub Actions** | PR/Issue上のメンションからのエージェント起動 | ターミナル外（ブラウザ上）でのレビューや修正 | GitHub連携 (エコシステムによるサポート) |

## 5. 並列化によるワークフロー (Multi-session Strategies)
| パターン | 具体的な進め方 | 期待される効果 | Geminiでの代替・相当機能 |
| :--- | :--- | :--- | :--- |
| **git worktree** | 物理的にディレクトリを分け、複数タスクを並列化 | 認証とUIを別々に進め、コンテキスト混濁を防ぐ | Git標準機能の活用 |
| **Writer / Reviewer** | 実装セッションとレビューセッションを分離 | 自己バイアスを排除し、コード品質を客観的に担保 | サブエージェント間の役割分担 |
| **TDD セッション** | テスト作成用と実装用のセッションを分ける | 仕様の穴を見つけやすくし、堅牢な実装を導く | 開発プロセスの運用 |
| **UI/UX 確認** | スクリーンショットとブラウザ操作のセット運用 | 見た目の崩れや操作性をClaude自身に評価させる | **Vision機能** (Gemini 2.0等) の活用 |
| **セッション使い分け** | Cloud / Desktop / Local の環境選択 | タスクの機密性、負荷、コストに応じた最適化 | 環境設定・モデル選択の柔軟性 |
| **自動承認運用** | 許可リスト、Sandbox、Hookによる安全担保 | 「止めるべき操作」を仕組み化し、安心して自動化 | **Policies** / **Hooks** による安全制御 |
