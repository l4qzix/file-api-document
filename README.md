# Screenshot Uploader API 公式ドキュメント

このドキュメントは、あなたが作成した Screenshot Uploader 用 API（Cloudflare Workers + Supabase Storage / Database）の**公式仕様書**です。

---

## 概要

Screenshot Uploader API は以下の機能を提供します。

* ファイルアップロード
* ファイルの直接配信
* 短縮URLの生成・管理
* OGP / Embed 対応の共有ページ
* 管理者向けファイル管理API

**ベースURL**

```
https://3.4x.f5.si
```

---

## 認証

管理者向けエンドポイントでは、以下のヘッダーが必須です。

```
x-upload-token: <UPLOAD_TOKEN>
```

`UPLOAD_TOKEN` は Cloudflare Workers の環境変数で設定されます。

---

## CORS

すべてのエンドポイントは CORS に対応しています。

許可ヘッダー：

* Access-Control-Allow-Origin: *
* Access-Control-Allow-Methods: GET, POST, OPTIONS
* Access-Control-Allow-Headers: Content-Type, x-upload-token

---

## エンドポイント一覧

### 1. ファイル取得（直接URL）

```
GET /:filename
```

#### 説明

Supabase Storage に保存されたファイルを直接配信します。

#### レスポンス

* 成功: `200 OK`
* 失敗: `404 Not Found`

#### 特徴

* 長期キャッシュ（1年）
* Content-Type 自動設定

---

### 2. ファイルアップロード

```
POST /upload
```

#### 認証

必須（x-upload-token）

#### リクエスト（multipart/form-data）

| フィールド | 型    | 必須 | 説明           |
| ----- | ---- | -- | ------------ |
| file  | File | ✅  | アップロードするファイル |

#### レスポンス例

```json
{
  "url": "https://3.4x.f5.si/1700000000000-test.png",
  "short_url": "https://3.4x.f5.si/u/Ab3XzQ",
  "filename": "1700000000000-test.png",
  "original_name": "test.png",
  "size": 12345,
  "type": "image/png"
}
```

---

### 3. 短縮URLアクセス

```
GET /u/:id
```

#### 説明

短縮IDに対応するファイルへリダイレクトします。

* embed 無効: 302 リダイレクト
* embed 有効: OGP付きHTMLを返却

---

### 4. 管理者：ファイル一覧取得

```
GET /admin/list
```

#### 認証

必須

#### レスポンス例

```json
[
  {
    "id": "Ab3XzQ",
    "name": "1700000000000-test.png",
    "original_name": "test.png",
    "url": "https://3.4x.f5.si/1700000000000-test.png",
    "short_url": "https://3.4x.f5.si/u/Ab3XzQ",
    "size": 12345,
    "type": "image/png",
    "embed_enabled": false
  }
]
```

---

### 5. Embed設定更新

```
POST /admin/embed/update
```

#### 認証

必須

#### リクエストJSON

```json
{
  "id": "Ab3XzQ",
  "embed_enabled": true,
  "title": "Sample Image",
  "description": "This is a screenshot",
  "image": "https://example.com/ogp.png"
}
```

---

### 6. 単一ファイルの短縮リンク生成

```
POST /admin/generate-single-link
```

#### 認証

必須

#### リクエストJSON

```json
{
  "name": "1700000000000-test.png"
}
```

---

### 7. 短縮リンク未生成ファイルを一括処理

```
POST /admin/generate-missing-links
```

#### 説明

Storage に存在するが short_links に存在しないファイルに対し、短縮URLを自動生成します。

---

### 8. ファイル削除

```
POST /admin/delete
```

#### 認証

必須

#### リクエストJSON

```json
{
  "name": "1700000000000-test.png"
}
```

---

### 9. デバッグ：リンク整合性チェック

```
GET /debug/link-check?id=:id
```

#### 説明

* short_links の存在確認
* Storage 上のファイル存在確認
* URL計算結果を返却

---

## 短縮ID仕様

* 長さ: 6文字
* 使用文字: a-z A-Z 0-9
* crypto.getRandomValues による生成

---

## Embed / OGP 仕様

embed_enabled が true の場合、以下のOGPが付与されます。

* og:title
* og:description
* og:image
* og:url
* twitter:card (summary_large_image)

---

## 使用例（Examples）

### 1. ブラウザ拡張からスクリーンショットをアップロード

```js
const formData = new FormData();
formData.append("file", blob, "screenshot.png");

const res = await fetch("https://3.4x.f5.si/upload", {
  method: "POST",
  headers: {
    "x-upload-token": UPLOAD_TOKEN,
  },
  body: formData,
});

const data = await res.json();
console.log(data.short_url);
```

---

### 2. curl を使ったファイルアップロード

```bash
curl -X POST https://3.4x.f5.si/upload \
  -H "x-upload-token: YOUR_TOKEN" \
  -F "file=@image.png"
```

---

### 3. 短縮URLへアクセス

```text
https://3.4x.f5.si/u/Ab3XzQ
```

* embed 無効 → 直接ファイルへリダイレクト
* embed 有効 → OGP 付き HTML を返却

---

### 4. 管理画面でファイル一覧を取得

```js
const res = await fetch("https://3.4x.f5.si/admin/list", {
  headers: {
    "x-upload-token": UPLOAD_TOKEN,
  },
});

const files = await res.json();
console.log(files);
```

---

### 5. Embed（OGP）を有効化

```js
await fetch("https://3.4x.f5.si/admin/embed/update", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "x-upload-token": UPLOAD_TOKEN,
  },
  body: JSON.stringify({
    id: "Ab3XzQ",
    embed_enabled: true,
    title: "Screenshot",
    description: "Uploaded via extension",
  }),
});
```

---

### 6. 既存ファイルに短縮URLを付与

```js
await fetch("https://3.4x.f5.si/admin/generate-single-link", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "x-upload-token": UPLOAD_TOKEN,
  },
  body: JSON.stringify({ name: "1700000000000-test.png" }),
});
```

---

### 7. 短縮リンク未作成ファイルを一括処理

```bash
curl -X POST https://3.4x.f5.si/admin/generate-missing-links \
  -H "x-upload-token: YOUR_TOKEN"
```

---

### 8. ファイル削除

```js
await fetch("https://3.4x.f5.si/admin/delete", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "x-upload-token": UPLOAD_TOKEN,
  },
  body: JSON.stringify({ name: "1700000000000-test.png" }),
});
```

---

## 想定ユースケース

* ブラウザ拡張（Screenshot Uploader）
* Discord / X (Twitter) 共有用画像アップロード
* Gyazo 互換の個人用アップローダー

---

## 技術構成

* Cloudflare Workers
* Supabase Storage
* Supabase PostgREST
* 独自ドメイン（3.4x.f5.si）

---

## セキュリティ注意事項

* Service Role Key は必ず Worker 内部のみで使用
* x-upload-token は公開しないこと
* 公開ファイルは誰でもアクセス可能

---

以上が Screenshot Uploader API の公式ドキュメントです。
