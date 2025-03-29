# AWS CDKを活用したRSSリーダーシステムのデプロイ

*投稿日: 2025年3月29日*

## はじめに

前回までの記事では、AWS公式ブログから最新情報を自動的に取得し、AIを活用して日本語に要約・翻訳するシステムの全体像とAI処理部分について解説しました。今回は、このシステムをAWS Cloud Development Kit (CDK)を使用してデプロイする方法に焦点を当てます。

AWS CDKは、TypeScript、Python、Java、C#などのプログラミング言語を使用してAWSリソースを定義できるフレームワークです。従来のCloudFormationテンプレートと比較して、型安全性、再利用性、モジュール性に優れており、複雑なインフラストラクチャをコードとして効率的に管理できます。

## プロジェクト構成

RSSリーダーシステムのCDKプロジェクトは、以下のような構成になっています：

```
aws_cdk/
├── app.py                      # CDKアプリケーションのエントリーポイント
├── integration_saas/           # SaaS統合関連のスタック
│   ├── __init__.py
│   ├── monitor_stack.py        # モニタリングスタック
│   └── aws_lambda/             # Lambda関数のソースコード
│       └── RssReaderFunction/  # RSSリーダー関数
│           └── index.py        # Lambda関数の実装
├── lib/                        # 共通ライブラリ
│   └── commons.py              # 共通ユーティリティ
└── tests/                      # テストコード
    └── unit/                   # ユニットテスト
```

## CDKスタックの実装

RSSリーダーシステムのCDKスタックは、`monitor_stack.py`ファイルに実装されています。以下に、主要なコンポーネントの実装例を示します：

```python
from aws_cdk import (
    Stack,
    aws_lambda as _lambda,
    aws_lambda_python_alpha as lambda_python,
    aws_events as events,
    aws_events_targets as targets,
    aws_iam as iam,
    Duration,
)
from constructs import Construct

class RssReaderStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)
        
        # Lambda関数の作成
        rss_reader_function = lambda_python.PythonFunction(
            self, "RssReaderFunction",
            entry="integration_saas/aws_lambda/RssReaderFunction",  # Lambda関数のソースコードパス
            index="index.py",                                       # エントリーポイントファイル
            handler="lambda_handler",                               # ハンドラー関数名
            runtime=_lambda.Runtime.PYTHON_3_9,                     # ランタイム
            timeout=Duration.seconds(300),                          # タイムアウト設定
            memory_size=512,                                        # メモリサイズ
            environment={                                           # 環境変数
                "SLACK_BOT_TOKEN": "{{resolve:secretsmanager:SlackBotToken:SecretString:token}}",
                "SLACK_CHANNEL": "#aws-updates",
            },
        )
        
        # Amazon Bedrockへのアクセス権限を付与
        rss_reader_function.add_to_role_policy(
            iam.PolicyStatement(
                actions=[
                    "bedrock:InvokeModel",
                    "bedrock:Converse",
                ],
                resources=["*"],  # 本番環境では特定のモデルARNに制限することを推奨
            )
        )
        
        # EventBridge Schedulerの設定（毎日午前9時に実行）
        rule = events.Rule(
            self, "ScheduleRule",
            schedule=events.Schedule.cron(
                minute="0",
                hour="9",
                month="*",
                week_day="*",
                year="*",
            ),
        )
        
        # ターゲットとしてLambda関数を設定
        rule.add_target(targets.LambdaFunction(rss_reader_function))
```

## 主要なCDKコンポーネント

### 1. Lambda関数の定義

AWS CDKの`PythonFunction`コンストラクトを使用して、Python Lambda関数を定義しています。このコンストラクトは、依存関係の自動パッケージングなど、Python Lambda関数のデプロイを簡素化する機能を提供します。

```python
rss_reader_function = lambda_python.PythonFunction(
    self, "RssReaderFunction",
    entry="integration_saas/aws_lambda/RssReaderFunction",
    index="index.py",
    handler="lambda_handler",
    runtime=_lambda.Runtime.PYTHON_3_9,
    timeout=Duration.seconds(300),
    memory_size=512,
    environment={
        "SLACK_BOT_TOKEN": "{{resolve:secretsmanager:SlackBotToken:SecretString:token}}",
        "SLACK_CHANNEL": "#aws-updates",
    },
)
```

