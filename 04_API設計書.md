# API設計書

**プロジェクト**: ユーザー管理システム
**作成日**: 2025-11-15
**ベースURL**: `https://api.example.com/v1`

---

## API概要

**設計方針**
- RESTful API
- JSON形式
- JWT認証
- HTTPステータスコードでエラー判定

**認証**
- Authorizationヘッダーにトークンを含める
```
Authorization: Bearer <JWT_TOKEN>
```
- アクセストークン有効期限: 1時間
- リフレッシュトークン: 30日

---

## エンドポイント一覧（CRUD）

| メソッド | エンドポイント | 説明 | 認証 |
|----------|---------------|------|------|
| POST | /api/v1/users | ユーザー作成 | 不要 |
| GET | /api/v1/users | ユーザー一覧取得 | 必要 |
| GET | /api/v1/users/:id | ユーザー詳細取得 | 必要 |
| PUT | /api/v1/users/:id | ユーザー更新 | 必要 |
| DELETE | /api/v1/users/:id | ユーザー削除 | 必要 |

---

## 1. ユーザー作成（CREATE）

`POST /api/v1/users`

**認証**: 不要

### リクエストボディ
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

**パラメータ**
- firstName, lastName: 必須、1〜50文字
- email: 必須、Email形式、一意
- password: 必須、8〜128文字、英数字記号混在
- birthDate: 必須、yyyy-MM-dd、13歳以上
- gender: 任意、male/female/other/prefer_not_to_say

### レスポンス

**成功時（201 Created）**
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

**エラー時（400）**
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

## 2. ユーザー一覧取得（READ）

`GET /api/v1/users`

**認証**: 必要

### クエリパラメータ

```
/api/v1/users?page=1&limit=20&search=山田&status=active&sortBy=createdAt&order=desc
```

- page: ページ番号（デフォルト: 1）
- limit: 件数（デフォルト: 20、最大: 100）
- search: 名前・メールで部分一致検索
- status: active/inactive
- sortBy: createdAt/lastName/email（デフォルト: createdAt）
- order: asc/desc（デフォルト: desc）

### レスポンス

**成功時（200）**
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

## 3. ユーザー詳細取得（READ）

`GET /api/v1/users/:id`

**認証**: 必要

### リクエスト例
```
GET /api/v1/users/usr_1234567890
```

### レスポンス

**成功時（200）**
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

**エラー時（404）**
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

## 4. ユーザー更新（UPDATE）

`PUT /api/v1/users/:id`

**認証**: 必要（本人または管理者）

### リクエストボディ
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

**パラメータ**
- firstName, lastName: 必須、1〜50文字
- email: 必須、Email形式、一意（自分除く）
- birthDate: 必須、yyyy-MM-dd
- gender: 任意
- status: active/inactive（管理者のみ変更可）

### レスポンス

**成功時（200）**
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

**エラー時（403）**
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

## 5. ユーザー削除（DELETE）

`DELETE /api/v1/users/:id`

**認証**: 必要（管理者のみ）
**注意**: 論理削除（物理削除はしない）

### リクエスト例
```
DELETE /api/v1/users/usr_1234567890
```

### レスポンス

**成功時（200）**
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

**エラー時（403）**
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

**リクエストヘッダー**
- Content-Type: application/json（必須）
- Authorization: Bearer <JWT_TOKEN>（認証必要な場合）

**レスポンスヘッダー**
- Content-Type: application/json
- X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset

**レート制限**
- 認証済み: 1000リクエスト/時
- 未認証: 100リクエスト/時

---

## HTTPステータスコード

| コード | 意味 |
|-------|------|
| 200 | 成功 |
| 201 | 作成成功 |
| 400 | バリデーションエラー |
| 401 | 認証エラー（未認証） |
| 403 | 権限エラー |
| 404 | リソース不在 |
| 409 | 重複エラー |
| 500 | サーバーエラー |

**エラーレスポンス形式**
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": [...]
  }
}
```

---

## メモ

- すべてのレスポンスは`success`フィールドでtrue/falseを返す
- エラーは`error.code`で種類を判定
- ページネーションは`hasNext`/`hasPrevious`で次ページの有無を判定
- 日時は全てISO 8601形式（UTC）
