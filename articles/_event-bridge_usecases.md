---
title: "AWSのEventBridgeがサポートするサービスは多いので嬉しくて辛い"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "awscdk", "CICD"]
published: false
---

CloudWatch EventsからEventBridgeに名前が変わったのは2019年（さっき調べました）。
EventBridgeはイベント駆動の要ですね。

そんなEventBridgeをaws-cdkで触ろうとすると面倒すぎたので備忘録です。
CDK管理で抽象化するにしても、ターゲット毎にプロパティが異なるので初期調査は欠かせず、、


## 任意のイベントをEventBridgeからSlackに通知する
EventBridgeのターゲットはSNSで、ChatbotのSlack統合から通知されることになりますが、通知メッセージはmarkdownで記述しておきたいところ。

その場合は[RuleTargetInput](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_events.RuleTargetInput.html)を


## 終わりに
「inputTransformerのパターンをサポートしたL2コンストラクが欲しい」というissueはあるものの、ドキュメントで補足する程度で決着しそうでした。
https://github.com/aws/aws-cdk/issues/11210

自身でよく使ってきた組み合わせはスケジュール実行(LambdaやECSタスク)とSlack通知(SNS + ChatbotのSlack統合)くらいですが、イベントソースとターゲットの量は膨大です。

2024/07現在、[EventBridge targets](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-targets.html)に記載されたAWSサービスは30個ですが、[AWS services that generate events](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-service-event-list.html)は数える気が起きませんでした。

また、AWSパートナーのSaaSからもイベント受信したり、Pipes機能でターゲットへのイベント送信時にデータ補完したり、Scheduler機能でAWS APIを直接実行したり、、ユースケースは広がるばかり。

何をどうしたいか、曖昧なまま使い始めるにはAWSは向かないなーと再認識させられることは多いです。
