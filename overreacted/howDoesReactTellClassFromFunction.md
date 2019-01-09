# Reactはどうやって関数からクラスを見分けるの？

以下は[How Does React Tell a Class from a Function?](https://overreacted.io/how-does-react-tell-a-class-from-a-function/)  の日本語訳です。

関数として定義された`Greeting`コンポーネントについて考えてみましょう


    function Greeting() {
      return <p>Hello</p>;
    }

Reactはclassとしての定義もサポートしています

    class Greeting extends React.Component {
      render() {
        return <p>Hello</p>;
      }
    }

( [最近まで](https://reactjs.org/docs/hooks-intro.html),
ステートの機能を使うための唯一の方法でした）


`<Greeting />`を描画するとき、どのように定義されたか気にする必要はありません。


    // クラスもしくは関数 — なんでも.
    <Greeting />

しかしReact自身は違いを気にする必要があります！

 `Greeting`が関数ならReactは下記のように呼ぶ必要があります

    // あなたのコード
    function Greeting() {
      return <p>Hello</p>;
    }
    
    // React内部
    const result = Greeting(props); // <p>Hello</p>

しかし、もし`Greeting`がクラスの場合、Reactは`new`演算子と作成したインスタンスに対して`render`関数を呼ぶ必要があります。

    // あなたのコード
    class Greeting extends React.Component {
      render() {
        return <p>Hello</p>;
      }
    }
    
    // React内部
    const instance = new Greeting(props); // Greeting {}
    const result = instance.render(); // <p>Hello</p>

どちらのケースでもReactの目的は描画したノードを取得することです。（この例では`<p>Hello</p>`)
しかし、実際のステップはどのように`Greeting`が定義されたかということに依存しています。

**Reactはどのようにしてクラスか関数か知るのでしょうか？**

[前の投稿](/why-do-we-write-super-props/)のように、**Reactで生産的になるためにこれを知る必要はありません。**
私はこれを何年間も知りませんでした。どうかこれを面接の質問にしないでください。
実際、Reactについてというよりも、Javascriptについての投稿です。

このブログはReactがなぜこのように動いているのか知りたい好奇心の強い読者向けです。あなたはそのような人ですか？一緒に深掘りしてみましょう。

**This is a long journey. Buckle up. This post doesn’t have much information about React itself, but we’ll go through some aspects of `new`, `this`, `class`, arrow functions, `prototype`, `__proto__`, `instanceof`, and how those things work together in JavaScript. Luckily, you don’t need to think about those as much when you _use_ React. If you’re implementing React though…**


(If you really just want to know the answer, scroll to the very end.)

* * *

First, we need to understand why it’s important to treat functions and classes differently. Note how we use the `new` operator when calling a class:

    // If Greeting is a function
    const result = Greeting(props); // <p>Hello</p>
    
    // If Greeting is a class
    const instance = new Greeting(props); // Greeting {}const result = instance.render(); // <p>Hello</p>

Let’s get a rough sense of what the `new` operator does in JavaScript.

* * *

In the old days, JavaScript did not have classes. However, you could express a similar pattern to classes using plain functions. **Concretely, you can use _any_ function in a role similar to a class constructor by adding `new` before its call:**

    // Just a function
    function Person(name) {
      this.name = name;
    }
    
    var fred = new Person('Fred'); // ✅ Person {name: 'Fred'}
    var george = Person('George'); // 🔴 Won’t work

You can still write code like this today! Try it in DevTools.

If you called `Person('Fred')` **without** `new`, `this` inside it would point to something global and useless (for example, `window` or `undefined`). So our code would crash or do something silly like setting `window.name`.

By adding `new` before the call, we say: “Hey JavaScript, I know `Person` is just a function but let’s pretend it’s something like a class constructor. **Create an `{}` object and point `this` inside the `Person` function to that object so I can assign stuff like `this.name`. Then give that object back to me.**”

That’s what the `new` operator does.

    var fred = new Person('Fred'); // Same object as `this` inside `Person`

The `new` operator also makes anything we put on `Person.prototype` available on the `fred` object:

    function Person(name) {
      this.name = name;
    }
    Person.prototype.sayHi = function() {  alert('Hi, I am ' + this.name);}
    var fred = new Person('Fred');
    fred.sayHi();

This is how people emulated classes before JavaScript added them directly.

* * *

So `new` has been around in JavaScript for a while. However, classes are more recent. They let us rewrite the code above to match our intent more closely:

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

_Capturing developer’s intent_ is important in language and API design.

If you write a function, JavaScript can’t guess if it’s meant to be called like `alert()` or if it serves as a constructor like `new Person()`. Forgetting to specify `new` for a function like `Person` would lead to confusing behavior.

**Class syntax lets us say: “This isn’t just a function — it’s a class and it has a constructor”.** If you forget `new` when calling it, JavaScript will raise an error:

    let fred = new Person('Fred');
    // ✅  If Person is a function: works fine
    // ✅  If Person is a class: works fine too
    
    let george = Person('George'); // We forgot `new`
    // 😳 If Person is a constructor-like function: confusing behavior
    // 🔴 If Person is a class: fails immediately

This helps us catch mistakes early instead of waiting for some obscure bug like `this.name` being treated as `window.name` instead of `george.name`.

However, it means that React needs to put `new` before calling any class. It can’t just call it as a regular function, as JavaScript would treat it as an error!

    class Counter extends React.Component {
      render() {
        return <p>Hello</p>;
      }
    }
    
    // 🔴 React can't just do this:
    const instance = Counter(props);

This spells trouble.

* * *

Before we see how React solves this, it’s important to remember most people using React use compilers like Babel to compile away modern features like classes for older browsers. So we need to consider compilers in our design.

In early versions of Babel, classes could be called without `new`. However, this was fixed — by generating some extra code:

    function Person(name) {
      // A bit simplified from Babel output:
      if (!(this instanceof Person)) {
        throw new TypeError("Cannot call a class as a function");
      }
      // Our code:
      this.name = name;
    }
    
    new Person('Fred'); // ✅ Okay
    Person('George');   // 🔴 Cannot call a class as a function

You might have seen code like this in your bundle. That’s what all those `_classCallCheck` functions do. (You can reduce the bundle size by opting into the “loose mode” with no checks but this might complicate your eventual transition to real native classes.)

* * *

By now, you should roughly understand the difference between calling something with `new` or without `new`:

`new Person()`

`Person()`

`class`

✅ `this` is a `Person` instance

🔴 `TypeError`

`function`

✅ `this` is a `Person` instance

😳 `this` is `window` or `undefined`

This is why it’s important for React to call your component correctly. **If your component is defined as a class, React needs to use `new` when calling it.**

So can React just check if something is a class or not?

Not so easy! Even if we could [tell a class from a function in JavaScript](https://stackoverflow.com/questions/29093396/how-do-you-check-the-difference-between-an-ecmascript-6-class-and-function), this still wouldn’t work for classes processed by tools like Babel. To the browser, they’re just plain functions. Tough luck for React.

* * *

Okay, so maybe React could just use `new` on every call? Unfortunately, that doesn’t always work either.

With regular functions, calling them with `new` would give them an object instance as `this`. It’s desirable for functions written as constructor (like our `Person` above), but it would be confusing for function components:

    function Greeting() {
      // We wouldn’t expect `this` to be any kind of instance here
      return <p>Hello</p>;
    }

That could be tolerable though. There are two _other_ reasons that kill this idea.

* * *

The first reason why always using `new` wouldn’t work is that for native arrow functions (not the ones compiled by Babel), calling with `new` throws an error:

    const Greeting = () => <p>Hello</p>;
    new Greeting(); // 🔴 Greeting is not a constructor

This behavior is intentional and follows from the design of arrow functions. One of the main perks of arrow functions is that they _don’t_ have their own `this` value — instead, `this` is resolved from the closest regular function:

    class Friends extends React.Component {
      render() {    const friends = this.props.friends;
        return friends.map(friend =>
          <Friend
            // `this` is resolved from the `render` method        size={this.props.size}        name={friend.name}
            key={friend.id}
          />
        );
      }
    }

Okay, so **arrow functions don’t have their own `this`.** But that means they would be entirely useless as constructors!

    const Person = (name) => {
      // 🔴 This wouldn’t make sense!
      this.name = name;
    }

Therefore, **JavaScript disallows calling an arrow function with `new`.** If you do it, you probably made a mistake anyway, and it’s best to tell you early. This is similar to how JavaScript doesn’t let you call a class _without_ `new`.

This is nice but it also foils our plan. React can’t just call `new` on everything because it would break arrow functions! We could try detecting arrow functions specifically by their lack of `prototype`, and not `new` just them:

    (() => {}).prototype // undefined
    (function() {}).prototype // {constructor: f}

But this [wouldn’t work](https://github.com/facebook/react/issues/4599#issuecomment-136562930) for functions compiled with Babel. This might not be a big deal, but there is another reason that makes this approach a dead end.

* * *

Another reason we can’t always use `new` is that it would preclude React from supporting components that return strings or other primitive types.

    function Greeting() {
      return 'Hello';
    }
    
    Greeting(); // ✅ 'Hello'
    new Greeting(); // 😳 Greeting {}

This, again, has to do with the quirks of the [`new` operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new) design. As we saw earlier, `new` tells the JavaScript engine to create an object, make that object `this` inside the function, and later give us that object as a result of `new`.

However, JavaScript also allows a function called with `new` to _override_ the return value of `new` by returning some other object. Presumably, this was considered useful for patterns like pooling where we want to reuse instances:

    // Created lazilyvar zeroVector = null;
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
    var b = new Vector(0, 0);var c = new Vector(0, 0); // 😲 b === c

However, `new` also _completely ignores_ a function’s return value if it’s _not_ an object. If you return a string or a number, it’s like there was no `return` at all.

    function Answer() {
      return 42;
    }
    
    Answer(); // ✅ 42
    new Answer(); // 😳 Answer {}

There is just no way to read a primitive return value (like a number or a string) from a function when calling it with `new`. So if React always used `new`, it would be unable to add support components that return strings!

That’s unacceptable so we need to compromise.

* * *

What did we learn so far? React needs to call classes (including Babel output) _with_ `new` but it needs to call regular functions or arrow functions (including Babel output) _without_ `new`. And there is no reliable way to distinguish them.

**If we can’t solve a general problem, can we solve a more specific one?**

When you define a component as a class, you’ll likely want to extend `React.Component` for built-in methods like `this.setState()`. **Rather than try to detect all classes, can we detect only `React.Component` descendants?**

Spoiler: this is exactly what React does.

* * *

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

* * *

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

* * *

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

* * *

That’s not what React does though. 😳

One caveat to the `instanceof` solution is that it doesn’t work when there are multiple copies of React on the page, and the component we’re checking inherits from _another_ React copy’s `React.Component`. Mixing multiple copies of React in a single project is bad for several reasons but historically we’ve tried to avoid issues when possible. (With Hooks, we [might need to](https://github.com/facebook/react/issues/13991) force deduplication though.)

One other possible heuristic could be to check for presence of a `render` method on the prototype. However, at the time it [wasn’t clear](https://github.com/facebook/react/issues/4599#issuecomment-129714112) how the component API would evolve. Every check has a cost so we wouldn’t want to add more than one. This would also not work if `render` was defined as an instance method, such as with the class property syntax.

So instead, React [added](https://github.com/facebook/react/pull/4663) a special flag to the base component. React checks for the presence of that flag, and that’s how it knows whether something is a React component class or not.

Originally the flag was on the base `React.Component` class itself:

    // Inside React
    class Component {}
    Component.isReactClass = {};
    
   