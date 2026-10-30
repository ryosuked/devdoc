# herdr まとめ

調査日: 2026-10-30 / 対象: herdr（pre-1.0。バージョンや API は変わりやすいので、使う前に公式ドキュメントを確認すること）

## 1. herdr とは

- AI コーディングエージェント（Claude Code、Codex など）を複数同時に動かすための**ターミナルマルチプレクサ**。公式の言い方では「the runtime your coding agents live on」。
- **いま使っているターミナルの中で動く TUI**。ターミナルアプリを置き換えない。
- tmux と同じく、バックグラウンドのサーバーがターミナルのプロセスを持ち、クライアントが attach / detach する。
- tmux との大きな違いは 2 つ。各ペインのエージェントを検出して状態（working / blocked / idle / done）をサイドバーに出すこと。エージェント自身が herdr を操作できる CLI と Socket API を持つこと。

## 2. 概念モデル

| 単位 | 説明 |
|---|---|
| Session | バックグラウンドサーバーの名前空間。`herdr` はデフォルトのセッションに attach する。名前付きセッション同士は完全に別物 |
| Workspace | プロジェクト単位の入れ物。普通はリポジトリかタスクごとに 1 つ |
| Tab | Workspace の中のレイアウト（例: `agents`、`logs`、`server`） |
| Pane | 実体のあるターミナル。分割でき、detach しても生き続ける |
| Agent | ペインの中で検出されたエージェントのプロセス |

モードは 3 つ: terminal（キー入力をペインへ送る）、prefix（`ctrl+b` のあとにキーを 1 つ）、navigate。

### エージェントの状態

- **idle**: プロンプトで入力を待っている
- **working**: 処理中
- **blocked**: 承認・質問・権限の確認で止まっている（人の対応が必要）
- **done**: 終わったがまだ見ていない（見るまで表示が残る）
- サイドバーは一番急ぐ状態を上の階層へ伝える。どこかのエージェントが blocked なら、そのペイン・タブ・Workspace も blocked と表示される。

検出の仕組み: ペインのプロセスと画面の最下部を見て、検出マニフェストのルールで状態を判定する。マニフェストはリモートから自動更新され、`~/.config/herdr/agent-detection/<agent>.toml` で上書きできる。`herdr integration install <agent>` を入れると、エージェントが状態を直接報告するようになり、判定が正確になる。

## 3. 使い方

### インストール

```bash
curl -fsSL https://herdr.dev/install.sh | sh   # Linux / macOS
brew install herdr                              # Homebrew
mise use -g herdr                               # mise
# Windows はベータ（PowerShell 用のインストールスクリプトあり）
herdr --version
herdr update            # 直接インストールしたときの更新（brew / mise / nix では使えない）
```

### 最初の手順

```bash
cd ~/project && herdr                 # 起動 / attach
claude                                # ペインでエージェントを起動すると自動で検出される
herdr integration install claude      # 任意。状態の判定が正確になり、再起動後の会話 resume にも要る
```

- マウスで操作できる: クリックでフォーカス、境界のドラッグでリサイズ、右クリックでメニュー、ドラッグでコピー。
- `prefix+v` で右に分割、`prefix+-` で下に分割、`prefix+c` で新しいタブ、`prefix+?` でキー一覧。
- `prefix+q` で detach、もう一度 `herdr` で再 attach、`herdr server stop` で全部止める。
- `HERDR_ENV=1` ならすでに herdr のペインの中にいる（入れ子での起動はできない）。
- 設定ファイル: `~/.config/herdr/config.toml`（`[keys]`、`[theme]`、`[ui]`、`[terminal]`、`[update]` など）。`herdr --default-config` でデフォルト値を表示、`herdr server reload-config` で動いているサーバーに反映。

### CLI（スクリプトやエージェントから操作する）

ほとんどのコマンドは JSON を返す。ペイン ID は `w1:p1` の形。

