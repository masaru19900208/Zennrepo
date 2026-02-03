---
title: "NoSQL育ちがAWS AuroraでSQL実践してみた"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["backend", "rdb", "db", "database", "sql"]
published: true
---

## 1.はじめに

こんにちは。前回の「設計編」に引き続き、リレーショナルデータベース（RDB）の学習記録です。
本記事では、実際に手を動かして AWS Aurora（PostgreSQL互換）上にデータベースを構築し、SQLを実行します
https://zenn.dev/masaru0208/articles/d7ecc82124eda7

「頭で理解した設計思想が、実際のクエリでどう動くのか？」
この疑問を解消し、RDBを"武器"として扱えるようになることがゴールです。

## 2.対象読者

- リレーショナルデータベースを実際に触ったことが無い方
- 「とりあえず動く」データベースから、保守性の高い設計へステップアップしたい方
- AWS上でのRDB構築に興味がある方

## 3.記事を読むメリット

- AWSマネジメントコンソールを用いたAurora構築の流れを把握できる
- DBeaverを使った接続方法と基本操作を習得できる
- SQLの基本を、前回学んだ正規化されたテーブル構造で実践できる

## 4.開発環境構築

本記事で使用する環境は以下の通りです。

### 4.1.使用ツール・サービス

| 項目 | 使用するもの | 用途 |
| --- | --- | --- |
| データベース | Amazon Aurora (PostgreSQL互換) | クラウド上のRDB |
| クライアントツール | DBeaver 25.3.2 | DBへの接続・SQL実行 |
| OS | Windows 11 | 開発マシン |

### 4.2.DBeaverとは

https://dbeaver.io/

- DBeaverは、無料で利用できるデータベース管理ツール（GUIクライアント）です
- MySQL、PostgreSQL、Oracle、SQL Serverなど、ほとんどの主要なデータベースに対応
- ER図（テーブル間のリレーション図）を自動生成できます

#### 類似ツール

無償で全機能利用可能という点でDBeaverを選定しましたが、他にも以下のような類似ツールもあります。

- TablePlus: https://tableplus.com/
- Beekeeper Studio: https://www.beekeeperstudio.io/ja/
- HeidiSQL: https://www.heidisql.com/
- DbGate: https://www.dbgate.io/ja/
- DataGrip: https://www.jetbrains.com/ja-jp/datagrip/

## 5.AWS Aurora (PostgreSQL互換) の構築

### 5.1.なぜAuroraを選ぶのか

設計編でも触れましたが、AWS環境でRDBを使うなら **Amazon Aurora** が推奨されます。

https://zenn.dev/masaru0208/articles/d7ecc82124eda7#amazon-aurora-%E3%81%AE%E6%8E%A8%E5%A5%A8

今回は学習目的のため、コストを抑えた構成で構築します。

### 5.2.VPCとセキュリティグループの準備

Auroraを構築する前に、ネットワーク周りの設定を確認します。

#### 5.2.1.セキュリティグループの作成

1. AWSマネジメントコンソールで「VPC」を検索し、VPCダッシュボードを開く
2. 左メニューから「セキュリティグループ」を選択
3. 「セキュリティグループを作成」をクリック
4. 以下を設定

| 項目 | 設定値 |
|---|---|
| セキュリティグループ名 | `aurora-postgres-sg` |
| 説明 | `Security group for Aurora PostgreSQL` |
| VPC | デフォルトVPCまたは作成済みのVPC |

5. インバウンドルールを追加：

