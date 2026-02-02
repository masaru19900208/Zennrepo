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

- リレーショナルデータベースを実際に触ってみたい方
- 「とりあえず動く」データベースから、保守性の高い設計へステップアップしたい方
- AWS上でのRDB構築に興味がある方

## 3.記事を読むメリット

- AWSマネジメントコンソールを用いたAurora構築の流れを把握できる
- DBeaverを使った接続方法と基本操作を習得できる
- SQLの基本（DDL/DML）を、前回学んだ正規化されたテーブル構造で実践できる

## 4.開発環境構築

本記事で使用する環境は以下の通りです。

### 4.1.使用ツール・サービス

| 項目 | 使用するもの | 用途 |
| --- | --- | --- |
| データベース | Amazon Aurora (PostgreSQL互換) | クラウド上のRDB |
| クライアントツール | DBeaver Community Edition | DBへの接続・SQL実行 |
| OS | Windows 11 | 開発マシン |

### 4.2.DBeaverとは

https://dbeaver.io/

- DBeaverは、無料で利用できるデータベース管理ツール（GUIクライアント）です
- MySQL、PostgreSQL、Oracle、SQL Serverなど、ほとんどの主要なデータベースに対応
- ER図（テーブル間のリレーション図）を自動生成できます

#### 類似ツール

無償で全機能利用可能という点でDBeaverを選定したが、以下のような類似ツールもあります。

- TablePlus: https://tableplus.com/
- Beekeeper Studio: https://www.beekeeperstudio.io/ja/
- HeidiSQL: https://www.heidisql.com/
- DbGate: https://www.dbgate.io/ja/
- DataGrip: https://www.jetbrains.com/ja-jp/datagrip/

## 5.DBeaverを使ったローカル環境での開発方法

### 5.1.Mockデータ構築方法

- DBeaver起動時、もしくはヘルプ から `Create sample database` をクリック
- 

## .AWS Aurora (PostgreSQL互換) の構築

### 5.1.なぜAuroraを選ぶのか

設計編でも触れましたが、AWS環境でRDBを使うなら **Amazon Aurora** が推奨されます。

| 観点 | Aurora のメリット |
| --- | --- |
| パフォーマンス | MySQL/PostgreSQLの数倍の処理速度 |
| 可用性 | 自動フェイルオーバー、複数AZへのレプリケーション |
| 運用負荷 | 自動バックアップ、パッチ適用はAWSが管理 |
| スケーラビリティ | Aurora Serverless v2で負荷に応じた自動スケーリング |

今回は学習目的のため、コストを抑えた構成で構築します。

### 5.2.VPCとセキュリティグループの準備

Auroraを構築する前に、ネットワーク周りの設定を確認します。

**VPC（Virtual Private Cloud）とは？**
AWSクラウド内に作成する仮想ネットワークです。データベースはこのVPC内に配置され、外部からの不正アクセスを防ぎます。

**セキュリティグループとは？**
ファイアウォールのような役割を持ち、「どのIPアドレスから」「どのポート番号で」アクセスを許可するかを制御します。

#### 5.2.1.セキュリティグループの作成

1. AWSマネジメントコンソールで「VPC」を検索し、VPCダッシュボードを開く
2. 左メニューから「セキュリティグループ」を選択
3. 「セキュリティグループを作成」をクリック
4. 以下を設定：

| 項目 | 設定値 |
|---|---|
| セキュリティグループ名 | `aurora-postgres-sg` |
| 説明 | `Security group for Aurora PostgreSQL` |
| VPC | デフォルトVPCまたは作成済みのVPC |

5. インバウンドルールを追加：

| タイプ | ポート範囲 | ソース |
|---|---|---|
| PostgreSQL | 5432 | マイIP（自分のIPアドレス） |

:::message alert
**セキュリティに関する注意**
学習目的であっても、ソースを「0.0.0.0/0（どこからでもアクセス可能）」に設定するのは避けてください。不正アクセスのリスクが高まります。
:::

### 5.3.Auroraクラスターの作成

1. AWSマネジメントコンソールで「RDS」を検索し、RDSダッシュボードを開く
2. 「データベースの作成」をクリック
3. 以下の設定で作成：

#### 5.3.1.エンジンの選択

| 項目 | 設定値 |
|---|---|
| エンジンのタイプ | Amazon Aurora |
| エディション | Amazon Aurora PostgreSQL互換エディション |
| エンジンバージョン | 最新の安定版（例：Aurora PostgreSQL 15.x） |

#### 5.3.2.テンプレート

| 項目 | 設定値 |
|---|---|
| テンプレート | 開発/テスト（コスト削減のため） |

#### 5.3.3.設定

| 項目 | 設定値 |
|---|---|
| DBクラスター識別子 | `my-learning-aurora` |
| マスターユーザー名 | `postgres`（デフォルト） |
| マスターパスワード | 任意の強力なパスワード |

:::message
パスワードは後でDBeaverからの接続時に使用します。忘れないようにメモしておきましょう。
:::

#### 5.3.4.インスタンスの設定

