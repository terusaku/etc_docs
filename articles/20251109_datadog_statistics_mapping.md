---
title: "統計学の概念とDatadog機能のマッピング完全ガイド"
emoji: "📊"
type: "tech"
topics: ["datadog", "observability", "statistics", "monitoring", "sre"]
published: false
---

# 統計学の概念とObservability

Observabilityを理解するため、目先としてはDatadogを使いこなすため、統計学の基礎知識を振り返りつつ、Datadogの各仕様に入門する。

本記事では、統計学の各概念がDatadogのどの機能やクエリに対応しているかを体系的にマッピングし、実践的なObservabilityの実現方法を解説します。

## Observabilityとは
システム内で「あの時どこで何が起きていたか」を知る能力。和訳では可観測性。
単なる監視だけでなく、分散トレーシング、プロファイリング(性能評価)、デバッグも含まれる。

個人的印象としては、分散トレーシングと一緒の文脈でObservabilityの重要性を言われることが多く、分散システムが流行り始めた頃と同時期に必要とされた非機能要件なのかと感じている。




| モニタ対象 | 尺度分類 | Datadog機能 | 監視条件(例) |  
|------------|------------|------------|------------|
| リソース使用量 | 
| ログメッセージ | 比例尺度 | ー |  |
| イベント数 | 比例尺度 | ー | アプリの起動数が希望のスケーリング数を下回っていないか |
| ー | 比例尺度 | ー | 
| ー | 名義尺度 | ー | 
| ー | 順序尺度 | ー | 


<!-- 以下、記事サンプル -->

## 基本統計量

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **平均（Mean）** | `avg:system.cpu.user{*}`<br>`avg(last_5m):` | 比尺度 | CPU使用率、メモリ使用量、リクエスト数 |
| **中央値（Median）** | `median:trace.request.duration{*}`<br>パーセンタイル集計の`p50` | 比尺度 | レイテンシ、応答時間、処理時間 |
| **最大値（Max）** | `max:system.disk.used{*}`<br>`max(last_1h):` | 比尺度 | メモリピーク、最大同時接続数、ディスク使用量 |
| **最小値（Min）** | `min:system.load.1{*}`<br>`min(last_1h):` | 比尺度 | 最小スループット、最低応答時間 |
| **合計（Sum）** | `sum:trace.requests.hits{*}`<br>`sum(last_1d):` | 比尺度 | リクエスト総数、エラー総数、データ転送量 |
| **カウント（Count）** | `count:logs{status:error}`<br>`count(last_15m):` | 比尺度 | ログ件数、イベント件数、トレース数 |

---

## パーセンタイルと分布

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **パーセンタイル<br>（p50, p95, p99）** | `p95:trace.servlet.request.duration{*}`<br>`percentile(metric, 99)` | 比尺度 | APIレイテンシ、データベースクエリ時間<br>ページロード時間（RUM） |
| **ヒストグラム<br>（Histogram）** | Distribution Metrics<br>`distribution:custom.metric{*}` | 比尺度 | レイテンシ分布、リクエストサイズ分布<br>DDSketchによる実装 |
| **分位点<br>（Quantile）** | Distribution Metricsから計算<br>`p(0.5)`, `p(0.95)`, `p(0.99)` | 比尺度 | パーセンタイルと同義<br>SLO設定の基準値 |

---

## ばらつきの指標

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **標準偏差（σ）** | Anomaly Detection内部計算<br>`anomalies(metric, 'basic', 2)` | 比尺度 | メトリクスのばらつき評価<br>（異常検知の基準） |
| **分散（Variance）** | Anomaly Detection内部計算<br>グラフで`avg ± stddev`表示可能 | 比尺度 | 性能の安定性評価<br>レイテンシのばらつき |
| **信頼区間<br>（Confidence Interval）** | Anomaly Detection bounds<br>上限・下限バンド表示 | 比尺度 | μ ± 2σ の範囲表示<br>予測値の信頼範囲 |

---

