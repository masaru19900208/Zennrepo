---
title: "json-serverで複数jsonファイルをモックサーバーとして利用する"
emoji: "🦴"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","nodejs","npm","memo"]
published: true
---
:::message
 初心者による初心者(自分)の為の備忘録です。語句の間違い等ありましたがご指摘願います
:::
<br>

結論だけ確認する場合は [Summary](#summary) まで飛ばしてください

# json-serverとは
[json-server](https://www.npmjs.com/package/json-server)
> Get a full fake REST API with zero coding in less than 30 seconds (seriously)

RestAPIを30秒以下で作れるとのことで、実際簡単なRestAPIであればコマンドのみで作成可能でした。
とても便利なんですが今回私がやりたいことに対しては少々工夫が必要でしたので備忘録を残します。
類似機能を構築できるExpressも参照しましたが今回は極力json-serverで行くこととしています。
理由は後述

# そもそもやりたいこと
弊社研修にて2週間で簡単なWebアプリケーションをチームで作成することになりました。
期間も限られていることからAWSでサーバーを立てる等のようなことはせずlocalhostで全てまかなうことを前提として実装を目指します。


### 前提条件
**テーブル毎にjsonファイルは分割する**
ただ単に管理の都合上です

**json-serverを利用する**
Expressはjson-serverと比較し学習コストが高いわりに将来的な業務での利用が見込めない

ぶっちゃけ私が弱弱エンジニアである為の前提ですね・・・


# 想定される障害
json-serverはmock利用できるjsonファイルが１つと限定されます。
そのためテーブル毎にjsonファイルを分けた場合上手く利用できませんのでこれを回避する為、工夫が必要です

## 要件定義

参考にさせて頂いた記事
https://qiita.com/nouka/items/47f26eecfd40712875e1

参考記事よりバックエンドでファイルを統括したマージjsonファイルを生成することにより障害を突破できることがわかりましたが、参考記事の場合、分割していたjsonファイル側を変更するとマージjsonファイルを更新するという仕様です。
json-serverはマージjsonファイル側をmock利用する為、今回のやりたい事と比較すると変更するファイルと更新させるファイルの方向が逆だと考えました

### 今回のアプリケーション設計でのあるべき姿
- アプリケーションはマージjsonファイルを参照し更新(CRUD)を行う
- マージjsonファイルが更新されたら分割jsonファイルも更新する
- 手作業でjsonファイルを更新する場合は分割jsonファイル側を更新する
- 分割jsonファイルが更新されたあとはマージjsonファイルにマージされる
    > 最後2つは開発初期段階のレアケースであるためアプリ起動時のみ動作すれば良いしそのうち無くせる


# Summary

### json-serverインストール
```shell
npm install json-server
```

### ファイル作成
- root(node_modulesと同じ階層)にapiフォルダを作成しその中に分割jsonファイルを保存します
- srcフォルダ内にscriptsフォルダを作成しその中にjsファイルを2つ作成します(srcフォルダ内でなくてもどこでもいいですがファイルパスの設定だけ注意してください)
    - merge_mock_json.js
    - split_mock_json.js

```js:merge_mock_json.js
const fs = require('fs');
const path = require('path');
const root = path.resolve('./', 'api');
let apiRes = {}
const allFile = fs.readdirSync(root, { withFileTypes: false })
allFile.forEach((file) => {
    const endpoint = path.basename(file, path.extname(file));
    apiRes[endpoint] = JSON.parse(fs.readFileSync(root + '/' + file, 'utf-8'));
})
fs.writeFileSync(root + '/../mock.json', JSON.stringify(apiRes), function (err) {
    if (err) throw err;
});
```

これは分割されたjsonファイルをmock.jsonというファイルに統合します
jsファイル起動時１度だけ実施します
<br>


```js:split_mock_json.js
const fs = require('fs');
const path = require('path');

const mockFile = path.resolve('./', 'mock.json');
let apiRes = {}
fs.watch(mockFile, function (event, changeFile) {
    if (changeFile === 'mock.json') {
        apiRes = JSON.parse(fs.readFileSync(mockFile, 'utf-8'))
    }
    Object.keys(apiRes).forEach(e => {
        fs.writeFileSync(`./api/${e}.json`, JSON.stringify(apiRes[e]), (err) => {
            console.log(err)
        })
        console.log(`update ${e}.json file`)
    });
});
```
これはmock.jsonを監視しmock.jsonに変更が発生した場合は分割jsonに更新をかけます。
アプリが起動している間は常にmock.jsonを監視し続け、変更が発生した度に分割jsonを更新します。
差分取得してその範囲だけ更新することも検討しましたが、現状悪さするようなケースを思いつかなかったことと、テーブル数も莫大に多いわけでもないので現状これでいいかと考えています。


### Scriptが動作するようにする

[npm-run-all](https://www.npmjs.com/package/npm-run-all)をインストールする
npm-run-allとは
>A CLI tool to run multiple npm-scripts in parallel or sequential.
```shell
npm install npm-run-all
```


npm-scriptを1コマンドで複数実行するために利用します。初めて利用しましたがとても便利です


package.jsonのscriptsを下記のように更新します
```json:package.json
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject",
    "serve": "npm-run-all -p create-mock watch-mock boot-mock start",
    "create-mock": "node ./src/scripts/merge_mock_json.js",
    "watch-mock": "node ./src/scripts/split_mock_json.js",
    "boot-mock": "json-server --watch ./mock.json --port 4000"
  },
```
これにより`npm run serve`と実行するだけで
1. merge_mock_json.jsの実行
2. split_mock_json.jsの実行
3. json-serverでmock.jsonをport4000で起動する
4. npm startする
これらを全て実行してくれます

# 動作イメージ
apiフォルダに下記のような2つのファイルをおいた場合を想定します

```json:user.json
[
    {
        "id":1,
        "username":"hoge"
    },
    {
        "id":2,
        "username":"foo"
    }
]
```
```json:meal.json
[
    {
        "id":1,
        "menuname":"bread"
    },
    {
        "id":2,
        "menuname":"rice"
    }
]
```

mock.jsonは下記のように生成されます
```json:mock.json
{
    "user":[
        {
            "id":1,
            "username":"hoge"
        },
        {
            "id":2,
            "username":"foo"
        }
    ],
    "meal":[
        {
            "id":1,
            "menuname":"bread"
        },
        {
            "id":2,
            "menuname":"rice"
        }
    ]
}
```

### アプリケーションでアクセスする

mock.jsonを利用した場合
[http://localhost:4000/user](http://localhost:4000/user)
[http://localhost:4000/meal](http://localhost:4000/meal)
これらにアクセスすることでそれぞれのデータを利用できます

読み込み
```js:test.jsx
    const [users, setUsers] = useState({})
    useEffect(() => {
        fetch("http://localhost:4000/user")
            .then(response => response.json())
            .then(userData => {setUsers(userData)})
    }, [])
```

取得データは下記のようになります
```json:users
[
    {
        "id":1,
        "username":"hoge"
    },
    {
        "id":2,
        "username":"foo"
    }
]
```

# 注意点
サーバーが起動している状態において
mock.jsonをVSCode等のエディターソフトで上書き保存するとエラーが発生し強制終了することがあります。真面目に検証したわけではありませんが、私はVSCodeの設定で保存と同時にフォーマットするようにしております。これにより上書き保存が2度発生することが起因していると考えています。
Windows標準の「メモ帳」等を利用するとこれを防げるには防げます。
使い勝手があまりいいものではないですが、そもそもmock.jsonを手作業で変更することは避けたほうが無難です

## 手作業でjsonファイルを更新する場合
jsonファイルを手作業で更新したい場合は下記手順を踏みましょう
1. json-serverが終了していることを確認
2. apiフォルダにある分割jsonファイルを更新する
3. `npm run serve` or `npm run create-mock`を実行しmock.jsonに反映させる

少し手間ですがどっちを更新したっけ？等の混乱を防げるかと思います


実際の運用には検証不足な部分もあるので運用をしてみて不足している部分がありましたら修正します

<br>
以上です
