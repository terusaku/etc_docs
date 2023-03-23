---
title: "Synthetics CanaryをAWS CDKで構築してみる"
# emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "awscdk", "IaC" ,"CloudWatchSynthetics"]
published: true
published_at: 2023-03-23 19:00
---

AWS CDKにも慣れてきたため、色々なリソースで練習中。
CloudWatch Synthetics Canaryを`cdk deploy`するまでの備忘メモ。


## スタックの抽象化

1つのスタックにまとめつつ、Constructを使って抽象化するとこうなりました。
```bash
app.py
├── __init__.py
├── alarm.py
├── canary.py
├── cw_syn_stack.py
└── src
    └── index.py
```

webdriverによるテストシナリオは`src/index.py`で記述していますが、今回は省略します。


### cw_syn_stack.py
設定値で使いたい変数はStackクラスでまとめ、Constructクラスに引数として渡しています。

```python
import os
import boto3
import aws_cdk as cdk
from aws_cdk import (
    Stack,
)
from constructs import Construct
from .canary import Canary
from .alarm import Alarm


class CwSynStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        account_id = boto3.client('sts').get_caller_identity().get('Account')
        output_bucket = f"cw-syn-results-{account_id}-ap-northeast-1"
        
        env_var = {
            'Env': os.getenv('Env', 'stg'),
            #####
        }
        if env_var['Env'] == 'prod':
            retention = 60
            schedule = 'rate(10 minutes)'
            alarm_period = 10 * 60
        else:
            retention = 15
            schedule = 'rate(30 minutes)'
            alarm_period = 30 * 60

        config = {
            'output_bucket': output_bucket,
            'retention': retention,
            'canary_schedule': schedule,
            'alarm_period': alarm_period,
        }

        canary = Canary(self, "canary", env_var, config)
        alarm = Alarm(self, "alarm", canary, config)
```

### canary.py

- `env_var`を環境変数に設定
- 後続のアラーム設定でcanary名称を参照したいため、値オブジェクト`canary_name`を定義
- 使う予定はないがCFnテンプレートのOutput同様のことをできるか、`CfnOutput`を使ってみる
    - できました

```python
import aws_cdk as cdk
from aws_cdk import (
    aws_synthetics as synthetics,
    aws_iam as iam,
    aws_s3 as s3,
)
from constructs import Construct


class Canary(Construct):
    @property
    def canary_name(self):
        return self._canary.name

    def __init__(self, scope: Construct, id: str, env_var: dict, config: dict, **kwargs):
        super().__init__(scope, id, **kwargs)

        canary_name = '.....'

        # Synthetics Canary Code
        with open('./cw_syn/src/index.py', 'r') as src:
            src_code = src.read()

        result_bucket = s3.CfnBucket(self, "outputBucket",
            bucket_name=config['output_bucket'],
            # versioning_configuration=cdk.CfnBucket.VersioningConfigurationProperty(
            #     status="Enabled"
            # ),
            tags=[cdk.CfnTag(
                key="Key",
                value="Value"
            )],
        )

        # IAM Role for CloudWatch Synthetics
        syn_role = iam.CfnRole(self, "synthRole",
            role_name=f"cw-synth-execution-{canary_name}",
            assume_role_policy_document={
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "lambda.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    },
                ],
            },
            description="AWS Synthetics execution Role",
            policies=[
                iam.CfnRole.PolicyProperty(
                    policy_name="syntheticsCanary-basic",
                    policy_document={
                        "Version": "2012-10-17",
                        "Statement": [
                            {
                                "Effect": "Allow",
                                "Action": [
                                    "logs:CreateLogKey",
                                    "logs:CreateLogStream",
                                    "logs:PutLogEvents",
                                ],
                                "Resource": "arn:aws:logs:*:*:*"
                            },
                            {
                                "Effect": "Allow",
                                "Action": [
                                    "s3:List*",
                                ],
                                "Resource": "arn:aws:s3:::*"
                            },
                            {
                                "Effect": "Allow",
                                "Action": [
                                    "s3:PutObject",
                                    "s3:GetBucketLocation",
                                ],
                                "Resource": [
                                    f"arn:aws:s3:::{result_bucket.bucket_name}",
                                    f"arn:aws:s3:::{result_bucket.bucket_name}/*",
                                ]
                            },
                            {
                                "Effect": "Allow",
                                "Action": [
                                    "cloudwatch:PutMetricData",
                                ],
                                "Resource": "*"
                            },
                        ]
                    },
                )
            ],
            tags=[cdk.CfnTag(
                key="Key",
                value="Value"
            )],
        )

        self._canary = synthetics.CfnCanary(self, "Canary",
            artifact_s3_location=f"s3://{result_bucket.bucket_name}",
            code=synthetics.CfnCanary.CodeProperty(
                handler="index.handler",
                # s3_bucket=sourcecode_bucket,
                # s3_key=f"{canary_name}/index.py",
                script=src_code,
            ),
            execution_role_arn=syn_role.attr_arn,
            failure_retention_period=config['retention'], # unit: day
            name=canary_name,
            runtime_version="syn-python-selenium-1.3",
            run_config=synthetics.CfnCanary.RunConfigProperty(
                active_tracing=False, # syn-python-selenium-1.3 does not support Active tracing
                memory_in_mb=960,
                timeout_in_seconds=120,
                environment_variables=env_var,
            ),
            schedule=synthetics.CfnCanary.ScheduleProperty(
                duration_in_seconds="120",
                expression=config['canary_schedule'],
            ),
            start_canary_after_creation=True,
            success_retention_period=15,
            tags=[cdk.CfnTag(
                key="Key",
                value="Value"
            )],
        )

        cdk.CfnOutput(self, "cw-syn-resultBucket",
            value=result_bucket.bucket_name,
            description="S3 Bucket for Synthetics Canary Result",
        )
```