## 時系列分析

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **移動平均<br>（Moving Average）** | `.rollup(avg, 60)` - 60秒間隔で平均<br>予測機能の内部処理 | 比尺度 | CPU使用率のスムージング<br>トラフィックトレンド |
| **指数移動平均<br>（EWMA）** | Anomaly Detection（Agile型）<br>直近データに高いウェイト | 比尺度 | 動的ベースライン<br>適応的な閾値 |
| **微分<br>（Derivative）** | `.derivative()`<br>時系列の傾き | 比尺度 | メトリクスの変化速度<br>増加率、減少率 |
| **累積<br>（Cumulative）** | `.cumsum()`<br>累積グラフ | 比尺度 | 累積リクエスト数<br>累積エラー数 |
| **差分<br>（Difference）** | `.diff()`<br>前回値との差 | 比尺度 | 増分計算<br>カウンター型メトリクスの差分 |
| **時系列分解<br>（Decomposition）** | Anomaly Detection（季節性対応）<br>Watchdog（パターン認識） | 比尺度 | 平日/週末パターン<br>時間帯別トラフィックパターン |

---

## 異常検知と予測

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **異常検知<br>（Anomaly Detection）** | `anomalies(metric, 'basic', 2)`<br>`anomalies(metric, 'agile', 3)`<br>`anomalies(metric, 'robust', 2)` | 比尺度 | 正常範囲からの逸脱検知<br>トラフィック異常、エラー率急増 |
| **外れ値検出<br>（Outlier Detection）** | Outlier Monitors<br>`outliers(metric, 'dbscan', 2)` | 比尺度 | 他と異なる挙動のホスト検知<br>異常なサーバーの特定 |
| **予測・回帰<br>（Forecasting）** | `forecast(metric, 'linear', 1w)`<br>Forecast Monitors | 比尺度 | ディスク容量予測<br>メモリ枯渇予測、成長率予測 |
| **変化率<br>（Change Rate）** | Change Alerts<br>`change(avg(last_5m), last_5m):` | 比尺度 | 前時間帯比の増減<br>エラー率の急増検知 |

---

## 相関と関係性

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **相関係数<br>（Correlation）** | Watchdog Insights<br>メトリクス相関分析機能 | 比尺度 | メトリクス間の関連性<br>（CPU↑とレイテンシ↑の相関） |

---

## レートと比率

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **レート・頻度<br>（Rate）** | `.as_rate()`<br>`per_second(count:requests{*})` | 比尺度 | 毎秒リクエスト数（RPS）<br>毎秒エラー数、スループット |
| **比率・割合<br>（Ratio）** | `(error_count / total_count) * 100`<br>SLI計算 | 比尺度 | エラー率、成功率<br>SLO達成率（99.9%） |

---

## データ処理と集計

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **集計・ロールアップ<br>（Aggregation）** | `.rollup(avg, 3600)` - 1時間ごと<br>Auto Rollup（保存期間管理） | 比尺度 | 15秒→1分→1時間データへの集約<br>ストレージ最適化 |
| **サンプリング<br>（Sampling）** | APM Ingestion Controls<br>Trace Sampling Rules | 比尺度<br>（サンプリング率） | トレースの10%をサンプリング<br>エラートレースは100%保持 |
| **正規化<br>（Normalization）** | Min-Max scaling in formulas<br>`(x - min) / (max - min)` | 比尺度 | 異なるスケールのメトリクス比較<br>0-100%への正規化 |
| **加重平均<br>（Weighted Average）** | カスタムメトリクス計算<br>`(metric1*w1 + metric2*w2)/(w1+w2)` | 比尺度 | 複数リージョンの重み付き平均<br>重要度による加重 |

---

## カテゴリカルデータ処理

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **グループ化<br>（Group By）** | `{*} by {host}`<br>`{*} by {service,env}` | 名義尺度<br>（グループキー） | ホスト別CPU使用率<br>サービス別エラー率 |
| **フィルタリング<br>（Filtering）** | `{env:prod}`<br>`{status:error}` | 名義尺度<br>（条件） | 本番環境のみ<br>エラーログのみ |
| **カーディナリティ<br>（Cardinality）** | `count_nonzero(metric)` by tag<br>Metrics Without Limits | 名義尺度<br>（ユニーク数） | ユニークホスト数<br>ユニークユーザー数（RUM） |

