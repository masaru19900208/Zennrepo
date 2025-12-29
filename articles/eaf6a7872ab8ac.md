---
title: "FastAPIについてまとめてみた"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "fastapi", "backend"]
published: false
---


## 1.はじめに

こんにちは。
Python Backendで利用されるFastAPIについてドキュメントを参考にまとめを行いました。
私は普段、社内限定の小さいPoCや現場の業務改善に取り組みを行っております。
各Projectは非常に小さいので、BE構成はサーバーレスがほとんどで、稀にNode.js+Expressの構成を取ります。
将来的に大きなProjectへ参画したく休みの期間にInputをまとめておく目的で本記事を記載します。

## 2.対象読者

- FastAPIの導入を検討している
- FastAPIの初学者
- Pythonについての理解がある程度ある

## 3.記事を読むメリット

- 公式ドキュメントからの抜粋で概要と要約を日本語で読める

## 4.FastAPIについて

### 4.1.概要

https://fastapi.tiangolo.com/
https://github.com/fastapi/fastapi

その名の通り、主にバックエンドのAPI実装で高速かつ安全な開発を行うためのWebフレームワーク。

開発者から特に選ばれるポイントは以下
- Python最速クラスのパフォーマンスを誇りNode.jsやGOに匹敵する速度を誇るWebフレームワーク
- 型安全性による開発効率の向上
- ドキュメントの完全自動生成

#### 4.1.1.処理速度能力について

---

