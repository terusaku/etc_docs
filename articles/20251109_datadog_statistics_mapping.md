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

## 統計学の尺度とデータタイプ

Datadogで扱うデータを理解する上で、統計学における「尺度」の概念が重要です。

### 尺度の種類

| 尺度分類 | 特徴 | Observabilityでの例 | 代表的な統計量 |
|----------|------|---------------------|----------------|
| **比尺度** | 絶対的な0点を持つ数値 | CPU使用率、レイテンシ、リクエスト数 | 平均、中央値、標準偏差、比率計算 |
| **間隔尺度** | 相対的な0点を持つ数値 | 経過時間、前月比 | (移動平均や比率の統計量で副次的に使われる) |
| **順序尺度** | 順序関係のみ意味を持つ | HTTPステータスコード、ログレベル(DEBUG < INFO < WARN < ERROR) | ヒストグラム |
| **名義尺度** | カテゴリ分類のみ | 環境名(prod/staging)、サービス名、ホスト名 | カウント、モード |

### Datadogにおける代表的な監視対象とマッピング

| モニタ対象 | 尺度分類 | Datadog機能 | 監視条件(例) |
|------------|------------|------------|------------|
| リソース使用量 | 比例尺度 | Infrastructure Monitoring | CPU > 80% が5分間継続 |
| レイテンシ | 比例尺度 | APM, Distribution Metrics | p95 > 500ms でアラート |
| エラー率 | 比例尺度 | Logs, APM | エラー率 > 5% でアラート |
| スループット | 比例尺度 | APM, Custom Metrics | RPS < 100 で警告 |
| イベント数 | 比例尺度 | Events, Monitors | デプロイ失敗が3回以上 |
| サービス名 | 名義尺度 | Tags, Group By | サービス別のエラー集計 |
| ログレベル | 順序尺度 | Log Management | ERROR以上でアラート |
| 環境 | 名義尺度 | Tags, Filters | 本番環境のみ監視 |

---

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

## 実践的なユースケース

### ケース1: APIレイテンシの異常検知

**課題**: ECサイトのAPIレスポンスタイムが時々遅くなるが、原因が分からない

**統計的アプローチ**:
- **パーセンタイル分析** で外れ値を検出(p50, p95, p99を比較)
- **異常検知** で過去のパターンから逸脱した場合にアラート
- **相関分析** でCPU使用率やDB接続数との関連を確認

**Datadogでの実装**:
```python
# p95レイテンシの異常検知
anomalies(p95:trace.web.request{service:api}, 'agile', 3)

# パーセンタイルの比較
p50:trace.web.request{service:api}  # 中央値
p95:trace.web.request{service:api}  # 95パーセンタイル
p99:trace.web.request{service:api}  # 99パーセンタイル
```

---

### ケース2: エラー率のSLO監視

**課題**: 99.9%の可用性を保証するSLOを設定したい

**統計的アプローチ**:
- **比率計算** で成功率を算出
- **時間窓集計** で30日間の移動平均を評価
- **しきい値判定** でSLO違反時にアラート

**Datadogでの実装**:
```python
# エラー率の計算
(sum:trace.requests{status:error}.as_count() /
 sum:trace.requests{*}.as_count()) * 100
```

---

### ケース3: ディスク容量枯渇の予測

**課題**: ディスク容量が満杯になる前に事前対処したい

**統計的アプローチ**:
- **線形回帰** で将来の使用量を予測
- **時系列分析** で季節性・週次パターンを考慮
- **予測アラート** で7日後に90%超える予測でアラート

**Datadogでの実装**:
```python
# 1週間先のディスク使用率を予測
forecast(avg:system.disk.used{*}, 'linear', 1w)

# アラート条件: 7日後の予測値が90%超
forecast(avg:system.disk.used{*}, 'linear', 1w) > 90
```

---

### ケース4: 外れ値ホストの検出

**課題**: 1000台のサーバーから異常な挙動のホストを特定したい

**統計的アプローチ**:
- **外れ値検出** でDBSCAN法を使用
- **カーディナリティ分析** でホスト数の推移を監視
- **グループ化** で正常/異常グループに分類

**Datadogでの実装**:
```python
# CPU使用率が他と異なるホストを検出
outliers(avg:system.cpu.user{*} by {host}, 'dbscan', 2)

# メモリ使用率の外れ値検出(MAD法)
outliers(avg:system.mem.used{*} by {host}, 'mad', 3)
```

---

## ベストプラクティス

### 1. 適切な統計量の選択

| 目的 | 推奨統計量 | 理由 |
|------|-----------|------|
| 典型的な値 | 中央値(p50) | 外れ値の影響を受けにくい |
| SLO設定 | p95, p99 | ユーザーの95-99%をカバー |
| 容量計画 | 最大値(max) | ピーク時の負荷に対応 |
| コスト最適化 | 平均値(avg) | 全体的な傾向を把握 |
| 異常検知 | 標準偏差 | ばらつきから逸脱を検知 |

### 2. アラート設定のアンチパターン回避

**❌ 悪い例**: 瞬間的なスパイクで誤報
```python
avg:system.cpu.user{*} > 80
```

