# コンテナ内に限定した Claude Code 開発環境の提案（herdr 組み込み）

作成: 2026-10-30 / 状態: **提案（机上の検討。まだ何も構築・検証していない）**

> ツールの選択肢は、まだ一つに絞らない。§3 にすべて候補として並べる。§4 以降の構成と例は、その中の一つの組み合わせ（共通の devbox + mise + herdr）で書いた**たたき台**で、採用を決めたものではない。

## 0. 要約

- macOS 上で直接 Claude Code を使うのをやめ、**Claude が動くのは Linux のコンテナの中だけ**にする。このドキュメントでは、作業用のコンテナを「devbox」と呼ぶ（共通の 1 つにするか、プロジェクトごとの devcontainer にするかは §3.2）。
- たたき台では、devbox の中で **herdr サーバーを主プロセスとして常に動かす**。エージェントのペインはコンテナが動いている限り生き続け、Mac 側からは herdr のクライアントで接続する。tmux や Zellij なども候補として残す（§3.5）。
- Mac 側の herdr は 1 つのウィンドウに 2 つの場所を並べる。
  - **devbox**: エージェントが作業する場所
  - **local（Mac）**: 人が手で操作する場所（Xcode、実機・シミュレーター、git push、compose の操作）
- 対象の開発がどこで動くかは、次のように分かれる。
  - **サーバーサイド・CLI（Ruby / Go / Rust / Python）と TS フロントエンド**: ほぼすべてコンテナの中で完結する。
  - **Swift / Flutter のネイティブ**: 「編集・解析・単体テスト・Web」はコンテナの中、「実機向けのビルド・シミュレーター・署名」は Mac の上、に分ける。
- 骨組みから段階的に作る（§9）。最初はドキュメント作業と 1 つの言語だけを移して試す。

---

## 1. 目的と前提

### 目的（優先度は §10 で確定させる）

| 目的 | 内容 |
|---|---|
| 隔離 | Claude が触れる範囲を、マウントしたワークスペースとコンテナの中に限る。Mac の Keychain、`~/.ssh`、他のディレクトリに届かせない |
| 任せやすさ | 隔離が十分なら、コンテナの中では権限の確認を減らして任せられる |
| 再現性 | 環境をコード（Dockerfile / compose）で持ち、作り直せるようにする |
| 並列化 | herdr で複数のエージェントを並べて動かし、状態を一覧で見張る |

### 前提

- ホスト: Apple Silicon の Mac、macOS 26。
- 対象:
  - サーバーサイド・CLI: Ruby、Go、Rust、Python
  - フロントエンド: TypeScript
  - ネイティブ: Swift（iOS / macOS）、Flutter
- 今まではホストの Claude をターミナルで直接使ってきた。シンプルさは大事にしたい。
- Xcode が Linux のコンテナで動かないことは承知のうえ。

---

## 2. 全体像

