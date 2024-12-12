---
title: "EventBridgeからmarkdown形式でChatbot通知するCDKスタック"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "awscdk", "CICD", "eventbridge", "chatbot"]
published: true
published_at: 2024-12-13 08:00
---

CloudWatch EventsからEventBridgeに名前が変わったのは2019年（さっき調べました）。
イベント駆動のサービスとしてサポートするソース・ターゲットが増えているので、復習ついでに触りました。

## 今回作ったもの
左側に表示されているスタックが今回作った部分です。（右側は機会があれば別途...）
Powered by https://pypi.org/project/aws-pdk/
![](/images/cdk_graph_noFilter.png)


## 任意のイベントをEventBridgeでトリガーする
よくあるユースケースですが、AWSのイベント通知すぐにでも設定できます。
ただ、通知メッセージをmarkdown形式にして見やすくすることを重視して、それをCDKで実装し直してみました。

ちょっとした見やすくする作業は大抵後回しにしがちですが、その分、実用性は高まるのでCDK実装で使い回しが効く形を作っておくと先々で役立ってくれそうです。

今回の場合、[RuleTargetInput](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_events.RuleTargetInput.html)を使うことになり、markdown形式では`static fromObject(obj)`が必要になっています。

https://docs.aws.amazon.com/chatbot/latest/adminguide/custom-notifs.html
https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-transform-input-rule.html
https://docs.aws.amazon.com/ja_jp/chatbot/latest/adminguide/custom-content.html

そのインターフェイスがドキュメント不足で迷いましたが・・出来たものはこちら。（抜粋）
`from_object`の中身は、上記ページあたりが何とか参考になりました。

```python
        # event rule at Console Login
        login_event = events.Rule(
            self, "ConsoleLoginEventRule",
            enabled=True,
            rule_name="ConsoleLogin_ChatbotNotify",
            description="Console Login",
            event_pattern={
                "source": ["aws.signin"],
                "detail": {
                    "eventSource": ["signin.amazonaws.com"],
                    "eventName": ["ConsoleLogin"],
                },
            }
        )
        login_event.add_target(events_targets.SnsTopic(
            chatbot_topic,
            # custom markdown
            message=events.RuleTargetInput.from_object({
                "version": "1.0",
                "source": "custom",
                "content": {
                    "text_type": "client-markdown",
                    "title": f"{events.EventField.from_path('$.detail.eventType')}, {account_id} at {events.EventField.from_path('$.detail.awsRegion')}",
                    "description": f"""
*EventTime:* {events.EventField.from_path('$.detail.eventTime')}
*EventName:* {events.EventField.from_path('$.detail.eventName')}
*UserArn:* {events.EventField.from_path('$.detail.userIdentity.arn')}
*SourceIP:* {events.EventField.from_path('$.detail.sourceIPAddress')}""",
                }
            })
        ))
```

### Slack通知例
![](/images/events_chatbot.png)


## 終わりに
以前から「inputTransformerのパターンをサポートしたL2コンストラクが欲しい」というissueはあるものの、ドキュメントで補足する程度で決着しそうでした。
https://github.com/aws/aws-cdk/issues/11210

2024/12現在、[EventBridge targets](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-targets.html)に記載されたAWSサービスは30個ですが、[AWS services that send events to EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-service-event.html)は数える気が起きないくらい膨大になっています。。

AWSパートナーのSaaSからもイベント受信したり(*1)、Pipes機能でターゲットへのイベント送信時にデータ補完したり、Scheduler機能でAWS APIを直接実行したり(*2)、、ユースケースは大量に用意されているので、それらを効果的に使う実装を考える必要性は高くなったと再認識しました。

(*1)
https://help.okta.com/oie/ja-jp/content/topics/reports/log-streaming/add-aws-eb-log-stream.htm

(*2)
https://aws.amazon.com/jp/blogs/aws/how-vercel-shipped-cron-jobs-in-2-months-using-amazon-eventbridge-scheduler/