```bash
# ペイン
herdr pane split w1:p1 --direction right --cwd ~/project
herdr pane run w1:p2 "npm test"                       # 入力して Enter
herdr pane read w1:p2 --source recent --lines 50
herdr pane wait-output w1:p2 --match "ready" --timeout 60000

# エージェント
herdr agent list
herdr agent start reviewer --kind claude --pane w1:p3
herdr agent prompt reviewer "このブランチの差分をレビューして" --wait
herdr agent wait reviewer --until idle --timeout 120000
herdr agent attach reviewer           # UI 全体を出さず、そのエージェント 1 つにだけつなぐ
herdr agent explain reviewer --json   # その状態と判定された理由

# Workspace / worktree / セッション
herdr workspace create --cwd ~/project --label api --no-focus
herdr worktree create --cwd ~/project --branch review   # Git worktree を Workspace として開く
herdr session list / attach <name> / stop <name>
```

### Socket API

- 1 行に 1 リクエストの JSON（NDJSON）。Unix ドメインソケットを使う（Windows は名前付きパイプ）。
- パス: `~/.config/herdr/herdr.sock`。名前付きセッションは `sessions/<name>/herdr.sock`。
- 形式: `{"id":"req_1","method":"pane.read","params":{"pane_id":"w1:p2"}}` → `result` か `error` が返る。
- `events.subscribe` で、Workspace・タブ・ペインの変化やエージェントの状態変化を受け取れる。履歴の再送はない。購読側が遅れると `events_lost` が来るので、`session.snapshot` を取り直す。
- `herdr api schema --json` でスキーマ全体を表示。たいていの用途は CLI のラッパーで足りる。

### リモート

```bash
herdr --remote workbox                    # SSH 越し。手元は薄いクライアントで、手元のキー設定が使える
herdr machine add workbox --label "Build machine"
herdr --machine "Build machine" agent list   # 保存したマシンへ API を転送
```

手元と保存した SSH マシンを 1 つのウィンドウに並べ、エージェント一覧もまとめて見られる。

### エージェントにスキルを入れる

`npx skills add herdrdev/herdr --skill herdr -g` で、エージェントに「herdr の操作方法」を教えるスキルを入れられる。エージェントが自分でペインを増やし、別のエージェントにプロンプトを送って、終わるのを待つ、といったことができるようになる。

## 4. 何が残るか（セッションの永続性）

| 場面 | プロセス | レイアウト | 直近の画面 | エージェントの会話 |
|---|---|---|---|---|
| detach → 再 attach | 残る | 残る | 残る | 残る |
| サーバーの再起動 | 消える | 残る | `pane_history` を有効にしたときだけ | native resume があるときだけ |
| `herdr update --handoff` | 引き継ぎに成功すれば残る | 残る | 成功すれば残る | 成功すれば残る |

- 再起動したときは、ペインが「保存されていたディレクトリで開く新しいシェル」として戻る。開発サーバーやテストは止まる。
- 画面履歴の再生（`[experimental] pane_history = true`）はデフォルトで無効。出力に秘密情報が含まれうるため。
- エージェント会話の resume はデフォルトで有効。integration が要る（例: Claude Code は integration のバージョン 6 以上）。
- レイアウトのスナップショットは最大 48 個（多くて 15 分に 1 回）を `session-snapshots/` に保存する。

## 5. 利点

1. **どのエージェントが待っているかが一目でわかる**。ペインを順に覗かなくても、サイドバーで blocked や done に気づける。これが一番の価値。
2. **エージェントからエージェントを操作できる**。`agent prompt --wait` や `agent wait --until` があるので、「実装役 → レビュー役」のような連携を、ポーリングせずにシェルスクリプトで組める。
3. **設定なしでマウスで使える**。tmux のように設定ファイルを育てる必要がない。prefix は `ctrl+b` なので、tmux に慣れた人もすぐ使える。
4. **永続性**。スリープ、回線の切断、SSH の切断があってもエージェントは動き続ける。VPS で動かしっぱなしにして、手元から `--remote` でつなぐ使い方に合う。
5. **いまのターミナルのまま使える**。Ghostty、iTerm2、WezTerm などの中で動くので、使い慣れたターミナルを替えなくてよい。
6. **Git worktree との統合**。`herdr worktree create` で、ブランチごとに隔離した Workspace を作れる。
7. **プラグイン**（ファイルビューア、コードレビュー用のサイドバー、Telegram 通知など）。まだ若いが増えている。