公式では以下のように記される
> Fast: Very high performance, on par with NodeJS and Go (thanks to Starlette and Pydantic).
[Link](https://fastapi.tiangolo.com/#:~:text=Fast%3A%20Very%20high%20performance%2C%20on%20par%20with%20NodeJS%20and%20Go%20(thanks%20to%20Starlette%20and%20Pydantic).)

この「速さ」について実際に比較した値を確認

##### 確認方法
TechEmpower Web Framework Benchmarks
https://www.techempower.com/benchmarks/#section=data-r23&test=db&l=gcuscf-p31

##### 前提条件
- 比較対象言語： JavaScript, TypeScript, Go, Python
- Classification（フレームワーク分類）：Micro（ルーティング等の最低限機能を持つがDB操作などのORM[^1]は標準で含まない）のみ

##### 比較対象のフレームワーク結果
![benchmark result](/images/fastapi/benchmark.png)

同分類の著名（と思われる）フレームワークとの比較し、GO言語のプロジェクトより速いケースもあり、Node.jsやGO言語でによるフレームワークに匹敵する「速さ」があると言える。

|順位|フレームワーク|言語|パフォーマンススコア|スコア比較値(FastAPI基準)|Github Star|
|---|---|---|---|---|---|
|13|Echo|GO|432,423|263.8%|32k|
|19|Gin|GO|238,344|145.4%|87.5k|
|24|express'[postgres.js]'|JS|214,177|130.7%|68.4k|
|41|FastAPI|Python|163,894|100%|93.5k|
|56|Falcon|Python|106,883|100%|9.8k|
|65|nestjs|JS|84,218|51.4%|74.1k|

:::details 4.1.2.FastAPIを選ぶ理由

Pythonかつ同分類に限った話で、fastAPIよりベンチマーク結果が高い（ハイパフォーマンス）結果だったフレームワークは複数存在する。

|順位|フレームワーク|パフォーマンススコア|スコア比較値(FastAPI基準)|Github|
|---|---|---|---|---|
|20|fastwsgi|231.555|141.3%|https://github.com/jamesroberts/fastwsgi|
|22|BlackSheep|222,590|135.8%|https://github.com/Neoteroi/BlackSheep|
|28|Panther|211,719|129.2%|https://github.com/AliRn76/Panther|
|35|apidaora|186,082|113.5%|https://github.com/dutradda/apidaora|
|36|starlette|178,268|108.8%|https://github.com/Kludex/starlette|
|38|emmett55|171,116|104.4%|https://github.com/emmett-framework/emmett55|

パフォーマンススコアのみで比較すると上記に挙げる他のフレームワークが選択肢に入れても良さそうだがGithub Star数でFastAPIが圧倒している理由を簡単に調査。

##### 結論

- 機能差
- 開発の活発性

##### 調査してわかったこと

- いくつかのフレームワークは開発が終了している（脆弱性対応が見込めない等）。
- 開発が現在（2025/12）も継続しており、Star数から見て比較的利用されているフレームワークは以下２点
  - BlackSheep
  - Starlette

各フレームワークを選択しない理由

|No|フレームワーク|不採用理由|
|---|---|---|
|1|BlackSheep|2.3kのStarを持つフレームワークで開発が現在も行われているもののあまり活発とは言えない状況（稼働が月に数回、かつCommitユーザーは１人という時もある|
|2|Starlette|11.8kのStarを持ちBlackSheepと比較すると開発も活発。そもそもFastAPIがStarletteを継承し開発を行っている。つまりStarletteに機能追加されたものがFastAPIと言えるため機能差がある（型ヒント駆動の入出力定義、バリデーション、依存性注入、OpenAPI生成、Swagger/ReDocなど）|

:::


### 4.2.インストールと開発環境

---

#### 4.2.1.検証環境
|No|項目|値|
|---|---|---|
|1|OS|Windows11|
|2|Python|3.14.2|


#### 4.2.2.仮想環境構築からインストールまで（コマンドプロンプト）

```shell
python -m venv .venv
.\.venv\Scripts\activate

python -m pip install --upgrade pip
pip install "fastapi[standard]"
```

- 公式の[チュートリアル](https://fastapi.tiangolo.com/virtual-environments/)でも丁寧な解説がある
- `"fastapi[standard]"`については以下の記事が参考になりました

https://zenn.dev/ryuu/scraps/139081511afdb4


#### 4.2.3.最小限構成起動

main.pyを作成し以下を記述

```python:main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

上書き保存した後に以下のコマンド実行で起動

```shell
fastapi dev main.py
```

以下のURLでサーバーレスポンスやドキュメントが生成されていることを確認できる。

|No|URL|表示内容|
|---|---|---|
|1|http://127.0.0.1:8000/|`{"message": "Hello World"}`が表示される|
|2|http://127.0.0.1:8000/docs|Swagger形式の自動生成OpenAPIドキュメントが表示される|
|3|http://127.0.0.1:8000/redoc|OpenAPIドキュメントから生成されるReDocドキュメントが表示される|
|4|http://127.0.0.1:8000/openapi.json|OpenAPIドキュメントの生json|


使いそうなfastapiコマンドオプションテーブル
|No|Optiohn|指定方法|意味|例|
|---|---|---|---|---|
|1|--host|text|バインドするIPアドレス|`fastapi main.py --host 0.0.0.0`|
|2|--port|int|待ち受けるポート番号|`fastapi main.py --port 3000`|

### 4.3.基本的な使用方法

---
#### 4.3.1.HTTPメソッド（operation）

Pythonメソッドのデコレーターで定義する

```python
@app.get("/")
def root():
    return {"message": "Hello World"}

@app.post("/items/")
async def create_item(item: Item):
    return item

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item):
    return {"item_name": item.name, "item_id": item_id}

@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    return {"item_id": item_id}

```

#### 4.3.2.パス

- デコレーター内で指定
- 動的値(`id`等)が入るパスは`{}`で記載する

```python
@app.post("/items/")
async def create_item(item: Item):
    return item

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item):
    return {"item_name": item.name, "item_id": item_id}
```

##### 4.3.2.1.同層で固定パスと変動パスが同居

動的なパスに加えて同階層に固定のパスがある場合（`/items/me`等）メソッドを定義する順番で上から処理されるので注意。
つまり `/items/me`を`/items/{item_id}`より先に宣言する必要がある

##### 4.3.2.2.パス全体の受け取り

<!-- TODO おかしいので修正 -->

以下のようにすることで`/files`以降のパスを一括で引き数に受け取れる
つまり`/files/hoge/foo/test`というリクエストをしたケースにおいて
- `:path`を宣言しない場合
  - `@app.get("/files/hoge/foo/test")`のデコレーターを持つ関数がないとエラーになる
- `:path`を宣言した場合
  - `@app.get("/files/file-path:path")`のデコレーターを持つ関数の引数で`hoge/foo/test`を受け取れる

```python
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    return {"file_path": file_path}

```

#### 4.3.3.型指定

<!-- ここら辺からバリデーションでまとめてしまう -->

各メソッドの宣言時は型宣言をすることで以下の恩恵が受けられる
- /docs及び/redocで生成されるドキュメント
- 実際のサーバー処理におけるバリデーション

##### 4.3.3.1.Enumの利用

パス指定で`id`のようなランダム値ではなく、ある程度数えられる選択肢の範囲内で宣言したい場合に利用する

```python
from enum import Enum

