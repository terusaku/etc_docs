---
title: "SecurityHubの評価ルールをAWS CDKのバリデーションとして実装する"
emoji: "✅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["awscdk", "securityhub", "validation"]
published: true
published_at: 2024-07-19 08:50
---

AWSではSecurity Hubの標準リファレンスとして、セキュリティのベストプラクティスが提供されている。
https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/fsbp-standard.html

ちょうど仕事でSecurity Hubの標準をを有効化して一通りチェックする機会があったが、コントロールを無効化したり、リソース毎に抑止したり、「標準と言っても一概には言えない」というケースは色々ありました。

そこで、CDK管理するリソースに対して、Security Hubの評価ルールをバリデーション実装すれば話が早い、ということで`Aspects`と`validate()`の2つの方法で動作を検証した。

参考:
https://aws.amazon.com/jp/blogs/news/align-with-best-practices-while-creating-infrastructure-using-cdk-aspects/
https://dev.to/aws-builders/validation-with-aws-cdk-addvalidation-20lo


`Aspects`は「特定のAWSリソース全てを検証したい場合」、`validate()`は「[Token](https://docs.aws.amazon.com/cdk/v2/guide/tokens.html)に基づくような動的な参照値を含む設定を検証したい場合」、として使い分けています。

## Aspectsを使ったバリデーション
https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/rds-controls.html#rds-7
あるCDKスタック内のRDSクラスタが全て削除保護が有効化されているか、の検証です。削除保護は本番環境では必須としておきたいところですが、「本番環境のみバリデーションを有効にする」ことはaws-cdkでは簡単に対応できます。

- CDKスタックに対するAspects適用例
```ts
const rdsStack = new RdsStack(
  app,
  'HopRdsStack',
  context,
  stackProps,
);
if (profileName === 'prod') {
  cdk.Aspects.of(rdsStack).add(new RdsDeleteProtectionAspect());
}
```

```ts
import * as rds from 'aws-cdk-lib/aws-rds';
import { IAspect, Annotations } from 'aws-cdk-lib';
import { IConstruct } from 'constructs';

import { RdsDeleteProtectionAspect } from '@/lib/aspects/rds-aspects';


export class RdsDeleteProtectionAspect implements IAspect {
  public visit(node: IConstruct) {
    if (node instanceof rds.CfnDBCluster) {
      // deletionProtectionが存在しない or falseの場合
      if (! node.deletionProtection) {
        Annotations.of(node).addError('RDS Cluster must have deletionProtection enabled, in pruduction.');
      }
    }
  }
}
```

`deletionProtection`のように0/1で検証可能なものは`Aspects`に向いている。

対象のCDKスタックで複数のRDSクラスタを作成していても、RDSクラスタ毎にバリデーションが実行されるため、エラー発生時もどのCDKリソースが原因かは判別できました。


## validate()を使ったバリデーション
https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/rds-controls.html#rds-11
RDSクラスタ作成時、自動バックアップ期間が7日以上で設定しているか、の検証です。本番・テストなど環境差異によって検証したい値は変わる前提として、バリデーションを作りました。

```ts
import { clusterProps } from '@/types/common-context';

// RDSクラスタ作成クラス例
class HopRdsCluster extends Construct {
  public readonly endpoint: rds.Endpoint;
  public readonly cluster: rds.IDatabaseCluster;

  constructor(scope: Construct, id: string, props: clusterProps) {
    super(scope, id);

    const {
      // クラスタパラメータ色々
    } = props;

    this.node.addValidation(
      new RdsBackupValidator({
        ...props,
      })
    );
    // 以下略.....
  }
}
```

```ts
import { IValidation } from 'constructs';

import { clusterProps } from '@/types/common-context';
import { RdsBackupValidator } from '@/lib/validators/rds-validators';

export class RdsBackupValidator implements IValidation {
  constructor(private readonly props: clusterProps) {}

  public validate(): string[] {
    const validateMessages: string[] = [];
    
    // profileNameを条件に値をセットする
    const backupRetentionDays: number = this.props.profileName === 'prod' ? 7 : 1;

    if (this.props.backupProps) {
      if (this.props.backupProps.retention.toDays() < backupRetentionDays) {
      validateMessages.push(`Backup Retention(${this.props.profileName}) must be ${backupRetentionDays} days or more.....Now, ${this.props.backupProps.retention.toDays()} days`);
      }
    } else {
      validateMessages.push(`Backup Retention(${this.props.profileName}) must be ${backupRetentionDays} days or more.....Now, Not set`);
    }
    return validateMessages;
  }
}
```

上記では評価基準のRetentionをハードコードしているが、「本番環境なら7日以上、それ以外の環境だったら1日以上」としている。もし自動バックアップ設定が無効な場合は`undefined`なので、CDKのバリデーションとしてはその考慮も必要です。

Aspects同様、RDSクラスタ毎にバリデーションが実行され、エラー発生時にはどのCDKリソースが失敗しているか判別できる。


## 終わりに
CDK管理するようになっても設定ミスが横展開されてしまうことは避けたいので、バリデーション実装はコストパフォーマンスが高いと感じました。

何より構築したシステム・アプリケーションでは何を重視しているのか、設計ポリシーをバリデーションに反映することができるので、ベストプラクティスの実務として理想の運用に近づけることができそう。