```
┌──────────────────────── macOS（ホスト） ────────────────────────┐
│                                                                  │
│  ターミナル ── herdr クライアント（Mac）                          │
│               ├─ machine: devbox  ← SSH（ホスト → コンテナの一方向）│
│               │    └ エージェントのペイン（中身は devbox にある）  │
│               └─ local                                           │
│                    ├ Xcode / xcodebuild / シミュレーター          │
│                    ├ flutter run（iOS）、Android エミュレーター    │
│                    ├ git push                                    │
│                    └ docker compose（devbox と DB の起動・停止）   │
│                                                                  │
│  ~/Documents/workspace ──────────┐ bind mount（同じパス）         │
│                                  ▼                               │
│  ┌──────────── Linux VM（実行基盤は §3.1） ─────────────────┐      │
│  │  ┌──────── devbox コンテナ ────────┐   ┌─ postgres ─┐  │      │
│  │  │ herdr server（主プロセス）       │   ├─ redis ────┤  │      │
│  │  │  ├ claude（impl / reviewer …）   │──▶│  …         │  │      │
│  │  │  ├ 開発サーバー、テスト、ログ    │   └────────────┘  │      │
│  │  │ mise: ruby / go / rust / python / node / flutter    │      │
│  │  │ sshd（Mac の herdr からの接続だけを受ける）          │      │
│  │  │ /Users/ryo/Documents/workspace（マウント）           │      │
│  │  │ /home/dev（ボリューム: ~/.claude, herdr, mise）      │      │
│  │  └─────────────────────────────────┘                    │      │
│  └────────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

方針:

- **接続は Mac からコンテナへの一方向**。コンテナには、Mac に入るための鍵も、Mac の herdr のソケットも渡さない。
- コンテナに `docker.sock` は渡さない（渡すとホストの root とほぼ同じ権限になる）。compose の操作は Mac 側で人が行う。
- `git push` は Mac 側の local ペインで行う。コンテナの中は commit まで（§6）。

---

## 3. ツールの選択肢（すべて候補として残す）

### 3.1 実行基盤

| 基盤 | 特徴 | 気になる点 |
|---|---|---|
| OrbStack | 軽く、ファイル共有（virtiofs）が速い。Rosetta による x86_64 のエミュレーション、`host.docker.internal`、SSH agent の転送がある | 商用利用は有料 |
| Colima（Lima ベース） | 無料・OSS。`--vm-type vz --vz-rosetta` で Rosetta も使える。docker CLI がそのまま使える | ファイル共有の速さと手軽さは OrbStack に劣る |
| Docker Desktop | 定番で、情報が多い。Dev Containers との相性がよい | 重い。大きな組織での利用は有料 |
| Podman（podman machine） | daemon がなく、rootless。OSS | compose や devcontainer との互換で、細かい差が出ることがある |
| Apple `container` | macOS 26 標準。コンテナごとに軽い VM で、隔離が強い | まだ若い。compose や devcontainer との相性はこれから |
| Lima（VM を直接使う） | コンテナを使わず、Linux の VM 1 台を作業場にする | イメージによる再現性は自分で作る必要がある |

Rosetta が使えるかどうかは、§5.3 の Android ツールチェーンで効いてくる可能性がある（要検証）。

### 3.2 コンテナの持ち方

| 案 | 内容 | 利点 | 欠点 |
|---|---|---|---|
| A. 共通の devbox | 作業用コンテナを 1 つ常に動かし、全部入れる | 今の使い方に一番近く、シンプル | プロジェクトごとの差は、バージョン管理のツールで吸収する必要がある |
| B. プロジェクトごとの devcontainer | `.devcontainer/` をリポジトリに置く | 再現性が高く、チームで共有できる | イメージとキャッシュがプロジェクトの数だけ増える（ダウンロード量も） |
| C. A と B の併用 | 普段は A。特殊なリポジトリだけ B | 両方の良いところを取れる | 2 つの方式を管理することになる |

### 3.3 言語のバージョン管理

| 候補 | 特徴 |
|---|---|
| mise | 1 つのツールで全言語を扱える。`.tool-versions` にも対応。たたき台はこれで書いた |
| asdf | mise の元になった方式。プラグインが豊富 |
| 言語ごとの公式・定番ツール | Ruby は rbenv と ruby-build、Go は公式の配布物、Rust は rustup、Python は uv か pyenv、Node は fnm / nvm / Volta、Flutter は fvm |
| Nix（nix-shell / devenv / Devbox / flox） | 完全に再現できる環境を宣言的に書ける。学ぶことが多い |
| イメージに焼き込む | Dockerfile で特定の版を入れる。単純だが、版を変えるたびにビルドし直す |

### 3.4 パッケージマネージャ

| 言語 | 候補 |
|---|---|
| Python | uv / pip と venv / Poetry |
| Node / TS | pnpm / npm / Yarn / Bun |
| Ruby | Bundler |
| Go / Rust | 標準（go mod / cargo） |
| Flutter / Dart | 標準（pub） |

### 3.5 ターミナル上の作業場（エージェントの持続と並列化）

| 候補 | 特徴 |
|---|---|
| herdr | エージェントの状態の表示、CLI と Socket API での連携、`machine` で複数のマシンを 1 つのウィンドウに。pre-1.0。たたき台はこれで書いた |
| tmux | 枯れていて、どこにでもある。エージェントの状態は見えない |
| Zellij | レイアウトや resurrect が組み込み。エージェントの状態は見えない |
| Claude Squad | tmux と git worktree の上で、エージェントを並べて動かす |
| 使わない | `docker exec` で入るたびに `claude` を起動する。端末を閉じると作業も終わる |
| GUI のエージェント管理（cmux、Conductor、Superset など） | Mac のアプリとして動く。コンテナの中のエージェントを扱えるかは、ツールごとに要確認 |

### 3.6 コンテナへの入り方・エディタ

| 候補 | 特徴 |
|---|---|
| `docker exec` / `docker compose exec` | 一番簡単。Mac に何も足さなくてよい |
| SSH（コンテナの中に sshd を置く） | herdr の `--remote` / `machine` や、普通の ssh で入れる |
| VS Code の Dev Containers 拡張 | IDE と Claude Code の連携がコンテナの中で動く |
| JetBrains Gateway / Dev Containers | JetBrains の IDE をリモート接続で使う |
| Mac のエディタでマウントしたファイルを直接開く | 編集は Mac、実行はコンテナ。LSP は Mac 側に言語環境が要る |

### 3.7 Claude Code の入れ方

| 候補 | 特徴 |
|---|---|
| ネイティブ版のインストーラ | Node に依存しない |
| npm でグローバルに入れる | Node の環境と一緒に管理できる |
| Anthropic の devcontainer の参考実装をそのまま使う | ファイアウォールまで含めた土台が手に入る |

### 3.8 隔離に関わる選択肢

| 項目 | 候補 |
|---|---|
| ファイルの置き場所 | Mac 側に置いて bind mount する（同じパスにするか、`/workspace` のような別のパスにするか）／コンテナや VM の中だけに置く |
| `~/.claude` | ホストと共有する（ホストの Claude を残すなら、hooks を書き換えられる危険がある）／コンテナ専用のボリュームに分ける |
| git push | Mac 側で push する／リポジトリを限った fine-grained token／SSH agent を転送する／`~/.ssh` をマウントする |
| テストからの docker の利用 | 使わない／Docker in Docker のサイドカー（privileged）／Sysbox などのランタイム／そのテストだけ Mac 側か CI で回す |
| ネットワーク制限 | 最初から許可リスト／観察してから締める／制限しない |

### 3.9 ネイティブのビルドの手段

§5.4 の方法 1〜4 と、macOS の VM の候補（Tart / UTM）。Flutter の Android は、コンテナでやるか Mac でやるか（§5.3）。

---

## 4. devbox の構成（たたき台）

この章は、§3 の候補から「A. 共通の devbox + mise + herdr + 同じパスでの bind mount + `~/.claude` の分離」を選んだ場合の構成例。他の候補を選んだら、該当する箇所を置き換える。

### 4.1 イメージ（ベースは小さく、ツールチェーンはボリュームに入れる）

- **イメージに入れるもの（共通の土台）**
  - Debian か Ubuntu（arm64）、ビルドに要る基本のパッケージ（build-essential、libssl、libyaml、libffi など。Ruby や Python をビルドするのに要る）
  - git、gh、ripgrep、fd、jq、zsh、openssh-server
  - Claude Code（ネイティブ版のインストーラ）
  - herdr（Linux 用の単一バイナリ）
  - mise（言語のバージョン管理）
  - pandoc（ドキュメント用）
  - 土台は Anthropic の Claude Code 用 devcontainer の参考実装（Dockerfile とファイアウォールのスクリプト）から作る。
- **ボリュームに入れるもの（イメージに焼かない）**
  - mise が入れる Ruby、Go、Rust、Python、Node、Flutter。プロジェクトの `mise.toml` / `.tool-versions` に書いた版を、使うときに入れる。
  - 各パッケージのキャッシュ（gem、Go の module と build、cargo の registry と target、uv、pnpm の store、pub）
  - Playwright のブラウザ、Android SDK（必要になってから）

こうするとイメージが小さく保てて作り直しが速く、プロジェクトごとの版の違いにも mise で対応できる。

### 4.2 ファイルの置き場所

| 対象 | 置き場所 | 理由 |
|---|---|---|
| ソースとドキュメント | Mac の `~/Documents/workspace` を、コンテナでも**同じパス**にマウント | Mac のアプリ（エディタ、Obsidian、Finder）からも見られる。Claude のプロジェクトごとのメモリと履歴（パスで区別される）、ログやメモに書いたパスがそのまま通じる |
| `~/.claude`（設定、メモリ、認証情報） | ボリューム `devbox-home` | ホストと共有しない（§6）。最初に一度だけ `CLAUDE.md` とメモリをコピーする |
| herdr の状態（`~/.config/herdr`） | ボリューム `devbox-home` | コンテナを作り直してもレイアウトを復元できる |
| mise のツールチェーン、キャッシュ | ボリューム `devbox-cache` | §4.1 |
| Rust の `target/` | `CARGO_TARGET_DIR` でボリュームへ | マウント越しだと巨大で遅い。全プロジェクトで共有するので、同じ名前のバイナリは上書きされる。困るなら、プロジェクトごとに分ける |
| Ruby の gem | `BUNDLE_PATH` でボリュームへ | ネイティブ拡張は Linux 用になり、Mac 側では使えないので、ワークスペースに置かない |
| `node_modules` | まずはマウントのまま。遅ければプロジェクトごとにボリュームを重ねる | 共通の作業用コンテナでは、プロジェクトごとのボリューム指定が煩雑になる。速さを見てから決める（§9 の検証項目） |
| Python の `.venv` | プロジェクトの中（マウント）のまま | ホストではもう Python を動かさないので、Linux 用でも困らない |

### 4.3 起動の形

- `entrypoint.sh` で sshd を起動したあと、`exec herdr server`（サーバーを前面で動かす）。tini（`init: true`）を PID 1 にする。
- つまり **コンテナの起動 = herdr サーバーの起動**。コンテナを再起動すると、herdr はレイアウトと作業ディレクトリを復元し、integration を入れた Claude は会話を再開する。開発サーバーなどのプロセスは止まるので、起動し直す。

### 4.4 compose の骨組み（たたき台。未検証）

```yaml
# ~/Documents/workspace/devbox/compose.yaml
services:
  devbox:
    build: .
    image: devbox:latest
    hostname: devbox
    init: true
    entrypoint: ["/usr/local/bin/entrypoint.sh"]   # sshd → exec herdr server
    user: dev
    volumes:
      - /Users/ryo/Documents/workspace:/Users/ryo/Documents/workspace
      - devbox-home:/home/dev
      - devbox-cache:/home/dev/.cache
    environment:
      CARGO_TARGET_DIR: /home/dev/.cache/cargo-target
      BUNDLE_PATH: /home/dev/.cache/bundle
      GOMODCACHE: /home/dev/.cache/go-mod
      GOCACHE: /home/dev/.cache/go-build
    ports:
      - "127.0.0.1:2222:22"            # Mac の herdr → devbox（SSH）
      - "127.0.0.1:3000-3010:3000-3010" # 開発サーバー（Mac のブラウザから開く）
      - "127.0.0.1:8080:8080"          # flutter web など
    # cap_add: [NET_ADMIN]             # 段階 3 でファイアウォールを有効にするとき
  postgres:
    image: postgres:17
    environment: { POSTGRES_PASSWORD: dev }
    volumes: [pg-data:/var/lib/postgresql/data]
  redis:
    image: redis:7

