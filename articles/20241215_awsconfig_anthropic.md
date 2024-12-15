---
title: "AIOpsでAWS Configの変更履歴を検索する"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aiops", "anthropic", "awsconfig", "athena"]
published: true
publishedAt: "2024-12-15 20:30"
---


AWS Configログは、監査などの内部統制や変更履歴の証跡として重要なんですが、もう少し能動的に使えないかと生成AIに手伝ってもらいました。

## つくったもの
AWS Lambdaの関数URLにAWS Configの対象年月をPOSTすると、変更履歴をサマリとして応答します。
現状、検索はAthenaクエリなので、検索クエリを動的に組み立てるにはFunction Callingのような機能が必要なんですが、今後の課題として今回は未着手です。

```sh
# target_year, target_monthを指定してPOST
$ curl -X POST ${url} \
-H "Content-Type: application/json" \
-d '{"target_year": "2024", "target_month": "11"}' | jq -r '.body[].text'
AWS リソースの変更履歴を分析した結果、以下の特徴的な変更が確認されました：

2024年11月の主な変更点：

1. Cassandra Keyspaceの変更
- 4つのKeyspaceリソース（system, system_multiregion_info, system_schema, system_schema_mcs）がそれぞれ2回の設定変更を記録

2. IAMロールの作成/変更
- AWSServiceRoleForCostOptimizationHub という新しいサービスロールが1回の変更を記録

3. Lambda関数の変更
- 合計32個のLambda関数で設定変更を確認
- 主な用途：
  - コスト管理関連（Athena_Cost-Query, CurReport など）
  - ChatGPT連携（chatgpt-api-llm など）
  - Slack/LINE Bot連携
  - モニタリング関連

4. S3バケットの変更
- コストエクスポート用のバケット（.....）で1回の変更を記録

特筆すべき点：
- すべての変更は ap-northeast-1 リージョンで発生（IAMロールを除く）
- 特にLambda関数の変更が多く、サーバーレスアーキテクチャの更新が活発
- コスト最適化関連のリソース変更が複数確認され、コスト管理の取り組みが進められている様子
```

