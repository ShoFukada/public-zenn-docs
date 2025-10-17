---
title: 'JWTについて'
---

## JWTとは何か（本質）

**JWT (JSON Web Token)** は、JSONデータをトークン文字列に変換し、デジタル署名で改ざんを検知できるようにする**データフォーマットの仕様**です（RFC 7519）。

```text
JWTの本質:
  JSONオブジェクト  ←→  署名付きトークン文字列
```

これ以上でも以下でもありません。

:::message
「JWT認証」という言葉がよく使われますが、**JWTは認証プロトコルではありません**。
JWTは認証の仕組みというのは誤解であり、JWTはデータをトークンに変換する方法という方が正しいです。
Tokenの形式として (たとえばOIDCのID Tokenなら必ず)JWTが使用されることがある、ということです。
:::

### JWTの特徴

JWTの特徴は以下の通りです。

- self-contained（自己完結型）
  - トークン自体に情報（claims）が含まれている
  - データベース照会なしで検証・利用可能
- Signed（署名付き）
  - 改ざん検知が可能
  - 発行者の正当性を検証可能
- URL-safe
  - URLやHTTPヘッダーで安全に送信可能
- デフォルトでは暗号化されていない
  - 内容は誰でも読める（Base64デコードするだけ）
  - ※機密情報を入れないこと

## JWTの構造

JWTは**3つの部分をドット(.)で連結した文字列**です。以下は一例です。

```text
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

形式: <Header>.<Payload>.<Signature>
       ヘッダー  ペイロード   署名
```

### 各部分の役割

3つの部分の役割について説明します。

#### 1. Header（ヘッダー）

署名アルゴリズムなどのメタデータ

```json
{
  "alg": "RS256", // 署名アルゴリズム
  "typ": "JWT", // トークンタイプ
  "kid": "abc123" // 鍵ID（どの公開鍵で検証するか）
}
```

これをBase64URLエンコード → `eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9`

#### 2. Payload（ペイロード）

実際のデータ（claims = 主張する情報）

```json
{
  // 標準クレーム
  "iss": "https://accounts.google.com", // 発行者
  "sub": "248289761001", // ユーザーID
  "aud": "your-client-id", // 対象者
  "exp": 1701238167, // 有効期限（Unix timestamp）
  "iat": 1701234567, // 発行時刻

  // カスタムクレーム（自由に追加可能）
  "name": "田中太郎",
  "email": "tanaka@example.com",
  "role": "admin"
}
```

これをBase64URLエンコード → `eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0`

#### 3. Signature（署名）

改ざん検知のための署名。以下は一例のコードです。

```javascript
// 署名の生成（RS256の場合）
const dataToSign = base64UrlEncode(header) + '.' + base64UrlEncode(payload)
const signature = RSA_SHA256(dataToSign, privateKey)

// 検証
const isValid = RSA_SHA256_VERIFY(dataToSign, signature, publicKey)
```

## Access TokenとID TokenにおけるJWT

### ID Token: 必ずJWT形式

OpenID Connect仕様で明確に規定されています。

:::message
なぜID TokenはJWT形式なのか？　→　クライアントが自分で検証する必要があるからです。

1. 認証サーバーに問い合わせずユーザー情報を取得
2. 署名を検証して正当性を確認

:::

### Access Token: JWT形式とは限らない

OAuth 2.0仕様はトークン形式を規定していません。

```text
Access Tokenの形式は実装次第

JWT形式の場合:
  - Resource Serverがトークンを自己検証できる
  - マイクロサービス環境で便利
  - ステートレス

Opaque Token（不透明）の場合:
  - ランダムな文字列（中身は見えない）
  - Resource Serverは認証サーバーに問い合わせて検証
  - 即座に失効可能（セキュリティ重視）
```

```javascript
// JWT形式のAccess Token
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0...
// → デコード可能、Resource Serverが自己検証

// Opaque Token
2YotnFZFEjr1zCsicMWpAA
// → ランダム文字列、認証サーバーに問い合わせが必要
```

## まとめ

- **JWTは「データをトークンに変換する仕様」**
  - 認証プロトコルではない。
  - JSONオブジェクト ↔ 署名付きトークン文字列の変換。
- **暗号化されていない**
  - 誰でも内容を読める（Base64デコード）。
  - 署名は「改ざん検知」のため。
- **ID Token: 必ずJWT形式（OpenID Connect仕様）**
  - クライアントが署名検証して使用。
- **Access Token: JWT形式とは限らない（OAuth 2.0仕様）**
  - JWT: ステートレス、マイクロサービス向き。
  - Opaque: セキュリティ重視、失効処理が必要な場合。

## 次のステップ

次のチャプターでは、エンタープライズで使用されるSAMLについて学びます。
