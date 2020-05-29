# 学习async/await内部的执行

### 为什么要引入async/await

<br />在[上一篇](./熟悉Promise.md) 中，我们从微任务的角度讲解了使用Promise解决异步回调的问题，并手动实现了一个简单的Promise，我们先回顾下Promise的三个特性：<br />

- 回调函数延迟绑定
- 错误冒泡机制
- 链式调用


<br />虽然promise能很好的解决回调地狱的问题，但是必须使用then方法，如果流程特别复杂的话，那么代码就充满整个then，给人的直观上不好，阅读起来也麻烦。<br />
<br />
<br />比如使用下面的场景：我们请求shushi.pro的内容，等返回信息后再请求另外一个资源，下面使用了fetch来请求，返回的也是Promise，只不过浏览器原生支持。<br />

```javascript
fetch("https://shushi.pro")
  .then((res) => {
    console.log(res);
    return fetch("https://shushi.pro/test");
  })
  .then((res) => {
    console.log(res);
  })
  .catch((err) => {
    console.log(err);
  });
```

<br />从上述代码中，我们可以看到，频繁使用promise会相当负责，虽然代码看起来比较线性化，但是包含了太多的then函数，使得阅读起来不方便。那么有没有更好的解决方案的，答案 `是有的` ，ES7引入了 `async+await。`<br />
<br />
<br />[根据MDN介绍](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/async_function)：`**async function**` 用来定义一个返回 [`AsyncFunction`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/AsyncFunction) 对象的异步函数。异步函数是指通过事件循环异步执行的函数，它会执行异步操作然后隐式返回 [`Promise`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise) 结果。如果你在代码中使用了异步函数，就会发现它的语法和结构会更像是标准的同步函数。<br />

```javascript
async function asyncCall() {
  console.log('calling');
  const result = await fetch("https://shushi.pro");
  console.log(result);
}

asyncCall();
```

<br />通过上面的代码，我们可以发现，处理异步的逻辑都在同步代码中实现，非常符合人的线性思维。<br />
<br />

<a name="54fbdf90"></a>
### 生成器VS协程

<br />那么async/await是什么用同步的方式来实现异步操作呢，在了解工作原理之前，我们要先知道async/await底层是Generator跟Promise一起来实现的，我们先介绍下生成器（Generator），然后在讲解Generator底层机制实现--协程（Coroutine），通过Generator + Promise来分析async/await是怎么实现异步代码的。<br />
<br />

<a name="1a34b8d9"></a>
#### 生成器

<br />

<a name="10dcdc5b"></a>
##### 生成器函数是一个带 * 标记的函数，可以「暂停/恢复」执行的


```javascript
function* genDemo() {
  console.log('开始执行第一次')
  yield 'gen start1'

  console.log('开始执行第二次')
  yield 'gen start2'

  console.log('开始执行第三次')
  yield 'gen start3'
  
  console.log('执行完成')
  return 'success'
}
console.log('start 0')

const gen = genDemo()
console.log(gen.next().value)
console.log('start 1')
console.log(gen.next().value)
console.log('start 2')
console.log(gen.next().value)
console.log('start 3')
console.log(gen.next().value)
console.log('start 4')
```

<br />通过上面的代码，我们可以发现，genDemo函数不是一次执行完成后的，是通过 `next` 方法执行的，如果不执行next方法，genDemo内部代码就不会执行，这就是 `生成器` ，它可以暂停执行，也可以恢复执行，下面我们来看下生成器函数的具体使用方式：<br />

- 在16行代码中，我们把genDemo赋值给gen，但是这个时候genDemo内部代码是没有执行的，这说明genDemo内部代码被暂停执行
- 在17行，我们调用了 `next` 方法，它会让genDemo开始内部代码开始执行，会输出「开始执行第一次」，之后会遇到 `yield` 关键字，这个时候，**js引擎将返回关键字后面的内容给外部，并且暂停genDemo的执行**
- 这个时候代码会执行在第18行，输出「start 1」，然后再执行19行代码，调用 `next` 方法，恢复genDemo函数执行，以此类推，直到执行完成


<br />上面我们多次提到了函数的「暂停」和「恢复」，相信你一定会很好奇其中原理，现在我们一起来学习下，js引擎是如何实现一个函数的暂停跟恢复的。<br />
<br />

<a name="ebe98654"></a>
#### 协程