---

## アラートと監視

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **しきい値判定<br>（Threshold）** | Metric Monitors<br>`> 80` or `< 10` | 比尺度 | CPU > 80%でアラート<br>エラー率 > 5%で通知 |
| **範囲判定<br>（Range Check）** | Composite Monitors<br>Service Level Objectives | 比尺度 | レイテンシが100ms~500msの範囲<br>正常範囲内の監視 |
| **時間窓集計<br>（Time Window）** | `last_5m`, `last_1h`, `last_1d`<br>Evaluation Window | 比尺度 | 過去5分間の平均<br>過去1時間の最大値 |

---

## イベント関連

| 統計学の概念 | 該当するDatadogの機能や設定 | 尺度分類 | データタイプ（Observability具体例） |
|------------|--------------------------|---------|--------------------------------|
| **イベントカウント<br>（Event Count）** | Event Stream<br>Events API | 比尺度（件数）<br>名義尺度（タイプ） | デプロイ回数<br>アラート発生回数 |

---

## Datadogデータタイプ別の統計適用例

### 1. Metrics（メトリクス）

| データ例 | 適用統計 | 尺度 | 典型的クエリ |
|---------|---------|------|------------|
| CPU使用率 | 平均、最大、p95 | 比尺度 | `avg:system.cpu.user{*}` |
| メモリ使用量 | 最大、異常検知 | 比尺度 | `max:system.mem.used{*}` |
| ディスク使用率 | 予測、しきい値 | 比尺度 | `forecast(disk.used, 'linear', 1w)` |
| ネットワークトラフィック | レート、移動平均 | 比尺度 | `per_second(net.bytes_sent)` |
| リクエスト数 | 合計、レート | 比尺度 | `sum:requests.count{*}.as_rate()` |

### 2. APM / Traces（トレース）

| データ例 | 適用統計 | 尺度 | 典型的クエリ |
|---------|---------|------|------------|
| レイテンシ | p50, p95, p99 | 比尺度 | `p95:trace.servlet.request{*}` |
| エラー率 | 比率、しきい値 | 比尺度 | `error_rate = errors / total` |
| スループット | レート、合計 | 比尺度 | `hits_per_second` |
| Span継続時間 | 分布、ヒストグラム | 比尺度 | Distribution metrics |
| サービス名 | グループ化、フィルタ | 名義尺度 | `{service:api}` |

### 3. Logs（ログ）

| データ例 | 適用統計 | 尺度 | 典型的クエリ |
|---------|---------|------|------------|
| ログ件数 | カウント、レート | 比尺度 | `count:logs{*}` |
| ログレベル | フィルタ、グループ化 | 名義尺度 | `status:error` |
| レスポンスサイズ | 平均、p95 | 比尺度 | `avg:@http.response_size` |
| 処理時間 | パーセンタイル | 比尺度 | `p99:@duration` |
| ソース | グループ化 | 名義尺度 | `source:nginx` |

### 4. RUM（Real User Monitoring）

| データ例 | 適用統計 | 尺度 | 典型的クエリ |
|---------|---------|------|------------|
| ページロード時間 | p75, p90 | 比尺度 | `p75:@view.loading_time` |
| ユーザーアクション数 | カウント、合計 | 比尺度 | `count:@action` |
| エラー率 | 比率 | 比尺度 | `error_rate` |
| ブラウザ | グループ化 | 名義尺度 | `@browser.name:Chrome` |
| 地域 | グループ化、フィルタ | 名義尺度 | `@geo.country:JP` |

### 5. Synthetics（合成監視）

| データ例 | 適用統計 | 尺度 | 典型的クエリ |
|---------|---------|------|------------|
| レスポンスタイム | 平均、異常検知 | 比尺度 | `avg:synthetics.http.response.time` |
| 稼働率 | 比率、SLO | 比尺度 | `uptime = success / total` |
| テスト結果 | カウント、フィルタ | 名義尺度 | `result:passed` |
| ステップ継続時間 | パーセンタイル | 比尺度 | `p95:step.duration` |

