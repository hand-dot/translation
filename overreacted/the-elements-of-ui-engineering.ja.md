---
title: 'UI開発の要素'
date: '2018-12-30'
langs: ['en', 'ja']
spoiler: UI開発を難しくするものは何ですか?
---

[前の記事で](/things-i-dont-know-as-of-2018/)、知識のギャップについて認めるということについて話しました。
あなたは私が平凡な提案をすると結論づけるかもしれませんが、違います！この記事は広い分野を扱います。

私はあなたが「どこからでも始める」ことができ、特定の順序で技術を学ぶ必要はないと強く信じます。しかし私は専門知識を得ることにも大きな価値を置いています。個人的には、主にユーザーインターフェイスの作成に興味を持っていました。

**私は、自分が知っていて価値のある物を熟考し続けています。** 確かに、私はいくつかのテクノロジ（JavaScriptやReactなど）に精通していますが、経験から得られるもっと重要な教訓はとらえどころがありません。私はそれらを言葉にしようとはしませんでしたが、それらを一覧にして述べることは私にとって最初の試みです。

---

テクノロジとライブラリに関する「学習ロードマップ」はたくさんあります。2019年にどのライブラリが流行するのでしょうか？ 2020はどうですか？ あなたはVueかReactを学ぶべきですか？ Angularですか？ReduxやRxはどうですか？ アポロを学ぶ必要がありますか？ RESTまたはGraphQL？ 迷子になるのは簡単です。 作者が間違っているとどうなりますか？

**私の最大の画期的な学びは特定のテクノロジに関するものではありませんでした。** むしろ、特定のUIの問題を解決するのに苦労したときに最も多くを学びました。私を助けてくれたライブラリやパターンを時々後から発見するでしょう。 他には、自分自身の解決策（良いものも悪いものも）を考え出すでしょう。

それは*問題*を理解すること、*解決策*を試すこと、そして私の人生で最もやりがいのある学びをもたらした*異なる*戦略*を適用すること の組み合わせです。 **この記事は問題だけに焦点を当てています**

---

ユーザーインターフェースを加工したら直接もしくはライブラリを用いてこれらの課題にいくつか対処したことがあると思います。どちらの場合もライブラリを使用しない小さなアプリを作成し、これらの問題を再現して解決することをお勧めします。それらのどれに対しても正解はありません。学習は問題の領域を探り、さまざまな可能性のあるトレードオフを試すことから始まります。

---

* **整合性** 「いいね」ボタンをクリックすると、「あなたと他の3人の友人がこの投稿を気に入っています」というテキストが更新されます。もう一度クリックすると、テキストは反転します。 簡単そうですね。 しかし、おそらくこのようなラベルが画面上のいくつかの場所に存在します。 変更する必要がある他の視覚的な表示（ボタンの背景など）があるかもしれません。 以前にサーバーから取得され、ホバーに表示されていた「いいねした人」のリストに、クリックした今、あなたの名前が含まれているはずです。 別の画面に移動して戻ってきた場合、投稿はいいねしたことを「忘れて」はいけません。 ローカルの整合性だけでも、一連の課題が発生します。 しかし、他のユーザーも私たちが表示するデータを変更する可能性があります（たとえば、私たちが閲覧している投稿を「いいね」することによって）。 画面の異なる部分で同じデータをどのように同期し、ローカルデータをサーバーとどのようにしていつ整合性を持たせさせますか？

* **応答性** 人は限られた時間の間だけ行動に対する視覚的なフィードバックの欠如を容認することができます。ジェスチャーやスクロールのような*連続*のアクションの場合、この制限は低くなりますが（16msのフレームを1つスキップしても「がらくた」と感じます。）クリックのような*単発*のアクションでは、100ms未満の遅延を*連続*のアクションと同じくらいの速さに感じるという研究があります。 アクションに時間がかかる場合は、視覚的なインジケータを表示する必要があります。 しかし、直感に反する問題がいくつかあります。 ページレイアウトを「ジャンプ」させる、またはいくつかの「ステージ」をロードするインジケーターは、アクションを以前よりも*長く*感じさせます。 同様に、アニメーションのフレームレートを落として20ミリ秒以内にインタラクションを処理すると、フレームレートを落とさず、30ミリ秒以内に処理した場合よりも遅く感じることがあります。 脳はベンチマーカーではありません。 アプリをさまざまな種類の入力にどのように対応させますか？

