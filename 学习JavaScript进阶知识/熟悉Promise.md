# 熟悉Promise

# 学习Promise

> 在学习promise之前，要先知道为什么会存在promise，它的好处是什么，解决了什么问题。让我门带着这些疑问一步一步的深入promise

<a name="fb6d78f1"></a>
### 异步编程

<br />先看下面一个简单的demo<br />

```javascript
function onResolve(response){console.log(response) }
function onReject(error){console.log(error) }
 
let xhr = new XMLHttpRequest()
xhr.ontimeout = function(e) { onReject(e)}
xhr.onerror = function(e) { onReject(e) }
xhr.onreadystatechange = function () { onResolve(xhr.response) }

let URL = 'https://shushi.pro/'
xhr.open('Get', URL, true);
 
xhr.timeout = 3000
xhr.responseType = "text"
xhr.setRequestHeader("X_TEST","time.geekbang")
xhr.send();
```

<br />当我们执行上述代码的时候，可以看到输出结果。但是，这么点代码就执行了5次回调，这些回调导致代码不连贯，不符合人的直觉，这就是异步回调影响到了我们的编程方式。<br />
<br />

<a name="d30cb285"></a>
### 封装异步回调

<br />当我们书写一个异步代码的时候，我们关注的是「**输入**」跟「**输出**」，至于中间的一些请求信息，我们不想在代码里面体现太多，这会干扰代码的逻辑，让人阅读起来也非常不方便，所以我们将上面的代码进行封装。<br />

```javascript
function fetchRequest(url) {
    let request = {
        method: 'Get',
        url: url,
        headers: '',
        body: '',
        credentials: false,
        sync: true,
        responseType: 'text',
        referrer: ''
    }
    return request
}
```

<br />我们把输入的 HTTP 请求信息全部保存到一个 request 的结构中，包括请求地址、请求头、请求方式、引用地址、同步请求还是异步请求、安全设置等信息。request 结构如上所示。<br />
<br />然后我们在封装请求过程，我们将所有的请求封装在Fetch函数中。<br />

```javascript
function fetch(request, resolve, reject) {
    let xhr = new XMLHttpRequest()
    xhr.ontimeout = function (e) { reject(e) }
    xhr.onerror = function (e) { reject(e) }
    xhr.onreadystatechange = function () {
        if (xhr.status = 200)
            resolve(xhr.response)
    }
    xhr.open(request.method, URL, request.sync);
    xhr.timeout = request.timeout;
    xhr.responseType = request.responseType;
    // 其他请求信息
    //...
    xhr.send();
}
```

<br />这个时候，fetch需要接收三个参数：request「请求信息」、resolve「请求成功的回调」、reject「请求失败的回调」。有了这些，我们来开始使用这个方法。<br />

```javascript
fetch(fetchRequest('https://shushi.pro/'),
    function resolve(data) {
        console.log(data)
    }, function reject(e) {
        console.log(e)
    })
```

<br />上面代码正常执行，我们把请求进行的简单的封装，也比较符合人的线性思维。<br />
<br />

<a name="711eb30f"></a>
### 新的问题

<br />上面代码看起来似乎正常，但是当业务复杂的时候，就会出现很多回调，一层嵌一层，让代码变成了**回调地狱**<br />

```javascript
fetch(fetchRequest('https://shushi.pro/?category'),
      function resolve(response) {
          console.log(response)
          fetch(fetchRequest('https://shushi.pro/column'),
              function resolve(response) {
                  console.log(response)
                  fetch(fetchRequest('https://shushi.pro')
                      function resolve(response) {
                          console.log(response)
                      }, function reject(e) {
                          console.log(e)
                      })
              }, function reject(e) {
                  console.log(e)
              })
      }, function reject(e) {
          console.log(e)
})
```

<br />这段代码看起来很乱，跟一坨💩一样，为什么出现这种原因呢。<br />

- 嵌套调用，等上一个执行完才能执行下一个。
- 请求的不确定性，每个都要做错误处理


<br />让我们知道这些原因后，就要解决这些问题。<br />

- 消除嵌套使用
- 将错误合并



```javascript
function fetch(request) {
  function executor(resolve, reject) {
      let xhr = new XMLHttpRequest()
      xhr.open('GET', request.url, true)
      xhr.ontimeout = function (e) { reject(e) }
      xhr.onerror = function (e) { reject(e) }
      xhr.onreadystatechange = function () {
          if (this.readyState === 4) {
              if (this.status === 200) {
                  resolve(this.responseText, this)
              } else {
                  let error = {
                      code: this.status,
                      response: this.response
                  }
                  reject(error, this)
              }
          }
      }
      xhr.send()
  }
  return new Promise(executor)
}
```

