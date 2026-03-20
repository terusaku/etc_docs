---
title: "Auth0ログをDatadog AP1サイトに収集したい場合はEventBridgeを使ってAPI送信する"
emoji: "📡"
type: "tech"
topics: ["aws", "auth0", "eventbridge", "sre"]
published: true
published_at: "2026-03-21 09:00"
---

Auth0のログ収集先としてDatadog AP1サイトを使いたいものの、Auth0のLog Streamの公式連携がAP1に未対応でした。そこで、Auth0 Log StreamをEventBridgeに流し、EventBridgeのAPI DestinationでDatadogログ送信APIへ転送する構成でワークアラウンドしました。この記事では、動機、構成図、Terraformのスニペット、設計上の考慮点、運用効率の観点で得られた効果をまとめます。

## 動機
- Auth0 Log StreamのDatadog連携がAP1に対応していない
- AP1以外へ送るとデータ所在や運用上の整合性が崩れる
- 既存のAuth0 Log StreamはEventBridge連携を提供している

## 構成フロー
```mermaid
flowchart LR
  A[Auth0 Log Stream] -->|Partner Event Bus| B[EventBridge Rule]
  B --> C[EventBridge Target]
  C --> D[API Destination]
  D --> E[Datadog Logs Intake (AP1)]

  subgraph AWS
    B
    C
    D
  end
```

## Terraformスニペット
API Destinationは環境に1つで良いので、最初のユースケースで作成し、以降は出力を使い回します。

```hcl
module "auth0_logs_lecheck_to_datadog" {
  source = "../../modules/eventbridge_auth0_logs_to_datadog"

  rule_name                  = module.event_rule_name_auth0_logs_lecheck.name
  target_role_name           = module.target_role_name_datadog_ap1.name
  event_bus_name             = local.eventbridge.event_bus_name.auth0_lecheck
  connection_arn             = local.eventbridge.datadog.connection_arn_ap1
  invocation_endpoint        = local.eventbridge.datadog.invocation_endpoint_ap1
  api_destination_name       = module.destination_name_datadog_ap1.name
  api_destination_description = "Datadog AP1 log intake"
  maximum_retry_attempts     = local.eventbridge.datadog.maximum_retry_attempts
  tags                       = local.account_tags
}

module "auth0_logs_cloudsign_review_to_datadog" {
  source = "../../modules/eventbridge_auth0_logs_to_datadog"

  rule_name            = module.event_rule_name_auth0_logs_cloudsign_review.name
  target_role_arn      = module.auth0_logs_lecheck_to_datadog.invoke_role_arn
  api_destination_arn  = module.auth0_logs_lecheck_to_datadog.api_destination_arn
  event_bus_name       = local.eventbridge.event_bus_name.auth0_cloudsign_review
  maximum_retry_attempts = local.eventbridge.datadog.maximum_retry_attempts
  tags                 = local.account_tags
}
```

## 設計上の考慮点
- **API Destinationの共有**: 環境内に1つあれば良いので、ユースケース間で使い回す設計にしました。
- **IAMロールの共有**: EventBridgeがAPI Destinationを呼び出すIAMロールも共通化し、重複作成を回避しました。
- **可観測性**: EventBridgeのルール名やAPI Destination名を命名規則で統一し、運用時の発見性を高めました。
- **障害耐性**: 再試行回数やイベント保持時間を明示し、送信失敗時の回復性を確保しました。

## 運用効率性における効果
- **設定の一元化**: 送信先APIやIAMロールの変更が1か所で済み、運用負荷が下がります。
- **再利用性の向上**: 新しいAuth0テナントが追加されても、ルールとターゲットを追加するだけで展開できます。
- **監査性の向上**: 命名規則に沿ったリソース名で、監査・調査時の追跡が容易になります。

## まとめ
Auth0 Log StreamがAP1に対応していない場合でも、EventBridgeとAPI Destinationを使えばDatadog AP1へ安定してログ送信できます。公式対応が追いつくまでの間の現実的なワークアラウンドとして有効です。
