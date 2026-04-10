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

**レスポンス:** JWTトークン、ユーザー情報（id, email, name, role, organizationId, organizationName）

::: info ソフトデリート
削除済みユーザー（`deletedAt` が設定済み）はログインが拒否されます。
:::

### パスワードリセット要求

```
POST /api/auth/forgot-password
```

パスワードリセット用のメールを送信します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| email | string | Yes | 登録済みメールアドレス |

::: info レート制限
同一メールアドレスへのリセットトークン発行は5分間のクールダウンがあります。ユーザー列挙攻撃を防ぐため、存在しないメールアドレスでも同一のレスポンスを返します。
:::

### パスワードリセット実行

```
POST /api/auth/reset-password
```

リセットトークンを使って新しいパスワードを設定します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| token | string | Yes | リセットトークン |
| newPassword | string | Yes | 新しいパスワード（8文字以上） |

::: info セキュリティ
- トークンはSHA-256でハッシュ化して保存
- 有効期限: 1時間
- ワンタイム使用（使用済みトークンは再利用不可）
- パスワード更新とトークン消費はトランザクションで原子的に実行
:::

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

組織に紐づくブランドの一覧を取得します。superadmin は全組織のブランドを取得可能。各ブランドには `lastScan`（最終スキャン日時）と `monitoringStatus`（active / inactive / running / error）が付与されます。

### ブランド詳細取得

```
GET /api/brands/:id
```

ブランドの詳細情報を取得します。`brandDomains` と関連カウント情報を含みます。

### ブランド作成

```
POST /api/brands
```

新しいブランドを登録します。作成後に自動でフルスキャン（`runFullScan`）が実行されます。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| name | string | Yes | ブランド名 |
| domain | string | Yes | 正規ドメイン |
| keywords | string | No | 検索キーワード（カンマ区切り） |
| whitelistDomains | string | No | ホワイトリストドメイン（カンマ区切り） |
| organizationId | string | No | 組織ID（superadmin のみ指定可） |
| senderEmail | string | No | テイクダウンメール送信元アドレス |
| smtpHost | string | No | SMTPサーバーホスト |
| smtpPort | number | No | SMTPポート番号 |
| smtpUser | string | No | SMTP認証ユーザー |
| smtpPass | string | No | SMTP認証パスワード |

### ブランド更新

```
PUT /api/brands/:id
```

ブランド情報を更新します。`whitelistDomains` が変更された場合、自動的にフルスキャンが再実行されます。

### ブランド削除

```
DELETE /api/brands/:id
```

### ブランド統計取得

```
GET /api/brands/:id/stats
```

ブランドの脅威統計を取得します。

**レスポンスに含まれるデータ:**

| 項目 | 説明 |
|------|------|
| statusCounts | ステータス別の脅威件数（groupBy） |
| riskBands | リスクレベル別の件数（critical / high / medium / low） |
| avgRisk | 平均リスクスコア |
| dailyCounts | 直近30日間の日別検出数 |
| recentScans | 直近10件のスキャン結果 |

### スキャン履歴取得

```
GET /api/brands/:id/scans
```

ブランドのスキャン実行履歴をページネーション付きで取得します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| page | number | No | ページ番号（デフォルト: 1） |
| limit | number | No | 1ページあたりの件数（デフォルト: 10） |

### ロゴアップロード

```
POST /api/brands/:id/logo
```

ブランドのロゴ画像をアップロードします。`multipart/form-data` で送信。

| 制限 | 値 |
|------|-----|
| 最大サイズ | 2MB |
| 対応形式 | JPEG, PNG, GIF, WebP, SVG |
| 最小解像度 | 200×200px（SVGを除く） |

::: info
アップロード時に既存のロゴファイルは自動削除されます。
:::

### ロゴ削除

```
DELETE /api/brands/:id/logo
```

### 商標登録証アップロード

```
POST /api/brands/:id/trademark-cert
```

テイクダウン申請のエビデンスとして使用する商標登録証をアップロードします。`multipart/form-data` で送信。

| 制限 | 値 |
|------|-----|
| 最大サイズ | 5MB |
| 対応形式 | PDF, JPEG, PNG, WebP |

### 商標登録証削除

```
DELETE /api/brands/:id/trademark-cert
```

### スクリーンショット撮影

```
POST /api/brands/:id/capture-screenshot
```

ブランドの正規ドメインのスクリーンショットを撮影・保存します。

### ドメイン一覧取得

```
GET /api/brands/:id/domains
```

