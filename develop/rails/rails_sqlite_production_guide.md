# Rails 8 SQLite Production Configuration Guide

Rails 8以降、SQLiteは本番環境でも「First-class citizen（第一級市民）」として扱われています。本ドキュメントでは、公式に推奨されている構成、およびスケールアウトを実現するためのツールについて解説します。

---

## 1. 基本構成と最適化

Rails 8のデフォルト設定では、SQLiteのパフォーマンスを最大化するために以下の設定が適用されます。

### WAL (Write-Ahead Logging) モード
`journal_mode: WAL` を有効にすることで、読み取り操作が書き込み操作をブロックせず、同時実行性が大幅に向上します。

### Solid Stack によるインフラ統合
外部サーバー（Redis等）を不要にするため、SQLiteをバックエンドとした以下のGemが標準採用されています。
- **Solid Queue**: データベースベースのジョブキュー。
- **Solid Cache**: データベースベースのキャッシュストア。
- **Solid Cable**: Action Cableのバックエンド。

### データベースの分離 (`database.yml`)
負荷を分散するため、用途ごとにDBファイルを分ける構成が推奨されます。
```yaml
production:
  primary:
    adapter: sqlite3
    database: storage/production.sqlite3
    migrations_paths: db/migrate
  cache:
    adapter: sqlite3
    database: storage/production_cache.sqlite3
    migrations_paths: db/cache_migrate
  queue:
    adapter: sqlite3
    database: storage/production_queue.sqlite3
    migrations_paths: db/queue_migrate
```

---

## 2. Litestream (バックアップと耐久性)

**Litestream** は、SQLiteのデータを安全に外部保存するためのストリーミング・レプリケーションツールです。

- **役割**: データの継続的バックアップ。
- **仕組み**: SQLiteのWALファイルを監視し、変更があるたびにS3やCloudflare R2、Google Cloud Storageなどのオブジェクトストレージへリアルタイムで送信します。
- **メリット**: 
  - サーバーが完全に消失しても、数秒前までのデータを復元可能。
  - データベース自体のパフォーマンスへの影響が極めて低い。
- **リンク**: [Litestream 公式サイト](https://litestream.io/) / [GitHub](https://github.com/benbjohnson/litestream)

---

## 3. LiteFS (水平スケーリング)

**LiteFS** は、SQLiteデータベースを複数のサーバー間で共有・複製するための分散ファイルシステムです。

- **役割**: 読み取り負荷の分散（Read Replicas）と高可用性の実現。
- **仕組み**: 
  - FUSE（Filesystem in Userspace）を使用して、SQLiteの書き込みをキャプチャし、他のノードへレプリケーションします。
  - **Primaryノード**が書き込みを受け持ち、複数の**Replicaノード**が読み取りを受け持ちます。
- **HTTPプロキシ機能**: 
  - 書き込みリクエストがReplicaに届いた場合、自動的にPrimaryノードへ転送（Forwarding）する機能を持っています。
- **メリット**: 
  - 世界中のエッジサーバーにDBを配置し、超低遅延な読み取りを実現。
  - シングルサーバーの限界を超えたスケールアウトが可能。
- **リンク**: [LiteFS 公式ドキュメント](https://fly.io/docs/litefs/) / [GitHub](https://github.com/superfly/litefs)

---

## 4. 運用戦略の選び方

| フェーズ | 推奨構成 | ツール |
| :--- | :--- | :--- |
| **スタートアップ** | シングルサーバー (VPS/PaaS) | Rails 8 (Solid Stack) + Litestream |
| **成長期** | リージョン内での可用性向上 | Rails 8 + LiteFS (1 Primary + 1-2 Replicas) |
| **グローバル展開** | 世界各地へのデプロイ | Rails 8 + LiteFS (Global Read Replicas) |
| **超大規模** | 書き込み集中型 (秒間数百〜) | PostgreSQL / MySQL への移行 |

---

## 5. 参考資料
- [Rails 8.0: No PaaS Required](https://rubyonrails.org/2024/11/7/rails-8-0-no-paas-required) (Official Blog)
- [The SQLite WORM Stack](https://world.hey.com/dhh/the-sqlite-worm-stack-88484ba6) (DHH Blog)