## 6. 欠点・注意点

- **pre-1.0 で歴史が浅い**（2026 年春ごろに公開）。API や設定が変わりうる。開発者が実質 1 人という指摘もある（bus factor）。
- **サンドボックスはない**。エージェントは自分のユーザー権限のまま動く。隔離が要るなら、コンテナや VM と組み合わせる（その場合は `HERDR_AGENT` で検出を補う）。
- **状態の判定は画面を見て推測している**。integration なしだと誤判定がありうる。ドキュメント自身が「idle を信じて自動入力すると、承認ダイアログにキー入力してしまう恐れがある」と警告している。自動化するときは integration を入れ、`agent explain` で確かめる。
- **サーバーを再起動するとプロセスは消える**。tmux と同じだが、Zellij の resurrect のようにコマンドを再実行する仕組みはない（tmux も resurrect 系のプラグインを使えばできる）。
- tmux ほどの細かいカスタマイズ（ステータスラインのウィジェット、プロジェクトのテンプレートなど）はない。
- ペインが多いと描画が遅れるという報告がある。
- Windows はベータ。
- 差分のレビュー UI はない（外部ツールかプラグインで補う）。

## 7. 向き・不向き

### 向いている

- エージェントを **3 つ以上同時に**動かし、どれが止まっているかを見張るのがつらくなっている人
- 実装役・レビュー役・テスト役のような**マルチエージェントの連携をスクリプトで組みたい**人
- **リモートの開発機や VPS で、エージェントを動かしっぱなしにしたい**人
- tmux の設定を覚えたくないが、永続するセッションは欲しい人
- macOS や Linux で、ターミナルを中心に作業する人

### 向いていない

- tmux の設定を作り込んでいて、それ自体が資産になっている人（乗り換える価値が小さい）
- エージェントを使わない、または 1 つしか動かさない用途（tmux や Zellij で足りる）
- 本番サーバーの運用など、安定性と枯れていることを最優先する場面
- サンドボックスや権限の隔離が要る場面（単体では満たせない）
- GUI で差分レビューまで完結させたい人（Conductor などのほうが合う）
- Windows だけの環境

## 8. tmux との比較

| 観点 | herdr | tmux |
|---|---|---|
| 主な目的 | AI エージェントを並べて動かし、操作する | 汎用のターミナルマルチプレクサ |
| エージェントの状態の把握 | あり（サイドバー、上の階層への伝搬、通知） | なし（自前のスクリプトやフックで作るしかない） |
| マウス操作 | 標準 | 設定すれば一部できる |
| 設定の手間 | 小さい（設定なしで使える） | 大きい（そのぶん柔軟） |
| 永続性（detach） | あり | あり |
| サーバー再起動後の復元 | レイアウトと cwd、エージェント会話の resume | 標準ではなし（resurrect / continuum で復元） |
| 稼働中の更新 | `--handoff`（実験的） | なし |
| 自動化の API | JSON を返す CLI と Socket API、イベント購読、`agent wait` | `send-keys` / `capture-pane` などのテキスト操作 |
| リモート | `--remote`（手元が薄いクライアント）、複数マシンを 1 画面に | SSH の先で tmux を起動する |
| 階層 | Session > Workspace > Tab > Pane | Session > Window > Pane |
| エコシステム | 若い | 非常に成熟（TPM、膨大な設定例） |
| 安定性・歴史 | pre-1.0、2026 年〜 | 2007 年〜、どこにでもある |

**使い分けの目安**: エージェントを並べて回すための作業場として herdr を使い、サーバーの運用や汎用の作業は tmux のままにする。両方を並行して使うのが現実的。キーの prefix がどちらも `ctrl+b` なので、tmux の中で herdr を使うなら、どちらかの prefix を変える。

## 9. 他のツール