| 項目 | 設定値 |
|---|---|
| DBインスタンスクラス | `db.t3.medium`または`db.t4g.medium`（学習用） |

#### 5.3.5.可用性と耐久性

| 項目 | 設定値 |
|---|---|
| マルチAZ配置 | Auroraレプリカを作成しない（コスト削減） |

#### 5.3.6.接続

| 項目 | 設定値 |
|---|---|
| VPC | 先ほどセキュリティグループを作成したVPC |
| パブリックアクセス | はい（学習用。本番環境では「いいえ」推奨） |
| VPCセキュリティグループ | 先ほど作成した`aurora-postgres-sg` |

#### 5.3.7.データベース認証

| 項目 | 設定値 |
|---|---|
| データベース認証 | パスワード認証 |

4. 「データベースの作成」をクリック

作成完了まで5〜10分程度かかります。ステータスが「利用可能」になるまで待ちましょう。

### 5.4.エンドポイントの確認

Auroraクラスターが作成されたら、接続に必要な **エンドポイント** を確認します。

1. RDSダッシュボードで作成したクラスターを選択
2. 「接続とセキュリティ」タブを開く
3. 「エンドポイント」セクションから **ライターエンドポイント** をコピー

例：`my-learning-aurora.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com`

## 6.DBeaverからAuroraへの接続

### 6.1.接続設定

1. DBeaverを起動
2. 左上の「新しい接続」アイコン（プラグマーク）をクリック
3. 「PostgreSQL」を選択し「次へ」

4. 接続情報を入力：

| 項目 | 設定値 |
|---|---|
| Host | Auroraのライターエンドポイント |
| Port | 5432 |
| Database | postgres（デフォルト） |
| Username | postgres |
| Password | Aurora作成時に設定したパスワード |

5. 「テスト接続」をクリックし、接続が成功することを確認
6. 「完了」をクリック

:::message
初回接続時に「ドライバーのダウンロード」を求められたら、「ダウンロード」をクリックして自動インストールしてください。
:::

### 6.2.接続できない場合のトラブルシューティング

| 症状 | 確認ポイント |
|---|---|
| タイムアウトする | セキュリティグループのインバウンドルールに自分のIPが正しく設定されているか確認 |
| 認証エラー | ユーザー名・パスワードが正しいか確認 |
| ホストが見つからない | エンドポイントのコピーミスがないか確認 |

## 7.SQL実践：データベース・テーブルの作成

ここからは、設計編で正規化した「注文管理システム」のテーブルを実際に作成していきます。

### 7.1.データベースの作成

まず、学習用のデータベースを作成します。

```sql
-- データベースの作成
CREATE DATABASE order_management;
```

作成後、DBeaverで接続先データベースを `order_management` に切り替えます。
（接続を右クリック → 「データベースを編集」または新規接続を作成）

### 7.2.テーブルの作成（DDL）

設計編で正規化した4つのテーブルを作成します。

```sql
-- ユーザーテーブル
CREATE TABLE users (
    user_id VARCHAR(10) PRIMARY KEY,
    user_name VARCHAR(50) NOT NULL,
    user_email VARCHAR(100) NOT NULL UNIQUE
);

-- 商品テーブル
CREATE TABLE products (
    product_id VARCHAR(10) PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    unit_price INTEGER NOT NULL CHECK (unit_price > 0)
);

-- 注文テーブル
CREATE TABLE orders (
    order_id VARCHAR(10) PRIMARY KEY,
    order_date DATE NOT NULL,
    user_id VARCHAR(10) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- 注文明細テーブル（交差テーブル）
CREATE TABLE order_details (
    order_id VARCHAR(10),
    product_id VARCHAR(10),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

:::message
**DDL（Data Definition Language）とは？**
テーブルの「構造」を定義するためのSQL文です。`CREATE`、`ALTER`、`DROP`などが該当します。
:::

### 7.3.SQLの構文解説

上記のSQLで使用した主要な構文を解説します。

| 構文 | 意味 |
|---|---|
| `PRIMARY KEY` | 主キー。テーブル内で一意に行を特定するためのキー |
| `NOT NULL` | NULL（空）を許容しない制約 |
| `UNIQUE` | 重複する値を許容しない制約 |
| `CHECK` | 値の条件を指定する制約（例：0より大きい） |
| `FOREIGN KEY` | 外部キー。他テーブルの主キーを参照し、リレーションを形成 |
| `REFERENCES` | 外部キーが参照する先のテーブルとカラムを指定 |

### 7.4.ER図の確認

DBeaverでは、作成したテーブルのER図（Entity-Relationship図）を自動生成できます。

1. 左のデータベースナビゲータで `order_management` を展開
2. 「テーブル」を右クリック
3. 「ER図を表示」を選択

リレーションが線で結ばれた図が表示され、設計編で考えた正規化の構造が視覚的に確認できます。

## 8.SQL実践：データの投入

### 8.1.データの挿入（DML）

テーブルにサンプルデータを投入します。

```sql
-- ユーザーデータの挿入
INSERT INTO users (user_id, user_name, user_email) VALUES
    ('u_001', '田中太郎', 'tanaka@example.com'),
    ('u_002', '佐藤花子', 'sato@example.com'),
    ('u_003', '鈴木一郎', 'suzuki@example.com');

