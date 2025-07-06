# OpenID Connect (OIDC) UserInfo Endpoint の解説

UserInfo Endpoint は、認証されたユーザーの属性情報（クレーム）を取得するための保護されたリソースです。

## 1. 定義と目的
UserInfo Endpoint は、OIDC において **OpenID Provider (OP)** が提供する API エンドポイントです。
- **目的:** クライアント（RP）が、アクセストークンを使用してユーザーの名前、メールアドレス、プロフィール画像などの詳細情報を取得するために使用します。
- **セキュリティ:** このエンドポイントへのアクセスには、有効な **Access Token** が必須です。

## 2. 仕様の要点

### リクエスト
- **メソッド:** `GET` または `POST`
- **認証:** `Authorization` ヘッダーに `Bearer {Access Token}` を含めます。
- **スコープ:** `openid` スコープに加え、取得したい情報に応じたスコープ（`profile`, `email`, `address`, `phone` など）が認可リクエスト時に要求されている必要があります。

### レスポンス
- **形式:** 通常は `application/json` ですが、署名・暗号化された `application/jwt` で返される場合もあります。
- **ステータスコード:** 成功時は `200 OK`。トークンが無効な場合は `401 Unauthorized`。

## 3. 標準的なクレーム (Claims)
UserInfo Endpoint から返される代表的な項目は以下の通りです。

| クレーム名 | 内容 |
| :--- | :--- |
| `sub` | ユーザーの識別子 (Subject)。**必須。** |
| `name` | ユーザーのフルネーム |
| `given_name` | 名 |
| `family_name` | 姓 |
| `email` | メールアドレス |
| `picture` | プロフィール画像の URL |
| `locale` | 言語・地域設定 |

## 4. 実例

### リクエスト例 (curl)
```bash
curl -X GET https://www.googleapis.com/oauth2/v3/userinfo \
     -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

### レスポンス例 (JSON)
```json
{
  "sub": "248289761001",
  "name": "Jane Doe",
  "given_name": "Jane",
  "family_name": "Doe",
  "email": "jane.doe@example.com",
  "email_verified": true,
  "picture": "https://example.com/janedoe/me.jpg",
  "locale": "ja-JP"
}
```

## 5. IDトークンとの使い分け
- **IDトークン:** クライアント内でのセッション維持や、最低限のユーザー識別（`sub`）に使用します。フロントエンドでもデコード可能です。
- **UserInfo:** より詳細な、あるいは最新のプロフィール情報を取得するためにバックエンドなどから呼び出します。IDトークンのサイズ肥大化を防ぐ役割もあります。
