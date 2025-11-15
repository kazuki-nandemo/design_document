# API設計書（エンドポイント一覧）

## プロジェクト: ユーザー管理システム
**作成日**: 2025-11-15
**バージョン**: 1.0
**ベースURL**: `https://api.example.com/v1`

---

## 目次
1. [API概要](#api概要)
2. [認証](#認証)
3. [エンドポイント一覧](#エンドポイント一覧)
4. [共通仕様](#共通仕様)
5. [エラーレスポンス](#エラーレスポンス)

---

## API概要

### 設計方針
- RESTful API設計
- JSON形式でのデータ送受信
- JWTトークンによる認証
- HTTPステータスコードに基づくエラーハンドリング

### バージョン管理
- URLパスにバージョンを含める（例: `/v1/users`）
- 後方互換性のない変更時は新バージョンを作成

---

## 認証

### 認証方式
- JWT（JSON Web Token）ベースの認証
- Authorizationヘッダーにトークンを含める

### ヘッダー形式
```
Authorization: Bearer <JWT_TOKEN>
```

### トークン有効期限
- アクセストークン: 1時間
- リフレッシュトークン: 30日

---

## エンドポイント一覧

| No | メソッド | エンドポイント | 説明 | 認証 |
|----|----------|---------------|------|------|
| 1 | POST | /api/v1/users | ユーザー作成（Create） | 不要 |
| 2 | GET | /api/v1/users | ユーザー一覧取得（Read） | 必要 |
| 3 | GET | /api/v1/users/:id | ユーザー詳細取得（Read） | 必要 |
| 4 | PUT | /api/v1/users/:id | ユーザー更新（Update） | 必要 |
| 5 | DELETE | /api/v1/users/:id | ユーザー削除（Delete） | 必要 |

---

## 1. ユーザー作成（Create）

### 基本情報
- **メソッド**: `POST`
- **パス**: `/api/v1/users`
- **説明**: 新規ユーザーを作成
- **認証**: 不要

### リクエスト

#### ヘッダー
```
Content-Type: application/json
```

#### リクエストボディ
```json
{
  "firstName": "太郎",
  "lastName": "山田",
  "email": "taro.yamada@example.com",
  "password": "SecurePass123!",
  "birthDate": "1990-01-01",
  "gender": "male"
}
```

#### パラメータ詳細
| フィールド | 型 | 必須 | 説明 | 制約 |
|-----------|-----|------|------|------|
| firstName | string | ○ | 名 | 1〜50文字 |
| lastName | string | ○ | 姓 | 1〜50文字 |
| email | string | ○ | メールアドレス | Email形式、一意 |
| password | string | ○ | パスワード | 8〜128文字、英数字記号混在 |
| birthDate | string | ○ | 生年月日 | yyyy-MM-dd形式、13歳以上 |
| gender | string | - | 性別 | male/female/other/prefer_not_to_say |

### レスポンス

#### 成功時（201 Created）
```json
{
  "success": true,
  "data": {
    "id": "usr_1234567890",
    "firstName": "太郎",
    "lastName": "山田",
    "email": "taro.yamada@example.com",
    "birthDate": "1990-01-01",
    "gender": "male",
    "status": "active",
    "createdAt": "2025-11-15T10:30:00Z",
    "updatedAt": "2025-11-15T10:30:00Z"
  },
  "message": "ユーザーが正常に作成されました"
}
```

#### エラー時（400 Bad Request）
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に誤りがあります",
    "details": [
      {
        "field": "email",
        "message": "このメールアドレスは既に登録されています"
      }
    ]
  }
}
```

---

## 2. ユーザー一覧取得（Read）

### 基本情報
- **メソッド**: `GET`
- **パス**: `/api/v1/users`
- **説明**: ユーザー一覧を取得（ページネーション、検索、フィルタリング対応）
- **認証**: 必要

### リクエスト

#### ヘッダー
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

#### クエリパラメータ
```
GET /api/v1/users?page=1&limit=20&search=山田&status=active&sortBy=createdAt&order=desc
```

| パラメータ | 型 | 必須 | デフォルト | 説明 |
|-----------|-----|------|-----------|------|
| page | integer | - | 1 | ページ番号 |
| limit | integer | - | 20 | 1ページあたりの件数（最大100） |
| search | string | - | - | 名前・メールアドレスで部分一致検索 |
| status | string | - | - | ステータスフィルタ（active/inactive） |
| sortBy | string | - | createdAt | ソート項目（createdAt/lastName/email） |
| order | string | - | desc | ソート順（asc/desc） |

### レスポンス

#### 成功時（200 OK）
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": "usr_1234567890",
        "firstName": "太郎",
        "lastName": "山田",
        "email": "taro.yamada@example.com",
        "birthDate": "1990-01-01",
        "gender": "male",
        "status": "active",
        "createdAt": "2025-11-15T10:30:00Z",
        "updatedAt": "2025-11-15T10:30:00Z"
      },
      {
        "id": "usr_0987654321",
        "firstName": "花子",
        "lastName": "佐藤",
        "email": "hanako.sato@example.com",
        "birthDate": "1992-05-15",
        "gender": "female",
        "status": "active",
        "createdAt": "2025-11-14T15:20:00Z",
        "updatedAt": "2025-11-14T15:20:00Z"
      }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 5,
      "totalCount": 100,
      "limit": 20,
      "hasNext": true,
      "hasPrevious": false
    }
  }
}
```

---

## 3. ユーザー詳細取得（Read）

### 基本情報
- **メソッド**: `GET`
- **パス**: `/api/v1/users/:id`
- **説明**: 指定IDのユーザー詳細を取得
- **認証**: 必要

### リクエスト

#### ヘッダー
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| id | string | ○ | ユーザーID |

#### リクエスト例
```
GET /api/v1/users/usr_1234567890
```

### レスポンス

#### 成功時（200 OK）
```json
{
  "success": true,
  "data": {
    "id": "usr_1234567890",
    "firstName": "太郎",
    "lastName": "山田",
    "email": "taro.yamada@example.com",
    "birthDate": "1990-01-01",
    "gender": "male",
    "status": "active",
    "profileImage": "https://storage.example.com/users/usr_1234567890/profile.jpg",
    "createdAt": "2025-11-15T10:30:00Z",
    "updatedAt": "2025-11-15T10:30:00Z",
    "lastLoginAt": "2025-11-15T14:00:00Z"
  }
}
```

#### エラー時（404 Not Found）
```json
{
  "success": false,
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "指定されたユーザーが見つかりません"
  }
}
```

---

## 4. ユーザー更新（Update）

### 基本情報
- **メソッド**: `PUT`
- **パス**: `/api/v1/users/:id`
- **説明**: 指定IDのユーザー情報を更新
- **認証**: 必要
- **権限**: 本人または管理者

### リクエスト

#### ヘッダー
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| id | string | ○ | ユーザーID |

#### リクエストボディ
```json
{
  "firstName": "太郎",
  "lastName": "山田",
  "email": "taro.yamada.new@example.com",
  "birthDate": "1990-01-01",
  "gender": "male",
  "status": "active"
}
```

#### パラメータ詳細
| フィールド | 型 | 必須 | 説明 | 制約 |
|-----------|-----|------|------|------|
| firstName | string | ○ | 名 | 1〜50文字 |
| lastName | string | ○ | 姓 | 1〜50文字 |
| email | string | ○ | メールアドレス | Email形式、一意（自分除く） |
| birthDate | string | ○ | 生年月日 | yyyy-MM-dd形式 |
| gender | string | - | 性別 | male/female/other/prefer_not_to_say |
| status | string | - | ステータス | active/inactive（管理者のみ変更可） |

### レスポンス

#### 成功時（200 OK）
```json
{
  "success": true,
  "data": {
    "id": "usr_1234567890",
    "firstName": "太郎",
    "lastName": "山田",
    "email": "taro.yamada.new@example.com",
    "birthDate": "1990-01-01",
    "gender": "male",
    "status": "active",
    "createdAt": "2025-11-15T10:30:00Z",
    "updatedAt": "2025-11-15T16:45:00Z"
  },
  "message": "ユーザー情報が正常に更新されました"
}
```

#### エラー時（403 Forbidden）
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "このユーザーを更新する権限がありません"
  }
}
```

---

## 5. ユーザー削除（Delete）

### 基本情報
- **メソッド**: `DELETE`
- **パス**: `/api/v1/users/:id`
- **説明**: 指定IDのユーザーを削除（論理削除）
- **認証**: 必要
- **権限**: 管理者のみ

### リクエスト

#### ヘッダー
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| id | string | ○ | ユーザーID |

#### リクエスト例
```
DELETE /api/v1/users/usr_1234567890
```

### レスポンス

#### 成功時（200 OK）
```json
{
  "success": true,
  "data": {
    "id": "usr_1234567890",
    "deletedAt": "2025-11-15T17:00:00Z"
  },
  "message": "ユーザーが正常に削除されました"
}
```

#### エラー時（403 Forbidden）
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "このユーザーを削除する権限がありません"
  }
}
```

---

## 共通仕様

### リクエストヘッダー
| ヘッダー | 必須 | 説明 |
|---------|------|------|
| Content-Type | ○ | application/json |
| Authorization | 条件付き | Bearer <JWT_TOKEN>（認証必要なエンドポイント） |
| X-Request-ID | - | リクエスト追跡用UUID |

### レスポンスヘッダー
| ヘッダー | 説明 |
|---------|------|
| Content-Type | application/json |
| X-Request-ID | リクエスト追跡用UUID |
| X-RateLimit-Limit | レート制限の上限 |
| X-RateLimit-Remaining | 残りリクエスト数 |
| X-RateLimit-Reset | レート制限リセット時刻（Unix timestamp） |

### レート制限
- 認証済み: 1000リクエスト/時間
- 未認証: 100リクエスト/時間

---

## エラーレスポンス

### HTTPステータスコード
| コード | 説明 |
|-------|------|
| 200 | 成功 |
| 201 | 作成成功 |
| 400 | リクエストエラー（バリデーションエラー等） |
| 401 | 認証エラー（未認証） |
| 403 | 権限エラー（認証済みだが権限なし） |
| 404 | リソース不在 |
| 409 | 競合エラー（重複等） |
| 422 | 処理不可（ビジネスロジックエラー） |
| 429 | レート制限超過 |
| 500 | サーバーエラー |
| 503 | サービス利用不可（メンテナンス等） |

### エラーレスポンス形式
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": [
      {
        "field": "フィールド名",
        "message": "詳細メッセージ"
      }
    ],
    "requestId": "req_xyz789"
  }
}
```

### エラーコード一覧
| コード | HTTPステータス | 説明 |
|-------|---------------|------|
| VALIDATION_ERROR | 400 | バリデーションエラー |
| UNAUTHORIZED | 401 | 認証エラー |
| FORBIDDEN | 403 | 権限エラー |
| USER_NOT_FOUND | 404 | ユーザー不在 |
| EMAIL_ALREADY_EXISTS | 409 | メールアドレス重複 |
| INTERNAL_SERVER_ERROR | 500 | サーバーエラー |
| RATE_LIMIT_EXCEEDED | 429 | レート制限超過 |

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|------------|------|--------|----------|
| 1.0 | 2025-11-15 | 設計チーム | 初版作成 |
