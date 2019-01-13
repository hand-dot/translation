# setStateはどうやって何をするか知る？

以下は[How Does setState Know What to Do?](https://overreacted.io/how-does-setstate-know-what-to-do/)  の日本語訳です。

コンポーネントの中で `setState`を呼び出すとき、何が起こると思いますか？

```js
import React from 'react';
import ReactDOM from 'react-dom';

class Button extends React.Component {
  constructor(props) {
    super(props);
    this.state = { clicked: false };
    this.handleClick = this.handleClick.bind(this);
  }
  handleClick() {
    this.setState({ clicked: true });  }
  render() {
    if (this.state.clicked) {
      return <h1>Thanks</h1>;
    }
    return (
      <button onClick={this.handleClick}>
        Click me!
      </button>
    );
  }
}

ReactDOM.render(<Button />, document.getElementById('container'));
```  

そう、次の `{clicked：true}`の状態でReactはコンポーネントを再レンダリングし、返された `<h1> Thanks </h1>`要素と一致するようにDOMを更新します。
簡単そうに見えますね。しかし、待ってください _React_ がしますか？それとも _React DOM_ ？
DOMを更新することは、React DOMの責務のように思えます。
しかし、React DOMのものではない、`this.setState()`を呼び出しています。 そして私たちの `React.Component`クラスはReactの内部で定義されています。

それでは、どのようにして `React.Component`内の` setState()`がDOMを更新することができるのでしょうか。

**免責事項：このブログの他のほとんど([これとか](https://overreacted.io/why-do-react-elements-have-typeof-property/)[これとか](https://overreacted.io/how-does-react-tell-a-class-from-a-function/)[これ](https://overreacted.io/why-do-we-write-super-props/))の投稿と同じように、Reactを効率的に使うために実際に知っておく必要はありません。この記事は、カーテンの裏に何があるのかを知りたい人のためのものです。 完全にオプション！**

---

`React.Component`クラスはDOM更新ロジックを含んでいると思うかもしれません。

しかし、そうであれば、 `this.setState()`は他の環境でどのように機能するのでしょうか？ 例えば、React Nativeアプリケーションのコンポーネントは `React.Component`も継承しています。
これらは上記と同じように `this.setState()`を呼び出しますが、React NativeはDOMの代わりにAndroidおよびiOSのネイティブビューで動作します。

React Test RendererまたはShallow Rendererについてもお詳しいかもしれませんが、 どちらのテスト方法でも通常のコンポーネントをレンダリングしてその中で `this.setState()`を呼び出すことができますがどちらもDOMとは連携できません。

[React ART](https://github.com/facebook/react/tree/master/packages/react-art)のようなレンダラーを使用した場合は、ページ上で複数のレンダラーを使用することが可能であることもわかります。 （たとえば、ARTコンポーネントはReact DOMツリー内で機能します。）これにより、グローバルフラグまたは変数が保持できなくなります。


だからどういうわけか **`React.Component`は状態の更新を扱うことをプラットフォーム固有のコードに委任します。** これがどのように起こるかを理解する前に、パッケージがどのように分離されるかそしてその理由を深く掘り下げましょう。

---

Reactの「エンジン」は `react`パッケージの中にあるという一般的な誤解があります。 これは事実ではないです。

実際、パッケージが[React 0.14](https://reactjs.org/blog/2015/07/03/react-v0.14-beta-1.html#two-packages)で分割されて以来、`react`パッケージは意図的にコンポーネントを定義するためのAPIのみを公開しています。 Reactの実装の大部分は「レンダラー」にあります。

`react-dom`、` react-dom / server`、 `react-native`、` react-test-renderer`、 `react-art`はレンダラーの例です（そして[自分で作ることもできます](https://github.com/facebook/react/blob/master/packages/react-reconciler/README.md#practical-examples)）。

これはあなたがどのプラットフォームをターゲットにしているかに関わらず `react`パッケージが便利だからです。 すべてのエクスポートは以下のとおりです。`React.Component`、` React.createElement`、 `React.Children` そして（最終的には）[Hooks](https://reactjs.org/docs/hooks-intro.html) これらはターゲットプラットフォームから独立しています。

React DOM、React DOMサーバー、Reactネイティブのいずれを実行しても、コンポーネントはインポートして同じ方法で使用します。

対照的に、レンダラパッケージはReact階層をDOMノードにマウントすることを可能にする `ReactDOM.render()`のようなプラットフォーム特有のAPIを公開します。各レンダラーはこのようなAPIを提供しま
理想的には、ほとんどのコンポーネントはレンダラーから何かをインポートする必要はありません。 これにより、移植性が高まります。

**What most people imagine as the React “engine” is inside each individual renderer.** Many renderers include a copy of the same code — we call it the [“reconciler”](https://github.com/facebook/react/tree/master/packages/react-reconciler). A [build step](https://reactjs.org/blog/2017/12/15/improving-the-repository-infrastructure.html#migrating-to-google-closure-compiler) smooshes the reconciler code together with the renderer code into a single highly optimized bundle for better performance. (Copying code is usually not great for bundle size but the vast majority of React users only needs one renderer at a time, such as `react-dom`.)

The takeaway here is that the `react` package only lets you _use_ React features but doesn’t know anything about _how_ they’re implemented. The renderer packages (`react-dom`, `react-native`, etc) provide the implementation of React features and platform-specific logic. Some of that code is shared (“reconciler”) but that’s an implementation detail of individual renderers.

---

Now we know why _both_ `react` and `react-dom` packages need to be updated for new features. For example, when React 16.3 added the Context API, `React.createContext()` was exposed on the React package.

But `React.createContext()` doesn’t actually _implement_ the context feature. The implementation would need to be different between React DOM and React DOM Server, for example. So `createContext()` returns a few plain objects:

```js
// A bit simplified
function createContext(defaultValue) {
  let context = {
    _currentValue: defaultValue,
    Provider: null,
    Consumer: null
  };
  context.Provider = {
    $$typeof: Symbol.for('react.provider'),
    _context: context
  };
  context.Consumer = {
    $$typeof: Symbol.for('react.context'),
    _context: context,
  };
  return context;
}
```

When you use `<MyContext.Provider>` or `<MyContext.Consumer>` in the code, it’s the _renderer_ that decides how to handle them. React DOM might track context values in one way, but React DOM Server might do it differently.

**So if you update `react` to 16.3+ but don’t update `react-dom`, you’d be using a renderer that isn’t yet aware of the special `Provider` and `Consumer` types.** This is why an older `react-dom` would [fail saying these types are invalid](https://stackoverflow.com/a/49677020/458193).

The same caveat applies to React Native. However, unlike React DOM, a React release doesn’t immediately “force” a React Native release. They have an independent release schedule. The updated renderer code is [separately synced](https://github.com/facebook/react-native/commits/master/Libraries/Renderer/oss) into the React Native repository once in a few weeks. This is why features become available in React Native on a different schedule than in React DOM.

---

Okay, so now we know that the `react` package doesn’t contain anything interesting, and the implementation lives in renderers like `react-dom`, `react-native`, and so on. But that doesn’t answer our question. How does `setState()` inside `React.Component` “talk” to the right renderer?

**The answer is that every renderer sets a special field on the created class.** This field is called `updater`. It’s not something _you_ would set — rather, it’s something React DOM, React DOM Server or React Native set right after creating an instance of your class:

```js
// Inside React DOM
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactDOMUpdater;
// Inside React DOM Server
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactDOMServerUpdater;
// Inside React Native
const inst = new YourComponent();
inst.props = props;
inst.updater = ReactNativeUpdater;
```

Looking at the [`setState` implementation in `React.Component`](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react/src/ReactBaseClasses.js#L58-L67), all it does is delegate work to the renderer that created this component instance:

    // A bit simplified
    setState(partialState, callback) {
      // Use the `updater` field to talk back to the renderer!
      this.updater.enqueueSetState(this, partialState, callback);
    }

React DOM Server [might want to](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-dom/src/server/ReactPartialRenderer.js#L442-L448) ignore a state update and warn you, whereas React DOM and React Native would let their copies of the reconciler [handle it](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-reconciler/src/ReactFiberClassComponent.js#L190-L207).

And this is how `this.setState()` can update the DOM even though it’s defined in the React package. It reads `this.updater` which was set by React DOM, and lets React DOM schedule and handle the update.

---

We know about classes now, but what about Hooks?

When people first look at the [Hooks proposal API](https://reactjs.org/docs/hooks-intro.html), they often wonder: how does `useState` “know what to do”? The assumption is that it’s more “magical” than a base `React.Component` class with `this.setState()`.

But as we have seen today, the base class `setState()` implementation has been an illusion all along. It doesn’t do anything except forwarding the call to the current renderer. And `useState` Hook [does exactly the same thing](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react/src/ReactHooks.js#L55-L56).

**Instead of an `updater` field, Hooks use a “dispatcher” object.** When you call `React.useState()`, `React.useEffect()`, or another built-in Hook, these calls are forwarded to the current dispatcher.

```js
// In React (simplified a bit)
const React = {
  // Real property is hidden a bit deeper, see if you can find it!
  __currentDispatcher: null,

  useState(initialState) {
    return React.__currentDispatcher.useState(initialState);
  },

  useEffect(initialState) {
    return React.__currentDispatcher.useEffect(initialState);
  },
  // ...
};
```

And individual renderers set the dispatcher before rendering your component:

```js
// In React DOM
const prevDispatcher = React.__currentDispatcher;
React.__currentDispatcher = ReactDOMDispatcher;
let result;
try {
  result = YourComponent(props);
} finally {
  // Restore it back
  React.__currentDispatcher = prevDispatcher;
}
```

For example, the React DOM Server implementation is [here](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-dom/src/server/ReactPartialRendererHooks.js#L340-L354), and the reconciler implementation shared by React DOM and React Native is [here](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-reconciler/src/ReactFiberHooks.js).

This is why a renderer such as `react-dom` needs to access the same `react` package that you call Hooks from. Otherwise, your component won’t “see” the dispatcher! This may not work when you have [multiple copies of React](https://github.com/facebook/react/issues/13991) in the same component tree. However, this has always led to obscure bugs so Hooks force you to solve the package duplication before it costs you.

While we don’t encourage this, you can technically override the dispatcher yourself for advanced tooling use cases. (I lied about `__currentDispatcher` name but you can find the real one in the React repo.) For example, React DevTools will use [a special purpose-built dispatcher](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-debug-tools/src/ReactDebugHooks.js#L203-L214) to introspect the Hooks tree by capturing JavaScript stack traces. _Don’t repeat this at home._

This also means Hooks aren’t inherently tied to React. If in the future more libraries want to reuse the same primitive Hooks, in theory the dispatcher could move to a separate package and be exposed as a first-class API with a less “scary” name. In practice, we’d prefer to avoid premature abstraction until there is a need for it.

Both the `updater` field and the `__currentDispatcher` object are forms of a generic programming principle called _dependency injection_. In both cases, the renderers “inject” implementations of features like `setState` into the generic React package to keep your components more declarative.

You don’t need to think about how this works when you use React. We’d like React users to spend more time thinking about their application code than abstract concepts like dependency injection. But if you’ve ever wondered how `this.setState()` or `useState()` know what to do, I hope this helps.

---