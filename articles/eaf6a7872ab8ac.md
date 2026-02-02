---
title: "Python FastAPIで脱初心者！「とりあえず動く」を卒業するバックエンド構築"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "fastapi", "backend", "aws", "docker"]
published: true
---


## 1.はじめに

こんにちは。今回は FastAPI を用いた、大規模開発を見据えた設計思想の学習記録です。
私は普段、非IT系の事業会社（いわゆるJTC）にて「部署内DX」の推進役を担っており、現場の手元業務を即座に改善するためのPoCやツール活用がメインに行っております。
現状はサーバーレス等の軽量な構成が中心ですが、スピード重視で作るため「とりあえず動く」状態のものが多くあり、将来的にエンタープライズ水準のプロジェクトでも通用するアーキテクチャ設計や実装力を養いたいと考え、休暇を活用してインプットを行いました。本記事はその学習の記録です。

## 2.対象読者

- FastAPIの使い心地を簡単に確認したい方
- 「とりあえず動く」コードから、保守性の高い設計へステップアップしたい方
- Pythonの基礎理解はあるが、Webフレームワークの設計思想を学びたい方

## 3.記事を読むメリット

- 公式ドキュメントをベースにした、実務で使える設計思想（ディレクトリ構成、エラー処理）が学べる
- AWS ECSへのデプロイまでの一連の流れを把握できる
- 「なぜその設計にするのか」という現場目線の理由がわかる

## 4.概要

https://fastapi.tiangolo.com/
https://github.com/fastapi/fastapi

その名の通り、主にバックエンドのAPI実装で高速かつ安全な開発を行うためのWebフレームワーク。

開発者から特に選ばれるポイントは以下
- Python最速クラスのパフォーマンスを誇りNode.jsやGOに匹敵する速度を誇るWebフレームワーク
- 型安全性による開発効率の向上
- ドキュメントの完全自動生成

:::details 4.1.処理速度能力について

---