ブランドに紐づく `BrandDomain`（プライマリドメイン・保有ドメイン）の一覧を取得します。

### ドメイン追加

```
POST /api/brands/:id/domains
```

ブランドにドメインを追加します。追加されたドメインに一致する検出済み脅威は自動的に `false_positive` に再分類されます。プライマリドメインの追加時にはフルスキャンが自動実行されます。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| domain | string | Yes | ドメイン名 |
| type | string | Yes | ドメイン種別（`primary` / `owned`） |

::: info 重複チェック
同一組織内でのドメイン重複は拒否されます（異なるブランド間でも同一組織内では重複不可）。
:::

### ドメイン削除

```
DELETE /api/brands/:id/domains/:domainId
```

### ドメイン一括追加

```
POST /api/brands/:id/domains/bulk
```

複数ドメインを一括追加します。カンマ・セミコロン・改行区切りに対応。重複チェックおよび脅威の自動再分類が実行されます。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| domains | string | Yes | ドメインリスト（カンマ / セミコロン / 改行区切り） |

### ホワイトリストCSVインポート

```
POST /api/brands/:id/whitelist/import
```

CSVファイルからホワイトリストドメインを一括インポートします。

### ドメインCSVインポート

```
POST /api/brands/:id/domains/import-csv
```

CSVファイルからドメインを一括インポートします。ヘッダー自動検出、デリミタ自動判別（カンマ / タブ）、type カラム（primary / owned）に対応。

### ブランドCSV一括インポート

```
POST /api/brands/import-csv
```

CSVファイルから複数ブランドを一括登録します。日本語ヘッダーにも対応。superadmin は組織IDまたは組織名を指定可能（組織名指定時は自動作成）。各ブランド作成後にフルスキャンが自動実行されます。

### ドメイン同期

```
POST /api/brands/sync-domains
```

`Brand.whitelistDomains` フィールドと `BrandDomain` テーブルを同期します。superadmin のみ実行可能。

---

## 脅威管理（Threats）

### 脅威一覧取得

```
GET /api/threats
```

検出された脅威の一覧を取得します。フィルタリング・ソート・ページネーションに対応。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| status | string | No | ステータスでフィルタ（カンマ区切りで複数指定可） |
| category | string | No | カテゴリでフィルタ |
| minRiskScore | number | No | リスクスコアの下限 |
| maxRiskScore | number | No | リスクスコアの上限 |
| brandId | string | No | ブランドIDでフィルタ |
| excludeResolved | boolean | No | 解決済みを除外 |
| sortBy | string | No | ソート項目（デフォルト: `riskScore`） |
| order | string | No | ソート順（`asc` / `desc`、デフォルト: `desc`） |
| page | number | No | ページ番号（デフォルト: 1） |
| pageSize | number | No | 1ページあたりの件数（デフォルト: 20） |

**レスポンス:** `{ data, total, page, pageSize, totalPages }`

### 脅威詳細取得

```
GET /api/threats/:id
```

脅威の詳細情報を取得します。以下の関連データを含みます：

- ブランド情報（組織情報含む）
- AI分析結果（`ThreatAnalysis`、全履歴）
- 削除申請（`TakedownRequest`、全履歴）
- ブラウザ削除申請（`BrowserReport`、全履歴）
- ページ調査結果（`WebProbe`、直近5件）
- ステータス変更履歴（`ThreatStatusLog`、全履歴）

### コンテンツ分析実行

```
GET /api/threats/:id/content-analysis
```

検出ドメインに対してリアルタイムでコンテンツ分析を実行します。HTML解析（キーワードマッチ、ログインフォーム検知、パスワードフィールド検知、ロゴ検出）と画像類似度比較を行います。

### abuse連絡先取得

```
GET /api/threats/:id/abuse-contacts
```

検出ドメインのabuse連絡先をWHOIS/RDAP経由で取得します。

**レスポンス:** レジストラ名、abuse連絡先メールアドレス、情報ソース

### ステータス更新

```
PATCH /api/threats/:id/status
```

脅威のステータスを変更します。変更履歴が `ThreatStatusLog` に記録されます。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| status | string | Yes | 新しいステータス |

**有効なステータス:** `new_domain`, `analyzing`, `confirmed_threat`, `false_positive`, `takedown_sent`, `resolved`

---

## スキャン管理（Scans）

### スキャン実行

```
POST /api/scans/trigger
```

