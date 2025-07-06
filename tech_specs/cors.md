
```mermaid
sequenceDiagram
    participant Browser
    participant Server

    %% ステップ1: ブラウザがリクエストの種類をチェック
    note over Browser: (1) CORSが必要か判定<br>・オリジンが異なるか？<br>・シンプルリクエストか？
    
    alt シンプルリクエスト
        Browser->>Server: GET /api/data (直接リクエスト)
        note over Server: (2) オリジン許可の確認<br>・許可されたオリジンか？
        
        alt 許可されたオリジン
            Server->>Browser: HTTP 200 OK\nAccess-Control-Allow-Origin: https://example.com\nContent-Type: application/json\n{"data": "example"}
        else 許可されていないオリジン
            Server->>Browser: HTTP 403 Forbidden
        end
    else プリフライトリクエストが必要
        Browser->>Server: OPTIONS /api/data (Preflight Request)
        note over Server: (2) CORSチェック<br>・許可されたオリジンか？<br>・許可されたメソッドか？<br>・許可されたヘッダーか？
        
        alt 全て許可
            Server->>Browser: HTTP 200 OK\nAccess-Control-Allow-Origin: https://example.com\nAccess-Control-Allow-Methods: GET, POST\nAccess-Control-Allow-Headers: Content-Type, Authorization
            
            note over Browser: (3) レスポンスチェック<br>・Access-Control-Allow-Origin は一致するか？<br>・メソッドは許可されているか？<br>・ヘッダーは許可されているか？

            alt 許可されている
                Browser->>Server: GET /api/data (Actual Request)
                Server->>Browser: HTTP 200 OK\nAccess-Control-Allow-Origin: https://example.com\nContent-Type: application/json\n{"data": "example"}
            else 許可されていない
                Browser->>Browser: CORSエラーを発生
            end
        else 許可されていない
            Server->>Browser: HTTP 403 Forbidden
        end
    end
