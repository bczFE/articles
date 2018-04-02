# Promises 不够中立

译自[PROMISES ARE NOT NEUTRAL ENOUGH](https://staltz.com/promises-are-not-neutral-enough.html)，作者[André Staltz](https://staltz.com/)。

`Promise` 的问题影响了整个 JS 生态系统。本文会讨论其中的一些问题。

看完开头你可能觉得我是在对着电脑咆哮数小时之后特发此文来喷 `Promise` 的，但你错了。其实是今早我刚泡完咖啡，[有人在推特上问](https://twitter.com/tomasz_ducin/status/963671852693499904)我如何看待 `Promise`，然后我一边喝咖啡一边回复，后来有人提议可以总结出来，才整理成此文。

`Promise` 的初衷是为了表示一个**终将得到的**值，可能在下一个事件循环，也可能在下一分钟。也有很多**原语 primitives**可以实现同样的目的：回调、C# 的 Tasks、Scala 的 Futures、RxJS 的 Observable 等等。JS 里的 `Promise` 就是使用**终值 eventual values**实现的一种**原语 primitives**。

尽管达到了目的，但 `JS Promise` 仍是一种武断的**原语 primitives**，引入了很多古怪的东西。这种古怪已经散布到了 JS 语言和生态系统的其他角落。简单来说，`Promise` 不够中立是因为以下四点：

1. 是**立即 eager**，而非懒惰的
2. 是不可取消的
3. 不是同步的
4. `then()` 是 `map()` 和 `faltMap()` 的结合体

## Promise 是**立即 eager**的，而非懒惰的

当你创建 `Promise` 的时候，它就已经开始执行了：

```js
console.log('before');
const promise = new Promise(function fn(resolve, reject) {
  console.log('hello');
  // ...
});
console.log('after');
```

在控制台可以看到顺序输出了 `before`、`hello`、`after`。传给 `Promise` 的初始化函数会立即执行。如果在调用之外定义执行函数，你能看的更清楚：

```js
function fn(resolve, reject) {
  console.log('hello');
  // ...
}

console.log('before');
const promise = new Promise(fn); // fn() 会立即调用
console.log('after');
```

`Promise` 会**立即 eager**调用它的实现。注意，在上面的代码中，我们并没有使用赋值的 `promise`。也没有 `.then()` 跟在后面。创建一个 `Promise` 时它就会立即工作。这是一个重要的细节，有两个原因：（1）我们并不是每一次都想让它立即执行，（2）有可能你想得到一个可复用的异步任务，但是 `Promise` 只会调用 `fn()` 一次，所以不可能在 `Promise` 创建之后再去复用它。

通用的正确方法是用函数把这个 `Promise` 包起来：

```js
function fn(resolve, reject) {
  console.log('hello');
  // ...
}

console.log('before');
const promiseGetter = () => new Promise(fn); // fn() 不会立即调用
console.log('after');
```

函数是懒惰的，所以上面的例子不会立即执行了。但是我们现在又不能用 `.then()` 把它们串联起来了。所以人们通常手动编写 `.then()` 给 `Promise Getter`，殊不知他们这样做是在修复 `Promise` 的可串联性和复用性。你应该看过这样的代码：

```js
// 这个函数是个 Promise Getter
function getUserAge() {
  // 下面的 fetch 也是一个 Promise Getter
  return fetch('https://my.api.lol/user/295712')
    .then(res => res.json())
    .then(user => user.age);
}
```

`Promise Getter` 是懒惰的，所以用它来构造和复用要好得多。但是如果 `Promise` 在一开始就是懒惰的话，我们就可以像这样直接构造：

```js
const getUserAge = betterFetch('https://my.api.lol/user/295712')
  .then(res => res.json())
  .then(user => user.age);
```

然后用 `getUserAge.run(cb)` 来调用。多次调用 `run`，也将多次触发**终值 eventual values**的链式执行。又能复用，还能串联。


立即执行不如懒惰，因为它限制了：不能复用**立即 eager**的**原语 primitives**。但是我们可以多次使用懒惰的**原语 primitives**，复用它的次数没有限制。

这也是**立即 eager**比懒惰更武断的原因。在 `C#` 中，`Tasks` 是懒惰的，有点像 `Promise`，但是 `C# Tasks` 有 `task.start()`，但是 `JS Promise` 没有。

用点餐打个比方，`Promise` 既是食谱（可串联的构造），也是食物（**终值 eventual values**），所以如果你吃了食物，你也会把食谱也吃了。

## 不可取消

创建 `Promise` 的时候你要想清楚：它会立即开始执行，在那之后你再也不能阻止它继续。

我相信这和 `Promise` 的立即性是有关的。[Yassine Elouafi](https://github.com/yelouafi/avenir) 举了一个好例子：


```js
var promiseA = someAsyncFn();
var promiseB = promiseA.then(/* ... */);
```


如果我们调用 `promiseB.cancel()`，我们是不是也要取消 `promiseA` 呢？在上面的例子中可能说得通，但是下面的例子呢？


```js
var promiseA = someAsyncFn();
var promiseB = promiseA.then(/* ... */);
var promiseC = promiseA.then(/* ... */);
```


如果我们调用 `promiseB.cancel()`，我们就不该取消 `promiseA` 了，因为它也被 `promiseC` 使用了。

由于 `Promise` 的立即性，向上取消变得复杂得多。Yassine 提出了计数的解决方案，但这也带来了更多的边界条件甚至 bug。

但如果 `Promise` 是懒惰的，有了 `.run()` 之后就简单得多了：

```js
var execution = promise.run();

// 过一会儿...
execution.cancel();
```

被赋值的 `execution` 将是向上的链的任务，沿着该链的每个任务都是专为这次执行创建的。因此，如果我们要执行 `executionC.cancel()`，就会调用 `executionA.cancel()`，但由于 `executionB` 具有其自己内部的 `executionA`，它将保持不变。执行多个任务 A 也很正常。如果不想多次执行 A，我们也可以为 A 构建一个特殊的“共享”方法，这样我们可以选择性的——而非总是引用计数。注意这里的“选择性”、“限制”、“总是”。如果一个行为是可选的，那它可以说是中立的。但如果一个行为总是强制执行，那它自然是武断的。

回到奇怪的食物类比，想象在餐厅点菜，但几分钟后你改变了主意想取消订单。但是你会被强迫着吃下去。只因为你点过餐。

## 不是同步的

`Promise` 的一个设计要点是尽快在事件循环的结尾 `resolve`，这样是为了促进同时创建的多个 `promise` 能尽快解决。这就意味着下面的代码：


```js
console.log('before');
Promise.resolve(42).then(x => console.log(x));
console.log('after');
```

会按顺序输出 `before`, `after`, `42`。无论你怎么创建这个 `Promise`，都不可能在两个 log 之间得到这个值。


事实告诉我们，可以把同步操作变成 `Promise`，但是不能把 `Promise` 变成同步操作。这是人为限制的，因为回调函数可以从同步转成回调，然后从回调转成同步，例如下面的 `Array ForEach`：

```js
console.log('before');
[42].forEach(x => console.log(x));
console.log('after');
```

控制台会顺序输出 `before`, `42`, `after`。

这种一旦转成 `Promise` 就不能变回同步的限制意味着，在代码中使用 `Promise` 会强制把上下的代码也都变成 `Promise` 的，这根本就说不通。异步代码会强制周围的代码也变成异步的，这我能理解，但是 `Promise` 糟糕在它把同步代码也变成了异步的。中立的**原语 primitives**不会声称数据是同步还是异步传递的。类比有损压缩，`Promise` 就是一种「有损抽象」，你可以把东西放进去，但拿出来就不是原本的东西了。

还是点餐的例子，想象你在连锁快餐店点了个汉堡，店员立马拿了个做好了的汉堡给你。但是你伸手拿的时候，店员却盯着你不松手。然后店员倒数三个数，松手了。你拿了汉堡走了，心想这地方有鬼。店员无故的想让你等待只是“以防万一”。

## `then()` 是 `map()` 和 `faltMap()` 的结合体

在 `Promise` 里调用 `then()` 的时候，返回正常值或者 `Promise` 都可以，有趣的是它们的结果还是一样的：


```js
Promise.resolve(42).then(x => x / 10);
// 一样
Promise.resolve(42).then(x => Promise.resolve(x / 10));
```

这是为了避免 `Promise` 套 `Promise`，所以内部的 then 会把返回值转成 `Promise`，然后再将其自动展开。

在你忘了某些细节的时候，这会帮助你填补空缺，但是如果在 `Promise` **内部** 有 `map`、`flatten`、`flatmap`（先 map 再 flatten）三个私有方法。而我们只能用 `then` 来做这三种事。知道这里的限制了吧？我想在已有的改变上做一些深层的操作时，却被限制了只能用 `then`。

在很久很久以前，索伦在末日山的不灭之火中锻造 `Promise` 的时候，人们曾在[这篇史诗级 Github 讨论](https://github.com/promises-aplus/promises-spec/issues/94)中建议把 `map` 和 `flatMap` 两个方法分开。但这些建议最终被归类为理论和函数式编程的幻想，并没有被实现。

这篇文章关于函数式编程只会提到一句话：不遵循数学便不可能创建中立的编程**原语 primitives**。数学在现实工程中并不是某种外星学科。数学只是定义了合理的东西，如果你希望一个东西不会自身重量压垮，你需要看看数学。

但我能轻松在讨论中给你总结几个函数式编程引起的担忧。如果 `Promise` 有 `map`、`faltMap`、`concat` 会怎样？有太多**原语 primitives**可以 `concat` 或是 `map` 了。例如 `Array` 有 `concat` 和 `map`, 也[即将有 `flatmap` 了](https://tc39.github.io/proposal-flatMap/)。如果你在使用 [`ImmutableJS`](http://facebook.github.io/immutable-js/)，就会发现里面就有很多**原语 primitives**可以使用 `map`、`flatMap`、`concat` 等等。这就很棒。

如果我能在**不考虑原语 primitives**的情况下使用 `map`、`flatMap`、`concat` 会是怎样呢？我只用考虑输入值有哪些方法。这样测试起来就方便多了，我能直接把数组当作测试数据使用。在生产中使用了 `ImmutableJS` 和异步任务的代码将会和测试中使用普通数组一样。函数式编程者会说 `范型 generics`、`用特定类型编程 programming with typeclasses`、`单子 monads` 等等，但是这意味着我们可以给这些**原语 primitives**以通用的命名。如果一个**原语 primitives**有 `concat` 方法，而另一个使用了 `concatenate`，它们做了同样的事，但是API和语义略有不同，这就没意思了。这就解释了为啥由于 `Promise` 是可以被串联的，它就应该拥有 `concat` 方法；为啥由于 `Promise` 是可以被映射的，它就应该拥有 `map` 方法；为啥由于 `Promise` 是可以被链接的，它就应该拥有 `faltMap` 方法。但不幸的是，事实并非如此，`Promise` 最终使用一些自动转换逻辑合并了 `map` 和 `flatMap`，仅仅是因为 `map` 和 `flatMap` 看上去太过相似，被认为没有必要单独拥有这两个方法。

## 总结

`Promise` 还可以用来做事情，岁月静好。不需要恐慌。它就是有点奇怪，还有点不幸的自以为是。它强制执行了某些说不通的行为。没关系我们可以解决的。它也难以被复用，没关系我们可以解决的。它也不能被取消，没关系就是让逻辑继续执行有点浪费。我们总是要解决它，这有点烦。现在有些 API、语法糖例如 async/await 也是基于 `Promise` 的，这也有点烦。所以我们必须在未来的几年里妥协这些怪异。然而，如果当时在设计时考虑到懒惰的细节，这一些都会不一样。

这儿有两个例子，如果当时用数学思维设计 `Promise` 将会是怎样：[fun-task](https://github.com/rpominov/fun-task) 和 [avenir](https://github.com/yelouafi/avenir)。这俩都是懒惰的，所以有些相似。不同点（可能？）在于命名和方法的可用性上。但是这俩都比 `Promise` 要中立的多，因为他们都：（1）是懒惰的、（2）允许同步的、（3）可以取消的。其中 `fun-task` 还分离了 `map` 和 `flatMap`。

`Promise` 是被发明的，不是被发现的。最好的**原语 primitives**是被发现的，因为它们通常是中性的，我们不能反驳它们。例如圆这样一个简单的数学概念，这就是为什么人们发现圆而不是发明圆。你的意见一般不能与某个圆“相悖”，因为它没有自带任何意见，而且它常常发生在自然界和系统中。