## 仕組み
1. AWS Configのログ保存先をS3バケットして、一つのAWSアカウントに集約する
2. そのS3バケットをソースとして、Glueテーブルを作成する
3. そのテーブルに対してAthenaからクエリを実行、変更履歴をjsonとして取得する
4. そのjsonを全て[Anthropicのmessages API](https://docs.anthropic.com/en/api/messages)に送信し、回答を生成する
    - クエリ検索の結果次第だが、jsonはそのまま`message.content`に含める方針
    - systemメッセージでは下記で固定して、データ構造について補足している
```
Here is AWS Athena query result from AWS Config logs.
there are resourceType, resourceName, awsAccountId, awsRegion, year, month, recordCount.
```
テーブル定義ができれば、特に難点はないので、少し詳しく書いておきます。

## Glueテーブルはパーティション射影を使う
パーティションのことを考慮してS3バケットにデータを保存していれば、パーティション射影は問題なく使えるはずなので、あとは設定するだけです。
Athenaにパーティションは重要ですがメンテナンスしたくないので、パーティション射影を使えるようにテーブル定義した方がお得です。

Tipsはクラメソさんの記事が詳しいので、この記事ではCDK実装例だけ紹介します。
L2コンストラクタがないので面倒でした。。誰かの参考になれば幸いです。
```python
class AwsConfigGlueStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        accountId = Aws.ACCOUNT_ID
        awsconfig_db_name = 'lake'
        awsconfig_table = 'awsconfig'
        awsconfig_bucket = 'orgs-.....'

        AwsConfigTable(self, "CreateGlueTable-AwsConfig", {
            'catalog_id': accountId,
            'db_name': awsconfig_db_name,
            'table_name': awsconfig_table,
            'location_bucket': awsconfig_bucket,

        })
```

パーティション射影は`storage.location.template`にパターンを定義して、`projection.{partition_key}.*`でパーティションの型を決めます。なお、`partition_keys`では全て`string`である必要があり、そこはパーティション射影の制約になってます。

:::details CDKによるGlueテーブル定義例：AwsConfigTable

```python
class AwsConfigTable(Construct):
    def __init__(self, scope: Construct, construct_id: str, glue_config: GlueConfig):
        super().__init__(scope, construct_id)

        accountId = glue_config['catalog_id']
        awsconfig_db_name = glue_config['db_name']
        awsconfig_table = glue_config['table_name']
        awsconfig_bucket = glue_config['location_bucket']

        glue.CfnTable(
            self, "AwsConfigTable",
            catalog_id=accountId,
            database_name=awsconfig_db_name,
            table_input=glue.CfnTable.TableInputProperty(
                description="AWS Config Table",
                name=awsconfig_table,
                parameters={
                    "classification": "json",
                    "compressionType": "gzip",
                    "storage.location.template": f"s3://{awsconfig_bucket}/AWSLogs/${{account_id}}/Config/${{region}}/${{year}}/${{month}}/${{day}}/",
                    "projection.enabled": "true",
                    "projection.account_id.type": "enum",
                    "projection.account_id.values": "123456789012", # 複数アカウントはカンマ区切りで指定可能だが、マルチアカウント構成の場合はパーティションをはずした方が良いかも
                    "projection.region.type": "enum",
                    "projection.region.values": "us-east-1,ap-northeast-1",
                    "projection.year.type": "integer",
                    "projection.year.range": "2021, 2999",
                    "projection.year.interval": "1",
                    "projection.month.type": "integer",
                    "projection.month.range": "1, 12",
                    "projection.month.interval": "1",
                    "projection.day.type": "integer",
                    "projection.day.range": "1, 31",
                    "projection.day.interval": "1",
                },
                partition_keys=[
                    glue.CfnTable.ColumnProperty(name='account_id',type='string'),
                    glue.CfnTable.ColumnProperty(name='region',type='string'),
                    glue.CfnTable.ColumnProperty(name='year',type='string'),
                    glue.CfnTable.ColumnProperty(name='month',type='string'),
                    glue.CfnTable.ColumnProperty(name='day',type='string'),
                ],
                storage_descriptor=glue.CfnTable.StorageDescriptorProperty(
                    columns=[
                        glue.CfnTable.ColumnProperty(name="fileversion", type="string"),
                        glue.CfnTable.ColumnProperty(
                            name="configurationitems",
                            # 見やすく改行を入れると正しく認識されないので1行で記述する
                            type="array<struct<configurationItemVersion:string,configurationItemCaptureTime:string,configurationStateId:bigint,awsAccountId:string,awsRegion:string,configurationItemStatus:string,resourceType:string,resourceId:string,resourceName:string,ARN:string,availabilityZone:string,configurationStateMd5Hash:string,resourceCreationTime:string,tags:map<string,string>>>",
                        ),
                    ],
                    location=f"s3://{awsconfig_bucket}/AWSLogs",
                    input_format="org.apache.hadoop.mapred.TextInputFormat",
                    output_format="org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
                    compressed=False,
                    serde_info=glue.CfnTable.SerdeInfoProperty(
                        serialization_library="org.apache.hive.hcatalog.data.JsonSerDe",
                        parameters={"serialization.format": "1"},
                    ),
                    stored_as_sub_directories=False,
                )
            )
        )
```
:::

## Claude Sonnetとの応答
自分は使う分だけ都度購入する方が合っているので、Anthropicを使っています。
アプリで使うなら時間あたりの合計トークン数をスロットリングできる仕様が必要そうですが、今回は`1000`で決めうちです。
https://docs.anthropic.com/ja/api/rate-limits


:::details Athenaのクエリ検索結果を渡して、変更履歴の特徴を生成する

```python
class TextBlock:
    def __init__(self, text, type):
        self.text = text
        self.type = type

    def to_dict(self):
        return {
            'text': self.text,
            'type': self.type
        }


def lambda_handler(event, context):
    ## 中略 ##

    # Athenaクエリ結果を取得
    result = client.get_query_results(QueryExecutionId=query_execution_id)

    # 結果をリストに変換
    rows = []
    for row in result['ResultSet']['Rows'][1:]:  # ヘッダー行をスキップ
        rows.append([col.get('VarCharValue', '') for col in row['Data']])

    claude_client = anthropic.Anthropic(
        # defaults to os.environ.get("ANTHROPIC_API_KEY")
        api_key=os.getenv('anthropic_api_key')
    )
    message = claude_client.messages.create(
        model="claude-3-5-sonnet-20241022",
        # 2024/12現在、1分あたり最大4万トークンまで
        max_tokens=1000,
        temperature=0,
        system='''
        Here is AWS Athena query result from AWS Config logs.
        there are resourceType, resourceName, awsAccountId, awsRegion, year, month, recordCount.
        ''',
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": f"""Show me annormal change history of AWS Resources by month, and tell me in Japanese. 
                        ```json
                        {rows}
                        ```
                        """
                    }
                ]
            }
        ]
    )
    
    anthropic_res: List[TextBlock] = message.content    

    res_body = {
        'status': 'SUCCEEDED',
        'body': [txt.to_dict() for txt in anthropic_res]
    }

    return {
        'statusCode': 200,
        'body': json.dumps(res_body, ensure_ascii=False)
    }
```
:::


この結果、冒頭のような回答が得られる様になります。
クエリ検索の結果レコードがない場合、`return`終了しても良いですが、このままにすると次のよう回答になります。
```markdown
提供されたデータが空（[]）のため、AWS リソースの異常な変更履歴を分析することができません。

有効なデータを提供していただければ、以下のような分析が可能です：
- 月ごとのリソース変更数の急激な増減
- 通常とは異なるリソースタイプの変更
- 特定のリージョンでの異常な活動
- アカウント間での異常な変更パターン
```

## 終わりに
これがAIOpsにつながるかは未定ですが、人が見るには辛い構成情報を生成AI経由で見ることは効率が良く、定点観測としての分析精度も悪くないと感じています。

バックエンドにAthenaを使っていてデータの品質は担保されている点も、コスパが良かったです。あとは検索クエリを動的に組み立てる機能があれば、ユースケースも広げられそうです。

余談としては、お金は多少かかりますが、エラーメッセージを生成AIに任せるのも良いな、と少し考えさせられました。

生成AIをアプリ実装する場合、入出力データは定義されているので、それなりに有用なエラーメッセージを生成してくれる気がしています。
