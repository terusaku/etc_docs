---
title: "統計学の概念とDatadog機能のマッピング完全ガイド"
emoji: "📊"
type: "tech"
topics: ["datadog", "observability", "statistics", "monitoring", "sre"]
published: true
published_at: 2025-11-10  07:00
---

Observabilityを理解するため、目先としてはDatadogを使いこなすため、統計学の基礎知識を振り返りつつ、Datadogの各機能に触れます。

Datadogの使い方を具体的に知りたい人には役立たないので、その仕様がなぜそうなっているのか、背景や違いを理解したい人向け。モニタリング機能は統計学の実装とも言える、という個人的見解が今回の動機です。

## Observabilityとは
システム内で「あの時どこで何が起きていたか」を知る能力。和訳では可観測性。
単なる監視だけでなく、分散トレーシング、プロファイリング(性能評価)、デバッグも含まれています。

個人的印象としては、分散トレーシングと一緒の文脈でObservabilityの重要性を言われることが多く、分散システムが流行り始めた頃と同時期に必要とされた非機能要件かと感じてます。

## 統計学の尺度とデータ

Datadogで扱うデータを理解するために、これから統計学における「尺度」を使って分類してます。


### 尺度の種類
| 尺度分類 | 特徴 | Observabilityでの例 | 代表的な統計量 |
|----------|------|---------------------|----------------|
| **比尺度** | 絶対的な0点を持つ数値 | CPU使用率、レイテンシ、リクエスト数 | 平均、中央値、標準偏差、比率計算 |
| **間隔尺度** | 相対的な0点を持つ数値 | タイムスタンプ(*1) | ー |
| **順序尺度** | 順序関係のみ意味を持つ | ログレベル(DEBUG < INFO < WARN < ERROR) | ヒストグラム |
| **名義尺度** | カテゴリや名前 | 環境名(prod/staging)、サービス名、HTTPステータスコード | カウント |

*1: 温度や西暦など代表例で、Observabilityには関係が薄いので割愛。

---

## Datadogにおける代表的な監視対象とマッピング

| Datadog機能 | モニタ対象 | 尺度分類 | 監視条件(例) |
|------------|------------|------------|------------|
| Infrastructure Monitoring | リソース使用量 | 比尺度 | CPU > 80% が5分間継続 |
| Logs, APM | エラー数、エラー率 | 比尺度 | `エラー率 > 5%` でアラート |
| Logs | ログステータス | 順序尺度 | `ERROR` は1回でアラート、`WARN`は複数回でアラート |
| APM | レイテンシ、スループット | 比尺度 | `p95 > 500ms`, `100rps 以上` でアラート |
| APM | HTTPステータス | 名義尺度 | `ステータスコード>= 500` でアラート |
| Events, Monitors | イベント数 | 比尺度 | 一定期間内の失敗が n回以上 |
| Tags, Group By | サービス名 | 名義尺度 | サービス別のエラー集計 |
| Tags, Filters | 環境、テナント | 名義尺度 | Prod/Stg, ユーザ毎の監視や集計 |

---

## 参考： Datadogが提供する統計関数
https://docs.datadoghq.com/ja/dashboards/functions/

tips的によく使う関数
- `.rollup()`: `last()`との違いは、ロールアップは指定期間内の集計を行うのに対し、`last()`は単に最後のデータポイントを取得する。
- `p95:__{metric}__`: APIのレイテンシなど、ユーザ体験の評価と言えばまずはこれ(パーセンタイル値は要検討)
- `count_nonzero()`, `exclude_null()`: ゼロに意味がない場合（例: 正常な時には何も記録されないイベント）に使う
   - `.fill()`を使ってゼロ埋めした後に計算する方法もある

これからもうちょっと使っていきたい関数
- `autosmooth()`: 傾向把握を定量的に行えるかも。
- `per_hour(__{metric}__)`, `diff(__{metric}__)`: 変化量を把握することで、因果関係・相関関係を議論しやすそう。
   - `.derivative()`(微分係数)の関数もある...

---

## Datadogの統計的ベストプラクティス

### 適切な統計量の選択

| 目的 | 推奨統計量 | 理由 |
|------|-----------|------|
| 典型的な値 | 中央値(p50) | 外れ値の影響を受けにくい |
| SLO設定 | p95, p99 | ユーザー体験の95-99%をカバー |
| サイジング | 最大/最小(max/min) | ピーク時負荷の大小に対応 |
| コスト最適化 | 移動平均値(avg) | 全体的な傾向を把握 |
| 異常検知 | 標準偏差 | ばらつきから外れ値を検知 |

### アラート設定のアンチパターン回避

**❌ 悪い例**: 瞬間的なスパイクで誤報
```python
avg:system.cpu.user{*} > 80
```
![](/images/datadog_non-rollup.png)
*rollupなし*

**✅ 良い例**: 10分間負荷が継続した場合にアラート
```python
avg:system.cpu.user{*}.rollup(avg, 600) > 80
```

![](/images/datadog_rollup(min,1800).png)
*rollupあり*


### カーディナリティの管理

**❌ 避けるべき**:
- ユーザーIDやリクエストIDをタグに使用
- 無制限のタグ値(IPアドレスなど)

**✅ 推奨**:
- タグでは事前定義したカテゴリのみ扱う(env, service, version)
   - クラウド環境では`host`も自然と高カーディナリティになるので、全て禁止する必要性はない
   - 個人的には、セッションIDをタグに使う企業ユースケースも時々耳にしている
- 基本的に、高カーディナリティのデータはLogsやTracesで管理する
- 費用対効果の方式としては、[Metrics Without Limits](https://docs.datadoghq.com/ja/metrics/metrics-without-limits/)で必要なタグのみ保持する

---

## まとめ

### 重要なポイント
- 比尺度（数値データ）: 測定、計算、集計、予測が可能。
- 順序尺度（質的データ）: アプリロジック含めての状態把握も可能（ログやトレースに必要な情報を意図的に含める）。
- 名義尺度（カテゴリデータ）: 分類、グループ化、フィルタリングの条件として使う

## リファレンス
- [Logs vs Metrics vs Traces - Microsoft Engineering Playbook](https://microsoft.github.io/code-with-engineering-playbook/observability/log-vs-metric-vs-trace/)
- [Datadog公式ドキュメント - 関数一覧](https://docs.datadoghq.com/ja/dashboards/functions/)
- [Metrics Without Limits](https://docs.datadoghq.com/ja/metrics/metrics-without-limits/)
- [Understanding Data Types and Scales of Measurement - Statistics Solutions](https://www.statisticssolutions.com/understanding-data-types-and-scales-of-measurement/)