### 6. Infrastructure（インフラ）

| データ例 | 適用統計 | 尺度 | 典型的クエリ |
|---------|---------|------|------------|
| ホスト数 | カウント | 比尺度 | `count:system.uptime{*} by {host}` |
| プロセスCPU | 平均、最大 | 比尺度 | `avg:process.cpu.pct{*}` |
| コンテナ数 | カーディナリティ | 比尺度 | `count_nonzero(container.cpu)` |
| ホスト名 | グループ化 | 名義尺度 | `{host:web-*}` |
| アベイラビリティゾーン | グループ化 | 名義尺度 | `{availability_zone:us-east-1a}` |

### 7. Network Performance Monitoring

| データ例 | 適用統計 | 尺度 | 典型的クエリ |
|---------|---------|------|------------|
| ネットワークレイテンシ | p50, p95 | 比尺度 | `p95:network.latency` |
| 転送バイト数 | 合計、レート | 比尺度 | `sum:network.bytes_sent.as_rate()` |
| TCP接続数 | カウント、最大 | 比尺度 | `max:network.tcp.connections` |
| 送信元IP | グループ化 | 名義尺度 | `{source_ip:10.0.0.1}` |
| プロトコル | フィルタ | 名義尺度 | `{protocol:tcp}` |

### 8. Security Monitoring

| データ例 | 適用統計 | 尺度 | 典型的クエリ |
|---------|---------|------|------------|
| セキュリティシグナル数 | カウント、レート | 比尺度 | `count:security.signal{*}` |
| 重大度 | フィルタ、グループ | 名義尺度 | `severity:high` |
| 攻撃元IP | カーディナリティ | 名義尺度 | `count_nonzero(@network.client.ip)` |
| ルール名 | フィルタ | 名義尺度 | `rule.name:"SQL Injection"` |

---

## 実装パターン

### 比尺度データの典型的な操作

```python
# 集計関数
avg(), max(), min(), sum(), count()

# パーセンタイル
percentile(metric, 95)

# 時系列操作
.rollup(avg, 60)        # 移動平均
.derivative()           # 微分（変化率）
.cumsum()              # 累積和
.diff()                # 差分

# 異常検知
anomalies(metric, 'basic', 2)
outliers(metric, 'dbscan', 3)

# 予測
forecast(metric, 'linear', 1w)

# レート変換
.as_rate()
per_second()
```

### 名義尺度データの典型的な操作

```python
# フィルタリング
{env:prod}
{status:error}
{service:api AND region:us-east-1}

# グループ化
{*} by {host}
{*} by {service, env}

# カーディナリティ
count_nonzero(metric)
count(distinct(field))

# 論理演算
AND, OR, NOT
```

---

## 学習パス

### 初級：基本統計量を使う

1. **平均・最大・最小を理解する**
   - `avg()`, `max()`, `min()`
   
2. **カウントと合計の違い**
   - `count()` vs `sum()`
   
3. **グループ化とフィルタ**
   - `by {tag}`, `{filter}`

### 中級：高度な統計を活用

4. **パーセンタイルでSLO設定**
   - `p95()`, `p99()`
   
5. **異常検知を使う**
   - `anomalies()`
   
6. **レート計算**
   - `.as_rate()`

### 上級：統計的分析

7. **予測とトレンド**
   - `forecast()`
   
8. **相関分析**
   - Watchdog Insights
   
9. **分布分析**
   - Distribution Metrics

---

## まとめ

- **比尺度（数値データ）**: 測定、計算、集計、予測が可能
- **名義尺度（カテゴリデータ）**: 分類、グループ化、フィルタリングが可能
- **統計学の理解なしには高度なObservabilityは実現できない**
- **Datadogの機能は統計学の実装である**

---

*このドキュメントはDatadog入門ガイドの一部として作成されました*