class ModelName(str, Enum):
    bird = "Bird"
    cat = "Cat"
    dog = "Dog"

@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    if model_name == ModelName.bird:
        return {"model_name": model_name, "message": "pi pi pi"}

    if model_name.value == "Cat":
        return {"model_name": model_name, "message": "meow"}

    return  {"model_name": model_name, "message": "bow wow"}
```

#### 4.3.4.クエリ

クエリパラメータを受けるけるAPIエンドポイント

```python
animal = [{"Bird": "pi pi pi"}, {"Cat": "meow"}, {"Dog": "bow wow"}]

@app.get("/animals/")
async def read_item(filter: str | None = None):
    if filter is None:
        return animal
    return [item for item in animal if filter in item]
```

- queryに`int`等の数値が含まれる場合はURLなので`str`型としてデータが渡されるがPython型で宣言してあると内部で自動的に`int`に変換され自身でキャストする必要はなくなる（バリデーション等も一緒）
- `bool`型をqueryで受け取ることも可能でリクエストでは`/animals?is_mammalian=True`,`/animals?is_mammalian=true`, `/animals?is_mammalian=on`等のリクエストにより、内部で自動的に変換をしてくれる
- queryパラメータはデフォルト値を設定しない場合、必須queryパラメータとなる。

#### 4.3.5.リクエスト

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.post("/items/")
async def create_item(item: Item):
    return item
```
##### 4.3.5.1.pydantic.BaseModelについて

- pydantic.BaseModelはEnum以外でデータ型クラスを作成する際にPOSTやPUT以外でも利用される。(例:GETにおいてデータベースから取得するデータレコードのデータ型宣言)
- Pythonのみを使用したデータ型クラスと比較し様々な機能がある（以下代表例）

|No|メソッド|機能|Docs|
|---|---|---|---|
|1|module_dump()|受け取ったデータをモデルに合わせて`dict`型に変換|[Docs](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump)|
|2|module_dump_json()|受け取ったデータをモデルに合わせてJSONの`str`型に変換|[Docs](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump_json)|

- 他にもバリデーションやドキュメント生成の自動はは他に同じく実施される

##### 4.3.5.2.明示的なバリデーション

###### クエリパラメータにおいて型宣言以上にバリデーションする方法

インポート
```python
from typing import Annotated
from fastapi import FastAPI, Query
```

文字数
```python
@app.get("/items/")
async def read_items(q: Annotated[str | None, Query(min_length=3, max_length=50)] = None)
```

正規表現
```python
@app.get("/items/")
async def read_items(q: Annotated[str | None, Query(pattern="^[b|B]ird$")] = None)
```

###### 再利用性の高いバリデーション

インポート
```python
from typing import Annotated, Literal
from fastapi import FastAPI, Query
from pydantic import BaseModel, Field
```
```python
class FilterParams(BaseModel):
    limit: int = Field(100, gt=0, le=100)
    offset: int = Field(0, ge=0)
    order_by: Literal["created_at", "updated_at"] = "created_at"
    tags: list[str] = []

@app.get("/items/")
async def read_items(filter_query: Annotated[FilterParams, Query()]):
    return filter_query
```
`Field(100, gt=0, le=100)` == Field(デフォルト値`100`、`0`より大きい、`100`以下)
Literal

|演算子|略称|英語|
|---|---|---|
|=|eq|equal to|
|!=|ne|not equal to|
|<|lt|less than|
|<=|le|less than or equal to|
|>|gt|greater than|
|<=|ge|greater than or equal to|

###### パスパラメータのバリデーション

インポート
```python
from typing import Annotated
from fastapi import FastAPI, Query, Path

```

比較演算(クエリパラメータでも利用可)
```python
@app.get("/items/{item_id}")
async def read_items(item_id: Annotated[int, Path(ge=1, le=1000)])
```

###### Bodyのバリデーション

