---
title: "Auth0を導入する際に考えた非機能要件の設計メモ"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zennfes2025infra"]
published: true

---

現在、SaaS開発組織のSREチームとして働いている。
ちょうど今年、そのプロダクトで「SSO・MFA対応」を目的としてAuth0導入を完了し、それも稼働して半年ほど経過した。なので、改めて導入時の設計メモ、そして運用Tips（後日談）をまとめてみる。

この記事の話題はインフラ領域が絞るため、（話題は色々ある）アプリ実装の設計・Tipsには触れない。

## terraformによるIaC管理
設計内容やAuth0の仕様理解を始める前に、IaC管理の方針だけ先に決めていた。
https://registry.terraform.io/providers/auth0/auth0/latest/docs

たまたま、開発チームのコアメンバーもterraformの知見があり、開発チーム・SREチーム、それぞれに1人ずつterraform担当がいるような状態だったため、理想的だったかもしれない。

:::details terraform構成(抜粋)
```sh
# environmentsフォルダ
$ tree -L2 ./environments 
./environments
├── develop
│   ├── main.tf
│   ├── terraform.tfvars_sample
│   └── variables.tf
# 以下略

# modulesフォルダ
$ tree -L2 ./modules                                 
./modules
├── actions
│   ├── main.tf
│   ├── scripts
│   └── variables.tf
├── apis
│   ├── main.tf
│   └── variables.tf
├── authentications
│   └── main.tf
├── base_applications
│   ├── main.tf
│   └── variables.tf
├── branding
│   ├── email_template
│   ├── main.tf
│   ├── prompt_custom_text
│   └── variables.tf
├── custom_domains
│   ├── main.tf
│   └── variables.tf
├── email_provider
│   ├── main.tf
│   └── variables.tf
├── log_streams
│   ├── main.tf
│   └── variables.tf
├── security
│   ├── main.tf
│   └── templates
└── tenant_settings
    ├── main.tf
    └── variables.tf
```
:::

## 非機能要件リスト
基本的には「Auth0仕様にならって作ればOK」というスタンスだが、その仕様理解のためにも[SQuaRE シリーズの品質モデル](https://www.ipa.go.jp/archive/files/000065855.pdf)を参考に非機能要件・設計項目を整理した。
ここでは詳細を割愛するが、次のような項目をタスク管理していた。
| 非機能要件 | 設計項目 | 必要性 | 内容 | 対応方針 | 備考 |
| --- | --- | --- | --- | --- | --- |
| 可用性 | テナント、Actionプロセス | - | - | - | - |
| バックアップ/リストア | - | - | - | - | - |
| モニタリング | アカウントロック、各種スロットリング、Actionプロセスエラー | - | - | - | - |
| インシデント管理 | ログ管理 | - | - | - | - |
| サービス運用 | Auth0ダッシュボード管理、アクセス許可URLの更新 | - | - | - | - |

## Auth0導入とサービス基盤構成
Auth0によってユーザ認証の仕組みが変わるので、アプリ実装やデータ移行の苦労が一番だったはず。
SREチームの領域では、構成変更というより、サービス基盤（主にAWS環境）とどう統合するか、という対応がメインになった。

### カスタムドメイン
Route 53で管理し、Auth0のブランディング設定に適用している。
カスタムドメインのサーバ証明書は、Auth0管理がデフォルトであり、技術的にはそれで全く問題はない。

### メール送信プロバイダ
サービス基盤（AWS環境）に合わせるため、今回はAmazon SESを採用。
SPF/DKIM/DMARC、一通り対応済みであり、Auth0用途で使うことになっても追加のセキュリティ設定は不要。

### SMS認証のサポート
SNS APIの実行権限をAuth0 Actionに付与し、`send-phone-message`というトリガーのActionをデプロイしている。
ログインフロー詳細は割愛するが、「条件をチェック」->「MFA要求をリクエスト」->「MFA実行」->「カスタムクレーム設定」、の辺りは少なくとも必要になってくるはず。

ちなみに、SMSの送信文面ではログイン先を明示するため、イベント変数を参照している。
```js
  const template = event.message_options.text;
  const text = `ログイン先サービス: ${event.client.name}
${template}`;
```

### Datadogによる監視
- Auth0ログをDatadogに転送し、アラート通知はDatadogで対応している
- 念のため、Auth0自体のテナント障害は `https://status.auth0.com/rss?domain=${Tenant_Domain}` を直接、Slack通知している
- 「国外から社内ユーザがアクセスしているか」、というイベント検知もしている

### S3によるログ管理
- 監視のためのログはDatadog、長期保存はAmazon S3、という方針をとっている
- "Eventbridge" + "Kinesis Firehose"により、パーティション化した上でS3にAuth0ログを長期保存する
    - 同様のユースケースはAWSでもサンプル公開されていた

https://github.com/aws-samples/amazon-eventbridge-integration-with-auth0


## 運用開始後のTips
### 定常
- Auth0テナント作成は手作業になってしまう
- また、Auth0ダッシュボードのユーザ管理についても、部分的に手運用で対応している
    - Auth0管理ユーザやその権限は、terraform対象外のため
- Auth0ダッシュボードのログインはSSO認証のみ可能にしている
    - 権限はテナント毎に付与できるので、本番の管理者権限は必要最小限に

https://auth0.com/docs/get-started/manage-dashboard-access/configure-single-sign-on-for-auth0-dashboard

- アカウントロックをAuth0ログで検知し、カスタマーサクセス(CS)チームにSlack通知するようになった
    - CS的に把握したい要望があり、DatadogからSlack通知している

### 随時
- 本番環境では、`TF_CLI_ARGS_plan="-parallelism=1"`がほぼ必須になった
    - 指定せずにterraform操作を実行すると、「Auth0 管理APIのグローバルレートリミット超過」の発生が増えてしまったため
    - サービスのアプリ機能でもAuth0管理APIを実行しているので、ユーザ影響を避ける必要がある