<br />然后，我们在用fetch在发送请求，代码如下：<br />

```javascript
var x1 = fetch(fetcheRequest('https://shishi.pro/?category'))
var x2 = x1.then(value => {
    console.log(value)
    return fetch(fetcheRequest('https://shishi.pro/column'))
})
var x3 = x2.then(value => {
    console.log(value)
    return fetch(fetcheRequest('https://shishi.pro'))
})
x3.catch(error => {
    console.log(error)
})
```

<br />上述代码执行的时候，我们引入了promise「先不管promise的实现，后面会讲」，这样一来，代码就变的很清晰了，是不是很棒。<br />
<br />
<br />细心的同学会发现，我们消除了**嵌套**，并且将**错误合并**了<br />
<br />

<a name="11c6e3c8"></a>
### 抛出疑问

<br />现在让我们带着疑问一起去了解promise<br />

- **promise是怎么消除嵌套的**
- **怎么实现回调函数返回值穿透的「链式调用」**
- **怎么合并错误处理的**


<br />**解决这三个问题之前，我们先了解下Promise为什么要引入微任务，**如果对微任务不了解的同学先网上查询资料，这里不做过多的讲解。<br />
<br />
<br />promise内部的执行函数是同步的，里面可以写异步操作，异步成功后执使用resolve方法，失败则调用reject方法，「resolve」「reject」这两个方法都会作为微任务放入到事件循环中，那么promise为什么要引入微任务呢？？<br />

> 微任务是在宏任务执行的时候产生的，宏任务包括「页面渲染」、「js脚本执行」、「网络请求」，而js中的宏任务包括了setTimeout、setInterval、IO、setImmediate


<br />微任务是执行宏任务的时候产生的，举个例子：把宏任务当作**开发时**的项目评审，每个宏任务就是评审时的需求，那么微任务就是**开发中**遇到的新需求<br />
<br />
<br />假设Promise是宏任务，大家都知道页面执行的时候，任务会被放入到**消息队列**中，通过**事件循环**的执行来执行，而**消息队列中的任务是先进先出**的，如果promise是宏任务，那么执行promise的时候会当消息队列的尾部，等到最后才执行，如果队列特别的长呢？？那么回调就要等很久才执行。<br />
<br />
<br />而微任务是执行宏任务的时候产生的，微任务的执行顺序是  `主函数执行结束后，宏任务执行结束前` 执行的，所以Promise采取了微任务的方案<br />
<br />

<a name="90c93933"></a>
### 实现Promise

<br />在开始实现promise之前，我们需要了解一下promise的状态<br />

- PENDING(等待)
- FULFILLED(成功)
- REJECTED(失败)


<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/512535/1589288837375-e95c7093-1710-4d4b-9ee5-73d63cfdb75d.png#align=left&display=inline&height=280&margin=%5Bobject%20Object%5D&name=image.png&originHeight=560&originWidth=1242&size=35880&status=done&style=none&width=621#align=left&display=inline&height=560&margin=%5Bobject%20Object%5D&originHeight=560&originWidth=1242&status=done&style=none&width=1242)<br />初始化的状态是PENDING，成功的状态是FULFILLED，失败的状态是REJECTED，成功后可以一只调用.then方法<br />
<br />开始写一个简单的promise<br />

```javascript
const PENDING = "pending";
const FULFILLED = "fulfilled";
const REJECTED = "rejected";

function MyPromise(executor) {
  this.value = null;
  this.error = null;
  this.status = PENDING;
  this.onFulfilledCallbacks = [];
  this.onRejectedCallbacks = [];

  const resolve = (value) => {
    if (this.status !== PENDING) return;

    setTimeout(() => {
      this.status = FULFILLED;
      this.value = value;
      this.onFulfilledCallbacks.forEach((callback) => {
        callback(this.value);
      });
    });
  };

  const reject = (error) => {
    if (this.status !== PENDING) return;

    setTimeout(() => {
      this.status = REJECTED;
      this.error = error;
      this.onRejectedCallbacks.forEach((callback) => {
        callback(this.error);
      });
    });
  };

  executor(resolve, reject);
}

MyPromise.prototype.then = function (onFulfilled, onRejected) {
  onFulfilled = typeof onFulfilled === "function" ? onFulfilled : value => value;
  onRejected = typeof onRejected === "function" ? onRejected : error => { throw error };
  
  if (this.status === PENDING) {
    this.onFulfilledCallbacks.push(onFulfilled);
    this.onRejectedCallbacks.push(onRejected);
  } else if (this.status === FULFILLED) {
    onFulfilled(this.value);
  } else {
    onRejected(this.error);
  }

  return this;
};
```

<br />在上述代码中，我们完成了简化版的promise的书写，我们现在来测试一下<br />

