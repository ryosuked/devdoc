
# フレームワーク別ミドルウェア機能比較表（拡張版）

Goで利用される代表的なWebフレームワークにおけるミドルウェアの充実度・対応状況の比較表です。

| 機能                         | Gin                           | Echo                          | Fiber                         | net/http（標準）             |
|------------------------------|--------------------------------|-------------------------------|-------------------------------|-------------------------------|
| ロギング                     | ✅ gin.Logger                  | ✅ echo/middleware.Logger     | ✅ fiber/middleware.Logger    | 🔶 logパッケージで実装可能    |
| リカバリ（パニック処理）     | ✅ gin.Recovery                | ✅ echo/middleware.Recover    | ✅ fiber/middleware.Recover   | 🔶 recover構文で実装可能      |
| CORS                         | ✅ gin-contrib/cors           | ✅ echo/middleware.CORS       | ✅ fiber/middleware.CORS      | 🔶 net/http + 外部実装        |
| セッション管理              | 🔶 gin-contrib/sessions       | 🔶 echo-contrib/session       | ✅ fiber/middleware/session   | 🔶 外部ライブラリ（gorilla/sessions など） |
| CSRF                         | ✅ gin-contrib/csrf           | ✅ echo/middleware.CSRF       | ✅ fiber/middleware/csrf      | 🔶 gorilla/csrf などを利用    |
| 認証（JWT）                 | ✅ gin-contrib/jwt            | ✅ echo/middleware.JWT        | ✅ fiber/middleware/jwt       | 🔶 github.com/golang-jwt/jwt など |
| OAuth2 対応                 | 🔶 外部パッケージ (e.g., osin) | 🔶 外部パッケージ (e.g., goth) | 🔶 外部パッケージ (e.g., goth) | 🔶 外部パッケージ必須         |
| OpenID Connect (OIDC) 対応  | 🔶 coreos/go-oidc など         | 🔶 coreos/go-oidc など         | 🔶 coreos/go-oidc など         | 🔶 coreos/go-oidc など         |
| FIDO / パスキー対応         | 🔶 外部実装が必要 (e.g., webauthn) | 🔶 外部実装が必要         | 🔶 外部実装が必要         | 🔶 外部実装が必要              |
| WebSocket対応               | 🔶 gorilla/websocketなど       | 🔶 echo に内包（独自ラッパー）| ✅ fiber 内包           | 🔶 gorilla/websocketなど       |
| Staticファイル提供          | ✅ gin.Static                  | ✅ echo.Static                | ✅ fiber.Static              | 🔶 http.FileServerなどで実装   |

> ✅: 標準/公式サポートあり  
> 🔶: 外部ライブラリ/サードパーティによるサポートあり

## 備考

- **OAuth2/OIDC** はどのフレームワークも公式には内包しておらず、`goth` や `osin`、`coreos/go-oidc` などの外部パッケージを使う必要があります。
- **FIDO (WebAuthn/Passkey)** についても、どのフレームワークも直接的なミドルウェアは持っておらず、実装例を元に組み込む必要があります。
