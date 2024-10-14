---
title: "Invoke使うと何がいいの?"
emoji: "🤔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp"]
published: false
---
:::message
 初心者による初心者(自分)の為の備忘録。
:::

## 背景

JTS勤めの私ですが2023年1月よりIT企業への出向を拝命し日々勉強しながら何とか食らいついている状態。<br>
実務では有識者の実装にならって実装していたものの、正直🤔??と思いながら実装していた点が多々あるので振り返りをする。<br>
実装業務の際は常時作業遅れが発生していて、とてもじゃないが調べてる余裕が無かったので今更ながら。

## 調査対象

### Invokeってやつ

* ユーザー操作や通信等、特定タイミングで処理を行いたいって時に使うイベントとか言うやつ
* 代表的な例は以下のようなコード

```csharp
public delegate void MyEventHandler(object sender, EventArgs e);

public class MyClass
{
    public event MyEventHandler MyEvent;
    public void TriggerEvent()
    {
        MyEvent?.Invoke(this, EventArgs.Empty);
    }
}

public class Subscriber
{
    public void OnMyEventTriggered(object sender, MyEventArgs e)
    {
        Console.WriteLine("Event triggered with no additional data.");
    }
}

class Program
{
    static void Main()
    {
        MyClass myClass = new MyClass();
        Subscriber subscriber = new Subscriber();
        myClass.MyEvent += (sender, e) => subscriber.OnEventTriggered(sender, e);
    }
}
```

以下のような形でよくない？
複雑だしわかりにくくね？

```csharp
public class Subscriber
{
    public void OnMyEventTriggered()
    {
        Console.WriteLine("Event triggered with no additional data.");
    }
}

class Program
{
    static void Main()
    {
        MyClass myClass = new MyClass();
        Subscriber subscriber = new Subscriber();
    }

    private void Hoge(){
        // 必要なタイミングで以下のメソッドを都度呼び出す
        subscriber.OnEventTriggered();
    }
}
```

### 素人目線の観測

* 普通にメソッド呼び出すだけだったらinvokeとかワンクッション挟む必要なくね？
* 素人目線ではどこに伝播してるのかわかりにくくて難しくない？
そんな風に思っていて意味が分からなかったのでChatGPTに色々聞いた結果をまとめる

## 調査結果

### 疎結合にする事が出来る

* `subscriber.OnEventTriggered(sender, e);`を直接呼び出していることは密結合にあたる。
  * 具体的には`Program`クラスは`Subscriber`クラスの存在と詳細を完全に把握している
  * つまり`Subscriber`クラスのメソッド名や引数などが変更された場合は`Program`クラスの修正も必要になる。
* 仮に`subscriber.OnEventTriggered(sender, e);`が複数のクラス複数の箇所で呼び出しをしている場合（密結合が多い状態）で`Subscriber`クラスに変更をかけようものならエラーの嵐になる。

以下のようにすることでイベントの発火側とそれを受け取る側が互いを認識せずにイベントを受け渡すことが可能となる

---

#### Eventの仲介役

* PublishEventメソッドを実行することでイベントを伝播させる

```csharp
public class EventAggregator
{
    public event EventHandler<MyEventArgs> OnEventPublished;

    public void PublishEvent(string message)
    {
        OnEventPublished?.Invoke(this, new MyEventArgs(message));
    }
}
```

#### Eventの引数

* `public delegate void MyEventHandler(object sender, EventArgs e);`のような作り方でも問題ないがカスタムクラスにするとより柔軟な引数を受け渡すことが出来る。

```csharp
public class MyEventArgs : EventArgs
{
    public string Message { get; }

    public MyEventArgs(string message)
    {
        Message = message;
    }
}
```

#### Eventを送る側

* `EventAggregator.PublishEvent`を必要なタイミングで呼び出す
  * 必要なタイミングの例：ユーザー操作等

```csharp
public class Publisher
{
    private readonly EventAggregator _eventAggregator;

    public Publisher(EventAggregator eventAggregator)
    {
        _eventAggregator = eventAggregator;
    }

    public void DoSomething()
    {
        Console.WriteLine("Publisher: Doing something important...");
        
        // 何かの処理の後、イベントを発火
        _eventAggregator.PublishEvent("Something happened!");
    }
}
```

#### Eventを受け取る側

* `EventAggregator.OnEventPublished`を`+=`演算子で購読しメソッドに紐づける。これにより`EventAggregator.PublishEvent`をどこかの誰かが実行したら`OnEventReceived`が実行されるようになる。`Subscriber`は誰が`EventAggregator.PublishEvent`を実行したのか知らない。

```csharp
public class Subscriber
{
    public Subscriber(EventAggregator eventAggregator)
    {
        // イベントにサブスクライブ
        eventAggregator.OnEventPublished += OnEventReceived;
    }

    private void OnEventReceived(object sender, MyEventArgs e)
    {
        Console.WriteLine($"Subscriber: Received event with message: {e.Message}");
    }
}
```

#### Eventのマネジメント

* `EventAggregator`のインスタンスを持ち各クラスに関連付ける。

```csharp
class Program
{
    static void Main()
    {
        EventAggregator eventAggregator = new EventAggregator();

        // Publisher と Subscriber が互いを知らず、EventAggregator を通じて通信する
        Publisher publisher = new Publisher(eventAggregator);
        Subscriber subscriber = new Subscriber(eventAggregator);

        publisher.DoSomething();
    }
}
```

---

* このように完全にイベントの発行クラスと購読クラスが関わることなくイベントを伝播させる疎結合を作り出すことが出来る。
* 疎結合により互いの変化に対する影響が最小限に抑えられ変化に強いプログラムが出来上がる

### その他

* テストコードが書きやすくなる
* コードの見通しが良くなる
