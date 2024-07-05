---
title: "AWSのサービス仕様が少し物足りずにaws-cdkで作り込んだこと"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "aws-cdk", "CI/CD"]
published: true
published_at: 2024-06-24 9:00
---


## FireLens for Amazon ECS
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/firelens-taskdef.html

正直、ドキュメントを読んでも何をどのように設定すれば良いのか分かりずらい...
そのため任意のタスク定義を指定してFireLens設定を追加するコンストラクタを作成した。

aws-cdkを使って良いことの一つは「誰か1人が苦労した仕様もコーディング技術で真似が簡単になる」ということ。

ECSタスク定義はすでに存在している前提として、

1. firelensコンテナ(fluent-bit)の作成
- 任意のfluent-bit設定ファイルを含むコンテナをビルドしてECRにプッシュする
- 今回はS3バケットとDatadog、両方に同じログを送信する方針とする


2. firelensコンテナを既存のタスク定義に追加する
- `new ecs.FirelensLogRouter`を実行するだけではあるが、、ここが一番分かりづらい仕様だった
- コンテナ設定一般のほか、`FirelensConfig`というインターフェイスに沿って設定する必要がある
- また、コンテナ環境変数に`DD_*`を渡してfluent-bit設定で変数として扱えるようにする


3. firelensコンテナに必要なIAM権限を定義する
- ログ送信先のS3バケットへのアクセス権限
- SSMパラメータストアの参照
- SecretsManagerの参照





```ts
export function addFirelensLogRouter(
  scope: Construct,
  appContext: CommonContext,
  serviceName: string,
  taskDef: ecs.TaskDefinition,
  appServerContext: ApplicationServerContext,
  datadogApiKey: ecs.Secret,
  projectName: string
): void {}
```