* **待ち時間** 計算とネットワークアクセスの両方に時間がかかります。デバイスで応答性を損なわない場合、計算コストを無視できることがあります。(ローエンドのデバイスでアプリケーションを確認してみてください). しかし、ネットワークの待ち時間を処理することは避けられません - それは数秒かかることがあります！私たちのアプリはデータやコードがロードされるのを待つだけでフリーズすることはできません。 これは、新しいデータ、コード、またはアセットに依存するすべてのアクションが潜在的に非同期であり、「ロード」を処理する必要があることを意味します。 しかし、それはほとんどすべての画面で起こります。 スピナーやローダーを表示せずに、どのようにして遅延を適切に処理するのでしょうか。 どのように我々は「飛び跳ねる」レイアウトを避けますか？ コードを毎回「再配線」せずに非同期依存関係を変更するにはどうすればよいでしょうか？

* **ナビゲーション** UIは操作しても「状態が固定されている」ことが期待されます。 いきなり物が消えてはいけません。 アプリ内で開始されているか(例:リンクをクリックする)、外部イベントが原因であるか(例:「戻る」ボタンをクリックする)でもナビゲーションもこの原則を尊重する必要があります。たとえば、プロフィール画面で `/profile/likes`と`/profile/follow`のタブを切り替えても、外側の検索入力を消去することはできません。*別の*画面に遷移するのは、部屋に入っていくようなものです。 人々は後で戻って残した物を見つけることを期待しています。(おそらく、いくつかの新しいアイテムがあります) フィードの途中にいる場合、プロフィールをクリックしてから戻ると、フィードでの自分の位置が失われたり、再度読み込みを待つとイライラします。重要なコンテキストを失うことなく任意のナビゲーションを処理するようにアプリをどのように設計しますか？

* **古さ** We can make the “back” button navigation instant by introducing a local cache. In that cache, we can “remember” some data for quick access even if we could theoretically refetch it. But caching brings its own problems. Caches can get stale. If I change an avatar, it should update in the cache too. If I make a new post, it needs to appear in the cache immediately, or the cache needs to be invalidated. This can become difficult and error-prone. What if the posting fails? How long does the cache stay in memory? When we refetch the feed, do we “stitch” the newly fetched feed with the cached one, or throw the cache away? How is pagination or sorting represented in the cache?

* **古さ** ローカルキャッシュを導入することで「戻る」ボタンのナビゲーションを即座にすることができます。 そのキャッシュでは、理論的に再フェッチできたとしても、すばやくアクセスできるようにデータを「記憶」できます。 しかし、キャッシュが古くなることがあり、問題をもたらします。 私がアバターを変更した場合、キャッシュも更新されるべきです。 私が新しい投稿をする場合、それは直ちにキャッシュするか、今のキャッシュを無効する必要があります。 これはプログラムの難易度を上げ、エラーを発生しやすくします。 また、投稿が失敗した場合はどうなりますか？ キャッシュはどのくらいの期間メモリ内に留まりますか？ フィードを再取得したとき、新しく取得したフィードをキャッシュされたフィードと統合しますか？それともキャッシュを破棄しますか？ ページネーションまたはソートはキャッシュでどのように表されますか？

* **Entropy.** The second law of thermodynamics says something like “with time, things turn into a mess” (well, not exactly). This applies to user interfaces too. We can’t predict the exact user interactions and their order. At any point in time, our app may be in one of a mind-boggling number of possible states. We do our best to make the result predictable and limited by our design. We don’t want to look at a bug screenshot and wonder “how did _that_ happen”. For *N* possible states, there are *N×(N–1)* possible transitions between them. For example, if a button can be in one of 5 different states (normal, active, hover, danger, disabled), the code updating the button must be correct for 5×4=20 possible transitions — or forbid some of them. How do we tame the combinatorial explosion of possible states and make visual output predictable?

* **Entropy.** 熱力学の第二法則は、「時間が経てば物事は混乱する」というようなことを言います。（まあ、正確にではありませんが） これはユーザーインターフェイスにも当てはまります。 正確なユーザー操作とその順序を予測することはできません。どの時点でも、私たちのアプリは驚くべき数の状態のうちの1つにあるかもしれません。 私達は結果を予測可能にし、設計によって制限されるように最善を尽くします。 私たちはバグのスクリーンショットを見て、「どうしてこうなったのだろう」と思ってはいけません。 *N*個の状態がある場合、それらの間に *N×(N – 1)*個の遷移が可能です。たとえば、ボタンが5つの異なる状態（通常、アクティブ、ホバー、危険、無効）のいずれかになる可能性がある場合、ボタンを更新するコードは5×4=20の遷移に対して正しいものでなければなりません。どのようにして可能性のある状態の組み合わせ的爆発を抑え、視覚的出力を予測可能にするのでしょうか。

