---
title: "DuckDB + S3をStreamlitアプリで簡単検索"
emoji: "🦆"
type: "tech"
topics: ["duckdb", "s3", "streamlit", "sre", "aws"]
published: true
published_at: 2026-02-16 09:00
---

SREチームでは「あの時何が起きていたか」をログから素早く特定する場面が日常的にある。
Datadogなどの監視SaaSでアラートは拾えるが、長期間のログを横断検索したいとき、S3に保存されたログを直接SQLで叩けると何かと便利。

今回、 **DuckDB + Streamlit** で、ブラウザからS3上のログをSQL検索できる簡易ツールを(主にClaudeが)作った話を書く。
自分が手を動かしたのは、DuckDBの初期設定とSQLクエリのチューニング辺り。


## 背景：S3ログ検索の選択肢
S3に保存されたログを検索する方法はいくつかある。
「柔軟にSQLで集計できるもので、Athenaほど大げさではない(誰でも使いやすい)もの」、としてDuckDBを採用した。

| 観点 | DuckDB（今回のツール） | Athena |
|------|----------------------|--------|
| **用途** | アドホック調査、インシデント対応 | 定期レポート、大規模集計 |
| **データ量** | GB単位まで実用的 | TB単位でもOK |
| **コスト** | S3 APIリクエスト数課金 | スキャン量課金 |
| **準備** | テーブル定義不要 | Glueカタログ or CREATE TABLE |

仕事では両方を使い分けることが適当だろうが、プライベートでは大抵、DuckDBで十分なケースも多い。日常の調査はDuckDBでアドホック対応し、月次レポートや長期間の分析はAthenaに任せる、という棲み分けになるイメージ。


## DuckDBの何が嬉しいか

### S3上のファイルを直接SQLで読める
テーブル定義不要で、`read_json()`や `read_parquet()`など、用意された関数にS3パスを渡すだけでクエリできる。

```sql
SELECT detail.data.user_name, COUNT(*) AS cnt
FROM read_json('s3://${my-bucket}/logs/year=${year}/month=${month}/day=*/events-*.json',
               hive_partitioning = true)
GROUP BY detail.data.user_name
ORDER BY cnt DESC
LIMIT 20;
```

- Hiveパーティション対応: `year=YYYY/month=MM/day=DD` のディレクトリ構造をそのまま認識し、パーティションプルーニングが効く
- ネストしたJSONの直接参照: `detail.data.user_name` のようにドット記法でネスト構造を辿れる
- ワイルドカード: `day=*` で月全体、ファイル名の `*.json` で複数ファイルを一括読み込み

### httpfs拡張でS3透過アクセス
DuckDBの `httpfs` と `aws` 拡張を組み合わせると、ローカルの AWS 認証情報をそのまま使ってS3にアクセスできる。

AWS認証済みの環境ならこれだけで動くが、`@st.cache_resource` でプロセスレベルにキャッシュすることで、クエリごとに接続を再作成するオーバーヘッドを削減している。
```python
@st.cache_resource
def get_connection():
    conn = duckdb.connect()
    conn.execute("INSTALL httpfs; LOAD httpfs;")
    conn.execute("INSTALL aws; LOAD aws;")
    conn.execute("CREATE SECRET (TYPE S3, PROVIDER CREDENTIAL_CHAIN, REFRESH auto);")
    return conn
```

### インメモリで高速
例えば、1日分のAuth0ログなら1秒以内で返ってくる。1ヶ月分(1万件のNDJSON)でも2分程度だった。


## 今回の簡易実装メモ

### 依存関係は最小限
3つのライブラリだけ。`uv`でパッケージ管理しつつ、`uv run streamlit run app.py` で起動できる。

```toml
[project]
name = "duckdb-s3-explorer"
requires-python = ">=3.13"
dependencies = [
    "streamlit>=1.31.0",
    "duckdb>=1.1.0",
    "pandas>=2.2.0",
]
```


### 動的なWHERE句の組み立て
Claudeに実装を色々お任せしたら、サイドバーの各フィルタ入力から、WHERE句を動的に組み立てる、というUIになった。
`ILIKE` による大文字小文字を無視した部分一致検索が、ログ調査では使い勝手が良い。

```python
where_parts = []
if keyword:
    kw = keyword.replace("'", "''")
    where_parts.append(
        f"(detail.data.user_name ILIKE '%{kw}%' "
        f"OR detail.data.client_name ILIKE '%{kw}%' ...)"
    )
if event_type:
    where_parts.append(f"detail.data.type = '{event_type}'")
# ... 他のフィルタも同様

where_sql = f"WHERE {' AND '.join(where_parts)}" if where_parts else ""
```

こうしたUIがあることでプリセットではなく、ユーザ自身が検索条件を決定できるので、アドホックな調査に向いている。


## SREとしての使いどころ
- インシデント調査
    - 「特定ユーザーが〇〇時頃からログインできない」という問い合わせに対して、ユーザー名とイベントタイプでフィルタをかけ、失敗イベントの時系列を即座に確認できる。
- 傾向分析
    - イベント種別ごとの件数をGROUP BYで集計したり、特定のエラー率の推移を確認したり。

## まとめ
「S3に長期保存しているログをちょっと確認したい」を最短距離で実現するツールとして、DuckDB + Streamlitの組み合わせはおすすめ。

- DuckDBの `read_json()` + `httpfs` でS3ログをテーブル定義なしにSQL検索できる
    - AWSコンソール操作やパーティショニング構成を意識せずに、クエリだけで必要なデータにアクセス可能
- SREのインシデント調査やアドホック分析において、Athenaテーブルを作るまでもない場面で重宝しそう
    - Streamlitと組み合わせると、170行程度のPythonでブラウザベースのログ検索UIが作れる
- ユースケースが決まったら、Athenaを作成したりBigQueryにデータ連携したり、データ分析基盤にシフトすると良い

## リファレンス
 https://duckdb.org/docs/stable/data/data_sources
 
 https://duckdb.org/docs/stable/data/overview
 
 https://docs.streamlit.io/develop/api-reference/caching-and-state/st.cache_resource