volumes:
  devbox-home:
  devbox-cache:
  pg-data:
```

- ポートは `127.0.0.1` にだけ公開する。開発サーバーはコンテナの中で `0.0.0.0` に bind する。
- DB などはサービス名（`postgres`、`redis`）で devbox から参照する。

### 4.5 想定するファイル構成

```
~/Documents/workspace/devbox/
├── Dockerfile
├── compose.yaml
├── entrypoint.sh            # sshd 起動 → exec herdr server
├── firewall/
│   ├── init-firewall.sh     # 参考実装をもとにする（段階 3）
│   └── allowlist.txt        # 許可するドメイン（言語ごとに分ける）
├── seed/
│   └── CLAUDE.md            # devbox 用のグローバル指示（最初に ~/.claude へコピーする）
├── host/
│   ├── ssh_config.snippet   # Mac 側の ~/.ssh/config に足す devbox の設定
│   └── host-runner          # 段階 4: Mac 側のビルド用の入口（任意）
└── README.md
```

---

## 5. 開発対象ごとの扱い

### 5.1 サーバーサイド・CLI（Ruby / Go / Rust / Python）

| 言語 | コンテナでの扱い | 注意点 |
|---|---|---|
| Ruby | mise で入れる（ソースからのビルドで数分かかる）。gem は `BUNDLE_PATH` へ | ビルドに要るライブラリを、イメージに入れておく |
| Go | mise で入れる。`GOMODCACHE` / `GOCACHE` はボリュームへ | macOS 向けのバイナリは `GOOS=darwin`（CGO なし）でクロスコンパイルできる |
| Rust | mise か rustup で入れる。`CARGO_TARGET_DIR` はボリュームへ | **macOS 向けのクロスコンパイルは難しい**（SDK が要る）。Mac で動かす CLI の最終確認は、Mac 側でビルドする（§5.4 の入口を使う）か、CI に任せる |
| Python | uv（mise で入れる）。`.venv` はプロジェクトの中 | 重い依存（科学計算や機械学習系）のダウンロードは事前に確認する |
| 共通 | DB やキャッシュは compose の別サービスにする | Testcontainers のように、テストの中から docker を使うものは困る。privileged な Docker in Docker のサイドカーを立てる（隔離が弱まる）か、そのテストだけ Mac 側か CI で回す |

### 5.2 TS フロントエンド

- Node と pnpm は mise で入れ、pnpm の store はキャッシュのボリュームに置く。
- 開発サーバーは `0.0.0.0` に bind し、Mac のブラウザで `http://localhost:3000` を開いて確認する。
- **ファイル変更の検知**: マウント越しでの inotify の伝わり方は基盤によって違う。HMR が反応しないときは、ポーリング（`CHOKIDAR_USEPOLLING` や Vite の `server.watch.usePolling`）に切り替える。
- **Playwright**: コンテナの中で Linux 版の Chromium / Firefox / WebKit を動かす。ブラウザの取得（数百 MB）は事前に確認する。Safari 本体での確認だけは Mac 側でする。
- `node_modules` の速さは段階 2 で測る（§4.2）。