公式では以下のように記される
> Fast: Very high performance, on par with NodeJS and Go (thanks to Starlette and Pydantic).
[Link](https://fastapi.tiangolo.com/#:~:text=Fast%3A%20Very%20high%20performance%2C%20on%20par%20with%20NodeJS%20and%20Go%20(thanks%20to%20Starlette%20and%20Pydantic).)

この「速さ」について実際に比較した値を確認

#### 確認方法
TechEmpower Web Framework Benchmarks
https://www.techempower.com/benchmarks/#section=data-r23&test=db&l=gcuscf-p31

#### 前提条件
- 比較対象言語： JavaScript, TypeScript, Go, Python
- Classification（フレームワーク分類）：Micro（ルーティング等の最低限機能を持つがDB操作などの[ORM](https://qiita.com/devmatsuko/items/be3bd5a7ddeb5c5e4036)は標準で含まない）のみ

#### 比較対象のフレームワーク結果
![benchmark result](/images/fastapi/benchmark.png)

同分類の著名（と思われる）フレームワークとの比較し、GO言語のプロジェクトより速いケースもあり、Node.jsやGO言語でによるフレームワークに匹敵する「速さ」があると言える。

|順位|フレームワーク|言語|パフォーマンススコア|スコア比較値(FastAPI基準)|Github Star|
|---|---|---|---|---|---|
|13|Echo|GO|432,423|263.8%|32k|
|19|Gin|GO|238,344|145.4%|87.5k|
|24|express|JS|214,177|130.7%|68.4k|
|41|FastAPI|Python|163,894|100%|93.5k|
|56|Falcon|Python|106,883|65.2%|9.8k|
|65|nestjs|JS|84,218|51.4%|74.1k|
:::

:::details 4.2.FastAPIを選ぶ理由

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

### 結論

- 機能差
- 開発の活発性

### 調査してわかったこと

- いくつかのフレームワークは開発が終了している（脆弱性対応が見込めない等）。
- 開発が現在（記事執筆時点）も継続しており、Star数から見て比較的利用されているフレームワークは以下２点
  - BlackSheep
  - Starlette

### 各フレームワークを選択しない理由

|No|フレームワーク|不採用理由|
|---|---|---|
|1|BlackSheep|2.3kのStarを持つフレームワークで開発が現在も行われているもののあまり活発とは言えない状況（稼働が月に数回、かつCommitユーザーは１人という時もある|
|2|Starlette|11.8kのStarを持ちBlackSheepと比較すると開発も活発。そもそもFastAPIがStarletteを継承し開発を行っている。つまりStarletteに機能追加されたものがFastAPIと言えるため機能差がある（型ヒント駆動の入出力定義、バリデーション、依存性注入、OpenAPI生成、Swagger/ReDocなど）|

:::


## 5.インストール

### 5.1.検証環境

- OS: Windows11
- Python: 3.14.2


### 5.2.仮想環境構築とインストール（コマンドプロンプト）

参照：[Virtual Environments](https://fastapi.tiangolo.com/virtual-environments/)

```shell
python -m venv .venv
.\.venv\Scripts\activate

python -m pip install --upgrade pip
pip install "fastapi[standard]"
```

### 5.3.最小限構成起動

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
:::message
fastapi dev はホットリロード（コード変更の即時反映）が有効な開発用モードです。
本番環境で運用する際は、パフォーマンスと安定性を確保するため fastapi run コマンドを使用することが推奨されています。 
::: 

以下のURLでサーバーレスポンスやドキュメントが生成されていることを確認できる。

|No|URL|表示内容|
|---|---|---|
|1|http://127.0.0.1:8000/|`{"message": "Hello World"}`が表示される|
|2|http://127.0.0.1:8000/docs|Swagger形式の自動生成OpenAPIドキュメントが表示される|
|3|http://127.0.0.1:8000/redoc|OpenAPIドキュメントから生成されるReDocドキュメントが表示される|
|4|http://127.0.0.1:8000/openapi.json|OpenAPIドキュメントの生json|

よく使いそうなfastapiコマンドオプションテーブル
|No|Optiohn|指定方法|意味|例|
|---|---|---|---|---|
|1|--host|text|バインドするIPアドレス|`fastapi main.py --host 0.0.0.0`|
|2|--port|int|待ち受けるポート番号|`fastapi main.py --port 3000`|

- ドキュメント生成は後述する様々な場面で登場する型指定やバリデーションがそのままドキュメントに反映されるため、保守のコストを大幅に削減できる。
- **軽く調査した限り、標準でこのような機能を持つフレームワークはあまりない**（プラグインや追加設定が多くの場合で必要になる）

## 6.メソッドの指定

参照：[FastAPI Class](https://fastapi.tiangolo.com/reference/fastapi/)
Pythonのメソッドデコレーターで定義する

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

:::message
**脱初心者ポイント：async def と def の使い分け**
FastAPI では `async def` と `def` のどちらも定義できますが、明確な使い分けがあります。
- **async def**: DBアクセスや外部API通信など、**I/O待ち（待機時間）が発生する処理**に使用します。
- **def**: 計算処理など、**CPUを占有する処理**に使用します。FastAPIはこれをスレッドプールで実行し、メインのイベントループをブロックしないよう制御します。
「とりあえず全部 `async`」にすると、CPUバウンドな処理でサーバー全体の応答が遅れる原因になるため注意が必要です。
:::

## 7.パス指定

参照：[Path Parameters](https://fastapi.tiangolo.com/tutorial/path-params/)
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

## 8.リクエストの受け取り

### 8.1.パスパラメータ

参照：[Path Parameters](https://fastapi.tiangolo.com/tutorial/path-params/)
基本的に内部的にルーティングされリクエストに応じたメソッドが発火するが、動的に変化させるパスを持つケースにおいて受け取りは以下のようにする。

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```

パスの変数とメソッドの引数の命名は一致している必要がある。
パスを全文取得（任意の場所以降のパス文字列全体で受け取る）するには`:path`を使用する

```python
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    return {"file_path": file_path}
# URL: /files/hoge/foo.txt
# Response: {"file_path": "hoge/foo.txt"}
```

- 固定パスと変動パスが同層に存在するケースでは上から順番に処理されるためメソッドの宣言順番に注意が必要
- 以下のようなケースでは`/items/admin`へのリクエストも`read_item()`で処理されてしまう

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}

@app.get("/items/admin")
async def read_admin():
```

### 8.2.クエリパラメータ

参照：[Query Parameters](https://fastapi.tiangolo.com/tutorial/query-params/)
URLの末尾に`?`で宣言するクエリの受け取りは以下のようにする

```python
animal = [{"Bird": "pi pi pi"}, {"Cat": "meow"}, {"Dog": "bow wow"}]

@app.get("/animals")
async def read_item(filter: str | None = None):
    if filter is None:
        return animal
    return [item for item in animal if filter in item]
# URL: /animals?filter=Bird
# Response: [{"Bird": "pi pi pi"}]
```

- パスに動的な値が使用されておらず、メソッド引数に単一のオブジェクトとして宣言されている場合はクエリパラメータとして処理される。
- クエリの指定は変数名と命名が一致している必要がある。
- クエリにbool値が含まれる場合は以下のようにする。この時、内部的な処理によりクエリは`/animals?is_mammalian=True`,`/animals?is_mammalian=true`, `/animals?is_mammalian=on`等が許容される。

```python
@app.get("/animals")
async def read_item(is_mammalian: bool = False):
    if is_mammalian:
        return [item for item in animal if "Dog" in item or "Cat" in item]
    return animal
# URL: /animals?is_mammalian=True
# Response: [{"Cat": "meow"}, {"Dog": "bow wow"}]
```

- クエリの指定でデフォルト値を設定しない場合、必須クエリパラメータとなる。

```python
@app.get("/animals")
async def read_item(filter: str, is_mammalian: bool = False):
    if is_mammalian:
        return [item for item in animal if "Dog" in item or "Cat" in item]
    return [item for item in animal if filter in item]
# URL: /animals?filter=Bird&is_mammalian=True
# Response: [{"Cat": "meow"}, {"Dog": "bow wow"}]
```

### 8.3.ボディパラメータ

参照：[Request Body](https://fastapi.tiangolo.com/tutorial/body/)
ボディパラメータはリクエストボディで送信されるデータを受け取るために使用され、受け取りは以下のようにする。

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

- `BaseModel()`もしくは`Body()`クラスを宣言し型定義をする。
- リクエストBodyの最上層にキーが複数ある場合は、それぞれの`BaseModel()`で`Item`や`User`を宣言する
- 単一のデータでは`Body()`を使用し内部的にリクエストボディとして扱うよう明示的に指示できる。`Body()`を使用しない場合、内部的にクエリパラメータとして解釈されてしまう。

```json
{
    "item": {
        "name": "Foo",
        "description": "The pretender",
        "price": 42.0,
        "tax": 3.2
    },
    "user": {
        "username": "dave",
        "full_name": "Dave Grohl"
    },
    "importance": 5
}
```
```python
@app.put("/items/")
async def update_item(item: Item, user: User, importance: int = Body(embed=True)):
```

- 使い分けはリクエストBodyの内容が単一キーか否かだと思われる（そのほかのガイドラインがあるかもしれない）。
- またリクエストBodyの最上層にキーが単一の場合は以下のようにする。

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

@app.put("/items")
async def update_item(item: Annotated[Item, Body(embed=True)]):
    results = {"item": item, "user": user, "importance": importance}
    return results
```

仮に`Body(embed=True)`を利用しない場合は、以下のようにペイロードをネストしたモデルで表現する必要があります。

```python
class ItemContents(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

class ItemPayload(BaseModel):
    item: ItemContents

@app.put("/items")
async def update_item(payload: ItemPayload):
    results = {"received_item": payload.item}
    return results
```

### 8.4.まとめて受け取り

- パスパラメータ、クエリパラメータ、リクエストBodyのすべてを受け取りそれぞれに合わせた処理をするためには以下のようにする。

```python
from typing import Annotated

from fastapi import FastAPI, Path, Query
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
    item_id: Annotated[int, Path()], #パスパラメータ
    q: Annotated[str | None, Query()] = None,   #クエリパラメータ
    item: Item | None = None,   # bodyパラメータ
):
```

### 8.5.Cookies

参照：[Cookie Parameter Models](https://fastapi.tiangolo.com/tutorial/cookie-param-models/)
- Cookieはセッション管理としてサーバーからブラウザに一時保管させログインユーザーの管理等に使用する。
- `Body()`と同様に`Cookie()`として宣言をしないと内部的にクエリパラメータとして処理されてしまうので注意。

```python
from fastapi import Cookie, FastAPI

class Cookies(BaseModel):
    session_id: str
    fatebook_tracker: str | None = None
    googall_tracker: str | None = None

@app.get("/items/")
async def read_items(cookies: Annotated[Cookies, Cookie()]):
    return cookies
```

### 8.6.Header

参照：[Header Parameter Models](https://fastapi.tiangolo.com/tutorial/header-param-models/)
- Headerは一般的に`Content-Type`等のケバブケースで宣言される。FastAPIでは内部的にスネークケース`content_type`に変換をしてくれる。
- `Body()`と同様に`Header()`として宣言をしないと内部的にクエリパラメータとして処理されてしまうので注意。

```python
from typing import Annotated
from fastapi import Header, FastAPI

app = FastAPI()

class CommonHeaders(BaseModel):
    host: str
    save_data: bool
    if_modified_since: str | None = None
    traceparent: str | None = None
    x_tag: list[str] = []

@app.get("/items/")
async def read_items(headers: Annotated[CommonHeaders | None, Header()] = None):
    return headers
```

### 8.7.ファイルアップロード

参照：[File Upload](https://fastapi.tiangolo.com/tutorial/request-files/)
- アップロードされるファイルは最大サイズ制限までメモリに保存され、超過するとディスクに保存される。

```python
from fastapi import FastAPI, File, UploadFile

@app.post("/files/")
async def create_file(file: Annotated[bytes, File()]):
    return {"file_size": len(file)}

@app.post("/uploadfile/")
async def create_upload_file(file: UploadFile):
    return {"filename": file.filename}

@app.post("/uploadfile/")
async def create_upload_files(files: list[UploadFile]):
    return {"filenames": [file.filename for file in files]}
```

## 9.バリデーション

### 9.1.パスパラメータバリデーション

参照：[Path Parameters and Numeric Validations](https://fastapi.tiangolo.com/tutorial/path-params-numeric-validations/)
- パスパラメータの変動値を範囲内で検証したいケースで利用する

`Enum`による検証

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

`Path`による検証
https://fastapi.tiangolo.com/reference/parameters/#fastapi.Path

```python
from fastapi import FastAPI, Query, Path

@app.get("/items/{item_id}")
async def read_items(item_id: Annotated[int, Path(ge=1, le=1000)]) # パスパラメータ1以上1000以下
```

### 9.2.クエリパラメータバリデーション

参照：[Query Parameters and String Validations](https://fastapi.tiangolo.com/tutorial/query-params-str-validations/)
- クエリパラメータを検証したいケースで利用する

`Query`による検証
https://fastapi.tiangolo.com/reference/parameters/#fastapi.Query

```python
from typing import Annotated
from fastapi import FastAPI, Query

@app.get("/items/")
async def read_items(q: Annotated[str | None, Query(min_length=3, max_length=50)] = None) # 文字数制限

@app.get("/animals/")
async def read_animals(q: Annotated[str | None, Query(pattern="^[b|B]ird$")] = None) # 正規表現
```

### 9.3.ボディパラメータバリデーション

- リクエストBodyを検証したいケースで利用する

`Field`による検証
https://docs.pydantic.dev/latest/api/fields/

```python
from typing import Annotated
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str
    description: str | None = Field(default=None, title="The description of the item", max_length=300) # 文字数
    price: float = Field(gt=0, description="The price must be greater than zero") # 値の範囲

@app.put("/items/{item_id}")
async def update_item(item: Item):
    results = {"item": item}
    return results
```

もう少し現実的な複雑さを加味すると以下のようになる
```python
from fastapi import FastAPI, Query, Body
from pydantic import BaseModel, Field, HttpUrl

class Image(BaseModel): #サブモデル
    url: HttpUrl #カスタムタイプ
    name: str

class Item(BaseModel):
    name: str
    description: str | None = Field(default=None, title="The description of the item", max_length=300)
    price: float = Field(gt=0, description="The price must be greater than zero")
    tax: float | None = None
    images: list[Image] | None = None #サブモデル

class User(BaseModel):
    username: str
    full_name: str | None = None

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item, user: User, importance: Annotated[int, Body()]):
    results = {"item_id": item_id, "item": item, "user": user, "importance": importance}
    return results
```
データ
```json
{
    "item": {
        "name": "Foo",
        "description": "The pretender",
        "price": 42.0,
        "tax": 3.2,
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

- `HttpUrl`のようなカスタムモデルは開発のバリデーション実装の手間を省くことができる。

Pydantic
https://docs.pydantic.dev/latest/api/networks/
FastAPI
https://fastapi.tiangolo.com/tutorial/extra-data-types/

#### 9.3.1.サンプルデータや説明について

- `Field()`で宣言する`title`, `description`, `examples`, `json_schema_extra`はいずれも生成ドキュメントの為に使用され検証のためには使用されるものではない。[Customizing JSON Schema](https://docs.pydantic.dev/latest/concepts/fields/#customizing-json-schema)
- 同じようにFastAPIから利用できる、`body()`, `Query()`, `Path()`等は以下のようにサンプルデータを宣言することができる

Pydantic
https://docs.pydantic.dev/latest/concepts/json_schema/#field-level-customization

FastAPI
https://fastapi.tiangolo.com/tutorial/schema-extra-example/?h=openapi_examples#using-the-openapi-examples-parameter

`Body()`を使ったRequestの説明とサンプルデータ
```python
@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    user: User,
    item: Annotated[
        Item,
        Body(
            openapi_examples={
                "Normal": {
                    "summary": "A normal example",
                    "description": "A **normal** item works correctly.",
                    "value": {"name": "Foo", "price": 100},
                },
                "invalid": {
                    "summary": "Invalid data is rejected with an error",
                    "value": {
                        "name": "Baz",
                        "price": "thirty five point four",
                    },
                },
            }
        ),
    ],
):
```

`Query()`を使ったクエリパラメータの説明とサンプルデータ
```python
@app.get("/items/")
async def read_items(q: Annotated[str | None, Query(
    title="bird",
    description="I don't particularly like birds.",
    pattern="^[b|B]ird$"
)] = None)
```

#### 9.3.2.モデル設定による堅牢化 (ConfigDict)
「脱初心者」を目指すなら、Pydanticの ConfigDict を活用してモデルの挙動を厳格に制御しましょう。 
デフォルトでは、定義されていないフィールドがリクエストに含まれていても無視されるだけですが、extra='forbid' を設定することでエラーとして検知できます。
これにより、クライアント側のタイポや不正なデータの混入を未然に防ぐことができます。
```python
from pydantic import BaseModel, ConfigDict

class StrictItem(BaseModel):
    model_config = ConfigDict(
            extra='forbid',
            str_strip_whitespace=True,
        )
    name: str
    price: float

@app.post("/strict_items/")
async def create_strict_item(item: StrictItem):
    return item
```

### 9.4.レスポンスバリデーション

参照：[Response Model - Return Type](https://fastapi.tiangolo.com/tutorial/response-model/)
- クライアントへ返答するResponseの検証をで利用する
- デコレーターに`response_model`を追加する

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.get("/items/", response_model=Item)
async def get_item(item: Item):
     return item
```

`response_model`をつけないと内部的な検証に加え、生成ドキュメントのレスポンス欄が正しく生成されないので注意
![response_model](/images/fastapi/response_model.png)

また、`response_model`は出力データの制限及びフィルタリングにも活用される。
以下のようにIN/OUTを別々に宣言し、`response_model`にOUTの型を指定することで、関数内で明示的な除外処理をせずともパスワードのような機密情報をレスポンスから除外できる。

```python
class UserIn(BaseModel):
    username: str
    password: str
    email: EmailStr
    full_name: str | None = None

class UserOut(BaseModel):
    username: str
    email: EmailStr
    full_name: str | None = None

@app.post("/user/", response_model=UserOut)
async def create_user(user: UserIn) -> UserOut:
    return user
```


### 9.5.カスタムバリデーション

- FastAPIとpydanticの標準機能に加えてカスタムの検証を行うには以下のようにする。

https://fastapi.tiangolo.com/tutorial/query-params-str-validations/?h=aftervalidator#custom-validation

```python
from pydantic import AfterValidator

def check_valid_id(id: str):  # 特定文字列開始
    if not id.startswith(("isbn-", "imdb-")):
        raise ValueError('Invalid ID format, it must start with "isbn-" or "imdb-"')
    return id

async def read_items(q: Annotated[str | None, AfterValidator(check_valid_id)] = None):
    return {"q": q}
```

`AfterValidator`は通常の内部的な検証の後に実行される
https://docs.pydantic.dev/latest/api/functional_validators/#pydantic.functional_validators.AfterValidator

## 10.エラー処理

https://fastapi.tiangolo.com/reference/status/

### 10.1.基本的な例外

```python
from fastapi import FastAPI, HTTPException, status

@app.get("/items/{item_id}")
async def read_item(item_id: str):
    if item_id not in items:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND , detail="Item not found")
    return {"item": items[item_id]}
```

### 10.2.カスタム例外ハンドラー

1. `Exception`を継承したエラークラス（以下では`UnicornException`）を作成する
2. `raise`で作成したエラーモデルの例外を発火する
3. するとtry句ブロック内になくても同じエラークラスを継承している`@app.exception_handler()`でキャッチされる

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class UnicornException(Exception):
    def __init__(self, name: str):
        self.name = name

@app.exception_handler(UnicornException)
async def unicorn_exception_handler(request: Request, exc: UnicornException):
    return JSONResponse(
        status_code=418,
        content={"message": f"Oops! {exc.name} did something. There goes a rainbow..."},
    )

@app.get("/unicorns/{name}")
async def read_unicorn(name: str):
    if name == "yolo":
        raise UnicornException(name=name)
    return {"unicorn_name": name}
```

### 10.3.バリデーションエラーのオーバーライド

1. `/items/{item_id}`の`item_id`は`int`を期待するが`str`のアクセスが発生した前提
2. Pydanticが内部でエラーを検知
3. FastAPIが内部で`raise RequestValidationError(errors)`を実行
4. `@app.exception_handler(RequestValidationError)`がキャッチ
5. 通常であればFastAPIのデフォルトエラーメッセージが返答されるがカスタムしたテキストを返答できる

```python
from fastapi.responses import PlainTextResponse
from fastapi import FastAPI, HTTPException
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc: RequestValidationError):
    message = "Validation errors:"
    for error in exc.errors():
        message += f"\nField: {error['loc']}, Error: {error['msg']}"
    return PlainTextResponse(message, status_code=400)

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id == 3:
        raise HTTPException(status_code=418, detail="Nope! I don't like 3.")
    return {"item_id": item_id}
```

### 10.4.HTTPExceptionのオーバーライド

1. 明示的なエラーに対するリクエストが発生した前提
2. コード内で`raise HTTPException()`を実行
3. `@app.exception_handler(StarletteHTTPException)`がキャッチ

```python
from fastapi.responses import PlainTextResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

@app.exception_handler(StarletteHTTPException)
async def http_exception_handler(request, exc):
    return PlainTextResponse(str(exc.detail), status_code=exc.status_code)

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id == 3:
        raise HTTPException(status_code=418, detail="Nope! I don't like 3.")
    return {"item_id": item_id}
```

`import HTTPException as StarletteHTTPException`と命名変更をしているのはFastAPIの`HTTPException`とstarletteの`HTTPException`が同名で存在しており、場合によっては共存させるケースもあるため。

### 10.5.FastAPIのHTTPException

- FastAPIのデフォルトエラーメッセージをベースで使用し出力を追加するケースで利用する

リクエストバリデーション
1. `@app.exception_handler(RequestValidationError)`でキャッチ
2. `print`の出力
3. FastAPIのデフォルトエラーメッセージを出力

明示的なエラー
1. `raise HTTPException(`を実行
2. `@app.exception_handler(StarletteHTTPException)`がキャッチ
3. `print`の出力
4. FastAPIのデフォルトエラーメッセージを出力

```python
from fastapi import FastAPI, HTTPException
from fastapi.exception_handlers import (
    http_exception_handler,
    request_validation_exception_handler,
)
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException

@app.exception_handler(StarletteHTTPException)
async def custom_http_exception_handler(request, exc):
    print(f"OMG! An HTTP error!: {repr(exc)}")
    return await http_exception_handler(request, exc)

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    print(f"OMG! The client sent invalid data!: {exc}")
    return await request_validation_exception_handler(request, exc)

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id == 3:
        raise HTTPException(status_code=418, detail="Nope! I don't like 3.")
    return {"item_id": item_id}
```

### 10.6.一般的なエラー処理パターン

※ 本節は一般的な開発におけるハンドリング方法を参考に、筆者の解釈を加えて整理したものです。

エラーハンドリングは主に3層に分けて設計を行うことが多く、特に特色が出て層が厚くなるのがNo2の部分であると考える。

|No|処理内容|対応方法|
|---|---|---|
|1|入力チェック|FastAPIの`RequestValidationError`を使用|
|2|業務エラー|独自プロセスやロジック上発生するエラー。例外を明示的に用意しraiseさせることがメイン。|
|3|システムエラー|予期せぬバグ。インフラの停止などが該当。全体の`Exception`をキャッチし500系で返答する。開発/運用への通達必須。|

No2の構造を実際に実装する場合は以下のようになると予想した。

エラーハンドラーの基底クラスを作成する。
あくまでエラークラスでデコレーター等も使用しないため`exceptions`等、ディレクトリやファイルを分離する。
```python:exceptions.py
class BusinessException(Exception):
    """業務エラーの基底クラス"""
    def __init__(self, message: str, code: str, status_code: int = 400):
        self.message = message
        self.code = code  # フロントエンド識別用の独自エラーコード（例: "E001"）
        self.status_code = status_code
```

基底クラスを継承し具体的なビジネスロジックに落とし込んだエラーハンドラークラスを作成する
```python:exceptions.py
class ItemNotFoundError(BusinessException):
    def __init__(self, item_id: str):
        super().__init__(
            message=f"商品ID {item_id} は存在しません。",
            code="ITEM_NOT_FOUND",
            status_code=404
        )

class InsufficientStockError(BusinessException):
    def __init__(self, current: int, required: int):
        super().__init__(
            message=f"在庫不足です (残り: {current}, 要求: {required})",
            code="INSUFFICIENT_STOCK",
            status_code=400
        )
```

小規模であれば`main.py`で良いが、中規模以上であれば`hendler`等、ディレクトリやファイルを分離する。
```python:handlers.py
from fastapi import Request
from fastapi.responses import JSONResponse
from .exceptions import BusinessException

async def business_exception_handler(request: Request, exc: BusinessException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
            }
        },
    )
```

```python: main.py
from fastapi import FastAPI
from .exceptions import BusinessException, ItemNotFoundError, InsufficientStockError
from .handlers import business_exception_handler

app = FastAPI()
# 親クラス(BusinessException)に対してハンドラーを登録する
# これで子クラス(ItemNotFoundなど)も全てここでキャッチされる
app.add_exception_handler(BusinessException, business_exception_handler)
```

実際の呼び出し
```python
@app.get("/items/{item_id}")
def read_item(item_id: str):
    if item_id == "unknown":
        raise ItemNotFoundError(item_id)
    if item_id == "soldout":
        raise InsufficientStockError(current=0, required=1)
    return {"item_id": item_id}
```

#### 10.6.1.エラー追跡性

- 上記を拡張し十分な情報をクライアントに返せればよいが、実際には開発者の追跡経路を確保する必要がある。
- 実装の際には以下のようなログ出力機能は必須と考える。

```python:logger.py
import logging
import json
from datetime import datetime
import traceback

# 初期化処理

def log_error_for_cloud(exc: Exception, request_info: dict):
    """
    CloudWatch / Datadog 向けの構造化ログを出力する
    """
    tb_str = traceback.format_exc()

    log_payload = {
        "level": "ERROR",
        "timestamp": datetime.now().isoformat(),  # 発生時刻
        "error_type": exc.__class__.__name__,     # クラス名 (ItemNotFoundErrorなど)
        "message": str(exc),                      # エラーメッセージ
        "stack_trace": tb_str,                    # エラー発生場所
        "request": {                              # リクエスト情報
            "method": request_info.get("method"),
            "url": request_info.get("url"),
            "client_ip": request_info.get("client_ip"),
        }
    }

    print(json.dumps(log_payload, ensure_ascii=False))
```

以下にログ出力を追加
```python:handlers.py
from fastapi import Request
from fastapi.responses import JSONResponse
from .exceptions import BusinessException
from .logger import log_error_for_cloud

async def business_exception_handler(request: Request, exc: BusinessException):
    req_info = {
        "method": request.method,
        "url": str(request.url),
        "client_ip": request.client.host if request.client else "unknown",
    }
    log_error_for_cloud(exc, req_info)

    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
            }
        },
    )
```

## 11.ミドルウェア

参照：[Middleware](https://fastapi.tiangolo.com/tutorial/middleware/)

### 11.1.そもそもミドルウェアとは

> A "middleware" is a function that works with every request before it is processed by any specific path operation. And also with every response before returning it.
>> 「ミドルウェア」とは、特定のパス操作によって処理される前にすべてのリクエストを処理する関数です。また、レスポンスを返す前にもすべてのレスポンスを処理します。

- 知識が浅い私はバックエンド構造である、`Routes`, `Controllers`, `Services` 等の機能分離による階層分けとの違いに初め戸惑いました。
- ミドルウェアはこれらよりより手前の段階で処理することを目的としているもの。
- 特定のAPIに対する処理ではなく、すべてのAPIに対する処理を行う。
主な用途は以下のようなケースがある。
  - CORS
  - GZip圧縮
  - 処理時間計測
  - アクセスログ

### 11.2.実装

- ミドルウェアにリクエストが届いた時点で`start_time`に時刻を記録する
- `call_next`が`request`を対応するパスへ渡し、対応するパスで生成された`response`が返答される。
- `call_next`の実行で経過した時間を`process_time`に記録する
- `response`へ`process_time`をカスタムヘッダーに追加し、クライアントに渡す

```python
from fastapi import FastAPI, Request
import time

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.perf_counter()
    response = await call_next(request)
    process_time = time.perf_counter() - start_time
    response.headers["X-Process-Time"] = str(process_time) # カスタムヘッダーの追加
    return response
```

- ミドルウェアは複数作成することができスタックを生成する
- 宣言の順番にスタックされ、宣言の順番でリクエストとレスポンスが処理される
- わざわざ複数作成するケースは、純粋に単一責任の原則に従うケースが多い

```python
app.add_middleware(Middleware_A) #一番内
app.add_middleware(Middleware_B) #中間
app.add_middleware(Middleware_C) #一番外
```
```txt
Request -> Middleware_C -> Middleware_B -> Middleware_A -> FastAPI App -> Middleware_A -> Middleware_B -> Middleware_C -> Response
```

### 11.3. 【必須】CORSミドルウェア

「とりあえず動く」環境で、フロントエンドとバックエンドを連携させる際、躓きがちなのがCORS（Cross-Origin Resource Sharing） です。
本番環境を見据えるなら、初期段階から設定しておくのが定石です。

```python
from fastapi.middleware.cors
import CORSMiddleware
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://myapp.com"], # 許可するオリジン
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```


## 12.ディレクトリ構造

参照：[Bigger Applications - Multiple Files](https://fastapi.tiangolo.com/tutorial/bigger-applications/)

※ 本節は参照ドキュメントに筆者の解釈を加えて整理した、中規模以上でのディレクトリ構造です。

```txt
/
├── app
│   ├── __init__.py
│   ├── main.py
│   ├── dependencies.py
│   └── routers
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── users.py
│   │   │   └── items.py
│   │   └── v2/
│   │
│   └── internal
│       ├── __init__.py
│       └── admin.py
```

|No|ディレクトリ|使い分け|
|---|---|---|
|1|main.py|FastAPIアプリケーションのインスタンス化、ルーターのインクルード、ミドルウェアや例外ハンドラの登録など、アプリ全体のエントリーポイント。|
|2|dependencies.py|複数のルーターで共有される依存性注入（DI）のための共通関数（例: DBセッションの取得、認証ユーザーの取得）を定義する。|
|3|routers/|APIのエンドポイントを機能ごと（例: users, items）に分割するためのディレクトリ。|
|4|routers/v1/items.py|主に"/items/"に関連するパス操作（エンドポイント）をまとめたファイル。|
|5|routers/v1/users.py|主に"/users/"に関連するパス操作をまとめたファイル。|
|6|internal/|管理者用のエンドポイントなど、一般のクライアントには公開しない内部向けのAPIを配置する。|
|7|internal/admin.py|ヘルスチェックやデバッグ用ツールなど、管理者向けの具体的なパス操作を定義する。|


```python:main.py
from app.routers import v1

app = FastAPI()

app.include_router(v1.router, prefix="/api/v1")
```
```python:routers/v1/__init__.py
from fastapi import APIRouter
from . import users, items

router = APIRouter()

router.include_router(users.router, prefix="/users", tags=["users"]) # tagsはドキュメント生成用
router.include_router(items.router, prefix="/items", tags=["items"])
```
```python:user.py
router = APIRouter()
@router.get("/") # -> /api/v1/users/ になる予定
def get_users():
```

- エラー処理等の共通基盤機能を追加するなら以下のようにすると想像。

```txt
/
├── app
│   ├── __init__.py
│   ├── main.py
│
├── core/
│   ├── __init__.py
│   ├── config.py        # 環境変数や設定 (pydantic-settingsなど)
│   ├── exceptions.py    # カスタム例外定義
│   ├── handlers.py      # エラーハンドラー
│   ├── logger.py        # ログ設定
│   └── middlewares.py
```

## 13.デプロイメント

### 13.1.ローカル起動

[FastAPI Cloud](https://fastapicloud.com/)を使ったデプロイも可能とのことだが今回は自社アカウントのAWSを想定したデプロイメントについて記す

参照：[FastAPI in Containers - Docker](https://fastapi.tiangolo.com/deployment/docker/)
※Windowsの場合、[Docker Desktop](https://www.docker.com/ja-jp/products/docker-desktop/)のダウンロードは必要

Dockerイメージ上で使用するために以下を実行しておく
```shell
pip freeze > requirements.txt
```

rootフォルダに`Dockerfile`を作成し以下を記述
```dockerfile:Dockerfile
FROM python:3.14
WORKDIR /code
COPY ./requirements.txt /code/requirements.txt
RUN pip install --no-cache-dir --upgrade -r /code/requirements.txt
COPY ./app /code/app
CMD ["fastapi", "run", "app/main.py", "--port", "80"]
```

Dockerが準備できたら、以下コマンドでイメージをビルド
```shell
docker build -t myimage .
```
![docker_build](/images/fastapi/docker_build.png)

上記のようにビルドが完了したら、以下のコマンドで実行
```shell
docker run -d --name mycontainer -p 80:80 myimage
```
![docker_run](/images/fastapi/docker_run.png)

筆者は以下の双方で動作確認ができた
- http://localhost/
- http://127.0.0.1/

![localhost](/images/fastapi/localhost.png)

### 13.2.CICD見据えたデプロイ設定(Github+AWS)

本質的ではないのでさらっと

#### 13.2.1 Githubでコード管理

リポジトリを作成して
```shell
git init
git remote add origin <url.git>
git add .
git commit -m "commit comment"
git push origin main
```

#### 13.2.2 .dockerignore

必要に応じ`.dockerignore`をrootディレクトリに作成
```dockerignore:.dockerignore
# --- セキュリティ（超重要） ---
.env
.env.*
*.pem
*.key
id_rsa
secrets/

# --- Python / 依存関係 ---
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
pip-log.txt
pip-delete-this-directory.txt

# --- Git ---
.git
.gitignore

# --- IDE / エディタ設定 ---
.vscode/
.idea/
*.swp

# --- テスト / ドキュメント ---
# 本番用イメージを極限まで軽くしたい場合は除外
tests/
docs/
htmlcov/
.pytest_cache/
.coverage

# --- Docker関連 ---
Dockerfile
docker-compose.yml
.dockerignore
```
pushしておく

#### 13.2.3.AWS IAM ロール作成

- AWSにてIAMロールを作成
- インラインポリシーにて以下を参考に必要な権限をアタッチ

https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/image-push-iam.html

- またGithubActionsからAWSへの操作時にOIDCによる認証を行うため以下を参考に`信頼ポリシー`を設定

https://docs.github.com/ja/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws#configuring-the-role-and-trust-policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<AWSアカウントID>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:<GitHubユーザー名>/<リポジトリ名>:*"
                },
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

#### 13.2.4.リポジトリにyamlファイルを追加

`.github/workflow/deploy.yaml`

```yaml:deploy.yaml
name: Deploy to Amazon ECS

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

env:

  AWS_REGION: "ap-northeast-1"
  ECR_REPOSITORY: "my-fastapi-app"
  ECS_CLUSTER: "my-cluster-name"
  ECS_SERVICE: "my-service-name"
  IAM_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}

jobs:
  deploy:
    name: Build & Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.IAM_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # ECRのリポジトリURIを構築
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:latest .

          # Push (特定のハッシュタグとlatestタグの両方を送る)
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

          # 次のステップ用に変数を出力
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Force new deployment for ECS Service
        run: |
          aws ecs update-service --cluster ${{ env.ECS_CLUSTER }} \
                                 --service ${{ env.ECS_SERVICE }} \
                                 --force-new-deployment
```


## 14.まとめ

調べてわかったFastAPIの魅力：開発者体験（DX）の向上
- 少ない実装量で大きな効果を得られる点
  - ドキュメント自動生成: OpenAPI (Swagger) 準拠の仕様書がコードと同期して自動生成されるため、フロントエンドエンジニアとの連携コストが劇的に下がる。
  - 型定義による恩恵: PythonのType HintsとPydanticを組み合わせることで、バリデーションだけでなく、エディタ（VS Code等）の強力な補完機能が効き、実行前にバグを特定できる。
- 処理速度（モダンな非同期処理）
  - Node.jsやGoに匹敵するパフォーマンスを持つ。
  - Python標準の async/await をネイティブサポートしており、高負荷なI/O処理（DBアクセスや外部API通信）が重なってもブロッキングしにくい構造になっている。
  - 今回はこれらを実感できるほどの実装をしていないので体感では不明

### Django vs FastAPI：脱初心者のための選定基準

「大規模向けはDjango、小規模はFastAPI」と単純に語られることもありますが、現代のバックエンド開発では**「プロジェクトの性質」**で選ぶのが正解です。

#### Djangoを選ぶべきケース（バッテリー同梱型の強み）
- **管理画面が必須**: Django Adminは非常に強力で、社内用管理ツールなどを爆速で作るなら右に出るものはありません。
- **「あるある」機能が多い**: ユーザー認証、権限管理、RDB操作（ORM）、セキュリティ対策などが標準装備されており、これらを独自実装する工数をかけたくない場合。
- **チームの規約統一**: ディレクトリ構成や書き方が強制されるため、メンバーのスキルレベルにばらつきがあっても品質を保ちやすい。

#### FastAPIを選ぶべきケース（マイクロフレームワーク型の強み）
- **SPA/モバイルアプリのバックエンド**: フロントエンドが別にある場合、Djangoのテンプレート機能は不要になります。APIサーバーとしての軽快さとドキュメント生成機能が活きます。
- **非同期処理・高パフォーマンス**: チャットアプリやリアルタイム通知、機械学習モデルの推論APIなど、高負荷なI/Oや低レイテンシが求められる場合。
- **技術選定の自由度**: 「ORMはSQLAlchemy 2.0を使いたい」「NoSQLを使いたい」など、要件に合わせてベストなライブラリを組み合わせたい場合。マイクロサービス化を見据えるならこちらが有利です。

結論として、**「標準機能で素早くCRUDアプリを作りたいならDjango」、「パフォーマンスや柔軟なアーキテクチャを重視するならFastAPI」**という使い分けが、脱初心者としての第一歩と言えるでしょう。

## 15.おわりに：脱初心者への第一歩とこれから

今回の学習を通じて、「とりあえず動くコード」と「実務で使えるコード」の間には、**エラーハンドリングへの配慮**や**ディレクトリ構成による責務の分離**といった、明確な設計思想の違いがあることを痛感しました。
FastAPIは自由度が高い分、こうした設計力が試されるフレームワークですが、型安全性やドキュメント自動生成といった強力な機能が、正しい設計への道標になってくれるとも感じました。

### 今後の展望

「脱初心者」を完全に果たすために、次は以下の技術課題に挑戦していきたいと考えています。

- **DB連携の深掘り**: SQLAlchemy 2.0 や Tortoise ORM を用いた、非同期DBアクセスの実践。
- **テストコードの記述**: Pytest を用いた単体テスト・結合テストを行い、リファクタリングに強いコードベースを作る。
- **認証・認可**: JWTを用いたセキュアな認証基盤の構築。

本記事が、私と同じように「ツール作成から一歩進んだバックエンド開発」を目指す方の参考になれば幸いです。