**✅ 良い例**: 5分間の平均で判定
```python
avg(last_5m):avg:system.cpu.user{*} > 80
# または異常検知を使用
anomalies(avg:system.cpu.user{*}, 'agile', 3)
```

### 3. カーディナリティの管理

**❌ 避けるべき**:
- ユーザーIDやリクエストIDをタグに使用
- 無制限のタグ値(IPアドレスなど)

**✅ 推奨**:
- 事前定義されたカテゴリのみ(env, service, version)
- 高カーディナリティデータはLogsやTracesで管理
- Metrics Without Limitsで必要なタグのみ保持

### 4. 時間窓の選択

| 用途 | 推奨時間窓 | 理由 |
|------|-----------|------|
| リアルタイムアラート | 1-5分 | 迅速な対応 |
| トレンド分析 | 1-24時間 | ノイズ除去 |
| 容量計画 | 7-30日 | 長期的傾向 |
| SLO評価 | 30日 | 業界標準 |

---

## よくある間違いと対処法

### 間違い1: 平均値だけを見る

**問題点**: 外れ値や分布の偏りを見逃す

**例**:
- API平均レイテンシ: 100ms(良好に見える)
- しかしp99は5秒(1%のユーザーは非常に遅い)

**対処法**:
```python
avg:trace.web.request{*}   # 100ms
p50:trace.web.request{*}   # 80ms
p95:trace.web.request{*}   # 300ms
p99:trace.web.request{*}   # 5000ms ← 問題発見!
```

---

### 間違い2: カウンター型メトリクスをそのまま使用

**問題点**: カウンターは累積値なので、そのままでは意味がない

**対処法**:
```python
# ❌ 累積リクエスト数(常に増加)
sum:http.requests.count{*}

# ✅ レート変換で毎秒リクエスト数に
sum:http.requests.count{*}.as_rate()
```

---

### 間違い3: 異常検知アルゴリズムの選択ミス

| アルゴリズム | 適用場面 | 特徴 |
|------------|---------|------|
| **Basic** | 季節性なし | シンプル、標準偏差ベース |
| **Agile** | 急速に変化 | 直近データに高い重み |
| **Robust** | 季節性あり | 週次・日次パターンを学習 |

**対処法**:
```python
# ❌ 平日/週末パターンがあるのにBasic
anomalies(avg:requests.count{*}, 'basic', 2)  # 週末に誤報

# ✅ 季節性を考慮したRobust
anomalies(avg:requests.count{*}, 'robust', 2)
```

---

### 間違い4: グループ化とフィルタの混同

**対処法**:
```python
# ❌ 本番のみ見たいのにグループ化
avg:system.cpu.user{*} by {env}  # staging/prod両方表示

# ✅ フィルタで絞り込む
avg:system.cpu.user{env:prod}

# ❌ サービス別に見たいのにフィルタ
avg:system.cpu.user{service:api}  # apiだけ

# ✅ グループ化で全サービス比較
avg:system.cpu.user{*} by {service}
```

---

### 間違い5: 高頻度データのロールアップ忘れ

**問題点**: 長期間クエリでパフォーマンス低下

**対処法**:
```python
# ❌ 30日間を15秒粒度で取得(172,800ポイント)
avg:system.cpu.user{*}

# ✅ 1時間ごとにロールアップ(720ポイント)
avg:system.cpu.user{*}.rollup(avg, 3600)
```

---

### 間違い6: SLOの指標選択ミス

**❌ 悪い例**: 平均レイテンシでSLO設定
- 問題: 一部のユーザーが極端に遅くても平均は良く見える

**✅ 良い例**: p95レイテンシでSLO設定
- 理由: 95%のリクエストが基準内であることを保証

**✅ さらに良い例**: 複合SLO
```
1. 可用性: 99.9%以上(エラー率 < 0.1%)
2. レイテンシ: p95 < 500ms
3. スループット: RPS > 1000
```

---

## まとめ

### 重要なポイント

- **比尺度（数値データ）**: 測定、計算、集計、予測が可能
- **名義尺度（カテゴリデータ）**: 分類、グループ化、フィルタリングが可能
- **統計学の理解なしには高度なObservabilityは実現できない**
- **Datadogの機能は統計学の実装である**

### 次のステップ

1. **基本統計量から始める**: avg, max, min, countの使い方をマスター
2. **パーセンタイルを活用**: p95, p99でユーザー体験を正確に把握
3. **異常検知を導入**: anomalies()で手動監視から解放される
4. **SLOを設定**: ビジネス目標と技術指標を結びつける
5. **予測機能を活用**: forecast()で問題を未然に防ぐ

### 参考リソース

- [Datadog公式ドキュメント](https://docs.datadoghq.com/)
- [Distribution Metricsガイド](https://docs.datadoghq.com/metrics/distributions/)
- [Anomaly Detection](https://docs.datadoghq.com/monitors/types/anomaly/)
- [SLO設定ガイド](https://docs.datadoghq.com/service_management/service_level_objectives/)

---

*このドキュメントはDatadog入門ガイドの一部として作成されました*
