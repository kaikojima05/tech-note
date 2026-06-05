## DuckDB について

読み方は DuckDB（ダックディービー）
<br />オープンソースのデータ集計・解析に特化した組み込みのサーバーレスなDB

## なぜ集計・解析に向いていると言えるのか？

一般的に使用されている MySQL や PostgreSQL は行単位で読み・書きを行っている
<br />これを `行指向` という

```markdown
Row1: { id: 1, name: "kai kojima", age: 29, address: "神奈川県厚木市" }
Row2: { id: 2, name: "yuki irioka", age: 40, address: "東京都杉並区" }
Row3: { id: 3, name: "shinya hino", age: 38, address: "神奈川県座間市" }
```

対して、DuckDB は `列指向`

```markdown
Column1: { id: 1, id: 2, id: 3 }
Column2: { name: "kai kojima" name: "yuki irioka", name: "shinya hino" }
Column3: { address: "神奈川県厚木市", address: "東京都杉並区", address: "神奈川県座間市" }
```

この時、MySQL や PostgreSQL は age の平均を計算しようとすると、全行を読んでから age の計算を行わなければいけないが、DuckDB は age の平均を算出するのに全行を読む必要がない（不要なカラムを読まない）ため、集計・解析に向いていると言える

```sql
SELECT AVG(age) FROM HogeHogeTable; -- 計算スピードが全く違う
```

## DuckDB の利点

1. MySQL や PostgreSQL では、csv の読み取りをするときは `LOAD DATA` を使用する必要があるが、DuckDB ではファイルパスを指定するだけで読み取りが可能
2. アプリケーションに組み込んで使用するため、セットアップが簡単

## DuckDB が向いていない領域

- DuckDB は一つのプロセスが書き込みを行っている間は他のプロセスは `read-only` であっても接続できないので、複数クライアント（プロセス）からの同時書き込み
- 小さな読み書きが大量に発生するアプリケーション