### 5.3 Flutter

| 作業 | 場所 | 補足 |
|---|---|---|
| 編集、`dart analyze`、`flutter test`（単体・widget） | devbox | Claude が自分で回せる |
| Web（`flutter run -d web-server --web-hostname 0.0.0.0 --web-port 8080`） | devbox | Mac のブラウザで確認する。hot reload は herdr のペインで |
| Android のビルド・実行 | **まずは Mac**。コンテナの中でやるのは検証してから | arm64 の Linux では、Android SDK の一部のツールが x86_64 版しか配布されてこなかった経緯がある。Rosetta などで動くかは**要検証**。Android SDK と NDK は数 GB と大きい |
| Android の実機・エミュレーター | Mac | エミュレーターはコンテナの中では動かない（入れ子の仮想化ができない）。Mac で動かす |
| iOS / macOS のビルド・実行 | Mac | Xcode が要る |

**ホストとコンテナで同じリポジトリを触るときの注意**: `.dart_tool/package_config.json` には Flutter SDK の絶対パスが書き込まれる。そのため、Mac とコンテナを行き来すると `flutter pub get` のやり直しが要る。対策として、コンテナの Flutter SDK を Mac と同じ絶対パスに置く方法がある（例: 両方で `/Users/ryo/.local/share/flutter`）。`build/` も共有されるので、使う側で clean する。

