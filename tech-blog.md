# AWSブログ自動要約システムの構築

*投稿日: 2025年3月29日*

## はじめに

テクノロジーの進化は日々加速しており、最新の技術動向を追跡することは開発者にとって重要かつ困難な課題です。特にAWSのような急速に進化するクラウドプラットフォームでは、新機能や最適なプラクティスが頻繁に更新されます。

この記事では、AWS公式ブログから最新情報を自動的に取得し、AIを活用して日本語に要約・翻訳するシステムの構築方法について解説します。このシステムにより、英語の技術記事を読む時間がない方でも、AWSの最新動向を効率的に把握することが可能になります。

## システムアーキテクチャ

このシステムは、AWS CDKを使用してインフラストラクチャをコードとして定義し、以下のコンポーネントで構成されています：

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  EventBridge    │───>│  Lambda Function │───>│  Amazon Bedrock │
│  (Scheduler)    │    │  (RSS Reader)    │    │  (Claude 3.5)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                              │                        │
                              │                        │
                              ▼                        ▼
                        ┌─────────────────┐    ┌─────────────────┐
                        │  AWS Blogs      │    │  Slack Channel  │
                        │  (Data Source)  │    │  (Output)       │
                        └─────────────────┘    └─────────────────┘
```

### 主要コンポーネント

1. **EventBridge Scheduler**: 定期的にLambda関数をトリガーするスケジューラー
2. **Lambda Function (RssReaderFunction)**: AWSブログからコンテンツを取得し、AIによる要約・翻訳を実行
3. **Amazon Bedrock**: Claude 3.5 Sonnetを使用した高度な自然言語処理
4. **Slack API**: 要約された情報をSlackチャンネルに投稿

## 実装詳細

### Lambda関数の実装

Lambda関数は、Python 3.xで実装されており、以下の主要な処理を行います：

1. **データ取得**: `requests`ライブラリを使用して、複数のAWS公式ブログからHTMLコンテンツを取得
2. **タイトル抽出**: HTMLパーサーを使用してブログのタイトルを抽出
3. **AI処理**: Amazon Bedrockの`invoke_model` APIを使用して、Claude 3.5 Sonnetに以下の処理を依頼
   - 最新の1週間分の情報を抽出
   - 内容を要約
   - 日本語に翻訳
   - 開発者やプロダクトマネージャーにとって価値のある革新的な情報に焦点を当てる
   - 元の記事に含まれるURLも保持
4. **結果配信**: Slack APIを使用して、要約された情報をSlackチャンネルに投稿

### コードハイライト

```python
def lambda_handler(event, context):
    # 初期化
    model_id = "anthropic.claude-3-5-sonnet-20241022-v2:0"
    inference_id = "apac.anthropic.claude-3-5-sonnet-20241022-v2:0"
    
    # AWSブログURLリスト
    urls = [
        "https://aws.amazon.com/blogs/aws/",
        "https://aws.amazon.com/blogs/architecture/",
        "https://aws.amazon.com/blogs/devops/",
    ]
    
    # 各ブログからコンテンツを取得し処理
    for url in urls:
        feed_content = requests.get(url).text
        feed_title = extract_title(feed_content)
        
        # コンテンツの一部をAIに送信（サイズ制限のため70%に制限）
        trimmed_content = feed_content[: int(len(feed_content) * 0.7)]
        
        # Amazon Bedrockを使用してAI処理
        bedrock_res = genAI_invokeModel(
            model_id=inference_id,
            system_prompt="This is html from AWS Tech Blog. Retrieve last 1 weeks information, summarize and translate in Japanese."
            + "The summary must focus on what is uniqueness and innovation, which attract web developers or product managers. If original information contain hyperlinks, answer with the url.",
            user_msg=trimmed_content,
        )
        
        # Slackに投稿
        post_slack_message(
            post_msg=f"{feed_title} \n {bedrock_res['content'][0]['text']}",
            post_attrs={"mrkdwn": True},
        )
```

## 技術的な課題と解決策

### 1. HTMLコンテンツの処理

**課題**: ブログのHTMLは構造が複雑で、関連情報の抽出が困難

**解決策**: HTMLパーサーを使用してタイトルを抽出し、AIの高度な言語理解能力を活用して関連コンテンツを識別。AIに生のHTMLを送信することで、DOM構造を解析する複雑なロジックを実装する必要がなくなりました。

### 2. トークン制限の対応

**課題**: LLMには入力トークン数の制限があり、ブログの全コンテンツを送信できない

**解決策**: コンテンツの先頭70%のみを送信し、最新情報が含まれる可能性が高い部分に焦点を当てる方法を採用。

```python
# コンテンツの一部をAIに送信（サイズ制限のため70%に制限）
trimmed_content = feed_content[: int(len(feed_content) * 0.7)]
```

### 3. 要約の品質向上

**課題**: 技術的な内容を正確かつ有用に要約する必要がある

**解決策**: システムプロンプトを詳細に設計し、以下の点を指定：
- 最新の1週間分の情報に焦点を当てる
- 開発者やプロダクトマネージャーにとって価値のある革新的な情報を優先
- 元のURLを保持して詳細情報へのアクセスを容易にする

## デプロイメント

このシステムはAWS CDKを使用してデプロイされています。CDKを使用することで、インフラストラクチャをコードとして管理し、バージョン管理やCI/CDパイプラインとの統合が容易になります。

主要なCDKコンポーネント：

1. Lambda関数の定義
2. IAMロールとポリシーの設定
3. EventBridge Schedulerの構成
4. 環境変数を通じたシークレット管理（Slack APIトークンなど）

## 今後の拡張計画

このシステムは以下の方向で拡張可能です：

1. **対象ブログの拡大**: より多くの技術ブログソースを追加
2. **カテゴリ分類**: AIを使用して記事をカテゴリ別に分類し、関心領域に基づいたフィルタリングを実現
3. **パーソナライズ**: ユーザーの興味に基づいて要約内容をカスタマイズ
4. **マルチモーダル対応**: 画像やダイアグラムの説明も含めた要約の提供
5. **履歴管理**: 過去の要約を検索可能なデータベースに保存

## 結論

このRSSリーダーシステムは、最新のAWS技術情報を効率的に追跡し、言語の壁を越えて日本語で提供することを可能にします。AWS CDK、Lambda、Amazon Bedrockなどのサーバーレステクノロジーを組み合わせることで、メンテナンスコストを最小限に抑えながら、高い価値を提供するシステムを構築できました。

このようなシステムは、技術情報の追跡だけでなく、様々な分野での情報収集と要約に応用可能です。AIの言語処理能力とクラウドのサーバーレスアーキテクチャを組み合わせることで、情報過多の時代における効率的な知識獲得を支援するツールとなります。

---

*このブログ記事は、実際のRSSリーダーシステムの実装に基づいています。コードやアーキテクチャは、特定の要件や環境に合わせてカスタマイズすることをお勧めします。*