```javascript
function testPromise() {
  return new MyPromise((resolve, rejected) => {
    const random = Math.random();
    setTimeout(() => {
      if (random > 0.5) {
        resolve("成功");
      } else {
        rejected("失败");
      }
    }, 1000);
  });
}

testPromise()
  .then((res) => {
    console.log(res);
    return testPromise();
  })
  .then((res) => {
    console.log(res);
  });
```

<br />可以看到，两次打印了一样的结果，而且中途没有1s的延迟，这是为什么呢？？大家可以先自行思考下。<br />
<br />
<br />
<br />
<br />
<br />
<br />
<br />
<br />
<br />
<br />
<br />
<br />现在我们来解开这个问题，大家可以看到，我们调用了两次then方法，而then方法执行的时候，比<br />setTimeout(() => {<br />     if (random > 0.5) {<br />       resolve("成功");<br />     } else {<br />       rejected("失败");<br />     }<br />   }, 1000)  执行要快，所以MyPromise的状态为PENDING，两个then方法的回调函数都会被存放在onFulfilledCallbacks/onRejectedCallbacks中，1s后执行resolve/rejetc的时候，都会执行then里面对应的队列<br />
<br />要解决这个问题，需要关注的点就是当状态为PENDING的时候，then里面的return<br />

```javascript
MyPromise.prototype.then = function (onFulfilled, onRejected) {
	//....
  return this;
};
```

<br />每次执行then的时候，return出来的永远是第一个MyPromise，所以执行第二个then的时候，拿到的this还是原来的MyPromise，下面我们对then进行优化。<br />

```javascript
function resolvePromise(bridgePromise, _promise, resolve, reject) {
  if (_promise instanceof MyPromise) {
    if (_promise.status === PENDING) {
      _promise.then(
        (res) => {
          resolvePromise(bridgePromise, res, resolve, reject);
        },
        (error) => {
          reject(error);
        }
      );
    } else {
      _promise.then(resolve, reject);
    }
  } else {
    resolve(_promise);
  }
}

MyPromise.prototype.then = function (onFulfilled, onRejected) {
  onFulfilled =
    typeof onFulfilled === "function" ? onFulfilled : (value) => value;
  onRejected =
    typeof onRejected === "function"
      ? onRejected
      : (error) => {
          throw error;
        };
  let bridgePromise;

  // 返回一个promise函数
  bridgePromise = new MyPromise((resolve, reject) => {
    if (this.status === FULFILLED) {
      setTimeout(() => {
        try {
          // 把then中成功或者失败后函数执行的结果获取到
          let x = onFulfilled(this.value);
          // 看一看是不是promise 如果是promise就让promise执行,取到最终这个promise的执行结果，让返回的promise成功或者失败
          resolvePromise(bridgePromise, x, resolve, reject);
        } catch (e) {
          reject(e);
        }
      }, 0);
    }
    if (this.status === REJECTED) {
      setTimeout(() => {
        try {
          let x = onRejected(this.reason);
          resolvePromise(bridgePromise, x, resolve, reject);
        } catch (e) {
          reject(e);
        }
      }, 0);
    }

    // 如果是等待态则将回调函数存放到响应的数组中
    if (this.status === PENDING) {
      this.onFulfilledCallbacks.push(() => {
        setTimeout(() => {
          try {
            // 把then中成功或者失败后函数执行的结果获取到
            let x = onFulfilled(this.value);
            // 看一看是不是promise 如果是promise就让promise执行,取到最终这个promise的执行结果，让返回的promise成功或者失败
            resolvePromise(bridgePromise, x, resolve, reject);
          } catch (e) {
            reject(e);
          }
        }, 0);
      });
      this.onRejectedCallbacks.push(() => {
        setTimeout(() => {
          try {
            let x = onRejected(this.reason);
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            reject(e);
          }
        }, 0);
      });
    }
  });
  return bridgePromise;
};
```

<br />这样就可以成功的执行，目前我们实现了promise的**链式调用**，不知道大家有么有发现，resolve跟reject里面用到了**定时器**，这是为什么呢？大家可以把定时器删除再试试，看看会发生什么事情<br />
<br />
<br />这是因为如果一开始就直接绑定，有可能会导致调用队列里面的函数的时候，then方法还没执行，从而报错，这就是**promise的延迟绑定**<br />
<br />
<br />最后我们再来实现catch方法<br />

```javascript
MyPromise.prototype.catch = function (onRejected) {
  return this.then(null, onRejected);
}
```

<br />是不是特别的简单，如果有错误，直接执行reject，通过事件冒泡的机制<br />
<br />
<br />promise三大特性：<br />

- 回调函数延迟绑定
- 错误事件冒泡
- 回调函数返回值穿透「链式调用」
