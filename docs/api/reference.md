# APIリファレンス

> **ひとことで：** ThreatGuard の REST API エンドポイント一覧です。

ThreatGuard は REST API を提供しており、すべての機能をAPIから利用できます。

## 認証

すべてのAPIリクエスト（公開エンドポイントを除く）には、`Authorization` ヘッダーにJWTトークンを含める必要があります。

```
Authorization: Bearer <token>
```

トークンは `/api/auth/login` エンドポイントで取得します。有効期限は7日間です。

---

## 認証（Auth）

### ログイン

```
POST /api/auth/login
```

メールアドレスとパスワードでJWTトークンを取得します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| email | string | Yes | メールアドレス |
| password | string | Yes | パスワード |

**レスポンス:** JWTトークン、ユーザー情報（id, email, name, role, organizationId）

### パスワードリセット要求

```
POST /api/auth/forgot-password
```

パスワードリセット用のメールを送信します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| email | string | Yes | 登録済みメールアドレス |

::: info レート制限
同一メールアドレスへのリセットトークン発行は5分間のクールダウンがあります。
:::

### パスワードリセット実行

```
POST /api/auth/reset-password
```

リセットトークンを使って新しいパスワードを設定します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| token | string | Yes | リセットトークン |
| password | string | Yes | 新しいパスワード |

---

## ヘルスチェック

### サーバーヘルス

```
GET /api/health
```

サーバーの稼働状態を確認します。認証不要。

### ブラウザヘルス

```
GET /api/health/browser
```

ヘッドレスブラウザ（Playwright/Chromium）の利用可否を確認します。

---

## ブランド管理（Brands）

### ブランド一覧取得

```
GET /api/brands
```

組織に紐づくブランドの一覧を取得します。

### ブランド作成

```
POST /api/brands
```

新しいブランドを登録します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| name | string | Yes | ブランド名 |
| domain | string | Yes | 正規ドメイン |
| keywords | string | No | 検索キーワード（カンマ区切り） |
| whitelistDomains | string | No | ホワイトリストドメイン（カンマ区切り） |

### ブランド詳細取得

```
GET /api/brands/:id
```

### ブランド更新

```
PUT /api/brands/:id
```

### ブランド削除

```
DELETE /api/brands/:id
```

### ロゴアップロード

ブランド作成・更新時にロゴ画像をアップロードできます。

| 制限 | 値 |
|------|-----|
| 最大サイズ | 2MB |
| 対応形式 | JPEG, PNG, GIF, WebP, SVG |

### 商標登録証アップロード

テイクダウン申請のエビデンスとして使用する商標登録証をアップロードできます。

| 制限 | 値 |
|------|-----|
| 最大サイズ | 5MB |
| 対応形式 | PDF, JPEG, PNG |

---

## 脅威管理（Threats）

### 脅威一覧取得

```
GET /api/threats
```

検出された脅威の一覧を取得します。フィルタリング・ソート・ページネーションに対応。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| status | string | No | ステータスでフィルタ（new_domain, analyzing, confirmed_threat, false_positive, takedown_sent, resolved） |
| category | string | No | カテゴリでフィルタ（phishing, brand_abuse, parked, legitimate, unknown） |
| minRiskScore | number | No | リスクスコアの下限 |
| maxRiskScore | number | No | リスクスコアの上限 |
| brandId | string | No | ブランドIDでフィルタ |
| excludeResolved | boolean | No | 解決済みを除外 |
| sortBy | string | No | ソート項目（riskScore, firstSeen, domain） |
| sortOrder | string | No | ソート順（asc, desc） |
| page | number | No | ページ番号 |
| limit | number | No | 1ページあたりの件数 |

### 脅威詳細取得

```
GET /api/threats/:id
```

---

## スキャン管理（Scans）

### スキャン実行

```
POST /api/scans/trigger
```

手動でスキャンを実行します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| brandId | string | Yes | 対象ブランドID |
| type | string | No | スキャン種別（ct_monitor, domain_generation） |

**レスポンス:** スキャンジョブID、ステータス

### スキャンジョブ一覧

```
GET /api/scans
```

スキャンジョブの実行履歴を取得します。

| ステータス | 説明 |
|-----------|------|
| pending | 実行待ち |
| running | 実行中 |
| completed | 完了 |
| failed | 失敗 |

### WHOISバックフィル

```
POST /api/scans/whois-backfill
```

WHOIS/RDAP情報が未取得のドメインに対して一括でWHOIS情報を取得します。superadmin のみ実行可能。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| forceRefresh | boolean | No | 既存のWHOIS情報を強制的に再取得 |

::: info 最適化
Raw SQLクエリによるPrismaタイムアウト回避、ページネーション対応。WHOIS取得に失敗したドメインは記録され、再試行を防止します。
:::

