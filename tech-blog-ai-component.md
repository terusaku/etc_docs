# Amazon Bedrockを活用した技術ブログの自動要約・翻訳システム

*投稿日: 2025年3月29日*

## はじめに

前回の記事では、AWS公式ブログから最新情報を自動的に取得し、日本語に要約・翻訳するシステムの全体像について解説しました。今回は、このシステムの中核となるAI処理部分に焦点を当て、Amazon BedrockとClaude 3.5 Sonnetを活用した自然言語処理の実装詳細について掘り下げていきます。

## Amazon Bedrockとは

Amazon Bedrockは、AWSが提供する生成AIサービスで、複数のファウンデーションモデル（FM）に統一されたAPIでアクセスできるマネージドサービスです。開発者は複雑なMLインフラストラクチャを管理することなく、高性能な生成AIモデルを利用できます。

主な特徴：
- Anthropic、AI21 Labs、Cohere、Meta、Stability AI、Amazonなど複数のプロバイダーのモデルにアクセス可能
- プライベートなデータと接続し、企業固有のユースケースに対応
- エンタープライズグレードのセキュリティとプライバシー
- サーバーレスで使いやすいAPI

## Claude 3.5 Sonnetの活用

本システムでは、AnthropicのClaude 3.5 Sonnetモデルを使用しています。このモデルを選択した理由は以下の通りです：

1. **高度な言語理解能力**: 複雑なHTMLコンテンツから関連情報を抽出する能力
2. **多言語対応**: 英語から日本語への高品質な翻訳能力
3. **長いコンテキスト処理**: ブログ記事の長文を処理できる十分なコンテキストウィンドウ
4. **要約能力**: 重要な情報を抽出し、簡潔に要約する能力

## Amazon Bedrock APIの実装

### モデル呼び出しの基本実装

Amazon Bedrockには主に2つのAPI呼び出し方法があります：`converse` APIと`invoke_model` APIです。本システムでは、より柔軟な制御が可能な`invoke_model` APIを採用しています。

```python
def genAI_invokeModel(
    model_id: str,
    system_prompt: str,
    user_msg: str,
    temperature: float = 0.7,
    top_k: int = 200,
) -> Dict[str, Any]:
    bedrock_client = boto3.client("bedrock-runtime")

    # リクエストボディの構築
    messages = {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": user_msg,
            }
        ],
    }

    request_body = {
        "anthropic_version": "bedrock-2023-05-31",
        "temperature": temperature,
        "top_k": top_k,
        "max_tokens": 10000,
        "system": system_prompt,
        "messages": [
            messages,
        ],
    }

    try:
        # モデル呼び出し
        response = bedrock_client.invoke_model(
            modelId=model_id, body=json.dumps(request_body)
        )

        # レスポンス解析
        response_body = json.loads(response["body"].read())

        # トークン使用量のログ記録
        if "usage" in response_body:
            usage = response_body["usage"]
            logger.info(f"Input Tokens: {usage.get('input_tokens', 'N/A')}")
            logger.info(f"Output Tokens: {usage.get('output_tokens', 'N/A')}")

        return response_body

    except Exception as e:
        logger.error(f"Failed to call Bedrock API: {e}")
        raise
```

### プロンプトエンジニアリング

AIモデルから高品質な要約・翻訳を得るためには、適切なプロンプト設計が不可欠です。本システムでは以下のようなシステムプロンプトを使用しています：

```python
system_prompt = "This is html from AWS Tech Blog. Retrieve last 1 weeks information, summarize and translate in Japanese." + "The summary must focus on what is uniqueness and innovation, which attract web developers or product managers. If original information contain hyperlinks, answer with the url."
```

このプロンプトには以下の重要な要素が含まれています：

1. **入力データの性質**: "This is html from AWS Tech Blog"
2. **時間的範囲**: "Retrieve last 1 weeks information"
3. **タスク定義**: "summarize and translate in Japanese"
4. **焦点**: "focus on what is uniqueness and innovation"
5. **ターゲットオーディエンス**: "which attract web developers or product managers"
6. **追加情報の保持**: "If original information contain hyperlinks, answer with the url"

これらの指示により、AIは単なる要約ではなく、特定のオーディエンスにとって価値のある情報に焦点を当てた、目的に合った出力を生成します。

## 技術的課題と最適化

### 1. トークン制限への対応

Claude 3.5 Sonnetは大きなコンテキストウィンドウを持ちますが、それでも入力トークン数には制限があります。ブログのHTMLコンテンツ全体を送信すると、この制限を超える可能性があります。

