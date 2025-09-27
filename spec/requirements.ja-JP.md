# プロジェクト要件定義書

## 1. プロジェクト概要

### 1.1 プロジェクト名
OAuth2-Proxy + Azure AD認証システム

### 1.2 プロジェクトの目的
Azure Active Directory (Entra ID)を利用したWebアプリケーションの認証・認可システムを構築し、静的コンテンツまたは外部Webアプリケーションへの安全なアクセスを提供する。

### 1.3 主要な機能
- Azure AD（Entra ID）による認証
- OAuth2/OpenIDConnect認証フロー
- セッション管理（Redis）
- SSL/TLS終端とリバースプロキシ
- 自動SSL証明書取得・更新

## 2. システム構成

### 2.1 アーキテクチャ概要
```
[ユーザー] → [Nginx (TLS終端)] → [OAuth2-Proxy] → [バックエンドアプリケーション]
                    ↓
               [Redis (セッション)]
```

### 2.2 システム構成要素

#### 2.2.1 認証プロキシ (oauth2-proxy)
- **目的**: Azure ADとの認証仲介、セッション管理
- **技術**: OAuth2-Proxy v7.2.1
- **ポート**: 4180 (内部)
- **機能**:
  - Azure AD OAuth2認証
  - Redisセッションストレージ
  - 認証済みユーザー情報のヘッダー注入
  - auth_requestエンドポイント提供

#### 2.2.2 Webサーバー (nginx)
- **目的**: TLS終端、リバースプロキシ、静的コンテンツ配信
- **技術**: Nginx latest
- **ポート**: 80 (HTTP), 443 (HTTPS)
- **機能**:
  - SSL/TLS終端 (Let's Encrypt証明書)
  - HTTP→HTTPS自動リダイレクト
  - OAuth2-Proxyとの認証連携 (auth_request)
  - 静的コンテンツ配信
  - 外部アプリケーションへのリバースプロキシ

#### 2.2.3 セッションストレージ (redis)
- **目的**: 分散セッション管理
- **技術**: Redis 7-alpine
- **ポート**: 6379 (内部)
- **機能**:
  - OAuth2-Proxyのセッション永続化

#### 2.2.4 SSL証明書管理 (certbot)
- **目的**: Let's Encrypt SSL証明書の自動取得・更新
- **技術**: Certbot latest
- **機能**:
  - ACME-01チャレンジによる証明書取得
  - 証明書の自動更新

## 3. 認証フロー

### 3.1 初回アクセス時の認証フロー
1. ユーザーがWebサイトにアクセス
2. Nginxがauth_requestでOAuth2-Proxyに認証状態を確認
3. 未認証の場合、Azure ADログインページにリダイレクト
4. ユーザーがAzure ADで認証
5. OAuth2-Proxyがコールバックを受信し、セッションを作成
6. 認証済みとしてWebサイトにリダイレクト

### 3.2 認証済みアクセス時のフロー
1. ユーザーがWebサイトにアクセス
2. Nginxがauth_requestでOAuth2-Proxyに認証状態を確認
3. セッションが有効な場合、直接コンテンツを提供

## 4. 環境変数設定

### 4.1 必須環境変数
- `OAUTH2_PROXY_CLIENT_ID`: Azure ADアプリケーションのクライアントID
- `OAUTH2_PROXY_CLIENT_SECRET`: Azure ADアプリケーションのクライアントシークレット
- `OAUTH2_PROXY_AZURE_TENANT`: Azure ADのテナントID
- `OAUTH2_PROXY_REDIRECT_URL`: OAuth2コールバックURL
- `OAUTH2_PROXY_COOKIE_SECRET`: セッションCookie暗号化キー
- `NGINX_SERVER_NAME`: サーバーのドメイン名
- `NGINX_SERVER_NAME_WWW`: WWWサブドメインを含むドメイン名
- `NGINX_PROXY_PASS`: バックエンドアプリケーションのURL

## 5. セキュリティ要件

### 5.1 SSL/TLS設定
- TLS 1.2/1.3の使用
- Let's Encrypt証明書の自動更新
- HTTP Strict Transport Security (HSTS)推奨

### 5.2 認証設定
- Secure Cookieの使用
- セッション情報のRedis永続化
- 適切なCookie Domain設定 (.potofo.net)

### 5.3 プロキシ設定
- 適切なHTTPヘッダーの転送
- X-Forwarded-*ヘッダーの設定
- WebSocket接続のサポート

## 6. デプロイメント要件

### 6.1 Docker Compose構成
- 全サービスのコンテナ化
- ネットワーク分離 (default-net)
- 永続化ボリューム設定
- 適切な依存関係定義

### 6.2 ファイル構成
```
/
├── docker-compose.yaml      # メインのDocker Compose設定
├── .env                     # 環境変数設定
├── .env.example            # 環境変数テンプレート
├── nginx/
│   ├── nginx.conf          # Nginxメイン設定
│   └── templates/
│       └── default.conf.template  # サイト設定テンプレート
├── web/                    # 静的コンテンツ
│   └── index.html
├── letsencrypt/            # SSL証明書ディレクトリ
└── certbot-www/            # ACME-01チャレンジディレクトリ
```

## 7. 運用要件

### 7.1 ログ管理
- 各コンテナのログ出力
- 認証ログの監視
- エラーログの集約

### 7.2 監視要件
- サービスヘルスチェック
- SSL証明書の有効期限監視
- Redis接続状態の監視

### 7.3 バックアップ要件
- Redis セッションデータ（必要に応じて）
- SSL証明書とキー
- 設定ファイル

## 8. 開発・テスト要件

### 8.1 開発環境
- Docker/Docker Composeでの環境構築
- ローカルでのSSL証明書テスト
- Azure ADテストテナントの利用

### 8.2 設定管理
- 環境別設定の分離
- シークレット情報の安全な管理
- 設定変更時の影響範囲の明確化

## 9. トラブルシューティング

### 9.1 よくある問題
- Azure AD設定の不備
- リダイレクトURL の不一致
- SSL証明書の期限切れ
- Redis接続エラー

### 9.2 デバッグ機能
- `/oauth2/userinfo` エンドポイントでユーザー情報確認
- 各種ログレベルの調整
- ヘルスチェックエンドポイント

## 10. 拡張性要件

### 10.1 スケーラビリティ
- 複数インスタンスでのセッション共有
- ロードバランサー対応
- Redis Cluster対応（将来的）

### 10.2 カスタマイズ
- 認証後のユーザー情報活用
- カスタムエラーページ
- 追加認証プロバイダーの対応

---

この要件定義書は、OAuth2-Proxy + Azure AD認証システムの構築・運用・保守において、開発チームと運用チームが共通理解を持つためのドキュメントです。