### ジオコードバックフィル

```
POST /api/scans/geocode-backfill
```

国コード情報が未取得のドメインに対してIPジオロケーション情報を一括取得します。ユニークIPアドレスごとにバッチ処理で効率化されています。

---

## 削除申請（Takedowns）

### 削除申請の作成

```
POST /api/takedowns
```

脅威に対する削除申請テンプレートを生成します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| detectedDomainId | string | Yes | 対象の検出ドメインID |
| recipientType | string | No | 送信先種別（registrar, police, jpcert） |

### 削除申請の更新

```
PUT /api/takedowns/:id
```

ステータスや却下理由を更新します。

### PDF ダウンロード

```
GET /api/takedowns/:id/pdf
```

削除申請テンプレートをPDF形式でダウンロードします。

### JPCERT連絡先情報

```
GET /api/takedowns/jpcert-info
```

JPCERT/CCの連絡先情報を取得します。

---

## 一括テイクダウン（Takedown Batches）

### バッチ作成

```
POST /api/takedown-batches
```

複数ドメインの削除申請をバッチとしてまとめて作成します。

### バッチ一覧取得

```
GET /api/takedown-batches
```

---

## ダッシュボード（Dashboard）

### 統計情報取得

```
GET /api/dashboard/stats
```

ダッシュボード用の統計データを取得します。

**レスポンスに含まれるデータ:**

| 項目 | 説明 |
|------|------|
| riskCounts | リスクレベル別の件数（danger, high, medium, low） |
| statusCounts | ステータス別の件数（action_needed, monitoring, resolved） |
| brandThreats | ブランド別の脅威件数 |
| takedownStats | 削除申請のステータス別件数 |
| timeline | 時系列データ |
| categoryBreakdown | カテゴリ別の内訳 |
| recentChanges | 最近のサイト変化（DNS、HTTPステータス、フォーム検出） |

---

## Webプローブ（Web Probe）

### プローブ実行

```
POST /api/web-probe
```

指定ドメインに対してWebプローブ（ページ調査）を実行します。

**調査内容:** DNS解決、HTTP接続、最終URL、IPアドレス、国コード（IPジオロケーション）、SSL証明書情報、HTMLスニペット、レスポンスヘッダー、スクリーンショット

---

## レポート（Reports）

### レポート生成

```
POST /api/reports
```

### レポート一覧取得

```
GET /api/reports
```

---

## フィッシングパターン（Phishing Patterns）

### パターン一覧取得

```
GET /api/phishing-patterns
```

### パターン報告

```
POST /api/phishing-patterns
```

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| brandId | string | Yes | 対象ブランドID |
| url | string | Yes | フィッシングサイトのURL |
| domain | string | Yes | フィッシングサイトのドメイン |
| description | string | No | 補足説明 |
| tags | string | No | 分類タグ（カンマ区切り） |

---

## アラート設定（Alerts）

### アラート設定更新

```
POST /api/alerts
```

ユーザーのメール通知設定を更新します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| alertEnabled | boolean | No | メール通知の有効/無効 |
| alertThreshold | number | No | 通知しきい値（リスクスコア） |

---

## SNS監視（Social Posts）

### 投稿一覧取得

```
GET /api/social-posts
```

### 投稿ステータス更新

```
POST /api/social-posts
```

---

## ブラウザ削除申請（Browser Reports）

### 申請送信

```
POST /api/browser-reports
```

Google Safe Browsing / Microsoft SmartScreen にフィッシング報告を送信します。

---

## 無料診断（Public）

### 診断実行

```
POST /api/public/diagnose
```

認証不要の公開エンドポイントです。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| url | string | Yes | 診断対象のURL |
| email | string | Yes | 結果通知先メールアドレス |

**レスポンス:** 診断ID、ステータス

::: info 診断結果の保持期間
診断結果は7日間保持されます。期限を過ぎると自動削除されます。
:::

---

## アクティビティログ（Activity Logs）

### ログ一覧取得

```
GET /api/activity-logs
```

全ユーザーの操作履歴を取得します。superadmin のみアクセス可能。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| page | number | No | ページ番号 |
| limit | number | No | 1ページあたりの件数 |

---

## 組織管理（Organizations）

### 組織一覧取得

```
GET /api/organizations
```

superadmin のみアクセス可能。

### 組織作成

```
POST /api/organizations
```

superadmin のみ実行可能。

---

## エラーレスポンス

すべてのエラーは以下の形式で返されます：

```json
{
  "error": "エラーメッセージ"
}
```

| HTTPステータス | 説明 |
|---------------|------|
| 400 | リクエストパラメータの不正 |
| 401 | 認証エラー（トークン未提供または無効） |
| 403 | 権限エラー（アクセス権限なし） |
| 404 | リソースが見つからない |
| 500 | サーバー内部エラー |
