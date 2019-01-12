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
ステートの機能を使うための唯一の方法でした）


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

どちらのケースでもReactの目的は描画したノードを取得することです。（この例では`<p>Hello</p>`)
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
（チェックなしで「ルーズモード」を選択することによってバンドルサイズを減らすことができますが、これは実際のネイティブクラスへの最終的な移行を複雑にするかもしれません。）
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
これはコンストラクタとして書かれた関数（上記の `Person`のように）には望ましいですが、関数のコンポーネントには混乱を招くでしょう：

```javascript
  function Greeting() {
    // ここで `this`が他の種類のインスタンスであるとも思わないでしょう
    return <p>Hello</p>;
  }
```

それは許容できるかもしれません。 この考えを殺すのには他に2つの理由があります。

---

The first reason why always using `new` wouldn’t work is that for native arrow functions (not the ones compiled by Babel), calling with `new` throws an error:
常に`new`を使用してもうまくいかない最初の理由は、ネイティブのarrow関数（Babelによってコンパイルされたものではない）では、`new`を指定して呼び出すとエラーが発生するためです。

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
しかし、`prototype`の欠如によってarrow関数を検出を試みることができます、そしてそれらに`new`はつきません。

```javascript
  (() => {}).prototype // undefined
  (function() {}).prototype // {constructor: f}
```

But this  for functions compiled with Babel. 
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

これもまた、[`new`演算子]（https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new）設計に関係しています。 前に見たように、 `new`はJavaScriptエンジンにオブジェクトを作成し、そのオブジェクトを関数の中での`this`にし、そして後で `new`の結果としてそのオブジェクトを渡すように伝えます。

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

There is just no way to read a primitive return value (like a number or a string) from a function when calling it with `new`. So if React always used `new`, it would be unable to add support components that return strings!

That’s unacceptable so we need to compromise.

---

What did we learn so far? React needs to call classes (including Babel output) _with_ `new` but it needs to call regular functions or arrow functions (including Babel output) _without_ `new`. And there is no reliable way to distinguish them.

**If we can’t solve a general problem, can we solve a more specific one?**

When you define a component as a class, you’ll likely want to extend `React.Component` for built-in methods like `this.setState()`. **Rather than try to detect all classes, can we detect only `React.Component` descendants?**

Spoiler: this is exactly what React does.

---

Perhaps, the idiomatic way to check if `Greeting` is a React component class is by testing if `Greeting.prototype instanceof React.Component`:

    class A {}
    class B extends A {}
    
    console.log(B.prototype instanceof A); // true

I know what you’re thinking. What just happened here?! To answer this, we need to understand JavaScript prototypes.

You might be familiar with the “prototype chain”. Every object in JavaScript might have a “prototype”. When we write `fred.sayHi()` but `fred` object has no `sayHi` property, we look for `sayHi` property on `fred`’s prototype. If we don’t find it there, we look at the next prototype in the chain — `fred`’s prototype’s prototype. And so on.

**Confusingly, the `prototype` property of a class or a function _does not_ point to the prototype of that value.** I’m not kidding.

    function Person() {}
    
    console.log(Person.prototype); // 🤪 Not Person's prototype
    console.log(Person.__proto__); // 😳 Person's prototype

So the “prototype chain” is more like `__proto__.__proto__.__proto__` than `prototype.prototype.prototype`. This took me years to get.

What’s the `prototype` property on a function or a class, then? **It’s the `__proto__` given to all objects `new`ed with that class or a function!**

    function Person(name) {
      this.name = name;
    }
    Person.prototype.sayHi = function() {
      alert('Hi, I am ' + this.name);
    }
    
    var fred = new Person('Fred'); // Sets `fred.__proto__` to `Person.prototype`

And that `__proto__` chain is how JavaScript looks up properties:

    fred.sayHi();
    // 1. Does fred have a sayHi property? No.
    // 2. Does fred.__proto__ have a sayHi property? Yes. Call it!
    
    fred.toString();
    // 1. Does fred have a toString property? No.
    // 2. Does fred.__proto__ have a toString property? No.
    // 3. Does fred.__proto__.__proto__ have a toString property? Yes. Call it!

In practice, you should almost never need to touch `__proto__` from the code directly unless you’re debugging something related to the prototype chain. If you want to make stuff available on `fred.__proto__`, you’re supposed to put it on `Person.prototype`. At least that’s how it was originally designed.

The `__proto__` property wasn’t even supposed to be exposed by browsers at first because the prototype chain was considered an internal concept. But some browsers added `__proto__` and eventually it was begrudgingly standardized (but deprecated in favor of `Object.getPrototypeOf()`).

