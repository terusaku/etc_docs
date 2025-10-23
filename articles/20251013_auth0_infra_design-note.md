---
title: "Auth0導入時にSREとして考えたこと"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zennfes2025infra"]
published: true
published_at: 2025-10-24 09:30
---

＊先だって、会社のテックブログ(note)を執筆したので、許可をもらって転載してます

https://note.com/lisse_developers/n/nf0162dc19dbd

現在、SaaS開発のSREチームで働いているので、得られた経験を時々、ブログとして残してみます。

(以下、転載)
---

ちょうど今年、当社サービスでは「SSO・MFA対応」を目的としてAuth0導入を完了し、それも稼働し始めて半年ほど経過しました。

https://lisse-law.com/release/notice/20250404/

そこで改めて、導入時の設計メモ、そして運用Tips（後日談）をまとめてみます。この記事の話題はインフラ領域に絞るため、（話題は色々あるけれど）アプリ実装には触れません 🙇

## terraformによるIaC管理

設計内容やAuth0の仕様理解を始める前に、IaC管理の方針だけ先に決めていました。

[Terraform Registry](https://registry.terraform.io/providers/auth0/auth0/latest/docs)

たまたま、開発チームのコアメンバーもterraformの知見があり、開発チーム・SREチーム、それぞれに1人ずつterraform担当がいる状態だったため、一から始めるにしても理想的だったかもしれません。

- terraform構成(抜粋)
    
```bash
# environmentsフォルダ
$ tree -L2 ./environments
./environments
├── develop
│   ├── main.tf
│   ├── terraform.tfvars_sample
│   └── variables.tf
# 中略

# modulesフォルダ
$ tree -L2 ./modules
./modules
├── actions  # Auth0 Actions（認証フロー中の カスタムロジック実行）の設定とソースコード
│   ├── main.tf
│   ├── scripts
│   └── variables.tf
├── apis  # Auth0で管理するAPI定義（スコープ、権限等）
│   ├── main.tf
│   └── variables.tf
├── authentications  # 認証方式の設定（DB、パスワードポリシー等）
│   └── main.tf
├── base_applications  # アプリケーション共通モジュール（他のアプリモジュールから利用）
│   ├── main.tf
│   └── variables.tf
├── xxxxx_applications  # アプリケーション設定
│   ├── main.tf
│   └── variables.tf
├── branding  # UI・送信メールのブランディング設定（ロゴ、色、メールテンプレート等）
│   ├── email_template
│   ├── main.tf
│   ├── prompt_custom_text
│   └── variables.tf
├── custom_domains  # カスタムドメイン設定
│   ├── main.tf
│   └── variables.tf
├── email_provider  # メール送信プロバイダ設定
│   ├── main.tf
│   └── variables.tf
├── log_streams  #  ログストリーム設定（AWS/Datadogへのログ転送）
│   ├── main.tf
│   └── variables.tf
├── security  # セキュリティ設定（ブルートフォース攻撃対策、不審なIP制限等）
│   ├── main.tf
│   └── templates
└── tenant_settings  # Auth0テナントの基本設定（ログイン方式、セッションタイムアウト等）
    ├── main.tf
    └── variables.tf
```

## 非機能要件リスト

基本的には「Auth0仕様にならって作ればOK」というスタンスでしたが、その仕様理解のためにも[SQuaRE シリーズの品質モデル](https://www.ipa.go.jp/archive/files/000065855.pdf)を参考に非機能要件・設計項目を整理しました。

要否の判断や内容詳細は非公開ですが、次のような項目で整理・タスク化していました。

| 非機能要件 | 設計項目 | 必要性 | 内容 | 対応方針 | 備考 |
| --- | --- | --- | --- | --- | --- |
| 可用性 | テナント、Actionプロセス | - | - | - | - |
| バックアップ/リストア | パスワードハッシュを含むユーザデータ | - | - | - | - |
| モニタリング | アカウントロック、各種スロットリング、Actionプロセスエラー | - | - | - | - |
| インシデント管理 | ログ管理、イベント検知 | - | - | - | - |
| サービス運用 | Auth0ダッシュボード管理、アクセス許可URLの更新 | - | - | - | - |

## Auth0導入とサービス基盤構成の考慮

Auth0によってユーザ認証の仕組みが変わるので、アプリ実装やデータ移行の苦労が一番でしたが、SREチームの領域では、サービス基盤（主にAWS環境）とどう統合するか、という対応がメインでした。

### カスタムドメイン

Route 53で管理し、Auth0のカスタムドメイン設定に適用しています。

このカスタムドメインはログイン画面（Universal Login）だけではなく、認証エンドポイントなど、Auth0 管理API全体にも適用されるので、ベースURLとして分かりやすい名前がおすすめです。

### メール送信プロバイダ

サービス基盤（AWS環境）に合わせるため、今回はAmazon SESを採用しています。
SPF/DKIM/DMARC、メール送信のセキュリティには一通り対応済みなので、Auth0導入にあたって追加のセキュリティ設定は不要でした。

### SMS認証のサポート

Amazon SNS APIの実行権限をAuth0 Actionに付与し、`send-phone-message`というトリガーのActionをデプロイしています。Actionフロー詳細は割愛しますが、次の辺りは少なくとも必要になってくるはず。

1. SSO/MFAの条件をチェック
2. 必要であればMFA要求をリクエスト
3. MFA実行（認証コードの送信）
4. MFA完了後、カスタムクレームを設定

ちなみに、SMSの送信文面ではログイン先を明示するため、イベント変数からアプリケーション名を参照しています。

```jsx
  const template = event.message_options.text;
  const text = `ログイン先サービス: ${event.client.name}
${template}`;
```

### Datadogによる監視

- [Auth0 - Log Streams](https://auth0.com/docs/customize/log-streams)を使い、アラート通知はDatadogで対応しています
- Auth0ログの監視によってインシデント検知も可能になってます
- 念のためAuth0自体のテナント障害は `https://status.auth0.com/rss?domain=${Tenant_Domain}` を直接、Slack通知しています

### S3によるログ管理

- 監視のためのログはDatadog、長期保存はAmazon S3、という方針をとっています
- "Eventbridge" + "Kinesis Firehose"により、パーティション化した上でS3にAuth0ログを長期保存しています

## 運用開始後のTips

### 定常オペ

- Auth0ダッシュボード運用について、部分的に手作業の運用している
    - Auth0ダッシュボードのユーザやロールは、terraform対象外のため
- Auth0ダッシュボードのログインは、SSO認証のみ許可
    - [Enterpriseプランのみ使える機能](https://auth0.com/docs/get-started/manage-dashboard-access/configure-single-sign-on-for-auth0-dashboard)で、Auth0サポートに依頼は必要ですが便利です
- Auth0ログによるイベントをDatadogでモニタリングし、関連チームにSlackメンション

### 随時オペ

- 設定変更時、Auth0単体で動作確認する方法
    
    ```jsx
    https://${AUTH0_DOMAIN}/authorize?client_id=${CLIENT_ID}&response_type=code&redirect_uri=${REDIRECT_URI}
    ```
    
    - OAuth仕様の認証であれば、上記のようなURLでUniversal Login画面に直接アクセスできるため、アプリケーション無しでログイン動作（SSO/MFA含む）を単体確認できます
    - （クエリ文字列は対象アプリの設定次第）
- 本番環境のterraform作業では、`TF_CLI_ARGS_plan="-parallelism=1"`がほぼ必須に…
    - モジュールやリソースの数が増えた結果、terraform操作によって「Auth0 管理APIのグローバルレートリミット超過」が発生するようになってしまったためです
    - サービスの管理機能でもAuth0管理APIを実行しているので、ユーザ影響を避けるため並列実行数を1に制限
- 負荷テストを許可されている時間は深夜のみ（詳細は参考ページ参照）
    - Enterpriseプランのみ、「テナントリージョン毎の指定時間帯」のみ負荷テストを許可されています
    - Auth0によるJWT認証でバックエンドAPIを認証・認可している場合、API負荷テストも同様の時間帯にしか実施できないので要注意です

## Auth0導入の感想

仕様のキャッチアップにはある程度時間をかけましたが、「Auth0仕様に沿って設計・実装する」というガードレールができたので、セキュリティ設計としては考えやすかった気がします。

また、SaaSマルチテナント構成についても、Auth0は「一つのテナントで複数のアプリと顧客を管理する」という仕様になっているため、構成レベルの設計はシンプルでした。

データ移行やアプリ実装は別途、対応や考慮が必要であったし、Tenant・Application・Organization、の分離方針はSaaS要件（ビジネスモデル）に依存するため、その辺りの答えは場合によって変わってくるはずです。

## 参考

### Auth0仕様全般

[Security Guidance](https://auth0.com/docs/secure/security-guidance)

[Check Error Messages](https://auth0.com/docs/troubleshoot/basic-issues/check-error-messages)

[Best Practices for Application Session Management](https://auth0.com/blog/application-session-management-best-practices/)

[The Not-So-Easy Art of Logging Out](https://auth0.com/blog/the-not-so-easy-art-of-logging-out/)

[Load Testing Policy](https://auth0.com/docs/troubleshoot/customer-support/operational-policies/load-testing-policy)

### Auth0 Action実装

[Action Coding Guidelines](https://auth0.com/docs/customize/actions/action-coding-guidelines)

### Auth0 管理APIのレートリミット

[Rate Limit Configurations](https://auth0.com/docs/troubleshoot/customer-support/operational-policies/rate-limit-policy/rate-limit-configurations)

### Auth0ログ管理

https://github.com/aws-samples/amazon-eventbridge-integration-with-auth0

### SQuaRE規格による品質モデル
https://webdesk.jsa.or.jp/common/W10K0620?id=1225