主要なパラメータ：
- `entry`: Lambda関数のソースコードが含まれるディレクトリパス
- `index`: エントリーポイントファイル名
- `handler`: 呼び出されるハンドラー関数名
- `runtime`: Lambda関数のランタイム環境
- `timeout`: 関数の最大実行時間
- `memory_size`: 割り当てられるメモリサイズ
- `environment`: 環境変数（Slack APIトークンなど）

### 2. IAMポリシーの設定

Lambda関数がAmazon Bedrockサービスにアクセスするために必要なIAMポリシーを設定しています：

```python
rss_reader_function.add_to_role_policy(
    iam.PolicyStatement(
        actions=[
            "bedrock:InvokeModel",
            "bedrock:Converse",
        ],
        resources=["*"],  # 本番環境では特定のモデルARNに制限することを推奨
    )
)
```

セキュリティのベストプラクティスとして、本番環境では特定のモデルARNに制限することをお勧めします。

### 3. スケジュール設定

EventBridge Ruleを使用して、Lambda関数を定期的に実行するスケジュールを設定しています：

```python
rule = events.Rule(
    self, "ScheduleRule",
    schedule=events.Schedule.cron(
        minute="0",
        hour="9",
        month="*",
        week_day="*",
        year="*",
    ),
)

rule.add_target(targets.LambdaFunction(rss_reader_function))
```

この例では、毎日午前9時にLambda関数を実行するように設定しています。

## シークレット管理

Slack APIトークンなどの機密情報は、AWS Secrets Managerを使用して安全に管理しています。CDKでは、以下のように環境変数を通じてシークレットを参照できます：

```python
environment={
    "SLACK_BOT_TOKEN": "{{resolve:secretsmanager:SlackBotToken:SecretString:token}}",
    "SLACK_CHANNEL": "#aws-updates",
}
```

この方法により、機密情報をソースコードに直接埋め込むことなく、安全に管理できます。

## CDKデプロイメントプロセス

### 1. CDKプロジェクトの初期化

新しいCDKプロジェクトを作成する場合は、以下のコマンドを実行します：

```bash
# CDKプロジェクトの初期化（Python）
cdk init app --language python

# 仮想環境のアクティベーション
python -m venv .venv
source .venv/bin/activate  # Linuxの場合
.venv\Scripts\activate     # Windowsの場合

# 依存関係のインストール
pip install -r requirements.txt
```

### 2. CDKアプリケーションの構成

`app.py`ファイルにCDKアプリケーションを構成します：

```python
#!/usr/bin/env python3
import os
from aws_cdk import App, Environment
from integration_saas.monitor_stack import RssReaderStack

app = App()

# スタックのデプロイ
RssReaderStack(
    app, "RssReaderStack",
    env=Environment(
        account=os.environ.get("CDK_DEFAULT_ACCOUNT"),
        region=os.environ.get("CDK_DEFAULT_REGION", "ap-northeast-1")
    ),
)

app.synth()
```

### 3. デプロイメント

CDKアプリケーションをデプロイするには、以下のコマンドを実行します：

```bash
# CloudFormationテンプレートの合成
cdk synth

# デプロイ
cdk deploy
```

初めてCDKをデプロイする場合は、ブートストラップが必要です：

```bash
cdk bootstrap
```

## CDKを使用する利点

### 1. インフラストラクチャのバージョン管理

CDKを使用することで、インフラストラクチャの変更をGitなどのバージョン管理システムで追跡できます。これにより、変更履歴の確認や、必要に応じて以前のバージョンへのロールバックが容易になります。

### 2. 再利用可能なコンポーネント

CDKでは、カスタムコンストラクトを作成して再利用することができます。例えば、特定の設定を持つLambda関数やモニタリング設定などを、複数のプロジェクトで再利用できます。

```python
class MonitoredLambda(Construct):
    def __init__(self, scope: Construct, id: str, *, code_path: str, **kwargs):
        super().__init__(scope, id)
        
        # Lambda関数の作成
        self.function = lambda_python.PythonFunction(
            self, "Function",
            entry=code_path,
            **kwargs
        )
        
        # CloudWatchアラームの設定
        alarm = cloudwatch.Alarm(
            self, "ErrorAlarm",
            metric=self.function.metric_errors(),
            threshold=1,
            evaluation_periods=1,
            alarm_description="Lambda function error alarm",
        )
        
        # SNSトピックへの通知設定
        # ...
```

