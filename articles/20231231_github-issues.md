---
title: "GitHub IssuesのOSS振り返り"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "github"]
published: true
published_at: 2023-12-31 18:00
---

この一年、OSS活動としていきなりPRは敷居が高いので、まずはGitHub Issuesにコメントすることを始めていました。単純に自身でOSSを使っているだけでも実装の壁やエラーにぶつかるため、その解決策をコミュニティに聞きたいというのがそもそもの動機。

改めて振り返ると、同じ目的に対して他人と気軽にコミュニケーションできることはコミュニティの基本的な価値だなーという所感です。

## loaclstackを使ってcdkを実行できない
https://github.com/localstack/aws-cdk-local/issues/76
cdkを本格的に実装してきた頃、細かい部分でAWSにデプロイすることが嫌だったので「localstackを使えばいいのでは」と思いつきで始めたら最初でつまづきました。

原因はNodejsバージョンによる仕様変更でIPv6がIPv4より優先されること...、これは自身で調べても分からなかったなーと実感しつつ感謝しました。

自身では既存のissueを補足から参加したが、クローズできたのは[joe4dev](https://github.com/joe4dev)さんのおかげ。aws-cdk-local 2.17.0がリリースされた。

## CodeDeployでデプロイ管理されたECS Fargateのタスク定義をcdkが更新できない
https://github.com/aws-cloudformation/cloudformation-coverage-roadmap/issues/1529
（最近になって挙動変わったような気がするのでステータスは要確認）

CodeDeployはECSサービスを更新できるがタスク定義は変更できないので、cdkがタスク定義を更新できる必要があるのにできない、、という致命傷のような不具合。issueの通り、CloudFormationが対応する必要のある要件。

## k8s 1.24からDockershimが削除されてcontainerdがデフォルトになったのでDatadog Agentの対応も必要
https://github.com/DataDog/datadog-agent/issues/16433

containerdでコンテナ起動することで他にも影響はあるかもしれないが、少なくとも自身の環境ではこのissueのエラーが再現していた。[Datadogのドキュメント](https://docs.datadoghq.com/ja/containers/guide/docker-deprecation/?tab=helm)にはちゃんと記載があるので予想されたエラーの範疇。

いつの間にかメンションされてお礼をもらっていたので嬉しい。

## 終わりに
issueの内容は玉石混合なのでコミュニティ側のコントリビューターも大変だな〜、と理解できました。その分、自身でissueを上げるときは再現性や目的に気をつけよう。

良いissueができれば問題の半分は解決に向かっているので、議論の場として使えればissueとして十分、という個人的見解です。

と言ってもissueだけ作成して放置されたり類似ケースが乱立したり、規模が広がれば一定のカオスは平常運転なのがOSSの世界なのかもしれない。
