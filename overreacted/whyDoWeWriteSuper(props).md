# なぜ私たちは `super(props)` を書くの?

以下は[Why Do We Write super(props)?](https://overreacted.io/why-do-we-write-super-props/)  の日本語訳です。

`Hooks`が最新でアツいって聞いたよ。皮肉なことだけどクラスコンポーネントの楽しい事実について述べてブログをスタートしたい。どうだ！

**これらの潜在的問題はReactを効率的に使うためには重要じゃない。でも、もしどうやって動いているか深く掘り下げることが好きなら面白いかもね**

これが最初のやつ。

---

私は人生で `super(props)`  必要以上に何度も書いたよ

```javascript
class Checkbox extends React.Component {
  constructor(props) {
    super(props); 
    this.state = { isOn: true };
  }
  // ...
}
```

もちろん、`class fields proposal `なら儀式(constructor)をスキップできる。

```javascript
class Checkbox extends React.Component {
  state = { isOn: true };
  // ...
}
```

2015年に`React 0.13`がプレーンクラスのサポートを追加したとき、こんな感じの構文が[計画](https://reactjs.org/blog/2015/01/27/react-v0.13.0-beta-1.html#es7-property-initializers)されていたよ。

コンストラクタの定義と`super(props)`の呼び出しは常にクラスフィールドが人間工学に基づいた代替手段を提供するまでの一時的な解決策だった。

**なぜ私たちはsuperを呼ぶの？呼ばなくてもいいの？もし呼ばないといけないなら、`props`を呼ばなかったらなにが起こるんですか？他の引数はあるの？確認してみましょう。**

---

JavaScriptではsuperは親クラスのコンストラクタを参照します。(この例では、親クラスはReact.Component実装を指しています。)

重要なのは、JavaScriptはあなたがコンストラクターで親のコンストラクターを呼ぶまで`this`は使わせてくれません。

```javascript
class Checkbox extends React.Component {
  constructor(props) {
    // 🔴 `this` はまだ使えない
    super(props);
    // ✅ 今なら使える
    this.state = { isOn: true };
  }
  // ...
}
```

あなたがthisを使う前にJavascriptが親のコンスラクターの実行を強制させるのには理由があります。クラス階層を考えてみてください

```javascript
class Person {
  constructor(name) {
    this.name = name;
  }
}

class PolitePerson extends Person {
  constructor(name) {
    this.greetColleagues(); // 🔴 ここでは許可されていない, 下記をご確認ください
    super(name);
  }
  greetColleagues() {
    alert('Good morning folks!');
  }
}
```

`super`の前に`this`の使用が許可されていた場合のことを想像してみてください。
一ヶ月後、`greetColleagues`に人の名前を入れるかもしれません。

```javascript
greetColleagues() {
    alert('Good morning folks!');
    alert('My name is ' + this.name + ', nice to meet you!');
  }
```

しかし`this.greetColleagues()`は`super()`の前に呼ばれていることを忘れちゃいました。そう、`this.name`はまだ定義されていません!
ご覧のとおり、このようなコードは考えるのが非常に難しいのです。

この落とし穴を避けるために **Javascriptはコンストラクターでthisを使いたい場合に`super`の呼び出しを強制します。**
そして、この制限はクラス定義されたReactのコンポーネントにも適用されます。

```javascript
constructor(props) {
    super(props);
    // ✅ ここから`this`が使える
    this.state = { isOn: true };
  }
```

他の疑問が残っています。なぜpropsを引数に渡すの？

---

基本的な`React.Component`でコンストラクターが`this.props`を初期化するために、`props`を`super`に渡すことが必要と思うかも知れません。

```javascript
// React内部
class Component {
  constructor(props) {
    this.props = props;
    // ...
  }
}
```

真実からそれほど遠くないですよ。[確かにやっています](https://github.com/facebook/react/blob/1d25aa5787d4e19704c049c3cfa985d3b5190e0d/packages/react/src/ReactBaseClasses.js#L22)

しかし、どういうわけか引数(`props`)なしの`super()`で呼び出しても、
`this.props`に`render`や他のメソッド内でアクセスできます。(信じないなら試してみて!)

どうやって動いているんだ？