**And yet I still find it very confusing that a property called `prototype` does not give you a value’s prototype** (for example, `fred.prototype` is undefined because `fred` is not a function). Personally, I think this is the biggest reason even experienced developers tend to misunderstand JavaScript prototypes.

---

This is a long post, eh? I’d say we’re 80% there. Hang on.

We know that when say `obj.foo`, JavaScript actually looks for `foo` in `obj`, `obj.__proto__`, `obj.__proto__.__proto__`, and so on.

With classes, you’re not exposed directly to this mechanism, but `extends` also works on top of the good old prototype chain. That’s how our React class instance gets access to methods like `setState`:

    class Greeting extends React.Component {  render() {
        return <p>Hello</p>;
      }
    }
    
    let c = new Greeting();
    console.log(c.__proto__); // Greeting.prototype
    console.log(c.__proto__.__proto__); // React.Component.prototypeconsole.log(c.__proto__.__proto__.__proto__); // Object.prototype
    
    c.render();      // Found on c.__proto__ (Greeting.prototype)
    c.setState();    // Found on c.__proto__.__proto__ (React.Component.prototype)c.toString();    // Found on c.__proto__.__proto__.__proto__ (Object.prototype)

In other words, **when you use classes, an instance’s `__proto__` chain “mirrors” the class hierarchy:**

    // `extends` chain
    Greeting
      → React.Component
        → Object (implicitly)
    
    // `__proto__` chain
    new Greeting()
      → Greeting.prototype
        → React.Component.prototype
          → Object.prototype

2 Chainz.

---

Since the `__proto__` chain mirrors the class hierarchy, we can check whether a `Greeting` extends `React.Component` by starting with `Greeting.prototype`, and then following down its `__proto__` chain:

    // `__proto__` chain
    new Greeting()
      → Greeting.prototype // 🕵️ We start here    → React.Component.prototype // ✅ Found it!      → Object.prototype

Conveniently, `x instanceof Y` does exactly this kind of search. It follows the `x.__proto__` chain looking for `Y.prototype` there.

Normally, it’s used to determine whether something is an instance of a class:

    let greeting = new Greeting();
    
    console.log(greeting instanceof Greeting); // true
    // greeting (🕵️‍ We start here)
    //   .__proto__ → Greeting.prototype (✅ Found it!)
    //     .__proto__ → React.Component.prototype 
    //       .__proto__ → Object.prototype
    
    console.log(greeting instanceof React.Component); // true
    // greeting (🕵️‍ We start here)
    //   .__proto__ → Greeting.prototype
    //     .__proto__ → React.Component.prototype (✅ Found it!)
    //       .__proto__ → Object.prototype
    
    console.log(greeting instanceof Object); // true
    // greeting (🕵️‍ We start here)
    //   .__proto__ → Greeting.prototype
    //     .__proto__ → React.Component.prototype
    //       .__proto__ → Object.prototype (✅ Found it!)
    
    console.log(greeting instanceof Banana); // false
    // greeting (🕵️‍ We start here)
    //   .__proto__ → Greeting.prototype
    //     .__proto__ → React.Component.prototype 
    //       .__proto__ → Object.prototype (🙅‍ Did not find it!)

But it would work just as fine to determine if a class extends another class:

    console.log(Greeting.prototype instanceof React.Component);
    // greeting
    //   .__proto__ → Greeting.prototype (🕵️‍ We start here)
    //     .__proto__ → React.Component.prototype (✅ Found it!)
    //       .__proto__ → Object.prototype

And that check is how we could determine if something is a React component class or a regular function.

---

That’s not what React does though. 😳

One caveat to the `instanceof` solution is that it doesn’t work when there are multiple copies of React on the page, and the component we’re checking inherits from _another_ React copy’s `React.Component`. Mixing multiple copies of React in a single project is bad for several reasons but historically we’ve tried to avoid issues when possible. (With Hooks, we [might need to](https://github.com/facebook/react/issues/13991) force deduplication though.)

One other possible heuristic could be to check for presence of a `render` method on the prototype. However, at the time it [wasn’t clear](https://github.com/facebook/react/issues/4599#issuecomment-129714112) how the component API would evolve. Every check has a cost so we wouldn’t want to add more than one. This would also not work if `render` was defined as an instance method, such as with the class property syntax.

So instead, React [added](https://github.com/facebook/react/pull/4663) a special flag to the base component. React checks for the presence of that flag, and that’s how it knows whether something is a React component class or not.

Originally the flag was on the base `React.Component` class itself:

    // Inside React
    class Component {}
    Component.isReactClass = {};
    
   