手動でスキャンを実行します。非同期で処理され、HTTP 202を返します。スキャン完了後、リスクスコア60以上の脅威が検出された場合はSlack/メール通知が送信されます。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| brandId | string (UUID) | Yes | 対象ブランドID |
| type | string | Yes | スキャン種別（`ct_monitor` / `domain_generation` / `manual`） |

**レスポンス:** HTTP 202、スキャンジョブ情報

### スキャンジョブ一覧

```
GET /api/scans
```

スキャンジョブの実行履歴を取得します。組織に紐づくブランドのスキャンのみ返却されます。最新50件、`startedAt` 降順。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| brandId | string | No | ブランドIDでフィルタ |

| ステータス | 説明 |
|-----------|------|
| pending | 実行待ち |
| running | 実行中 |
| completed | 完了 |
| failed | 失敗 |

### WHOISバックフィル

```
POST /api/scans/backfill-whois
```

WHOIS/RDAP情報が未取得のドメインに対して一括でWHOIS情報を取得します。superadmin のみ実行可能。HTTP 202を返し、バックグラウンドで処理します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| limit | number | No | 処理件数の上限（最大200） |
| delay | number | No | リクエスト間隔（ms、デフォルト: 2000） |
| offset | number | No | オフセット（ページネーション用） |
| refresh | boolean | No | 既存のWHOIS情報を強制的に再取得 |
| retry | boolean | No | 以前失敗したドメインを再試行 |

::: info 最適化
Raw SQLクエリによるPrismaタイムアウト回避、ページネーション対応。WHOIS取得に失敗したドメインは `whoisData` に `"FETCH_FAILED"` と記録され、通常モードでは再試行を防止します。
:::

### ジオコードバックフィル

```
POST /api/scans/backfill-geocode
```

国コード情報が未取得のWebProbeに対してIPジオロケーション情報を一括取得します。superadmin のみ実行可能。HTTP 202を返し、バックグラウンドで処理します。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| limit | number | No | 処理件数の上限（最大500） |

::: info 最適化
ユニークIPアドレスごとにバッチ処理を行い、同一IPを持つ全WebProbeレコードを一括更新します。ip-api.com のレート制限に対応するため、リクエスト間隔は約1.4秒に設定されています。
:::

---

## 削除申請（Takedowns）

### 削除申請の作成

```
POST /api/takedowns
```

脅威に対する削除申請テンプレートを生成します。送信先種別に応じたテンプレート生成関数が呼び出されます。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| detectedDomainId | string (UUID) | Yes | 対象の検出ドメインID |
| recipientType | string | No | 送信先種別（`registrar` / `police` / `jpcert`、デフォルト: `registrar`） |

### 削除申請の更新

```
PUT /api/takedowns/:id
```

ステータスを更新します。ステータスに応じてタイムスタンプが自動設定されます。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| status | string | Yes | 新しいステータス |

| ステータス | タイムスタンプ |
|-----------|-------------|
| draft | — |
| sent | `sentAt` を設定 |
| acknowledged | `respondedAt` を設定 |
| completed | `respondedAt` を設定 |
| rejected | `respondedAt` を設定 |

### PDF ダウンロード

```
GET /api/takedowns/:id/pdf
```

削除申請テンプレートをPDF形式でダウンロードします。Chromium によるHTML→PDF変換で生成されます。

### メール送信

```
POST /api/takedowns/:id/send
```

削除申請メールを abuse 連絡先に送信します。送信後、ステータスが自動的に `sent` に更新されます。

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| email | string | Yes | 送信先メールアドレス |

::: info エラー分類
送信失敗時のエラーは以下のように分類されます：
- **SMTP認証エラー**: メール送信設定の認証情報を確認してください
- **接続エラー**: メールサーバーへの接続に失敗しました
- **ブラウザエラー**: PDF生成用のブラウザ起動に失敗しました
:::

### JPCERT連絡先情報

```
GET /api/takedowns/jpcert-info
```

JPCERT/CCの連絡先情報（名称、メールアドレス、PGP鍵URL）を取得します。

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

**調査内容:** DNS解決、HTTP接続（HTTPS→HTTPフォールバック）、最終URL、IPアドレス、国コード（IPジオロケーション）、SSL証明書情報（発行者・有効期限・プロトコル・サブジェクト名）、HTMLスニペット（先頭5,000文字）、レスポンスヘッダー、スクリーンショット

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

**記録される情報:** 操作内容（action）、対象エンティティ（entityType / entityId）、メタデータ、IPアドレス、ユーザーエージェント

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