コンテナから Mac の Android エミュレーターを使う構成（段階 4 の候補・要検証）:
Mac 側で adb サーバーを外から受けられる形で起動し、コンテナ側は `ADB_SERVER_SOCKET=tcp:host.docker.internal:5037` で接続する。こうすると `flutter run` をコンテナから実行できる。ただし adb サーバーが待ち受けるポートの公開範囲には注意する。

### 5.4 Swift（iOS / macOS）

この構成で一番弱いところ。コンテナの中の Claude は Swift のファイルを編集できるが、UIKit や SwiftUI の型検査やビルドができないので、手応え（フィードバック）が Mac 側でのビルドに頼ることになる。

| 作業 | 場所 |
|---|---|
| 編集、Apple のフレームワークに依存しない Swift Package の `swift build` / `swift test` | devbox（Linux 版の Swift） |
| server-side Swift（Vapor など） | devbox |
| Xcode のプロジェクトのビルド、テスト、シミュレーター、署名 | Mac |

Mac 側でのビルドを Claude の手応えにつなぐ方法（段階 4 で選ぶ）:

1. **人が local ペインでビルドし、ログをワークスペースに出す**（最初はこれ）
   `xcodebuild … 2>&1 | tee .build-logs/ios.log`。コンテナの Claude がそのログを読んで直す。隔離は崩れない。
