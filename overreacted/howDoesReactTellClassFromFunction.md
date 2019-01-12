# Reactはどうやって関数からクラスを見分けるの？

以下は[How Does React Tell a Class from a Function?](https://overreacted.io/how-does-react-tell-a-class-from-a-function/)  の日本語訳です。

関数として定義された`Greeting`コンポーネントについて考えてみましょう

```javascript
  function Greeting() {
    return <p>Hello</p>;
  }

Reactはclassとしての定義もサポートしています

  class Greeting extends React.Component {
    render() {
      return <p>Hello</p>;
    }
  }
```

( [最近まで](https://reactjs.org/docs/hooks-intro.html),
ステートの機能を使うための唯一の方法でした)


`<Greeting />`を描画するとき、どのように定義されたか気にする必要はありません。

```javascript
  // クラスもしくは関数 — なんでも.
  <Greeting />
```

しかしReact自身は違いを気にする必要があります！

 `Greeting`が関数ならReactは下記のように呼ぶ必要があります

```javascript
  // あなたのコード
  function Greeting() {
    return <p>Hello</p>;
  }
  
  // React内部
  const result = Greeting(props); // <p>Hello</p>
```

しかし、もし`Greeting`がクラスの場合、Reactは`new`演算子と作成したインスタンスに対して`render`関数を呼ぶ必要があります。

```javascript
  // あなたのコード
  class Greeting extends React.Component {
    render() {
      return <p>Hello</p>;
    }
  }
  
  // React内部
  const instance = new Greeting(props); // Greeting {}
  const result = instance.render(); // <p>Hello</p>
```

どちらのケースでもReactの目的は描画したノードを取得することです。(この例では`<p>Hello</p>`)
しかし、実際のステップはどのように`Greeting`が定義されたかということに依存しています。

**Reactはどのようにしてクラスか関数か知るのでしょうか？**

[前の投稿](/why-do-we-write-super-props/)のように、**Reactで生産的になるためにこれを知る必要はありません。**
私はこれを何年間も知りませんでした。どうかこれを面接の質問にしないでください。
実際、Reactについてというよりも、Javascriptについての投稿です。

このブログはReactがなぜこのように動いているのか知りたい好奇心の強い読者向けです。あなたはそのような人ですか？一緒に深掘りしてみましょう。

**これは長い旅です。ベルトを締めてください。その投稿はReact自身についての十分な情報は持っていません。しかし、Javascriptで`new`, `this`, `class`, arrow functions, `prototype`, `__proto__`,`instanceof`のこれらがどのように機能するか説明します。幸運にもReactを使う時は、これらのことを考える必要がありませんでした。もしあなたがReactを実装しているなら...**

答えを知りたいだけなら最後までスクロールしてください。

---

はじめに、私たちはなぜ関数とクラスの違いを扱うことが大切なのか理解する必要があります。Note: クラスを呼び出す時に`new`演算子を使う方法


```javascript
  // Greetingが関数なら
  const result = Greeting(props); // <p>Hello</p>
  
  // Greetingがクラスなら
  const instance = new Greeting(props); // Greeting {}
  const result = instance.render(); // <p>Hello</p>
```

JavaScriptで `new`演算子がすることの大まかな意味を理解しましょう。

---

昔は、Javascriptはクラスを持っていませんでした。しかしながら普通の関数を使ってクラスと同じようなパターンを表現できます。
**具体的には呼び出しの前に`new`を追加することで任意の関数をクラスのコンストラクタに似た役割で使うことができます。**


```javascript
  // 単なる関数
  function Person(name) {
    this.name = name;
  }
  
  var fred = new Person('Fred'); // ✅ Person {name: 'Fred'}
  var george = Person('George'); // 🔴 動かない
```

今日でもこんなコードを書くことができます!DevToolsで試してみてください。

もし `Person('Fred')` を `new`なしで呼び出したら、その中の`this`はグローバルで無用なものを指すでしょう。(例えば `windows`や`undefined`)
だから、そのコードはクラッシュしたり、`window.name`に設定するような愚かなことをするでしょう。


呼び出しの前に`new`を追加することで、私たちはこう言います。
「やあJavascript、`Person`は単なる関数だってことは知っている。だけど、それをクラスコンストラクタのようなものにしよう。
**オブジェクト(`{}`)を作成し、`Person`関数内で`this`はそのオブジェクトを指すようにして、`this.name`に値を割り当てる。その後そのオブジェクトを返してください。**」


それが`new`演算子がすることです。

```javascript
  var fred = new Person('Fred'); // `Person`の中の`this`と同じオブジェクト 
```

The `new` operator also makes anything we put on `Person.prototype` available on the `fred` object:
`new`演算子は`Person.prototype`に追加したもの全てを`fred`オブジェクトで使えるようにします。

```javascript
  function Person(name) {
    this.name = name;
  }
  Person.prototype.sayHi = function() {  alert('Hi, I am ' + this.name);}
  var fred = new Person('Fred');
  fred.sayHi();
```

これはJavascriptが直接クラスを追加する前にクラスをエミュレートする方法です。

---

だから`new`は結構前からJavascriptに登場しています。しかしながらクラスはもっと最近です。最近のクラスはさらに直感的に上のコードを書き直すことができます。


```javascript
  class Person {
    constructor(name) {
      this.name = name;
    }
    sayHi() {
      alert('Hi, I am ' + this.name);
    }
  }
  
  let fred = new Person('Fred');
  fred.sayHi();
```

開発者の意図を捉えることは言語とAPI設計において重要です。

関数を書いたら、Javascriptはそれが`alert()`みたいに呼ばれることを意図しているのか、それとも`new Person()`みたいにコンストラクタとして呼ばれるのか推測できない。

**クラス構文は「これは関数じゃない、それはクラスでコンストラクタを持っている」と言ってくれる**
もし`new`をつけ忘れて呼ぶとJavascriptはエラーを発生させる。


```javascript
  let fred = new Person('Fred');
  // ✅  もしPersonが関数なら: うまく動く
  // ✅  もしPersonがクラスなら: これもうまく動く
  
  let george = Person('George'); // `new`つけるのを忘れた
  // 😳  もしPersonがコンスラクタみたいな関数なら: 混乱した振る舞いになる
  // 🔴  もしPersonがクラスなら: 即エラー
```

これは、`this.name`が` george.name`ではなく`window.name`として扱われるようなあいまいなバグを待つのではなく、早い段階でミスを見つけるのに役立ちます。


However, it means that React needs to put `new` before calling any class. It can’t just call it as a regular function, as JavaScript would treat it as an error!
しかしながらそれはReactはどんなクラスでも`new`を書かないといけないということを意味します。
Javascriptはそれをエラーとして扱うので、普通の関数を単に呼び出せない。

```javascript
  class Counter extends React.Component {
    render() {
      return <p>Hello</p>;
    }
  }
  
  // 🔴 React can't just do this:
  const instance = Counter(props);
```

これはトラブルの種です。

---

Reactがこれをどうやって解決するかを見る前に、Reactを使うほとんどの人がBabelのようなコンパイラを使って古いブラウザのためにクラスのような機能をコンパイルしていることを覚えておくことが重要です。だから我々は私たちのデザインでコンパイラを考慮する必要があります。

Babelの初期のバージョンはクラスは`new`なしで呼び出すことができました。しかし、これは下記のコードを生成することで修正されました。

```javascript
  function Person(name) {
    // Babelの出力から少し簡略化したもの
    if (!(this instanceof Person)) {
      throw new TypeError("Cannot call a class as a function");
    }
    // 自分のコード
    this.name = name;
  }
  
  new Person('Fred'); // ✅ OK
  Person('George');   // 🔴 Cannot call a class as a function
```
もしかしたらバンドルされたコード中で`_classCallCheck`というコードをみたことがあるかもしれません。上記の例がそれです。
(チェックなしで「ルーズモード」を選択することによってバンドルサイズを減らすことができますが、これは実際のネイティブクラスへの最終的な移行を複雑にするかもしれません。)
(ルーズモードのオプションでバンドルサイズを減らすことができますが、最終的にネイティブのクラスへの移行を複雑にするかもしれません。)
---

ここまでで、 `new`を付けて呼び出した場合と` new`を付けずに呼び出した場合の違いをおおまかに理解できるはずです。

|            | `new Person()`                  | `Person()`                           |
| ---------- | ------------------------------- | ------------------------------------ |
| `class`    | ✅ `this` is a `Person` instance | 🔴 `TypeError`                       |
| `function` | ✅ `this` is a `Person` instance | 😳 `this` is `window` or `undefined` |

そのため、Reactがコンポーネントを正しく呼び出すことが重要です。 **あなたのコンポーネントがクラスとして定義されている場合、Reactはそれを呼び出すときに `new`を使う必要があります。**

それでReactは何かがクラスであるかどうかを単にチェックすることができますか？

そう簡単ではありません！[JavaScriptの関数からクラスを見分ける]((https://stackoverflow.com/questions/29093396/how-do-you-check-the-difference-between-an-ecmascript-6-class-and-function))ことができたとしても、
これはまだBabelのようなツールで処理されたクラスにはうまくいかないでしょう。ブラウザにとっては、それらは単なる普通の関数です。 Reactは頑張ってください。

---

OK,もしかしたらReactは全ての呼び出しに`new`を使えばいい？残念なことに、それは常に正しく動くとは限りません。

通常の関数では、それらを `new`で呼び出すと、それらに` this`としてオブジェクトインスタンスが与えられます。
これはコンストラクタとして書かれた関数(上記の `Person`のように)には望ましいですが、関数のコンポーネントには混乱を招くでしょう：

```javascript
  function Greeting() {
    // ここで `this`が他の種類のインスタンスであるとも思わないでしょう
    return <p>Hello</p>;
  }
```

それは許容できるかもしれません。 この考えを殺すのには他に2つの理由があります。

---

The first reason why always using `new` wouldn’t work is that for native arrow functions (not the ones compiled by Babel), calling with `new` throws an error:
常に`new`を使用してもうまくいかない最初の理由は、ネイティブのarrow関数(Babelによってコンパイルされたものではない)では、`new`を指定して呼び出すとエラーが発生するためです。

```javascript
  const Greeting = () => <p>Hello</p>;
  new Greeting(); // 🔴 Greeting is not a constructor
```

この動作は意図的なもので、arrow関数の設計に基づいています。
arrow関数の主な利点の1つは、それらが独自の `this`値を持たないということです - 代わりに、`this`は最も近い通常の関数から解決されます。


```javascript
  class Friends extends React.Component {
    render() {
      const friends = this.props.friends;
      return friends.map(friend =>
        <Friend
          // `this`は` render`メソッドから解決されます
          size={this.props.size}
          name={friend.name}
          key={friend.id}
        />
      );
    }
  }
```

さて、それで**arrow関数はそれ自身の `this`を持っていません**しかしそれはそれらがコンストラクタとして全く役に立たないことを意味します！

```javascript
  const Person = (name) => {
    // 🔴 これは意味がない！
    this.name = name;
  }
```

Therefore, **JavaScript disallows calling an arrow function with `new`.** If you do it, you probably made a mistake anyway, and it’s best to tell you early. This is similar to how JavaScript doesn’t let you call a class _without_ `new`.
そのため、**JavaScriptでは `new`を使用してarrow関数を呼び出すことはできません。**これを実行した場合は、間違いを犯している可能性があります。
これは、JavaScriptがクラスを`new`無しで呼び出せないのと似ています。

これは素晴らしいことですが、それはまた私たちの計画を打ち立てます。 Reactはすべてのものに対して `new`を呼び出すだけでは不可能です。arrow関数が壊れるから！
しかし、`new`をつけず、`prototype`の欠如によってarrow関数を検出を試みることができます。

```javascript
  (() => {}).prototype // undefined
  (function() {}).prototype // {constructor: f}
```

しかしこれはBabelでコンパイルされた関数には[うまく動きません。](https://github.com/facebook/react/issues/4599#issuecomment-136562930)
これは大したことではないかもしれませんが、このアプローチを行き止まりにするもう1つの理由があります。

---

常に`new`を使うことができないもう一つの理由は、Reactが文字列や他のプリミティブ型を返すコンポーネントをサポートすることを妨げるということです。

```javascript
  function Greeting() {
    return 'Hello';
  }
  
  Greeting(); // ✅ 'Hello'
  new Greeting(); // 😳 Greeting {}
```

これもまた、[`new`演算子](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new)設計に関係しています。 前に見たように、 `new`はJavaScriptエンジンにオブジェクトを作成し、そのオブジェクトを関数の中での`this`にし、そして後で `new`の結果としてそのオブジェクトを渡すように伝えます。

しかしながら、JavaScriptでは、他のオブジェクトを返すことによって、`new`で呼び出された関数が`new`の戻り値をオーバーライドすることもできます。
おそらく、これはインスタンスを再利用したい場合のプーリングのようなパターンに役立つと考えられていました。

```javascript
  // 遅延作成
  var zeroVector = null;
  
  function Vector(x, y) {
    if (x === 0 && y === 0) {
      if (zeroVector !== null) {
        // Reuse the same instance      return zeroVector;    }
      zeroVector = this;
    }
    this.x = x;
    this.y = y;
  }
  
  var a = new Vector(1, 1);
  var b = new Vector(0, 0);
  var c = new Vector(0, 0); // 😲 b === c
```

ただし、関数がオブジェクトではない場合、`new`は関数の戻り値を完全に無視します。 あなたが文字列や数字を返す場合、それは `return`がまったくなかったようです。

```javascript
  function Answer() {
    return 42;
  }
  
  Answer(); // ✅ 42
  new Answer(); // 😳 Answer {}
```

`new`でそれを呼び出すときに、関数からプリミティブな戻り値(数字や文字列のような)を受け取る方法は全くありません。 そのため、Reactが常に `new`を使っていたら、文字列を返すサポートコンポーネントを追加することはできません。

それは受け入れられないので、諦める必要があります。

---

What did we learn so far? React needs to call classes (including Babel output) with `new` but it needs to call regular functions or arrow functions (including Babel output) without `new`. And there is no reliable way to distinguish them.
これまでに何を学びましたか？ Reactは `new`を使ってクラス(Babel出力を含む)を呼び出す必要がありますが、`new`を使わずに通常の関数やarrow関数(Babel出力を含む)を呼び出す必要があります。 そしてそれらを区別する信頼できる方法はありません。

**一般的な問題を解決できないなら、より具体的な問題を解決できますか？**

コンポーネントをクラスとして定義するとき、おそらく `this.setState()`のような組み込みメソッドのために `React.Component`を拡張します。
**すべてのクラスを検出しようとするのではなく、 `React.Component`の子孫だけを検出できますか？**

ネタバレ：これはReactがすることです。

---

おそらく、 `Greeting`がReactコンポーネントクラスかどうかをチェックする慣用的な方法は、` Greeting.prototype instanceof React.Component`かどうかをテストすることです。

```javascript
  class A {}
  class B extends A {}
  
  console.log(B.prototype instanceof A); // true
```

私はあなたが何を考えているのか知っています。 ここで何が起きたのですか？ これに答えるためには、JavaScriptプロトタイプを理解する必要があります。

もしかしたらあなたは“prototype chain”に精通しているかもしれません。
Javascriptでは全てのオブジェクトは“prototype”を持っています。
`fred.sayHi()`を書いたときに、`fred`オブジェクトが`sayHi`プロパティを持っていなかったら、`fred`のプロトタイプで`sayHi`を探します。
もしそこで見つからなかったら、チェーン内から次のプロトタイプである`fred`のプロトタイプのプロトタイプを探します。

**紛らわしいことに、クラスや関数の `prototype`プロパティはその値のプロトタイプを指し示すわけではありません。**冗談じゃないよ。
```javascript
  function Person() {}
  
  console.log(Person.prototype); // 🤪 Personのprototypeじゃない
  console.log(Person.__proto__); // 😳 Personのprototype
```

「プロトタイプチェーン」は `prototype.prototype.prototype`より` __proto __.__ proto __.__ proto__`ですね。 私はこれに何年も要しましたよ。

それでは、関数やクラスの `prototype`プロパティは何ですか？ **それはそのクラスまたは関数で `new`されたすべてのオブジェクトに与えられた` __proto__`です！**

```javascript
  function Person(name) {
    this.name = name;
  }
  Person.prototype.sayHi = function() {
    alert('Hi, I am ' + this.name);
  }
  
  var fred = new Person('Fred'); // `Person.prototype`に`fred.__proto__`を設定
```

そしてその `__proto__`チェーンがJavaScriptがプロパティを調べる方法です。

```javascript
  fred.sayHi();
  // 1. Does fred have a sayHi property? No.
  // 2. Does fred.__proto__ have a sayHi property? Yes. これを呼ぶ!
  
  fred.toString();
  // 1. Does fred have a toString property? No.
  // 2. Does fred.__proto__ have a toString property? No.
  // 3. Does fred.__proto__.__proto__ have a toString property? Yes. これを呼ぶ!
```

実際には、プロトタイプチェーンに関連するものをデバッグしているのでなければ、コードから直接 `__proto__`を直接触る必要はないはずです。
`fred .__ proto__`で利用可能にしたい場合は、それを` Person.prototype`に置くことになっています。
少なくともそれはもともと設計された方法です。


プロトタイプチェーンは内部概念と考えられていたため、 `__proto__`プロパティは最初はブラウザによって公開されることさえ想定されていませんでした。
しかし、いくつかのブラウザは `__proto__`を追加し、結局それはひどく標準化されました(しかし` Object.getPrototypeOf()`を支持して推奨されなくなりました)。

**それでもなお、 `prototype`と呼ばれるプロパティが値のプロトタイプを与えないことは非常に混乱します。例えば、`fred`は関数ではないので `fred.prototype`は未定義です。 個人的には、これが経験豊富な開発者でさえJavaScriptプロトタイプを誤解しがちな最大の理由だと思います。

---

これは長い記事ですね。 現在80％くらいの場所にいると思います。 あとちょっと。

`obj.foo`を実行したとき、JavaScriptは実際には`obj`の`foo`を探すとき、 `obj .__ proto__`、` obj .__ proto __.__ proto__`などの中で探を探します。

クラスでは、このメカニズムに直接さらされることはありませんが、 `extends`は古き良きプロトタイプチェーンの上でも機能します。 それが私たちのReactクラスインスタンスが `setState`のようなメソッドにアクセスする方法です：

```javascript
  class Greeting extends React.Component {
    render() {
      return <p>Hello</p>;
    }
  }
  
  let c = new Greeting();
  console.log(c.__proto__); // Greeting.prototype
  console.log(c.__proto__.__proto__); // React.Component.prototype
  console.log(c.__proto__.__proto__.__proto__); // Object.prototype
  
  c.render();      // Found on c.__proto__ (Greeting.prototype)
  c.setState();    // Found on c.__proto__.__proto__ (React.Component.prototype)
  c.toString();    // Found on c.__proto__.__proto__.__proto__ (Object.prototype)
```

言い換えれば、**クラスを使うとき、インスタンスの `__proto__`チェーンはクラス階層を反映しています。：**

```javascript
  // `extends` chain
  Greeting
    → React.Component
      → Object (implicitly)
  
  // `__proto__` chain
  new Greeting()
    → Greeting.prototype
      → React.Component.prototype
        → Object.prototype
```

---

`__proto__`チェーンはクラス階層を反映しているので、`Greeting.prototype`から始めて、
その`__proto__`チェーンをたどることで`Greeting`が `React.Component`を拡張しているかどうかをチェックすることができます。

```javascript
  // `__proto__` chain
  new Greeting()
    → Greeting.prototype // 🕵️ ここから始める
      → React.Component.prototype // ✅ 見つけた!
        → Object.prototype
```

便利なことに、 `x instanceof Y`はまさにこの検索を行います。 それは `x.__ proto__`チェーンで `Y.prototype`を探します。
通常は、何かがクラスのインスタンスであるかどうかを判断するために使用されます。

```javascript
  let greeting = new Greeting();
  
  console.log(greeting instanceof Greeting); // true
  // greeting (🕵️‍ ここから始める)
  //   .__proto__ → Greeting.prototype (✅ 見つけた!)
  //     .__proto__ → React.Component.prototype 
  //       .__proto__ → Object.prototype
  
  console.log(greeting instanceof React.Component); // true
  // greeting (🕵️‍ ここから始める)
  //   .__proto__ → Greeting.prototype
  //     .__proto__ → React.Component.prototype (✅ 見つけた!)
  //       .__proto__ → Object.prototype
  
  console.log(greeting instanceof Object); // true
  // greeting (🕵️‍ ここから始める)
  //   .__proto__ → Greeting.prototype
  //     .__proto__ → React.Component.prototype
  //       .__proto__ → Object.prototype (✅ 見つけた!)
  
  console.log(greeting instanceof Banana); // false
  // greeting (🕵️‍ ここから始める)
  //   .__proto__ → Greeting.prototype
  //     .__proto__ → React.Component.prototype 
  //       .__proto__ → Object.prototype (🙅‍ 見つからなかった)
```

しかし、あるクラスが別のクラスを継承しているかどうかを判断するのと同じようにうまくいきます。

```javascript
  console.log(Greeting.prototype instanceof React.Component);
  // greeting
  //   .__proto__ → Greeting.prototype (🕵️‍ We start here)
  //     .__proto__ → React.Component.prototype (✅ Found it!)
  //       .__proto__ → Object.prototype
```

そしてこのチェックは、Reactコンポーネントクラスなのか通常の関数なのかを判断する方法です。

---

しかしこれはReactがすることではありません。 😳

`instanceof`ソリューションの注意点の1つは、ページ上にReactのコピーが複数ある場合、それが機能しないことです。

1つのプロジェクトにReactの複数のコピーを混在させるのは、いくつかの理由で好ましくありませんが、私たちはこれまで可能な限り問題を避けるようにしてきました。
(Hooksの場合、重複排除を強制する[必要がある](https://github.com/facebook/react/issues/13991)かもしれません。)

もう1つの可能性のある発見的方法は、プロトタイプ上の `render`メソッドの存在をチェックすることです。
ただし、その当時は、コンポーネントAPIがどのように進化するのか[明確ではありませんでした](https://github.com/facebook/react/issues/4599#issuecomment-129714112)。
すべてのチェックにはコストがかかるため、複数を追加することは望ましくありません。 
クラスプロパティ構文のように `render`がインスタンスメソッドとして定義されている場合もこれは機能しません。

その代わりに、基本コンポーネントに特別なフラグをReact [追加](https://github.com/facebook/react/pull/4663)します。
Reactはそのフラグの存在をチェックし、それがReactコンポーネントクラスであるかどうかを知る方法です。

もともとフラグはReact.Componentクラス自体にありました：

```javascript
  // React内部
  class Component {}
  Component.isReactClass = {};

  // こんな感じでチェックできます。
  class Greeting extends Component {}
  console.log(Greeting.isReactClass); // ✅ Yes
```   

However, some class implementations we wanted to target [did not](https://github.com/scala-js/scala-js/issues/1900) copy static properties (or set the non-standard `__proto__`), so the flag was getting lost.
This is why React [moved](https://github.com/facebook/react/pull/5021) this flag to `React.Component.prototype`:

```javascript
  // React内部
  class Component {}
  Component.prototype.isReactComponent = {};

  // こんな感じでチェックできます。
  class Greeting extends Component {}
  console.log(Greeting.prototype.isReactComponent); // ✅ Yes
```

**そしてこれは文字通りすべてです。**

You might be wondering why it’s an object and not just a boolean. It doesn’t matter much in practice but early versions of Jest (before Jest was Good™️) had automocking turned on by default. The generated mocks omitted primitive properties, [breaking the check](https://github.com/facebook/react/pull/4663#issuecomment-136533373). Thanks, Jest.
なぜそれが単なるブール値ではなくオブジェクトであるのか疑問に思うかもしれません。 実際にはそれほど重要ではありませんが、Jestの初期のバージョン(JestがGood™️以前のバージョン)では、デフォルトで自動モックが有効になっていました。 生成されたモックはプリミティブプロパティを削除しました。[変更を見る。](https://github.com/facebook/react/pull/4663#issuecomment-136533373) ありがとう、Jest。

`isReactComponent`チェックは今日[Reactで使われています。](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L297-L300)

`React.Component`を継承しないのであれば、Reactはプロトタイプ上で` isReactComponent`を見つけることができず、コンポーネントをクラスとして扱うこともできません。
今、あなたは`Cannot call a class as a function`のエラーに対する[最も支持された答え](https://stackoverflow.com/questions/38481857/getting-cannot-call-a-class-as-a-function-in-my-react-project/42680526#42680526)が`extends React.Component`を追加することである理由はわかりますね。
最後に、`prototype.render`が存在するが` prototype.isReactComponent`が存在しない場合に警告する警告が追加されました。

---

もしかしたらあなたはこの物語が引っ掛けだと言うかもしれません。
**実際の解決策は非常に単純ですが、Reactがこの解決策を採用した理由とその代替案について説明するために、話を大きく脱線しました。**

私の経験ではこれは多くの場合ライブラリのAPIの場合です。
APIを使いやすくするためには、言語のセマンティクス（将来の方向性を含むいくつかの言語について）、実行時のパフォーマンス、コンパイルの手順、エコシステムの状態、およびパッケージソリューション、早期警告など、多くのことを考慮する必要があります。
 最終的な結果は必ずしも最も洗練されたものではないかもしれませんが、それは実用的でなければなりません。

**最終的なAPIが成功した場合、そのユーザーはこのプロセスについて考える必要はありません**。 代わりに、彼らはアプリの作成に集中することができます。

しかし、あなたも興味があれば…それがどのように動くのか知っているのはいいことです。