| ツール | 種類 | 特徴 | herdr との違い |
|---|---|---|---|
| **tmux** | 汎用のマルチプレクサ（TUI） | 枯れていて、どこにでもあり、カスタマイズの自由度が高い | エージェントを認識しない |
| **Zellij** | 汎用のマルチプレクサ（TUI） | レイアウト、フローティングペイン、組み込みの resurrect、WASM プラグインのサンドボックス | エージェントの状態を見られない。一般用途での完成度は herdr より上 |
| **Claude Squad** | tmux の上で動くエージェント管理 | エージェントごとに git worktree と tmux セッションを作る。シンプル | Socket API やプラグインはない。軽く始めるならこちら |
| **cmux** | macOS ネイティブのターミナル兼ブラウザ（GUI） | どの CLI エージェントでも監視できる | ターミナルそのものを置き換える。macOS 専用 |
| **Conductor** | macOS アプリ（GUI） | 隔離した Workspace、タスクの設定、差分レビューまで含めたフロー | タスクとレビューのフローまで用意されている。ターミナルの中では動かない |
| **Superset** | エージェントの並列実行環境（GUI） | herdr の代替を名乗っている | GUI。差分レビューを持つ |
| **Warp** | AI を組み込んだターミナル（GUI） | ターミナル自体に AI 機能がある | ターミナルを置き換える |
| **Tern** | ワークスペース型ターミナル | コミュニティのプラグインで herdr をペインに載せられる | herdr と組み合わせられる |
| **Superlogical Rex** | マルチプレクサを兼ねるターミナル | macOS のみ | ターミナルを置き換える |

大まかな位置づけ:

- **ターミナルの中で動き、エージェントの状態がわかる** → herdr
- **ターミナルの中で動き、汎用** → tmux、Zellij
- **tmux の上に worktree の管理を足しただけの軽いもの** → Claude Squad
- **GUI でタスクから差分レビューまで扱う** → Conductor、Superset、cmux

※ 各社の比較ページ（cmux、Superset）は自社に有利に書かれている。中立のベンチマークは見つからなかった。

## 10. 導入するときのおすすめ手順

1. 手元で `herdr` を起動し、Claude Code と Codex を 2 ペインで動かして、サイドバーの状態表示を試す。
2. 使うエージェントの `herdr integration install <agent>` を入れる（状態の判定と resume のため）。
3. `herdr worktree create` で、タスクごとに worktree の Workspace を分ける。
4. 必要になったら `herdr agent prompt --wait` や `agent wait` で連携のスクリプトを組む。自動入力するときは blocked の誤判定に注意する。
5. リモートで常時動かすなら `herdr machine add` で開発機を登録する。

## 11. 補足（実装・配布・ライセンス）

### herdr

- Rust 製の単一バイナリ（約 10 MB）。外部依存なし、Electron は使っていない。
- ライセンスは Apache-2.0。テレメトリなし、アカウント不要。
- リポジトリ: `github.com/ogulcancelik/herdr`（`herdrdev/herdr` の表記もあり、移管された可能性がある）／公式: https://herdr.dev

### 他のツールのライセンスなど

| ツール | 実装・ライセンス |
|---|---|
| tmux | C、ISC |
| Zellij | Rust |
| Claude Squad | AGPL |
| cmux | GPL |
| Conductor | 無料、プロプライエタリ |
| Tern | Rust |

### 参考

- 公式: https://herdr.dev ／ ドキュメント: https://herdr.dev/tui/docs/
- エージェント向けガイド: https://herdr.dev/agent-guide.md
- CLI リファレンス: https://herdr.dev/tui/docs/cli-reference/
- Socket API: https://herdr.dev/tui/docs/socket-api/
- セッションの状態: https://herdr.dev/tui/docs/session-state/
- エージェント: https://herdr.dev/tui/docs/agents/
- README: https://raw.githubusercontent.com/ogulcancelik/herdr/master/README.md
- レビュー（Bitdoze）: https://www.bitdoze.com/herdr-agent-multiplexer/
- Better Stack のガイド: https://betterstack.com/community/guides/ai/herdr-ai-agent/
- tecmint: https://tecmint.com/herdr-run-ai-coding-agents-in-linux-terminal
- 比較（各社）: https://cmux.com/compare/cmux-vs-conductor ／ https://superset.sh/compare/herdr-alternative