2. **Mac 側に、決まったコマンドだけを通す入口（host-runner）を作る**
   コンテナに専用の鍵を置き、Mac の `authorized_keys` に `command="host-runner",no-pty,no-port-forwarding,no-agent-forwarding` で登録する。host-runner は `SSH_ORIGINAL_COMMAND` を見て、許可した操作だけを実行する（例: `ios-test <リポジトリのパス> <scheme>`、`flutter-build-ios <パス>`。パスはワークスペースの中に限る）。
   **注意**: ビルドはプロジェクトのコードを実行する（Run Script の phase、SPM のプラグイン、CocoaPods のフックなど）。そのコードは Claude が編集できるので、入口を絞っても **実質的に Mac 上でコードを実行できてしまう**。そのため host-runner は、普段使っていない専用の macOS ユーザー（例: `builder`。Keychain、`~/.ssh`、自分のホームに触れない）として動かし、ワークスペースはグループの権限で共有する。シミュレーター向けのビルドに署名は要らないので、署名の鍵は builder に置かない。
3. **macOS の VM（Tart / UTM）の中に Xcode と Claude を入れる**
   隔離も本物の Xcode も手に入るが、数十 GB と重い。Swift の比重が大きくなったら検討する。
4. **例外として、Xcode の作業だけ Mac 上の Claude を使う**（権限を確認する運用で）
   一番手軽だが、「Claude はコンテナの中だけ」という方針の例外になる。

---

## 6. 認証情報と隔離

| 項目 | 方針 |
|---|---|
| Claude のログイン | devbox の中で `claude` を起動し、表示される URL を Mac のブラウザで開く（OAuth）。認証情報は `devbox-home` ボリュームの中のファイルに保存される（Linux なので Keychain は使わない） |
| `~/.claude` | たたき台では**ホストと共有しない**。共有すると、ホストの `settings.json` の hooks をコンテナから書き換えられ、それがホストの Claude で実行されてしまうため（共有する案も §3.8 に残す） |
| git の commit | コンテナの中。`user.name` / `user.email` だけを設定する |
| git push | たたき台では **Mac 側の local ペイン**で行い、コンテナには push の権限を置かない。他の案（fine-grained token、SSH agent の転送、`~/.ssh` のマウント）は §3.8 |
| gh CLI | 読み取りだけの token、または使わない |
| SSH | Mac 側の `~/.ssh/config` で、devbox の接続には `ForwardAgent no` を明示する（全体の設定で agent の転送が有効だと、コンテナに agent が渡ってしまう） |
| docker.sock | 渡さない |
| ネットワーク | 段階 3 で許可リストのファイアウォールを有効にする（§8） |
| 権限の確認を省く運用 | ファイアウォールを有効にし、push の権限がないことを確かめてからにする |

Mac 側の `~/.ssh/config` に足す設定（例）:

```
Host devbox
  HostName 127.0.0.1
  Port 2222
  User dev
  IdentityFile ~/.ssh/devbox_ed25519   # devbox に入るための専用の鍵（ホスト → コンテナ）
  ForwardAgent no
  StrictHostKeyChecking accept-new
```

---

## 7. herdr の組み込み方

### 7.1 役割

| 位置 | 役割 |
|---|---|
| devbox の中の herdr サーバー | エージェント、開発サーバー、テストのペインを持ち続ける。コンテナが動いている限り、Mac のターミナルを閉じても作業は続く |
| Mac の herdr クライアント | `machine` 機能で devbox を登録する。devbox のエージェントと、Mac の local ペインを 1 つのウィンドウで扱う。サイドバーで、devbox の中のエージェントの blocked や done にも気づける |
| Mac の local ペイン | 人だけが使う（Xcode、`flutter run` の iOS 版、git push、compose）。**ここでは Claude を起動しない** |

### 7.2 接続の仕方（段階に応じて）

```bash
# 段階 1: いちばん簡単。Mac に herdr は要らない
docker compose exec devbox herdr          # devbox の herdr サーバーに attach

# 段階 2: Mac 側にも herdr を入れ、devbox をマシンとして登録する
herdr --remote devbox                      # SSH で devbox に attach（キー設定は手元のもの）
herdr machine add devbox --label "devbox"  # local と devbox を 1 つのウィンドウに並べる
herdr --machine devbox agent list          # Mac から devbox のエージェントを CLI で操作する
```

