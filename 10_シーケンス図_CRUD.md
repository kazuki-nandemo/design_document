# シーケンス図（CRUD操作）

## プロジェクト: ユーザー管理システム
**作成日**: 2025-11-15
**バージョン**: 1.0

---

## 目次
1. [CREATE（ユーザー作成）](#1-createユーザー作成)
2. [READ（ユーザー一覧取得）](#2-readユーザー一覧取得)
3. [READ（ユーザー詳細取得）](#3-readユーザー詳細取得)
4. [UPDATE（ユーザー更新）](#4-updateユーザー更新)
5. [DELETE（ユーザー削除）](#5-deleteユーザー削除)

---

## 1. CREATE（ユーザー作成）

### シーケンス図

```mermaid
sequenceDiagram
    participant C as クライアント<br/>(ブラウザ)
    participant A as APIサーバー<br/>(Node.js/Express)
    participant V as バリデーション<br/>レイヤー
    participant B as ビジネスロジック<br/>レイヤー
    participant D as データベース<br/>(PostgreSQL)
    participant S as ストレージ<br/>(S3)

    Note over C,S: ユーザー新規作成フロー

    C->>A: POST /api/v1/users<br/>{firstName, lastName, email, password, ...}
    activate A

    A->>V: リクエストバリデーション
    activate V
    V->>V: 必須項目チェック
    V->>V: 形式チェック（Email, Password等）
    V->>V: 文字数チェック

    alt バリデーションエラー
        V-->>A: バリデーションエラー
        A-->>C: 400 Bad Request<br/>{error: "バリデーションエラー"}
    else バリデーション成功
        V-->>A: OK
        deactivate V

        A->>B: ユーザー作成処理
        activate B

        B->>D: メールアドレス重複チェック<br/>SELECT * FROM users WHERE email = ?
        activate D
        D-->>B: 結果
        deactivate D

        alt メールアドレス重複
            B-->>A: 重複エラー
            A-->>C: 409 Conflict<br/>{error: "メールアドレスが既に登録されています"}
        else 重複なし
            B->>B: パスワードハッシュ化<br/>(bcrypt, cost=12)
            B->>B: ユーザーID生成<br/>(usr_xxxxx)
            B->>B: プロフィールID生成<br/>(prf_xxxxx)

            B->>D: トランザクション開始<br/>BEGIN;
            activate D

            B->>D: ユーザー挿入<br/>INSERT INTO users (...)
            D-->>B: 成功

            B->>D: プロフィール挿入<br/>INSERT INTO user_profiles (...)
            D-->>B: 成功

            B->>D: コミット<br/>COMMIT;
            D-->>B: 完了
            deactivate D

            B->>S: デフォルトプロフィール画像コピー<br/>(オプション)
            activate S
            S-->>B: URL
            deactivate S

            B-->>A: ユーザーデータ
            deactivate B

            A->>A: レスポンス生成<br/>(パスワードハッシュを除外)

            A-->>C: 201 Created<br/>{success: true, data: {user}}
            deactivate A
        end
    end
```

### 処理フロー説明

1. **リクエスト受信**: クライアントから新規ユーザー作成リクエスト
2. **バリデーション**: 入力値の妥当性チェック
3. **重複チェック**: メールアドレスの重複確認
4. **パスワードハッシュ化**: bcryptでパスワードをハッシュ化
5. **ID生成**: ユーザーIDとプロフィールIDを生成
6. **トランザクション処理**:
   - usersテーブルにINSERT
   - user_profilesテーブルにINSERT
7. **デフォルト画像**: 必要に応じてデフォルトプロフィール画像を設定
8. **レスポンス**: 作成されたユーザー情報を返却

---

## 2. READ（ユーザー一覧取得）

### シーケンス図

```mermaid
sequenceDiagram
    participant C as クライアント<br/>(ブラウザ)
    participant A as APIサーバー<br/>(Node.js/Express)
    participant M as 認証ミドルウェア
    participant B as ビジネスロジック<br/>レイヤー
    participant D as データベース<br/>(PostgreSQL)
    participant R as Redis<br/>(キャッシュ)

    Note over C,R: ユーザー一覧取得フロー（ページネーション付き）

    C->>A: GET /api/v1/users?page=1&limit=20&search=山田<br/>Authorization: Bearer <token>
    activate A

    A->>M: Token認証
    activate M
    M->>M: JWT検証
    M->>M: 署名チェック
    M->>M: 有効期限チェック

    alt Token無効
        M-->>A: 認証エラー
        A-->>C: 401 Unauthorized
    else Token有効
        M->>M: ユーザー情報抽出
        M-->>A: ユーザーID
        deactivate M

        A->>B: ユーザー一覧取得<br/>(page, limit, search)
        activate B

        B->>B: クエリパラメータ検証
        B->>B: キャッシュキー生成<br/>(users:page:1:limit:20:search:山田)

        B->>R: キャッシュ確認
        activate R
        R-->>B: キャッシュミス
        deactivate R

        B->>D: ユーザー一覧取得<br/>SELECT u.*, p.profile_image_url<br/>FROM users u<br/>LEFT JOIN user_profiles p ON u.id = p.user_id<br/>WHERE deleted_at IS NULL<br/>AND (first_name LIKE ? OR last_name LIKE ? OR email LIKE ?)<br/>ORDER BY created_at DESC<br/>LIMIT 20 OFFSET 0
        activate D
        D-->>B: ユーザーリスト
        deactivate D

        B->>D: 総件数取得<br/>SELECT COUNT(*) FROM users<br/>WHERE deleted_at IS NULL AND ...
        activate D
        D-->>B: 総件数
        deactivate D

        B->>B: ページネーション情報計算<br/>(totalPages, hasNext, hasPrevious)

        B->>R: キャッシュ保存（TTL: 60秒）
        activate R
        R-->>B: OK
        deactivate R

        B-->>A: {users, pagination}
        deactivate B

        A-->>C: 200 OK<br/>{success: true, data: {users, pagination}}
        deactivate A
    end
```

### 処理フロー説明

1. **認証**: JWTトークンの検証
2. **パラメータ検証**: ページ番号、件数、検索条件のバリデーション
3. **キャッシュ確認**: Redisでキャッシュの有無を確認
4. **データ取得**: ユーザー一覧とプロフィール画像をJOIN取得
5. **総件数取得**: ページネーション用の総件数を取得
6. **ページネーション計算**: 総ページ数、次ページ有無等を計算
7. **キャッシュ保存**: 結果をRedisに保存（60秒間）
8. **レスポンス**: ユーザー一覧とページネーション情報を返却

---

## 3. READ（ユーザー詳細取得）

### シーケンス図

```mermaid
sequenceDiagram
    participant C as クライアント<br/>(ブラウザ)
    participant A as APIサーバー<br/>(Node.js/Express)
    participant M as 認証ミドルウェア
    participant B as ビジネスロジック<br/>レイヤー
    participant D as データベース<br/>(PostgreSQL)

    Note over C,D: ユーザー詳細取得フロー

    C->>A: GET /api/v1/users/usr_1234567890<br/>Authorization: Bearer <token>
    activate A

    A->>M: Token認証
    activate M
    M->>M: JWT検証
    M-->>A: ユーザーID
    deactivate M

    A->>B: ユーザー詳細取得<br/>(userId)
    activate B

    B->>D: ユーザー情報取得<br/>SELECT u.*, p.*<br/>FROM users u<br/>LEFT JOIN user_profiles p ON u.id = p.user_id<br/>WHERE u.id = ? AND u.deleted_at IS NULL
    activate D
    D-->>B: ユーザーデータ
    deactivate D

    alt ユーザー不在
        B-->>A: NotFoundError
        A-->>C: 404 Not Found<br/>{error: "ユーザーが見つかりません"}
    else ユーザー存在
        B->>B: センシティブ情報除外<br/>(password_hash等)
        B-->>A: ユーザーデータ
        deactivate B

        A-->>C: 200 OK<br/>{success: true, data: {user}}
        deactivate A
    end
```

### 処理フロー説明

1. **認証**: JWTトークンの検証
2. **データ取得**: usersとuser_profilesをJOINして取得
3. **存在チェック**: ユーザーの存在確認
4. **センシティブ情報除外**: パスワードハッシュ等を除外
5. **レスポンス**: ユーザー詳細情報を返却

---

## 4. UPDATE（ユーザー更新）

### シーケンス図

```mermaid
sequenceDiagram
    participant C as クライアント<br/>(ブラウザ)
    participant A as APIサーバー<br/>(Node.js/Express)
    participant M as 認証ミドルウェア
    participant P as 権限チェック
    participant V as バリデーション<br/>レイヤー
    participant B as ビジネスロジック<br/>レイヤー
    participant D as データベース<br/>(PostgreSQL)
    participant R as Redis<br/>(キャッシュ)

    Note over C,R: ユーザー更新フロー

    C->>A: PUT /api/v1/users/usr_1234567890<br/>Authorization: Bearer <token><br/>{firstName, lastName, email, ...}
    activate A

    A->>M: Token認証
    activate M
    M-->>A: ユーザーID (認証済みユーザー)
    deactivate M

    A->>P: 権限チェック
    activate P
    P->>P: 本人または管理者か確認

    alt 権限なし
        P-->>A: ForbiddenError
        A-->>C: 403 Forbidden<br/>{error: "権限がありません"}
    else 権限あり
        P-->>A: OK
        deactivate P

        A->>V: リクエストバリデーション
        activate V
        V->>V: 入力値チェック
        V-->>A: OK
        deactivate V

        A->>B: ユーザー更新処理
        activate B

        B->>D: ユーザー存在確認<br/>SELECT * FROM users WHERE id = ? AND deleted_at IS NULL
        activate D
        D-->>B: ユーザーデータ
        deactivate D

        alt ユーザー不在
            B-->>A: NotFoundError
            A-->>C: 404 Not Found
        else ユーザー存在
            B->>D: メールアドレス重複チェック<br/>SELECT * FROM users<br/>WHERE email = ? AND id != ? AND deleted_at IS NULL
            activate D
            D-->>B: 結果
            deactivate D

            alt メール重複
                B-->>A: ConflictError
                A-->>C: 409 Conflict<br/>{error: "メールアドレスが既に使用されています"}
            else 重複なし
                B->>D: ユーザー更新<br/>UPDATE users SET<br/>first_name = ?, last_name = ?, email = ?, ...,<br/>updated_at = CURRENT_TIMESTAMP<br/>WHERE id = ?
                activate D
                D-->>B: 更新完了
                deactivate D

                B->>D: 更新後データ取得<br/>SELECT u.*, p.* FROM users u<br/>LEFT JOIN user_profiles p ON u.id = p.user_id<br/>WHERE u.id = ?
                activate D
                D-->>B: ユーザーデータ
                deactivate D

                B->>R: キャッシュ削除<br/>(該当ユーザーの全キャッシュ)
                activate R
                R-->>B: OK
                deactivate R

                B-->>A: ユーザーデータ
                deactivate B

                A-->>C: 200 OK<br/>{success: true, data: {user}, message: "更新しました"}
                deactivate A
            end
        end
    end
```

### 処理フロー説明

1. **認証**: JWTトークンの検証
2. **権限チェック**: 本人または管理者のみ更新可能
3. **バリデーション**: 入力値の妥当性チェック
4. **存在確認**: 更新対象ユーザーの存在確認
5. **重複チェック**: 変更後のメールアドレスの重複確認
6. **更新実行**: usersテーブルをUPDATE
7. **キャッシュ削除**: 該当ユーザーのキャッシュを削除
8. **レスポンス**: 更新後のユーザー情報を返却

---

## 5. DELETE（ユーザー削除）

### シーケンス図

```mermaid
sequenceDiagram
    participant C as クライアント<br/>(ブラウザ)
    participant A as APIサーバー<br/>(Node.js/Express)
    participant M as 認証ミドルウェア
    participant P as 権限チェック
    participant B as ビジネスロジック<br/>レイヤー
    participant D as データベース<br/>(PostgreSQL)
    participant R as Redis<br/>(キャッシュ)
    participant Q as ジョブキュー<br/>(Bull/RabbitMQ)

    Note over C,Q: ユーザー削除フロー（論理削除）

    C->>A: DELETE /api/v1/users/usr_1234567890<br/>Authorization: Bearer <token>
    activate A

    A->>M: Token認証
    activate M
    M-->>A: ユーザーID (認証済みユーザー)
    deactivate M

    A->>P: 権限チェック（管理者のみ）
    activate P
    P->>P: 管理者権限確認

    alt 権限なし
        P-->>A: ForbiddenError
        A-->>C: 403 Forbidden<br/>{error: "管理者権限が必要です"}
    else 管理者権限あり
        P-->>A: OK
        deactivate P

        A->>B: ユーザー削除処理
        activate B

        B->>D: ユーザー存在確認<br/>SELECT * FROM users WHERE id = ? AND deleted_at IS NULL
        activate D
        D-->>B: ユーザーデータ
        deactivate D

        alt ユーザー不在
            B-->>A: NotFoundError
            A-->>C: 404 Not Found
        else ユーザー存在
            B->>D: トランザクション開始<br/>BEGIN;
            activate D

            B->>D: 論理削除実行<br/>UPDATE users SET<br/>deleted_at = CURRENT_TIMESTAMP,<br/>status = 'inactive'<br/>WHERE id = ?
            D-->>B: 更新完了

            B->>D: セッション無効化<br/>UPDATE user_sessions SET<br/>is_active = FALSE,<br/>revoked_at = CURRENT_TIMESTAMP<br/>WHERE user_id = ?
            D-->>B: 更新完了

            B->>D: コミット<br/>COMMIT;
            D-->>B: 完了
            deactivate D

            B->>R: キャッシュ削除<br/>(該当ユーザーの全キャッシュ)
            activate R
            R-->>B: OK
            deactivate R

            B->>Q: 削除後処理をキューに追加<br/>(メール送信、ファイル削除等)
            activate Q
            Q-->>B: ジョブID
            deactivate Q

            B-->>A: 削除完了データ
            deactivate B

            A-->>C: 200 OK<br/>{success: true, data: {id, deletedAt}, message: "削除しました"}
            deactivate A

            Note over Q: バックグラウンド処理
            Q->>Q: ユーザー削除通知メール送信
            Q->>Q: プロフィール画像削除（S3）
            Q->>Q: 関連データアーカイブ
        end
    end
```

### 処理フロー説明

1. **認証**: JWTトークンの検証
2. **権限チェック**: 管理者権限の確認
3. **存在確認**: 削除対象ユーザーの存在確認
4. **トランザクション処理**:
   - usersテーブルのdeleted_atを更新（論理削除）
   - user_sessionsの全セッションを無効化
5. **キャッシュ削除**: 該当ユーザーのキャッシュを削除
6. **バックグラウンド処理**: 削除後処理をジョブキューに追加
   - 削除通知メール送信
   - プロフィール画像削除
   - 関連データのアーカイブ
7. **レスポンス**: 削除完了情報を返却

---

## エラーハンドリングパターン

### 共通エラーレスポンス

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": [
      {
        "field": "email",
        "message": "メールアドレスの形式が正しくありません"
      }
    ],
    "requestId": "req_xyz789"
  }
}
```

### エラーハンドリングフロー

```mermaid
sequenceDiagram
    participant C as クライアント
    participant A as APIサーバー
    participant E as エラーハンドラー
    participant L as ロガー

    C->>A: リクエスト
    activate A

    A->>A: 処理実行

    alt エラー発生
        A->>E: エラーオブジェクト
        activate E
        E->>E: エラー種別判定
        E->>L: エラーログ記録
        activate L
        L-->>E: OK
        deactivate L
        E->>E: レスポンス生成
        E-->>A: エラーレスポンス
        deactivate E
        A-->>C: 4xx/5xx レスポンス
    else 正常終了
        A-->>C: 200/201 レスポンス
    end
    deactivate A
```

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|------------|------|--------|----------|
| 1.0 | 2025-11-15 | 設計チーム | 初版作成 |