### 3. テスト容易性

CDKでは、インフラストラクチャコードに対するユニットテストやスナップショットテストを実装できます。これにより、デプロイ前に潜在的な問題を検出し、インフラストラクチャの品質を向上させることができます。

```python
def test_rss_reader_stack():
    app = App()
    stack = RssReaderStack(app, "TestStack")
    template = Template.from_stack(stack)
    
    # Lambda関数リソースの検証
    template.has_resource_properties(
        "AWS::Lambda::Function",
        {
            "Handler": "index.lambda_handler",
            "Runtime": "python3.9",
            "Timeout": 300,
        }
    )
    
    # EventBridge Ruleの検証
    template.has_resource_properties(
        "AWS::Events::Rule",
        {
            "ScheduleExpression": "cron(0 9 * * ? *)",
        }
    )
```

## 運用上の考慮事項

### 1. モニタリングとアラート

Lambda関数のエラーや実行時間などを監視するために、CloudWatchアラームを設定することをお勧めします：

```python
# エラー率アラーム
error_alarm = cloudwatch.Alarm(
    self, "RssReaderErrorAlarm",
    metric=rss_reader_function.metric_errors(),
    threshold=1,
    evaluation_periods=1,
    alarm_description="RSS Reader Lambda function error alarm",
)

# 実行時間アラーム
duration_alarm = cloudwatch.Alarm(
    self, "RssReaderDurationAlarm",
    metric=rss_reader_function.metric_duration(),
    threshold=Duration.seconds(250).to_milliseconds(),  # 250秒（タイムアウトの83%）
    evaluation_periods=1,
    alarm_description="RSS Reader Lambda function duration alarm",
)
```

### 2. コスト最適化

Lambda関数のメモリサイズとタイムアウト設定は、パフォーマンスとコストのバランスを考慮して調整することが重要です：

```python
rss_reader_function = lambda_python.PythonFunction(
    self, "RssReaderFunction",
    # ...
    timeout=Duration.seconds(300),  # 5分
    memory_size=512,                # 512MB
    # ...
)
```

メモリサイズを増やすと処理速度が向上しますが、コストも増加します。実際の処理要件に基づいて適切な値を設定してください。

### 3. CI/CDパイプライン

CDKプロジェクトをCI/CDパイプラインと統合することで、コード変更の自動テストとデプロイを実現できます：

```python
# AWS CodePipelineの例
pipeline = codepipeline.Pipeline(
    self, "RssReaderPipeline",
    pipeline_name="RssReaderPipeline",
)

source_output = codepipeline.Artifact()
source_action = codepipeline_actions.CodeStarConnectionsSourceAction(
    action_name="Source",
    connection_arn="arn:aws:codestar-connections:region:account:connection/xxx",
    repository="your-repo",
    branch="main",
    output=source_output,
)

pipeline.add_stage(
    stage_name="Source",
    actions=[source_action],
)

# ビルドステージ
# デプロイステージ
# ...
```

## 結論

AWS CDKを使用することで、RSSリーダーシステムのインフラストラクチャをコードとして管理し、再現性の高いデプロイメントを実現できます。また、バージョン管理、テスト、CI/CD統合などの利点により、システムの信頼性と保守性が向上します。

特に、Lambda関数、IAMポリシー、スケジュール設定などの複雑な構成を、型安全なプログラミング言語で表現できることは、大きなメリットです。これにより、インフラストラクチャのバグを早期に発見し、デプロイメントの品質を向上させることができます。

AWS CDKは比較的新しいツールですが、その柔軟性と生産性の向上により、多くの組織でCloudFormationテンプレートの代替として採用されています。RSSリーダーシステムのような自動化ソリューションでは、CDKの利点を最大限に活用できるでしょう。

---

*この記事は、実際のRSSリーダーシステムのCDK実装に基づいています。具体的な実装は、プロジェクトの要件や組織のベストプラクティスに合わせてカスタマイズすることをお勧めします。*