### 7.3 devbox の中の初期設定

```bash
herdr integration install claude     # 状態の判定を正確にし、再起動後の会話の再開を有効にする
npx skills add herdrdev/herdr --skill herdr -g   # Claude に herdr の操作を教えるスキル（任意）
```

- 設定は `~/.config/herdr/config.toml`（ボリュームの中）。
- Mac 側でも tmux を使うなら、prefix がどちらも `ctrl+b` なので、どちらかを変える。
- `pane_history`（再起動後に画面の履歴を再生する機能）は無効のままにする。出力に秘密情報が残りうるため。

### 7.4 Workspace の切り方

- **リポジトリ × タスクごとに 1 つの Workspace**。`herdr worktree create --cwd <repo> --branch <topic>` で worktree を作り、そのまま Workspace として開く。
- タブの例: `agents`（impl / reviewer）、`server`（開発サーバーとコンテナのログ）、`test`。

### 7.5 作業の例: Go API、TS フロントエンド、Flutter アプリにまたがる機能追加

```bash
# devbox の中（または Mac から herdr --machine devbox …）
herdr worktree create --cwd /Users/ryo/Documents/workspace/api --branch feat-search
herdr worktree create --cwd /Users/ryo/Documents/workspace/web --branch feat-search
herdr worktree create --cwd /Users/ryo/Documents/workspace/app --branch feat-search

# 各 Workspace で claude を起動し、名前を付ける（impl-api / impl-web / impl-app、reviewer）
herdr agent prompt impl-api "検索 API を追加。OpenAPI も更新" --wait
herdr agent prompt impl-web "更新された OpenAPI を見て検索画面を作る" --wait
herdr agent prompt impl-app "同じ API を使う検索画面を Flutter で。flutter test まで" --wait
herdr agent prompt reviewer "3 つのリポジトリの feat-search をレビュー" --wait
herdr agent wait impl-app --until blocked    # 承認待ちになったら人が見る
```

人がやること（Mac の local ペイン）:

- iOS のシミュレーターでの `flutter run` と、Xcode でのビルドの確認。ログは `.build-logs/` に出す（§5.4 の方法 1）。
- 各リポジトリの `git push`。

### 7.6 herdr を使うときの注意

- pre-1.0 なので、コマンドや設定が変わりうる。困ったら tmux に置き換えられるようにしておく（devbox の主プロセスを tmux にするだけで済む）。
- 状態の判定に誤りがあると、blocked が idle に見えることがある。そのまま自動で入力すると承認ダイアログにキーを打ち込んでしまうので、自動化するなら integration を入れ、`herdr agent explain` で確かめる。
- devbox の中の herdr のソケットは、コンテナの外に出さない。Mac から操作するのは SSH 経由（`--machine`）だけにする。

---

## 8. ネットワーク制限（段階 3）

Anthropic の参考実装のファイアウォール（iptables と ipset。`NET_ADMIN` が要る）をもとに、許可リストを言語ごとに分けて管理する。

| 分類 | 許可するドメインの例 |
|---|---|
| Claude | `api.anthropic.com`、`claude.ai`（ログイン）、参考実装に載っているもの |
| git | `github.com`、`api.github.com`、`codeload.github.com`、`objects.githubusercontent.com` |
| Ruby | `rubygems.org`、`index.rubygems.org` |
| Go | `proxy.golang.org`、`sum.golang.org`、`storage.googleapis.com` |
| Rust | `crates.io`、`index.crates.io`、`static.crates.io`、`static.rust-lang.org` |
| Python | `pypi.org`、`files.pythonhosted.org` |
| Node / TS | `registry.npmjs.org` |
| Flutter / Dart | `pub.dev`、`storage.googleapis.com`、（Android を使うなら）`dl.google.com`、`maven.google.com` |
| ツールチェーンの取得 | mise がダウンロードに使う先（ツールごとに違うので、段階 2 で実際の通信を見て決める） |

- 最初から締めると、原因の切り分けが難しくなる。段階 2 までは開けておき、通信先を観察してから締める。
- ドキュメントの参照（Web の取得）を Claude に許すかどうかも、ここで決める。

---

## 9. 段階的な導入計画（骨組みから）

### 段階 1: 骨組み

