# Rails コンテナビルドガイド

このドキュメントでは、Rails アプリケーションのコンテナ化について解説します。Docker を用いたビルド戦略や、使用する Docker イメージの選定について説明します。

## 1. Docker イメージの選択

Rails の Docker イメージとして、以下の 3 種類の Ruby イメージを使用できます。

| イメージ種類 | ベース | 特徴 |
| -------------------------------------- | ------------ | --------------------------- |
| **通常版** (`ruby:<version>`)             | Debian       | 必要なパッケージが揃っているが、イメージサイズが大きい |
| **slim 版** (`ruby:<version>-slim`)     | Debian       | サイズが小さく、本番環境向け              |
| **alpine 版** (`ruby:<version>-alpine`) | Alpine Linux | 超軽量だが、追加パッケージが必要な場合がある      |

### **推奨イメージ**

- **開発環境**: `ruby:<version>`（通常版）
- **本番環境**: `ruby:<version>-slim`（軽量でバランスが良い）
- **軽量化を最優先する場合**: `ruby:<version>-alpine`（ただし、追加設定が必要）

## 2. Dockerfile の作成

以下のような **マルチステージビルド** を採用することで、不要なファイルを省きつつ、効率的なコンテナイメージを作成できます。

```dockerfile
# syntax = docker/dockerfile:1

ARG RUBY_VERSION=3.3.6
FROM ruby:$RUBY_VERSION-slim AS base

WORKDIR /rails

# 必要なパッケージをインストール
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y curl libjemalloc2 libvips freetds-dev && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development"

# ビルドステージ
FROM base AS build

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y build-essential git pkg-config && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

COPY Gemfile Gemfile.lock ./
RUN bundle install && \
    rm -rf ~/.bundle/ "$BUNDLE_PATH"/ruby/*/cache "$BUNDLE_PATH"/ruby/*/bundler/gems/*/.git && \
    bundle exec bootsnap precompile --gemfile

COPY . .
RUN bundle exec bootsnap precompile app/ lib/

# 本番用最終ステージ
FROM base

COPY --from=build "$BUNDLE_PATH" "$BUNDLE_PATH"
COPY --from=build /rails /rails

RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash && \
    chown -R rails:rails db log storage tmp
USER 1000:1000

ENTRYPOINT ["/rails/bin/docker-entrypoint"]

EXPOSE 3000
CMD ["./bin/rails", "server"]
```

## 3. docker-compose の利用

### **docker-compose.yml の作成**

以下の `docker-compose.yml` を作成し、Rails アプリケーションと MySQL を連携させます。

```yaml
version: '3.8'

services:
  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: myapp_production
      MYSQL_USER: myapp
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

  web:
    build: .
    depends_on:
      - db
    environment:
      RAILS_ENV: development
      DATABASE_URL: mysql2://myapp:password@db/myapp_production
    ports:
      - "3000:3000"
    volumes:
      - .:/rails
    command: ["./bin/rails", "server", "-b", "0.0.0.0"]

volumes:
  db_data:
```

## 4. コンテナのビルドと起動

### **ビルド**

```sh
docker-compose build
```

### **コンテナの起動**

```sh
docker-compose up -d
```

### **データベースのセットアップ**

```sh
docker-compose exec web rails db:create db:migrate
```

## 5. 開発環境でのホットリロード対応

本番環境ではコードの変更が自動で反映されませんが、開発環境では以下の方法でホットリロードを有効にできます。

### **Dockerfile の変更**

開発用の Dockerfile を作成し、`RAILS_ENV=development` に設定します。

```dockerfile
ENV RAILS_ENV="development" \
    BUNDLE_WITHOUT="production"
```

### **ボリュームマウント**

```sh
docker-compose up -d
```

これにより、ホストのコード変更がコンテナに即時反映されます。

### **開発環境向けDockerfile**

```dockerfile
# syntax = docker/dockerfile:1

# 開発環境用の Ruby イメージ
ARG RUBY_VERSION=3.3.6
FROM ruby:$RUBY_VERSION-slim

# 作業ディレクトリの設定
WORKDIR /rails

# 必要なパッケージのインストール
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y curl libjemalloc2 libvips freetds-dev build-essential git pkg-config && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# 環境変数の設定
ENV RAILS_ENV=development \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="production"

# Gemfile のコピーとバンドルインストール
COPY Gemfile Gemfile.lock ./
RUN bundle install

# アプリケーションのコードをコンテナにコピー
COPY . .

# 開発用のホットリロードを有効化
RUN gem install foreman

# ポートの公開
EXPOSE 3000

# サーバーの起動（ホットリロード対応）
CMD ["bundle", "exec", "foreman", "start", "-f", "Procfile.dev"]
```

## 7. まとめ

- **本番環境では ****\`slim 版\`**** を推奨**
- **マルチステージビルドで不要な依存を削減**
- **\`docker-compose\` を利用して MySQL と連携**
- **開発環境では ****\`development\`**** に設定し、ホットリロードを有効化**

この手順に従うことで、効率的な Rails コンテナの構築が可能になります。