<br />在了解函数的暂停跟恢复之前，我们先了解** [协程](https://www.liaoxuefeng.com/wiki/897692888725344/923057403198272) **的概念:<br />协程，也称为**微线程，**它比线程更加的轻量级，一个线程上可以存在多个协程，但是在js中，协程只能同时执行一个，无法多个一起执行，比如当前有A跟B两个协程，最开始执行了A协程，如果想执行B协程，那么A协程就要把主线程的控制权交给B协程，这个时候A协程会暂停执行，B协程开始执行。反之，B协程也可以把控制权交给A线程。<br />
<br />
<br />那么为什么要有协程呢？<br />因为协程不被操作系统内核管理，完全是由程序所控制的，这样切换协程的时候，不会像切换线程那么消耗资源，在性能上有很大的提升。<br />
<br />
<br />我们通过上面的代码，来看看协程是怎么工作。<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/512535/1589354116857-86e7e36f-3f6d-419d-bbf3-5f1b900eae6f.png#align=left&display=inline&height=161&margin=%5Bobject%20Object%5D&name=image.png&originHeight=704&originWidth=3260&size=237075&status=done&style=none&width=746#align=left&display=inline&height=704&margin=%5Bobject%20Object%5D&originHeight=704&originWidth=3260&status=done&style=none&width=3260)<br />从图中可以看到：<br />

- 通过调用genDemo函数来生成一个协程gen，当执行genDemo()的时候，函数内部没有被执行
- 通过gen.next来执行协程
- 协程进行的时候，遇到yield关键字来暂停gen协程，并返回主要信息给父协程
- 在协程执行期间，如果遇到了return，那么就会结束当前协程，并将return出来的值返回给父协程


<br />

<a name="b93300f9"></a>
#### Generator+Promise

<br />相信到了这里，你已经弄清楚了协程是什么工作的，如果还是不懂，那你自裁把。现在我们用promise来改造一下上面的代码，代码如下：<br />

```javascript
function* genDemo() {
  let res1 = yield fetch('https:/shushi.pro')
  console.log('res1')
  console.log(res1)

  let res2 = yield fetch('https:/shushi.pro')
  console.log('res2')
  console.log(res2)
}

function mGenerator(gen) {
  return gen.next().value
}

const gen = genDemo()

mGenerator(gen).then(res => {
  console.log(res)
  return mGenerator(gen)
}).then(res => {
  console.log(res)
})
```

<br />可以看到genDemo就是个生成器，内部用了同步代码的形式来实现异步操作，我们来分析下上面代码的执行流程：<br />

- 首先执行的是const gen = genDemo()，创建来一个gen协程
- 然后调用mGenerator函数，把gen协程传入进去，内部会调用gen的next方法。
- gen协程获取控制权，执genDemo内部代码，调用fetch函数，创建一个promise对象res1，然后通过yield暂停gen协程，将res1返回给父协程
- 父协程恢复执行后，调用res1.then方法等待异步请求结果
- then方法获取结果后，执行内部回调函数，打印console.log(res)，再执行mGenerator(gen)
- 这个时候又执行了next方法，以此类推


<br />我们通过promis+generator来实现异步请求，增强了代码的可读性，更符合了人的线性直观<br />
<br />

<a name="async"></a>
#### async

<br />上面说过，async是通过执行一个**异步操作**，然后**隐式返回**Promise作为结果的函数，我们先在控制台中执行下面代码，看看返回结果是什么.<br />

```javascript
async function test(){
    return 'test'
}
console.log(test()) // Promise {<resolved>: "test"}
```

<br />我们可以看到，返回的结果是 「Promise {: "test"}」，这证明了async函数返回的是个Promise<br />
<br />

<a name="await"></a>
#### await

<br />我们知道async函数返回的是个Promise，那么我们在函数中使用await，再看看具体的效果，代码如下：<br />

```javascript
async function test(){
    console.log(1)

    let t = await 'tqh';
    console.log(t)

    console.log(2)
}
console.log(0)
test()
console.log(3)
```

<br />按照人的线性思维，这段代码打印的循序应该是 0 -> 1 -> tqh -> 2 -> 3， 但是现实往往是那么的残忍，我们在控制台执行代码，可以看到打印结果是0 -> 1 -> 3 -> tqh -> 2，是不是很疑惑，不是说好了同步执行的嘛，怎么test函数还没执行完，就执行了console.log(3)，让我们带着这个疑惑来分析这段代码内部具体做了什么：<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/512535/1589364014789-841231af-246a-4884-8488-3bed22a93012.png#align=left&display=inline&height=438&margin=%5Bobject%20Object%5D&name=image.png&originHeight=876&originWidth=2940&size=167041&status=done&style=none&width=1470#align=left&display=inline&height=876&margin=%5Bobject%20Object%5D&originHeight=876&originWidth=2940&status=done&style=none&width=2940)<br />我们通过上图一起来分析下执行流程。<br />

- 首先是打印出 0 ，然后执行test函数
- 执行test的时候，由于test被async标记过，所以js引擎会保存当前调用栈的信息，接着执行console.log(1)
- 接下来遇到了await关键字
  - 首先会默认创建一个promise对象
    - let promise_ = new Promise((resolve,reject){


<br />resolve(100)<br />})<br />

- 当执行resolve的时候，js引擎会将该任务提交到微任务队列
- 暂停当前协程，将控制权给父协程，同时将promise_对象返回给父协程
- 父协程获取权限后就调用then方法，由于then方法是在微任务里面，所以会先执行同步任务console.log(3)
- 等父协程同步任务执行完成后，再去微任务队列里面执行任务
- 执行完成后把控制权交给test协程，并且把value值传递进去
- test协程执行console.log（t），console.log(2)
- 执行完成后，结束test进程，把控制权给父协程


<br />这就是async/await的执行流程。