- [ ] §3 の候補から、最初に試す組み合わせを選ぶ
- [ ] 実行基盤を入れる（§3.1 から選ぶ）
- [ ] `devbox/` を作る（Dockerfile、compose.yaml、entrypoint.sh）。中身は土台だけ（Claude、herdr、mise、git、pandoc）
- [ ] ワークスペースを同じパスでマウントし、`devbox-home` / `devbox-cache` を作る
- [ ] devbox の中で Claude にログインし、`seed/CLAUDE.md` とメモリをコピーする
- [ ] `docker compose exec devbox herdr` で attach し、`herdr integration install claude`
- [ ] **ドキュメント作業**と**言語 1 つ（例: Go）**だけを移して、数日使う

確かめること（試す前に相談する）:

- Claude のログインの流れ
- マウント越しのファイル操作の速さ
- コンテナを再起動したあと、herdr のレイアウトと Claude の会話が戻るか

### 段階 2: サーバーサイドとフロントエンドに広げる

- [ ] Ruby、Rust、Python、Node を mise で入れ、各キャッシュをボリュームに逃がす
- [ ] compose に postgres と redis を足し、ポートを公開する
- [ ] TS の開発サーバーと HMR、Playwright を確かめる。`node_modules` の速さを測る
- [ ] Mac に herdr を入れ、`machine add devbox` で 1 つのウィンドウにまとめる
- [ ] git push を Mac の local ペインに寄せる運用を始める

### 段階 3: 締める

- [ ] 許可リストのファイアウォールを有効にし、通信先を育てる
- [ ] コンテナの中で権限の確認を省く範囲を決める
- [ ] ホストの Claude をどうするか決める（消す、または Xcode 用の例外として残す）

### 段階 4: ネイティブ

- [ ] Flutter を devbox に入れ、analyze、test、web を回す。SDK のパスを Mac と揃えるか決める
- [ ] Android のビルドをコンテナでやるかを検証する（arm64 と Rosetta）。だめなら Mac で
- [ ] Swift: まずは方法 1（ログをワークスペースに出す）。必要になったら host-runner と builder ユーザー、または macOS の VM を検討する

---

## 10. 決めること（未決）

0. **§3 の各候補からどれを選ぶか**: 現時点ではすべて候補のまま残している。
1. **目的の優先度**: 隔離、任せやすさ、再現性、並列化のどれが一番か。段階 3 の締め方の強さが決まる。
2. **実行基盤**: §3.1 から選ぶ。個人利用か（OrbStack と Docker Desktop の有料条件）。今入っているものはあるか。
3. **ホストの Claude**: 最終的に消すか、Xcode 用の例外として残すか。
4. **Swift の手応えの取り方**: 方法 1〜4 のどれまで進めるか。
5. **Flutter の Android**: コンテナで頑張るか、最初から Mac に任せるか。
6. **テストからの docker の利用**: Testcontainers などを使っているか（使っているなら §5.1 の対策が要る）。
7. **プロジェクトごとの devcontainer**: 共通の devbox で足りないリポジトリはあるか。

---

## 11. ダウンロード量の目安（実施前に都度確認する）

| もの | 目安 | 段階 |
|---|---|---|
| 実行基盤（OrbStack / Colima と VM のイメージ） | 数百 MB | 1 |
| devbox のベースイメージと土台のパッケージ | 1〜1.5 GB | 1 |
| Ruby / Go / Rust / Python / Node のツールチェーン | 合計 2〜3 GB | 1〜2 |
| postgres / redis のイメージ | 数百 MB | 2 |
| Playwright のブラウザ（3 種類） | 0.5〜1 GB | 2 |
| Flutter SDK | 1〜2 GB | 4 |
| Android SDK / NDK（コンテナに入れる場合） | 5〜10 GB | 4 |
| macOS の VM と Xcode（方法 3 を選ぶ場合） | 数十 GB | 4 |

数字はおおまかな見積もりなので、実際の版と構成によって変わる。

---

## 12. 参考

- herdr のまとめ: [herdr.md](./herdr.md) ／ 操作チュートリアル: [herdr-tutorial.html](./herdr-tutorial.html)
- herdr docs: https://herdr.dev/tui/docs/ （connecting machines、session state、CLI reference）
- Claude Code の devcontainer の参考実装: Anthropic の Claude Code ドキュメントの「Development containers」