対応策として、コンテンツの先頭70%のみを送信する方法を採用しています：

```python
# コンテンツの一部をAIに送信（サイズ制限のため70%に制限）
trimmed_content = feed_content[: int(len(feed_content) * 0.7)]
```

この方法は、以下の仮定に基づいています：
- ブログ記事の重要な情報は通常、前半部分に集中している
- 最新の記事は通常、ブログの先頭に表示される

### 2. コスト最適化

AIモデルの利用にはコストがかかるため、以下の最適化を行っています：

1. **必要最小限のコンテンツ送信**: 前述のトリミング処理によりトークン数を削減
2. **適切なモデル選択**: タスクの複雑さに見合ったモデルを選択（Claude 3.5 Sonnet）
3. **バッチ処理**: 定期的なスケジュールでまとめて処理することでAPI呼び出し回数を最小化

### 3. エラーハンドリング

API呼び出しの失敗や予期せぬレスポンスに対応するため、適切なエラーハンドリングを実装しています：

```python
try:
    response = bedrock_client.invoke_model(
        modelId=model_id, body=json.dumps(request_body)
    )
    # レスポンス処理
except Exception as e:
    logger.error(f"Failed to call Bedrock API: {e}")
    raise
```

また、Slackへの投稿処理にも同様のエラーハンドリングを実装し、システムの堅牢性を確保しています。

## パフォーマンス評価

### 要約・翻訳の品質

Claude 3.5 Sonnetによる要約・翻訳の品質は非常に高く、以下の点で優れています：

1. **技術的正確性**: 専門用語や技術概念を正確に理解し翻訳
2. **自然な日本語**: 機械翻訳特有の不自然さが少ない流暢な日本語
3. **重要情報の抽出**: 記事の本質的な価値や革新性を適切に抽出
4. **構造化された出力**: 読みやすく整理された形式での情報提供

### 処理時間とコスト

Lambda関数の実行時間は、処理するブログの数と内容によって変動しますが、典型的なケースでは以下のパフォーマンスを示しています：

- **平均処理時間**: 1ブログあたり約5〜10秒
- **Lambda実行時間**: 全体で約30秒（3つのブログを処理）
- **推定コスト**: 
  - Lambda実行: 月あたり約$0.10（日次実行の場合）
  - Bedrock API: 月あたり約$5〜10（日次実行、3ブログ処理の場合）

これらのコストは、手動で同等の作業を行う人的コストと比較すると非常に低いと言えます。

## 今後の改善点

### 1. マルチモーダル対応

現在のシステムはテキストのみを処理していますが、ブログには画像やダイアグラム、動画などのマルチメディアコンテンツも含まれています。Claude 3.5のマルチモーダル機能を活用することで、これらの視覚的情報も含めた包括的な要約が可能になります。

```python
# マルチモーダル対応の例
messages = {
    "role": "user",
    "content": [
        {
            "type": "text",
            "text": "以下のブログ記事とその中の画像を要約してください"
        },
        {
            "type": "image",
            "source": {
                "type": "base64",
                "media_type": "image/jpeg",
                "data": base64_encoded_image
            }
        },
        {
            "type": "text",
            "text": blog_text
        }
    ],
}
```

### 2. ファインチューニング

特定のドメインや形式に特化した要約を生成するために、カスタムモデルのファインチューニングも検討できます。Amazon Bedrockは、独自データでのモデルカスタマイズをサポートしています。

### 3. フィードバックループの実装

要約の品質を継続的に向上させるため、ユーザーからのフィードバックを収集し、プロンプトや処理ロジックを改善するフィードバックループの実装も有効です。

## 結論

Amazon BedrockとClaude 3.5 Sonnetを活用することで、高品質な技術ブログの自動要約・翻訳システムを比較的少ない実装コストで実現できました。このようなAIを活用したシステムは、情報過多の時代において、必要な情報を効率的に抽出し、言語の壁を越えて提供する強力なツールとなります。

特に注目すべきは、HTMLのような構造化されていないデータからも関連情報を抽出できる点です。従来のルールベースのアプローチでは困難だった柔軟な情報抽出が、最新のLLMによって可能になっています。

今後、生成AIの能力はさらに向上し、コストも低減していくことが予想されます。このようなAIを活用した自動化システムの構築スキルは、エンジニアにとって重要な競争力となるでしょう。

---

*この記事は、実際のRSSリーダーシステムのAI処理部分の実装に基づいています。Amazon Bedrockの利用方法や最適なプロンプト設計は、特定の要件や使用するモデルによって異なる場合があります。*