| タイプ | ポート範囲 | ソース |
|---|---|---|
| PostgreSQL | 5432 | [自身のIPアドレス](https://www.cman.jp/network/support/go_access.cgi) |

### 5.3.Auroraクラスターの作成

1. AWSマネジメントコンソールで「RDS」を検索し、RDSダッシュボードを開く
2. 「データベースの作成」をクリック
3. 以下を設定

エンジンの選択

| 項目 | 設定値 |
|---|---|
| エンジンのタイプ | Aurora (PostgreSQL Compatible) |
| エンジンバージョン | 最新の安定版（例：Aurora PostgreSQL メジャーバージョンのデフォルト 17） |

テンプレート

| 項目 | 設定値 |
|---|---|
| テンプレート | 開発/テスト（コスト削減のため） |

設定

| 項目 | 設定値 |
|---|---|
| DBクラスター識別子 | `my-learning-aurora`等 |
| ユーザー名 | `postgres`（デフォルト） |
| 認証情報管理 | AWS Secrets Manager |

接続

| 項目 | 設定値 |
|---|---|
| コンピューティングリソース | 接続しない |
| VPC | 先ほどセキュリティグループを作成したVPC |
| パブリックアクセス | あり（学習用。本番環境では「いいえ」推奨） |
| VPCセキュリティグループ | 先ほど作成した`aurora-postgres-sg` |

4. 「データベースの作成」をクリック
5. ステータスが「利用可能」になるまで待つ

### 5.4.エンドポイントの確認

Auroraクラスターが作成されたら、接続に必要な情報を確認します

1. RDSダッシュボードで作成したクラスターを選択
2. 「接続とセキュリティ」タブを開く
3. 「エンドポイント」をクリックしエンドポイントセクションから **ライターエンドポイント** をコピー

例：`my-learning-aurora.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com`

## 6.DBeaverからAuroraへの接続

### 6.1.接続設定

1. DBeaverを起動
2. 左上の「新しい接続」アイコン（プラグマーク）をクリック
3. 「PostgreSQL」を選択し「次へ」をクリック

![dbeavercreatenewconnection](/images/rdb_practical/dbeaver_create_new_connection.png)

4. 以下を参照し接続情報を入力：

接続とセキュリティタブにて確認

| 項目 | 設定値 |
|---|---|
| Connect by | Host |
| Host | タイプ：ライターのエンドポイント |
| Port | 5432 |
| Database | postgres |
| 認証 | Database Native |
| Username | postgres（作成時に設定したマスターユーザー名） |
| Password | RDSダッシュボードから"シークレットマネージャー"と記載されたボタンをクリックし取得 |

![dbeaversettingauroraconfig](/images/rdb_practical/dbeaver_setting_aurora_config.png)

5. 「テスト接続」をクリックし、接続が成功することを確認してOKをクリック

![cinoleteconnectaurora](/images/rdb_practical/complete_connect_aurora.png)


### 6.2.タイムアウトする場合

セキュリティグループのインバウンドIP、VPCのルートテーブルが正しく設定されているか再度確認しましょう（[セキュリティグループ設定](#52vpcとセキュリティグループの準備)）


## 7.SQL実践：データベース・テーブルの作成

ここからは、設計編で正規化した「注文管理システム」のテーブルを実際に作成していきます。

### 7.1.テーブルの作成

- テーブル構造は簡単な受発注システムを想定したものとしています。
- 今回はGUIベースに作成していきます

```mermaid
erDiagram
  users{
    varchar user_id PK "ユーザーID"
    varchar user_name "ユーザー名"
    varchar user_email UK "メールアドレス"
  }
  products{
    varchar product_id PK "商品ID"
    varchar product_name "商品名"
    integer unit_price "単価"
  }
  orders{
    smallint order_id PK "注文ID"
    date order_date "注文日"
    varchar user_id FK "注文したユーザーID"
  }
  order_details{
    smallint order_id PK "注文ID"
    varchar product_id PK "商品ID"
    smallint quantity "数量"
  }

  users ||--o{ orders : "places"
  orders ||--|{ order_details : "contains"
  products ||--o{ order_details : "is_in"
```

https://zenn.dev/masaru0208/articles/d7ecc82124eda7#7.4.%E7%AC%AC3%E6%AD%A3%E8%A6%8F%E5%BD%A2-(3nf)-%3A-%E3%82%AD%E3%83%BC%E4%BB%A5%E5%A4%96%E3%81%AB%E4%BE%9D%E5%AD%98%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%E3%82%82%E3%81%AE%E3%82%92%E5%88%A5%E3%83%86%E3%83%BC%E3%83%96%E3%83%AB%E3%81%B8

#### 7.1.1.usersテーブル

- ユーザーテーブル`users`を作成します
- postgres > データベース > postgres > スキーマ > public > テーブル > 右クリック > `新しく作る表`をクリック

![databasetable](/images/rdb_practical/database_table.png)

- テーブル名を`users`に設定し、列セクションで右クリックし`新しく作る カラム`をクリックし、以下の項目を追加していきます

| colmun name | データタイプ | ヌルではない | keys |
|---|---|---|---|
| `user_id` | varchar | True | 主key |
| `user_name` | varchar | True | - |
| `user_email` | varchar | True | ユニークキー |

※ KeysのNameとは、Keys自体（どのColumnにどのような役割を持たせたか）を識別するための名前なので、自動入力内容でも識別できればOK。後から変更も可能。

`Ctrl+S`で保存することで、以下のようなSQLを使用しテーブル作成が実行されます

```sql
CREATE TABLE public.users (
	user_id varchar NOT NULL,
	user_name varchar NOT NULL,
	user_email varchar NOT NULL,
	CONSTRAINT users_pk PRIMARY KEY (user_id),
	CONSTRAINT users_email_unique UNIQUE (user_email)
);
```

![createusertable](/images/rdb_practical/create_user_table.png)

:::details 列作成のプロパティについて

https://dbeaver.com/docs/dbeaver/25.3/Creating-columns/#create

| 項目名                           | 説明                                                                                      |
| -------------------------------- | ----------------------------------------------------------------------------------------- |
| 身元 | Identityの日本語訳。いわゆるサロゲートキーのようなデータ追加時に自動的にインクリメントした番号を割り振る機能
| Collection                       | データソートする際の基準を示すもの。仮に日本語基準でソートをする予定があればここを`ja_JP` |
| Storage                          | データの圧縮方式らしいですが、この項目について記載のある公式ドキュメントを確認できませんでした。基本は空でよい。  |
| key | user_idやemail等、一意であることが重要な場合はチェックを入れます。生成されるNameは列名とは異なり制御ロジックの中で使用される名称なので自動生成されるものを利用でよさそうです                                                                                          |
:::


#### 7.1.2.productsテーブル

- 商品テーブル`products`を作成します
- 以下の項目を作成します

| colmun name | データタイプ | ヌルではない | keys |
|---|---|---|---|
| `product_id` | varchar | True | 主key |
| `product_name` | varchar | True | - |
| `unit_price` | integer | True | - |


- 追加で`unit_price`項目に`制約`を追加します
  - サイドメニューから、セクションを`制約`に切り替え > 右クリック > 新しく作る制約をクリック
  - タイプを`CHECK`に変更し以下のように入力します

![createcolumnconstraint](/images/rdb_practical/create_column_constraint.png)

最終的に以下のSQLが実行されます

```sql
CREATE TABLE public.products (
	product_id varchar NOT NULL,
	product_name varchar NOT NULL,
	unit_price integer NOT NULL,
	CONSTRAINT products_pk PRIMARY KEY (product_id),
	CONSTRAINT products_check_ge_zero CHECK (unit_price >= 0)
);
```

#### 7.1.3.ordersテーブル

- 注文テーブル`orders`を作成します
- 以下の項目を作成します

| colmun name | データタイプ | ヌルではない | keys |
|---|---|---|---|
| `order_id` | smallint | True | 主key |
| `order_date` | date | True | - |
| `user_id` | varchar | True | - |

- 追加で`user_id`項目に`外部キー`を追加します
- サイドメニューから、セクションを`外部キー`に切り替え > 右クリック > 新しく作る外部キーをクリック
- 参照表で`users`を選択
- ユニークキーで`users_pk`を選択（usersテーブルの主キー設定時の`Name`）
- Custom Nameは自動で入ります（画像は変更しちゃってますが基本変更しなくて大丈夫です
- 削除時（ON DELETE）と更新時（ON UPDATE）の動作を定義します

![createforeignkey](/images/rdb_practical/create_foreign_key.png)

:::details 外部キーの制約（RESTRIC、CASCADE等）について

https://dbeaver.com/docs/dbeaver/25.3/Utilizing-Foreign-Keys/#create

|項目|説明|
|---|---|
|CASCADE|親テーブルの行が削除された場合、子テーブルの関連する行も自動的に削除されます。|
|RESTRICT|親テーブルの行を削除しようとしたときに、子テーブルに関連する行が存在する場合、削除操作は拒否されます。|
|NO ACTION|RESTRICTと同様に、関連する行が存在する場合は削除操作を拒否しますが、標準SQLの動作では、制約チェックがトランザクションの最後に行われる点が異なります。|
|SET NULL|親テーブルの行が削除された場合、子テーブルの関連する外部キー列の値をNULLに設定します。|
|SET DEFAULT|親テーブルの行が削除された場合、子テーブルの関連する外部キー列の値をデフォルト値に設定します。|

:::

最終的に以下のSQLが実行されます

```sql
CREATE TABLE public.orders (
	order_id smallint GENERATED BY DEFAULT AS IDENTITY NOT NULL,
	order_date date NOT NULL,
	user_id varchar NOT NULL,
	CONSTRAINT orders_pk PRIMARY KEY (order_id),
	CONSTRAINT orders_users_fk FOREIGN KEY (user_id) REFERENCES public.users(user_id) ON DELETE RESTRICT ON UPDATE CASCADE
);
```

#### 7.1.4.order_detailsテーブル

- 注文明細テーブル（交差テーブル）`order_details`テーブルを作成します
- 以下の項目を作成します

| colmun name | データタイプ | ヌルではない | keys |
|---|---|---|---|
| `order_id` | smallint | True | 主key |
| `product_id` | varchar | True | 主key |
| `quantiry` | smallint | True | - |

- 追加で`order_id`と`product_id`項目に`外部キー`を追加します [参照](#713ordersテーブル)
- 追加で`quantity`項目に`制約`(0以上の値のみ)を追加します [参照](#712productsテーブル)

最終的に以下のSQLが実行されます

```sql
CREATE TABLE public.order_details (
	order_id smallint NOT NULL,
	product_id varchar NOT NULL,
	quantity smallint NOT NULL,
	CONSTRAINT order_details_pk PRIMARY KEY (order_id,product_id),
	CONSTRAINT order_details_check CHECK (quantity > 0),
	CONSTRAINT order_id FOREIGN KEY (order_id) REFERENCES public.orders(order_id) ON DELETE RESTRICT ON UPDATE CASCADE,
	CONSTRAINT product_id FOREIGN KEY (product_id) REFERENCES public.products(product_id) ON DELETE RESTRICT ON UPDATE CASCADE
);
```

### 7.2.ER図の確認

DBeaverでは、作成したテーブルのER図（Entity-Relationship図）を自動生成できます。

1. 左のデータベースナビゲータで `テーブル` を右クリック > `View Diagram`をクリック
2. リレーションが線で結ばれた図が表示され、設計通りの正規化構造が視覚的に確認できます。

![viewdiagram](/images/rdb_practical/view_diagram.png)

![viewdiagramdetails](/images/rdb_practical/view_diagram_details.png)

## 8.SQL実践：データの投入

### 8.1.データの挿入（DML）

- テーブルにサンプルデータを投入します。
- public > 右クリック > SQLエディタ > SQLエディタ をクリック
- 以下のスクリプトを入力したのち SQL文を実行(`Ctrl + Enter`)する をクリック

```sql
-- ユーザーデータの挿入
INSERT INTO users (user_id, user_name, user_email) VALUES
    ('u_001', '田中太郎', 'tanaka@example.com'),
    ('u_002', '佐藤花子', 'sato@example.com'),
    ('u_003', '鈴木一郎', 'suzuki@example.com');
```
```sql
-- 商品データの挿入
INSERT INTO products (product_id, product_name, unit_price) VALUES
    ('A', 'りんご', 100),
    ('B', 'みかん', 50),
    ('C', 'バナナ', 200),
    ('D', 'ぶどう', 300);
```
```sql
-- 注文データの挿入
INSERT INTO orders (order_id, order_date, user_id) VALUES
    ( 1, '2025-01-01', 'u_001'),
    ( 2, '2025-01-02', 'u_002'),
    ( 3, '2025-01-03', 'u_001');
```
```sql
-- 注文明細データの挿入
INSERT INTO order_details (order_id, product_id, quantity) VALUES
    ( 1, 'A', 2),
    ( 1, 'B', 5),
    ( 2, 'C', 1),
    ( 3, 'A', 3),
    ( 3, 'D', 2);
```

実行を終えると以下のようにデータが入っているのが確認できます
![inserteddata](/images/rdb_practical/inserted_data_in_table.png)

- もし確認ができなかった場合は、アプリメニューのファイル > 更新をクリック
- もしくは、各テーブル名を右クリック > 更新をクリック

### 8.2.データの確認

投入したデータを確認してみましょう

```sql
-- 各テーブルの全データを取得
SELECT * FROM users;
SELECT * FROM products;
SELECT * FROM orders;
SELECT * FROM order_details;
```

## 9.SQL実践：データの検索

### 9.1.基本的なSELECT文

```sql
-- 特定のカラムのみ取得
SELECT user_name, user_email FROM users;

-- 条件付き検索（WHERE句）
SELECT * FROM products WHERE unit_price > 100;

-- 並び替え（ORDER BY句）
SELECT * FROM products ORDER BY unit_price DESC;
```

WHERE句結果

| product_id | product_name | unit_price |
|---|---|---|
| C | バナナ | 200 |
| D | ぶどう | 300 |

### 9.2.テーブルの結合（JOIN）

正規化されたテーブルの真価は、JOINによる結合で発揮されます。

注文一覧に「ユーザー名」を付与して表示

```sql
SELECT
    o.order_id,
    o.order_date,
    u.user_name,
    u.user_email
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id;
```

結果

| order_id | order_date | user_name | user_email |
|---|---|---|---|
| 1 | 2025-01-01 | 田中太郎 | tanaka@example.com |
| 2 | 2025-01-02 | 佐藤花子 | sato@example.com |
| 3 | 2025-01-03 | 田中太郎 | tanaka@example.com |

注文明細に「商品名」「単価」を付与し、小計を計算

```sql
SELECT
    od.order_id,
    p.product_name,
    p.unit_price,
    od.quantity,
    p.unit_price * od.quantity AS subtotal
FROM order_details od
INNER JOIN products p ON od.product_id = p.product_id;
```

結果

| order_id | product_name | unit_price | quantity | subtotal |
|---|---|---|---|---|
| 1 | りんご | 100 | 2 | 200 |
| 1 | みかん | 50 | 5 | 250 |
| 2 | バナナ | 200 | 1 | 200 |
| 3 | りんご | 100 | 3 | 300 |
| 3 | ぶどう | 300 | 2 | 600 |

注文ごとの合計金額を算出

```sql
SELECT
    o.order_id,
    o.order_date,
    u.user_name,
    SUM(p.unit_price * od.quantity) AS total_amount
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
GROUP BY o.order_id, o.order_date, u.user_name
ORDER BY o.order_id;
```

結果

| order_id | order_date | user_name | total_amount |
|---|---|---|---|
| 1 | 2025-01-01 | 田中太郎 | 450 |
| 2 | 2025-01-02 | 佐藤花子 | 200 |
| 3 | 2025-01-03 | 田中太郎 | 900 |

### 9.3.JOINの種類

| JOIN種類 | 説明 |
|---|---|
| INNER JOIN | 両方のテーブルに一致するデータのみ取得 |
| LEFT JOIN | 左テーブルの全データ＋右テーブルの一致データを取得 |
| RIGHT JOIN | 右テーブルの全データ＋左テーブルの一致データを取得 |
| FULL OUTER JOIN | 両方のテーブルの全データを取得 |

### 9.4.集計関数

商品ごとの販売数量合計

```sql
SELECT
    p.product_name,
    SUM(od.quantity) AS total_quantity
FROM order_details od
INNER JOIN products p ON od.product_id = p.product_id
GROUP BY p.product_name
ORDER BY total_quantity DESC;
```

結果

| product_name | total_quantity |
|---|---|
| りんご | 5 |
| みかん | 5 |
| ぶどう | 2 |
| バナナ | 1 |

ユーザーごとの注文回数

```sql
SELECT
    u.user_name,
    COUNT(o.order_id) AS order_count
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_name;
```

結果

| user_name | oder_count |
|---|---|
| 田中太郎 | 2 |
| 鈴木一郎 | 0 |
| 佐藤花子 | 1 |

## 10.SQL実践：データの更新と削除

### 10.1.データの更新（UPDATE）

```sql
-- 商品の単価を変更
UPDATE products
SET unit_price = 120
WHERE product_id = 'A';

-- 確認
SELECT * FROM products WHERE product_id = 'A';
```

結果

| product_id | product_name | unit_price |
|---|---|---|
| A | りんご | 120 |

### 10.2.データの削除（DELETE）

```sql
-- 特定の注文明細を削除
DELETE FROM order_details
WHERE order_id = 3 AND product_id = 'D';

-- 確認
SELECT * FROM order_details WHERE order_id = 3;
```

結果

| order_id | product_id | quantity |
|---|---|---|
| 3 | A | 3 |

:::message alert
**削除操作の注意点**
`WHERE`句を付け忘れると、テーブルの全データが削除されます。本番環境では必ずトランザクション内で実行し、`COMMIT`前に確認することを習慣づけましょう。
:::

## 11.トランザクションの基礎

### 11.1.トランザクションとは

トランザクションとは、「複数のSQL操作をひとまとめにして、すべて成功するか、すべて失敗するか」を保証する仕組みです。
設計編で触れた「注文確定と在庫減少の矛盾を防ぐ」というRDBの強みは、このトランザクション機能によって実現されています。

### 11.2.基本的な使い方

```sql
-- トランザクション開始
BEGIN;

-- 注文を追加
INSERT INTO orders (order_id, order_date, user_id)
VALUES (4, '2025-01-04', 'u_003');

-- 注文明細を追加
INSERT INTO order_details (order_id, product_id, quantity)
VALUES (4, 'B', 10);

-- 問題なければ確定
COMMIT;

-- 問題があればロールバック（取り消し）
-- ROLLBACK;
```

orders

| order_id | order_date | user_id |
| -------- | ---------- | ------- |
| 1        | 2025-01-01 | u_001   |
| 2        | 2025-01-02 | u_002   |
| 3        | 2025-01-03 | u_001   |
| 4        | 2025-      | U_003   |

order_details

| order_id | product_id | quantity |
| -------- | ---------- | -------- |
| 1        | A          | 2        |
| 1        | B          | 5        |
| 2        | C          | 1        |
| 3        | A          | 3        |
| 4        | B          | 10       |

order_id: 3, product_id: D の注文は削除済み

### 11.3.ACID特性

トランザクションが保証する4つの特性を **ACID** と呼びます。

| 特性 | 英語 | 意味 |
|---|---|---|
| 原子性 | Atomicity | 処理は「全部成功」か「全部失敗」のいずれか |
| 一貫性 | Consistency | トランザクション前後でデータの整合性が保たれる |
| 独立性 | Isolation | 同時実行中のトランザクションが互いに干渉しない |
| 永続性 | Durability | 確定した変更は障害が発生しても失われない |

## 12.クリーンアップ（リソースの削除）

学習が終わったら、不要なAWSリソースを削除してコストを抑えましょう。

:::message alert
**課金に注意**
Auroraは起動している間、継続的に課金が発生します。学習を中断する際は、クラスターを **停止** するか **削除** することを忘れずに。
:::

## 13.ローカル環境のみで試す方法

- AWS環境を構築せず、ローカル環境のみで試したい場合は、以下の手順でMockデータ環境を構築できます。
- 仕事用以外にAWSアカウントを持っていない場合や、一時的に試したい場合に便利です

### 13.1.Mockデータ構築

- Windows環境では、WSL2, DockerDesktopをインストールしておく
- BE用プロジェクトディレクトリを作成する 例:`./backend_postgresql_db`
- ディレクトリに`docker-compose.yml`を作成し以下を記載

```yml
version: "3.9"

services:
  postgres:
    image: postgres:16
    container_name: postgres-local
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: sample_db
      POSTGRES_USER: sample_user
      POSTGRES_PASSWORD: sample_pass
      TZ: Asia/Tokyo
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  pgdata:
```

- PowerShellやコマンドプロンプトで`docker compose up -d`コマンドを実行（Docker desktopが起動していること）

![dockercompose](/images/rdb_practical/docker_compose_up.png)

- `docker ps`でデータ確認するかDockerDesktopの画面で以下のような情報が確認できればOK

![dockerps](/images/rdb_practical/docker_ps.png)

- DBeaverを起動し、新しい接続 > PostgreSQlを選択

![dbeavercreatenewconnection](/images/rdb_practical/dbeaver_create_new_connection.png)
- `docker-compose.yml`の設定を参照し接続情報を入力

![dbeaversettingconfig](/images/rdb_practical/dbeaver_setting_config.png)

| 項目 | 設定値 |
| --- | --- |
| Host | localhost |
| Port | 5432 |
| Database | sample_db |
| Username | sample_user |
| Password | sample_pass |

- テスト接続をクリック
- `ドライバファイルをダウンロードする`が出現した場合はダウンロードを行う

![downloaddriver](/images/rdb_practical/download_driver.png)

- 接続テストの完了を確認し、OKをクリック

![completeconnecttest](/images/rdb_practical/complete_connect_test.png)

![createddatabase](/images/rdb_practical/created_database.png)


## 14.おわりに

本記事では、設計編で学んだ正規化の概念を、実際にAWS Aurora上で動かすところまで実践しました。

**振り返り**
- DBeaverを使ったGUIベースのDB操作で、学習のハードルを下げられる
- JOINを使うことで、正規化されたテーブル間のデータを柔軟に結合できる
- トランザクションにより、データの整合性を保証できる
- 「設計」と「実装」の両輪が揃って初めて、RDBを武器として扱えるようになります
- 「とりあえず動く」から「意図して動かす」エンジニアを目指して、引き続き学習を進めていきましょう

## 15.参考リンク

- [Amazon Aurora ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html)
- [DBeaver 公式サイト](https://dbeaver.io/)
- [PostgreSQL 公式ドキュメント](https://www.postgresql.org/docs/)
- [SQL入門 - Progate](https://prog-8.com/courses/sql)
- [サクッと始めるデータベース構築【SQL / NoSQL / newSQL】](https://zenn.dev/umi_mori/books/331c0c9ef9e5f0/viewer/992632)