* **Priority.** Some things are more important than others. A dialog might need to appear physically “above” the button that spawned it and “break out” of its container’s clip boundaries. A newly scheduled task (e.g. responding to a click) might be more important than a long-running task that already started (e.g. rendering next posts below the screen fold). As our app grows, parts of its code written by different people and teams compete for limited resources like processor, network, screen estate, and the bundle size budget. Sometimes you can rank the contenders on a shared scale of “importance”, like the CSS `z-index` property. [But it rarely ends well.](https://blogs.msdn.microsoft.com/oldnewthing/20050607-00/?p=35413) Every developer is biased to think _their_ code is important. And if everything is important, then nothing is! How do we get independent widgets to *cooperate* instead of fighting for resources?

* **Accessibility.** Inaccessible websites are *not* a niche problem. For example, in UK disability affects 1 in 5 people. [(Here’s a nice infographic.)](https://www.abrightclearweb.com/web-accessibility-in-the-uk/) I’ve felt this personally too. Though I’m only 26, I struggle to read websites with thin fonts and low contrast. I try to use the trackpad less often, and I dread the day I’ll have to navigate poorly implemented websites by keyboard. We need to make our apps not horrible to people with difficulties — and the good news is that there’s a lot of low-hanging fruit. It starts with education and tooling. But we also need to make it easy for product developers to do the right thing. What can we do to make accessibility a *default* rather than an afterthought?

* **Internationalization.** Our app needs to work all over the world. Not only do people speak different languages, but we also need to support right-to-left layouts with the least amount of effort from product engineers. How do we support different languages without sacrificing latency and responsiveness?

* **Delivery.** We need to get our application code to the user’s computer. What transport and format do we use? This might sound straightforward but there are many tradeoffs here. For example, native apps tend to load all code in advance at the cost of a huge app size. Web apps tend to have smaller initial payload at the cost of more latency during use. How do we choose at which point to introduce latency? How do we optimize our delivery based on the usage patterns? What kind of data would we need for an optimal solution?

* **Resilience.** You might like bugs if you’re an entomologist, but you probably don’t enjoy seeing them in your programs. However, some of your bugs will inevitably get to production. What happens then? Some bugs cause wrong but well-defined behavior. For example, maybe your code displays incorrect visual output under some condition. But what if the rendering code *crashes*? Then we can’t meaningfully continue because the visual output would be inconsistent. A crash rendering a single post shouldn’t “bring down” an entire feed or get it into a semi-broken state that causes further crashes. How do we write code in a way that isolates rendering and fetching failures and keeps the rest of the app running? What does fault tolerance mean for user interfaces?

* **Abstraction.** In a tiny app, we can hardcode a lot of special cases to account for the above problems. But apps tend to grow. We want to be able to [reuse, fork, and join](/optimized-for-change/) parts of our code, and work on it collectively. We want to define clear boundaries between the pieces familiar to different people, and avoid making often-changing logic too rigid. How do we create abstractions that hide implementation details of a particular UI part? How do we avoid re-introducing the same problems that we just solved as our app grows?

---

Of course, there are many problems I haven’t mentioned. This list is by no means exhaustive! For example, I haven’t talked about the designer and engineering collaboration, or debugging and testing. Maybe another time.

It’s tempting to read about these problems with a particular view library or a data fetching library in mind as a solution. But I encourage you to pretend that these libraries don’t exist, and read again from that perspective. How would *you* approach solving these issues? Give them a try on a tiny app! (I’d love to see your experiments on GitHub — feel free to tweet me in response.)

What’s interesting about these problems is that most of them show up at any scale. You can see them both in small widgets like a typeahead or a tooltip, and in huge apps like Twitter and Facebook.

**Think of a non-trivial UI element from an app you enjoy using, and go through this list of problems. Can you describe some of the tradeoffs chosen by its developers? Try to recreate a similar behavior from scratch!**

I learned a lot about UI engineering by experimenting with these problems in small apps without using libraries. I recommend the same to anyone who wants to gain a deeper appreciation for the tradeoffs in UI engineering.
