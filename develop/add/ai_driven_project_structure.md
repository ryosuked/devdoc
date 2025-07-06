# AI駆動型マルチランゲージ・プロジェクト構成案

本ドキュメントでは、AIエージェント（Cline/Cursor等）が迷いなくコードを生成・修正できるよう最適化されたディレクトリ構造を定義する。

## 1. 全体構造 (Mono-repo 構成)

```text
/
├── .github/              # CI/CD (GitHub Actions)
├── docs/                 # 共通設計ドキュメント
├── schema/               # 真実の口 (Single Source of Truth)
│   ├── openapi/          # REST API定義 (Main App)
│   └── grpc/             # マイクロサービス間通信定義
├── services/             # 各サービスの実装
│   ├── api-gateway/      # Go: エントリポイント
│   ├── core-app/         # Ruby on Rails: メインロジック
│   └── payment-svc/      # Go: 特定マイクロサービス
├── shared/               # 言語を跨ぐ共有資産
│   └── scripts/          # コード生成スクリプト等
├── Makefile              # 共通オペレーションの抽象化
└── docker-compose.yml    # ローカル開発環境
```

---

## 2. 各サービスの内部構造詳細

### 2.1 API Gateway (Go)
AIが「ボイラープレート」と「カスタムロジック」を区別しやすい構造にする。
```text
/services/api-gateway/
├── cmd/
│   └── server/          # エントリポイント (main.go)
├── internal/
│   ├── auth/            # 認証・認可ロジック
│   ├── middleware/      # 共通ミドルウェア
│   └── proxy/           # Ruby側へのプロキシ処理
├── gen/                 # OpenAPI等からの自動生成コード（AIに触らせない）
└── go.mod
```

### 2.2 Core Application (Ruby on Rails)
AIに「どこがビジネスロジックか」を明示するため、標準的なRails構造に加え、Serviceクラス等を活用する。
```text
/services/core-app/
├── app/
│   ├── controllers/     # APIエンドポイント
│   ├── models/          # データベース・スキーマ
│   ├── services/        # ★ビジネスロジック（AIが主に編集する場所）
│   └── sorbet/          # 型定義 (T::Struct 等)
├── config/
├── spec/                # RSpecテスト（AIに必ず書かせる）
└── Gemfile
```

### 2.3 Microservice (Go)
1つのリポジトリ/ディレクトリ内に全ての責務を押し込めず、クリーンアーキテクチャを意識した構造。
```text
/services/payment-svc/
├── api/                 # gRPC定義に基づくハンドラ
├── domain/              # ドメインモデル
├── usecase/             # ユースケース（AIに実装させるコア）
├── infrastructure/      # DB/外部APIアクセス実装
└── main.go
```

---

## 3. AIを円滑に動かすための「共通ツール」

### 3.1 `schema/` ディレクトリの重要性
AIに「スキーマを読み取ってから実装せよ」と指示するために、プロジェクトのルートに配置する。
*   `schema/openapi/v1.yaml`
*   `schema/grpc/payment.proto`

### 3.2 `Makefile` による抽象化
AIに複雑なコマンドを覚えさせるのではなく、短いエイリアスを提供する。
*   `make gen`: スキーマから各言語のコードを自動生成。
*   `make test-all`: 全サービスのテストを実行。
*   `make dev`: 全サービスをローカルで立ち上げ。

## 4. AIエージェントへの指示（プロンプト）への活用
本構造を採用することで、以下のような簡潔な指示が可能になる。
> 「`schema/openapi/v1.yaml` に新しいエンドポイントを追加し、その変更を `services/core-app/` に反映し、対応するRSpecを書いてください。」