### alarm.py
- アラーム設定は対象メトリクスによってパラメータが変わるため、リソース数（メトリクス種類 × 監視対象）が増えても抽象化は難しめ
    - 対象メトリクスごとにクラス実装しても良いかもしれない
- 下記コードではSNSトピックまでは作成してサブスクリプションは省略している
    - 例えば、AWS Chatbotで通知したい場合はChatbotスタックでSNSトピックを参照する必要あり

```python
import aws_cdk as cdk
from aws_cdk import (
    aws_cloudwatch as cloudwatch,
    aws_sns as sns,
)
from constructs import Construct


class Alarm(Construct):

    def __init__(self, scope: Construct, id: str, canary, config: dict, **kwargs):
        super().__init__(scope, id, **kwargs)

        # SNS Topic for CloudWatch Alarm
        sns_topic = sns.CfnTopic(self, "synthAlarmTopic",
            display_name=f"Synthetics Canary Alarm on {canary.canary_name}",
            topic_name=f"synthetics-{canary.canary_name}",
            tags=[cdk.CfnTag(
                key="key",
                value="value"
            )],
        )
        
        alarm_metric = 'Duration'
        # CloudWatch Alarm for Synthetics Canary
        cfn_alarm = cloudwatch.CfnAlarm(self, 'CanaryAlarm',
            actions_enabled=False,
            namespace='CloudWatchSynthetics',
            metric_name=alarm_metric,
            comparison_operator='GreaterThanOrEqualToThreshold',
            evaluation_periods=2,
            dimensions=[cloudwatch.CfnAlarm.DimensionProperty(
                name='CanaryName',
                value=canary.canary_name
            )],
            alarm_actions=[sns_topic.attr_topic_arn],
            alarm_description=f"Alarm for Synthetics Canary",
            alarm_name=f"{alarm_metric}-{canary.canary_name}",
            ok_actions=[sns_topic.attr_topic_arn],
            period=config['alarm_period'], # unit: second
            statistic='Average',
            threshold=40 * 1000,
            unit='Milliseconds',
            treat_missing_data='ignore',
        )
```

## 途中で発生したエラー

### Invalid bucket name "...-${Token[AWS.AccountId.7]}": Bucket name must match the regex ..."

https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/tokens.html

CDK仕様とS3サービス仕様が競合している例。
トークンはデプロイ実行後に展開されるため、`cdk deploy`のビルドプロセスでは上記エラーが発生してしまう。今回はboto3の`get_caller_identity()`を使ってスルーした。


### Resource handler returned message: "Resource of type 'AWS::...' with identifier '...' already exists.

CFnテンプレートを変更＆デプロイしていると、よくあるエラー。
CDKでも一度デプロイ完了後にリファクタリングして`Construct`クラスを導入したりすると発生する。

https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/identifiers.html

エラー発生時は異なる論理IDによる同名リソースのデプロイとして認識されているため、リソース名称を1文字でも変更すればエラーは解消する。


##　最後に
`cdk deploy`のプロセスではCloudFormationテンプレートを生成しているため、型チェックや命名規則などの巨大なバリデーションが最初に実行されています。そのためか、デプロイが実際に動き始めるとコンパイルが成功したときのような安心感を得られるので、プログラマフレンドリーなIaCツールなのは確かです。

次はCDKのバリデーション実装で使い勝手の良いものを試していきたいです。