-- 商品データの挿入
INSERT INTO products (product_id, product_name, unit_price) VALUES
    ('A', 'りんご', 100),
    ('B', 'みかん', 50),
    ('C', 'バナナ', 200),
    ('D', 'ぶどう', 300);

-- 注文データの挿入
INSERT INTO orders (order_id, order_date, user_id) VALUES
    ('001', '2025-01-01', 'u_001'),
    ('002', '2025-01-02', 'u_002'),
    ('003', '2025-01-03', 'u_001');

-- 注文明細データの挿入
INSERT INTO order_details (order_id, product_id, quantity) VALUES
    ('001', 'A', 2),
    ('001', 'B', 5),
    ('002', 'C', 1),
    ('003', 'A', 3),
    ('003', 'D', 2);
```

:::message
**DML（Data Manipulation Language）とは？**
データの「操作」を行うためのSQL文です。`INSERT`、`UPDATE`、`DELETE`、`SELECT`などが該当します。
:::

### 8.2.データの確認

投入したデータを確認してみましょう。

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
SELECT * FROM products WHERE unit_price >= 100;

-- 並び替え（ORDER BY句）
SELECT * FROM products ORDER BY unit_price DESC;
```

### 9.2.テーブルの結合（JOIN）

正規化されたテーブルの真価は、JOINによる結合で発揮されます。

```sql
-- 注文一覧に「ユーザー名」を付与して表示
SELECT 
    o.order_id,
    o.order_date,
    u.user_name,
    u.user_email
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id;
```

```sql
-- 注文明細に「商品名」「単価」を付与し、小計を計算
SELECT 
    od.order_id,
    p.product_name,
    p.unit_price,
    od.quantity,
    p.unit_price * od.quantity AS subtotal
FROM order_details od
INNER JOIN products p ON od.product_id = p.product_id;
```

```sql
-- 注文ごとの合計金額を算出
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

### 9.3.JOINの種類

| JOIN種類 | 説明 |
|---|---|
| INNER JOIN | 両方のテーブルに一致するデータのみ取得 |
| LEFT JOIN | 左テーブルの全データ＋右テーブルの一致データを取得 |
| RIGHT JOIN | 右テーブルの全データ＋左テーブルの一致データを取得 |
| FULL OUTER JOIN | 両方のテーブルの全データを取得 |

### 9.4.集計関数

```sql
-- 商品ごとの販売数量合計
SELECT 
    p.product_name,
    SUM(od.quantity) AS total_quantity
FROM order_details od
INNER JOIN products p ON od.product_id = p.product_id
GROUP BY p.product_name
ORDER BY total_quantity DESC;

-- ユーザーごとの注文回数
SELECT 
    u.user_name,
    COUNT(o.order_id) AS order_count
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_name;
```

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

### 10.2.データの削除（DELETE）

```sql
-- 特定の注文明細を削除
DELETE FROM order_details 
WHERE order_id = '003' AND product_id = 'D';

-- 確認
SELECT * FROM order_details WHERE order_id = '003';
```

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
VALUES ('004', '2025-01-04', 'u_003');

-- 注文明細を追加
INSERT INTO order_details (order_id, product_id, quantity) 
VALUES ('004', 'B', 10);

-- 問題なければ確定
COMMIT;

-- 問題があればロールバック（取り消し）
-- ROLLBACK;
```

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

### 12.1.Auroraクラスターの削除

1. RDSダッシュボードで作成したクラスターを選択
2. 「アクション」→「削除」を選択
3. 最終スナップショットの作成有無を選択（学習用なら不要）
4. 確認入力を行い削除

### 12.2.セキュリティグループの削除

1. VPCダッシュボードでセキュリティグループを選択
2. 作成した`aurora-postgres-sg`を削除

:::message alert
**課金に注意**
Auroraは起動している間、継続的に課金が発生します。学習を中断する際は、クラスターを **停止** するか **削除** することを忘れずに。
:::

## 13.おわりに

本記事では、設計編で学んだ正規化の概念を、実際にAWS Aurora上で動かすところまで実践しました。

**振り返り**
- DBeaverを使ったGUIベースのDB操作で、学習のハードルを下げられる
- AWS Auroraの構築は、セキュリティグループの設定がポイント
- JOINを使うことで、正規化されたテーブル間のデータを柔軟に結合できる
- トランザクションにより、データの整合性を保証できる

「設計」と「実装」の両輪が揃って初めて、RDBを武器として扱えるようになります。
ぜひ本記事のSQLを実際に実行し、自分なりにアレンジしながら理解を深めてください。

「とりあえず動く」から「意図して動かす」エンジニアを目指して、引き続き学習を進めていきましょう。

## 14.参考リンク

- [Amazon Aurora ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html)
- [DBeaver 公式サイト](https://dbeaver.io/)
- [PostgreSQL 公式ドキュメント](https://www.postgresql.org/docs/)
- [SQL入門 - Progate](https://prog-8.com/courses/sql)