Request Bodyが以下のようになっているケース
```json
{
    "item": {
        "name": "Foo",
        "description": "The pretender",
        "price": 42.0,
        "tax": 3.2,
        "tags": ["rock", "metal", "bar"],
        "images": [
            {
                "url": "http://example.com/baz.jpg",
                "name": "The Foo live"
            }
        ]
    },
    "user": {
        "username": "dave",
        "full_name": "Dave Grohl"
    },
    "importance": 5
}
```
インポート
```python
from typing import Annotated
from fastapi import FastAPI, Query, Body
from pydantic import BaseModel, Field, HttpUrl
```
バリデーション付き
```python

class Image(aseModel): #サブモデル
    url: HttpUrl #カスタムタイプ
    name: str

class Item(BaseModel):
    name: str
    description: str | None = Field(default=None, title="The description of the item", max_length=300, examples=["A very nice Item"]) #バリデーション + 生成されるjsonスキーマに適用されるサンプルデータ
    price: float = Field(gt=0, description="The price must be greater than zero") #バリデーション
    tax: float | None = None
    tags: set[str] = set() #一意の文字列配列
    images: list[Image] | None = None #サブモデル

    model_config = {  # 生成されるjsonスキーマに適用されるサンプルデータ
        "json_schema_extra": {
            "examples": [
                {
                    "name": "Foo",
                    "description": "A very nice Item",
                    "price": 35.4,
                    "tax": 3.2,
                }
            ]
        }
    }

class User(BaseModel):
    username: str
    full_name: str | None = None

@app.put("/items/{item_id}")
async def update_item(
    item_id: int, item: Item, user: User, importance: Annotated[int, Body()]
):
    results = {"item_id": item_id, "item": item = Body(   # デフォルト値の仕組み利用によるSwagger向けの詳細指示
        openapi_examples={
            "Normal": {
                "summary": "通常の例",
                "description": "標準的な商品の例です。",
                "value": {"name": "Foo", "price": 100},
            },
        }
    ), "user": user, "importance": importance}
    return results
```
カスタムタイプの例
- [HttpUrl](https://docs.pydantic.dev/latest/api/networks/#pydantic.networks.HttpUrl)
- [EmailStr](https://docs.pydantic.dev/latest/api/networks/#pydantic.networks.EmailStr)

Bodyがシンプルである場合は以下のようにする
```json
{
    "item": {
        "name": "Foo",
        "description": "The pretender",
        "price": 42.0,
        "tax": 3.2
    }
}
```
```python
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.put("/items/{item_id}")
async def update_item(
    item_id: int, item: Annotated[Item, Body(embed=True)]):
    results = {"item_id": item_id, "item": item, "user": user, "importance": importance}
    return results
```

仮に`Body(embed=True)`を利用しない場合は以下のようにする必要がある
```python
class Item(BaseModel):
    Item: Item_child

class Item_child(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.put("/items/{item_id}")
async def update_item(
    item_id: int, item: Item):
    results = {"item_id": item_id, "item": item, "user": user, "importance": importance}
    return results
```

`embed=True`による階層の省略は１段階のみである


###### カスタムバリデーション

```python
from pydantic import AfterValidator
```

```python
def check_valid_id(id: str):
    if not id.startswith(("isbn-", "imdb-")):
        raise ValueError('Invalid ID format, it must start with "isbn-" or "imdb-"')
    return id

async def read_items(q: Annotated[str | None, AfterValidator(check_valid_id)] = None)
```

###### その他

クエリ説明（バリデーションではない）
```python
@app.get("/items/")
async def read_items(q: Annotated[str | None, Query(
    title="bird",
    description="I don't particularly like birds.",
    pattern="^[b|B]ird$"
)] = None)
```

エイリアス（バリデーションではない）
※ケバブパターンを採用しないといけないケース等
```python
@app.get("/items/")
async def read_items(q: Annotated[str | None, Query(alias="q-item")] = None)
```

##### 4.3.5.3.パラメータの引数宣言

```python
from typing import Annotated

from fastapi import FastAPI, Path
from pydantic import BaseModel
```
```python
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.put("/items/{item_id}")
async def update_item(
    item_id: Annotated[int, Path(title="The ID of the item to get", ge=0, le=1000)], #パスパラメータ
    q: Annotated[str | None, Query(min_length=3, max_length=50)] = None,   #クエリパラメータ
    item: Item | None = None,   # bodyパラメータ
):
```


### 4.-.BaseModel() Body()
BaseModelは複数のフィールドがセットになっているケース
BaseModelほどではなくフィールドがシンプルで少量なケース。Body内でキー対値が1対1になるケースでは内部的にクエリパラメータと誤認されないために明示必要
> If you declare it as is, because it is a singular value, FastAPI will assume that it is a query parameter.
But you can instruct FastAPI to treat it as another body key using Body:



### 4.-.デプロイメント

```shell
fastapi deploy
```


## 5.Express(Node)と比較

## 6.Django(Python)と比較

## 7.参考サイト

## 8.まとめ

---

[^1] [ORMとは](https://qiita.com/devmatsuko/items/be3bd5a7ddeb5c5e4036)