---
author: Mh
pubDatetime: 2020-08-08T11:26:34Z
title: 手实现 Promise A+（一）：实现 then 方法
postSlug: manual-implementation-promise-1
featured: false
draft: false
tags:
    - JavaScript
    - 前端
ogImage: ""
description: "Promise 原理和实现"
---

`promise` 如今已经深度融入前端开发技术当中，很多模块内部都依赖 `promise`，使用 promise 可以很好的解决异步回调问题。  

`promise` 内部有三种状态：预备态（`pending`）、成功态（`fulfilled`）和失败态（`rejected`）。初始状态是 `pending`，当操作完成时会变成成功态，如果操作出现异常而终止时它会变成失败状态（或拒绝态）。一旦 `promise` 被履行或者被拒绝，它就不可逆转，即不能从成功态再变回其他状态，状态之间不能再相互转换。此时的 promise 可以认为是完成状态（`settled`）。  

`then` 函数是 `promise` 中最常用的一个方法，它也是 promise 的核心 API。例如：  

```js
let p = new Promise((resolve, reject) => {
    resolve(123);
});
p.then(d => {
    console.log(d);
}, e => {
    console.log(e);
});
```
当执行上面代码后，将会履行成功态，执行 `then` 方法的第一个回调函数，打印出 `123`。 
`Promise` 构造函数接收一个 `exector` 函数，当构造实例时，这个函数会 **立即执行**。下面一个初级版的 `promise` 实现。

```js
const PENDING = 1;	// 等待状态
const FULFILLED = 2;	// 成功态
const REJECTED = 3;		// 失败态

class Promise{
    constructor (exector) {
        // 初始状态是 等待态
        this.status = PENDING;
        // 成功时的值
        this.value = undefined;
        // 失败时的值
        this.reason = undefined;
        const resolve = (value)  =>{
            // 用户在调用 resolve 函数后，就要改变状态
            // 如果状态是初始态，才改变，因为状态之间不可逆转
            // resolve 函数不能调用多次，reject 函数也是一样
            if (this.status === PENDING) {
                this.status = FULFILLED;
                this.value = value;
            }
        }
        const reject = (reason) => {
            if (this.status === PENDING) {
                this.status = REJECTED;
                this.reason = reason;
            }
        }

        try {
            // 立即调用，调用 exector 可能会出错
            exector(resolve, reject);
        } catch (error) {
            // 把异常传给 reject 失败函数
            reject(error);
        }
    }
	// then 方法接收两个回调函数，一个是成功的回调，一个是失败的回调
    then (onFulfilled, onRejected) {
        // 当状态转变时，就调用传入的状态函数
        if (this.status === FULFILLED) {
            onFulfilled(this.value);
        }
        if (this.status === REJECTED) {
            onRejected(this.reason);
        }
    }
}
```
上面的实现比较简单，我们知道，实际当中的 `then` 函数非常强大，可以链式调用，当前的返回值可以作为下一个 `then` 的接收值，而且 promsie 主要是为了解决异步回调问题。比如下面的例子：  

```js
var fn = function (num) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            // 大约1秒后把参数传给成功函数
            resolve(num);
        },1000);
    });
}
fn(1000).then(data => {
    return data + 111;  // 返回 1111
}).then(data => {       // data 是上一次 then 的返回值
    console.log('data === ',data);  // 1111
}, e => {
    console.log('error === ',e);
});
```
显然，我们刚刚实现的 promise 不能链式调用，也不能正确处理异步回调，我们的 `then` 方法是同步的代码，用户在异步的情况下使用 `resolve` 或者 `reject` 函数，在调用 `then` 方法时，`resolve` 或 `reject` 函数还没有执行（或者说定时器等异步函数还没有执行），`status` 就还没有改变，但 `then` 方法已经执行完了（状态并没有命中），这就导致没有接收到结果。 

## 处理异步代码
通过上面分析，了解到，调用 `then` 时，`this.status` 可能还是 `PENDING` 状态，说明是异步函数在使用 promise。为了解决这个问题，我们可以使用两个数组，分别用来存储失败和成功的回调，当 `status` 变化时，`value` 或 `reason` 也会跟着变化（这两个变量都在 resolve 或 reject 函数中改变），这时就遍历数组，执行函数。代码如下：  
```js
class Promise{
    constructor (exector) {
        this.status = PENDING;
        this.value = undefined;
        this.reason = undefined;
        // 成功的回调数组
        this.onResolvedCbs = [];
        // 失败的回调数组
        this.onRejectedCbs = [];
        // 失败时的回调
        var reject = (reason) => {
            if(this.status === PENDING){
                this.status = REJECTED; // 状态变化了
                this.reason = reason;   // 值变化了
                // 执行 then 中的回调，这时回调函数的参数中就能拿到值了
                this.onRejectedCbs.forEach(fn => fn());
            }
        };
        var resolve = (value) => {
            if(this.status === PENDING){
                this.status = FULFILLED;    // 状态变化了
                this.value = value;         // 值变化了
                // 执行 then 中的回调，这时回调函数的参数中就能拿到值了
                this.onResolvedCbs.forEach(fn => fn());
            }
        };
        try{
            exector(resolve, reject);
        }catch(e){
            reject(e);
        }
    }
    then (onFulfilled, onRejected) {
        if(this.status === FULFILLED){
            onFulfilled(this.value);
        }
        if(this.status === REJECTED){
            onRejected(this.value);
        }
        if(this.status === PENDING){
            // 异步，就先不执行函数，先把函数存储到数组中
            // 当 status 变化后，再遍历数组，执行函数
            // 在定时器里调用了 resolve 或者 reject
            this.onResolvedCbs.push(() => {
                onFulfilled(this.value);
            });
            this.onRejectedCbs.push(() => {
                onRejected(this.reason);
            });
        }
    }
}
```
## 链式调用
链式调用可以说是 Promise 的精华，有了链式调用，你的代码中基本就不会再有回调地狱了！  

在 Promise 中，上一次 `then` 方法返回的值会作为下一次 `then` 方法的回调中的参数。`then` 方法返回的值有三种：  

1. 在成功回调或者失败回调中抛出异常，会走到下一次 `then` 的失败回调里；
2. 成功或失败返回的是还是一个 `promise`，那么会用这个 （返回的）promise 的状态作为结果；
3. 普通值。即除了上面两种之外的值，promise 会把这个值传递给外层的下一个 `then` 方法中。  

下面代码是第一种情况的演示：  

```js
// 在成功回调或者失败回调中抛出异常，会走到下一次 `then` 的失败回调里；
var p = function(num){
    return new Promise(resolve => {
        // 这里如果是调用 reject
        // 会触发第一个 then 中的失败回调
        // 如果失败回调中抛出异常，错误会传递到第二个 then 方法的失败回调中
        resolve(num);
    });
}
p(4).then(n => {
    throw Error('出错了！' + n);
},e => {
    console.log('e: ', e);
}).then((d) => {
    console.log('d == ', d);
}, err => {     // 错误会走到这个失败回调中
    console.log('err == ', err);
});
```
下面的代码是情况2 的例子，在 then 方法中返回 promise 实例。  

```js
// 成功或失败返回的是还是一个 `promise`，那么会用这个 （返回的）promise 的状态作为结果；
var p = function(num){
    return new Promise((resolve, reject) => {
        resolve(num);
    });
}
p(4).then(n => {
    // then 内部又返回了 promise
    return new Promise((resolve, reject) => {
        resolve(n + 1);     // 调用成功的函数
        // 如果这里调用的 reject 函数，失败的状态会传递到下面 then 方法的失败回调中
    })
}).then((data) => {
    // 在这个 then 中可以接收到 resolve(n + 1) 的数据
    console.log('data === ', data);
},err => console.log('err: ', err));
```

甚至可以这样使用 promise：

```js
p(4).then(n => {    // n === 4
    return new Promise((resolve, reject) => {
        resolve(n + 1);
    }).then(d => {      // d === 5
        return new Promise((resolve, reject) => {
            resolve(d + 2);
        }).then(d => {      // d === 7
            return new Promise((resolve, reject) => {
                resolve(d + 3);
            });
        }).then(data1 => {      // data1 === 10
            // 这个 then 离 resolve(d + 3) 最近，因此会接收数据
            console.log('data1 === ', data1);   // 10
        });
    }).then(data2 => {      // undefined
        console.log('data2 === ', data2);
    })
}).then((data) => {     // undefined
    console.log('data === ', data);
}, err => {
    console.log('err: ', err);
});
```
仔细观察上面代码，可能你对 then 方法的执行顺序会有困惑。上面我们已经说过，`new Promise` 传入的 `exector` 函数会被立即执行，上面代码中，最上面的 `then` 在执行时，会立即执行 `new Promise`，exector 执行完后，最外层的 `then` 方法还没有执行完，因为内部又调用了 `then` 方法，直到执行到 `resolve(d + 3)`处，此时 `then(d => xxx)` 的代码才执行完，执行完后返回 promsie 实例，又调用了 `then` 方法（then(data1 => xxx)）,接收到 `resolve(d + 3)` 传来的值，打印出结果，这个 `then` 方法之所以能接收到值，是因为它满足了情况2，返回的 promise 是 resolve 态，然后就接收到了。`data1`这个 `then` 方法不再返回值，因此它下面的 `then` 方法打印的结果都是 `undefined`。

上面代码看起来实在是有些丑陋，在实际工作当中估计也不会这么写，但这能反映出 `then` 方法的特点，这也是 `then` 方法的强大之处。错误处理也是这样。**如果在失败回调中返回的是一个普通值，该值会传入下一个 `then` 方法的成功回调里**。例如：  

```js
var p = function(num){
    return new Promise((resolve, reject) => {
        reject(num);    // 调用失败函数，num 是一个普通值
    });
}
p(4).then(n => {
    
}, e => {
    return e;   // 返回普通值
}).then(d => {  // d === 4
    console.log('d === ', d);
}, err => {
    console.log('err === ', err);
});
```

`then` 方法的实现原理：通过返回一个新的 promise 实例来实现链式调用（而不是返回当前的实例）。不能返回当前实例，因为一旦是状态转变，就不可逆转。比如上面的代码，失败回调中返回普通值，结果值会传递到下一个成功回调里，而不是失败回调里，如果 then 方法返回的是当前实例，状态已经是失败态，状态不可逆转，也就不能把失败态变成成功态。当返回的是新的 promsie 实例时，把普通值传入 `resolve` 中就可以做到状态转变。  

### 重写 then 方法
代码如下：  

```js
then (onFulfilled, onRejected) {
    // 两个参数必须是回调函数，不是函数时替换成函数
    // 这样可以实现这种效果：promise.then().then().then(d => console.log(d));
    // 即使中间的 then 函数没有传参，后面的 then 函数也可以获取到值，这被称为“值穿透”
    onFulfilled = typeof onFulfilled === 'function' ?
            onFulfilled : v => v;   // 默认是把参数（this.value）直接返回
        onRejected = typeof onRejected === 'function' ? 
            onRejected : err => { throw err };
     // 创建新的 promise 实例
    let promise2 = new Promise((resolve, reject) => {
        /**
         * 根据 onFulfilled 回调执行后的返回值来判断是调用 resolve 还是 reject
         * onFulfilled 回调可能返回 promise，可能抛出异常，也可能返回的是一个普通值
         */
        if(this.status === FULFILLED){
            setTimeout(() => {
                try{        // 加 try-catch 是因为 onFulfilled 回调可能抛出异常
                    let x = onFulfilled(this.value);
                    resolvePromise(x, promise2, resolve, reject);
                }catch(e){
                    reject(e);
                }
            },0);
        }
        if(this.status === REJECTED){
            setTimeout(() => {
                try{
                    let x = onRejected(this.reason);
                    resolvePromise(x, promise2, resolve, reject);
                }catch(e){
                    reject(e);
                }
            },0);
        }
        if(this.status === PENDING){
            // 异步，就先不执行函数，先把函数存储到数组中
            // 当 status 变化后，在遍历数组，执行函数
            this.onResolvedCbs.push(() => {
                setTimeout(() => {
                    try{        // 加 try-catch 是因为 onFulfilled 回调可能抛出异常
                        let x = onFulfilled(this.value);
                        resolvePromise(x, promise2, resolve, reject);
                    }catch(e){
                        reject(e);
                    }
                },0);
            });
            this.onRejectedCbs.push(() => {
                setTimeout(() => {
                    try{
                        let x = onRejected(this.reason);
                        resolvePromise(x, promise2, resolve, reject);
                    }catch(e){
                        reject(e);
                    }
                },0);
            });
        }
    });
    return promise2;    // 返回 promise 实例，实现链式调用
}
```
调用 `onFulfilled` 或者调用 `onRejected` 函数的地方都加上了定时器和 `try-catch` 语句，用定时器包裹是因为在 promise2 构造函数内部使用了 promise2 实例，**在生成实例期间就想使用实例，显然是拿不到 promise2 实例的**，这时就要使用定时器，在下一个事件循环时再运行里面的代码，这样就能拿到实例了。try-catch 语句很明显是为了捕获可能抛出的异常，有异常就传给 `reject` 函数。如果没有异常，就交给 `resolvePromise` 函数去处理（普通值或者 promise）。  

`resolvePromise` 函数代码如下：  

```js
function resolvePromise(x, promise2, resolve, reject){
    if(x === promise2){     // x 不能与 promise2 相等，不然会造成死循环
        reject(new TypeError('TypeError: Chaining cycle detected for promise #<Promise>'));
    }
    let called;
    // 如果 x 是一个对象，或者是一个方法
    if((typeof x === 'object' && x !== null) || typeof x === 'function'){
        // 带有 then 方法的对象被称为 thenable
        try{        // 防止 then 取不到报错
            let then = x.then;  // 如果能获取到 then 方法，就认为 x 是一个 promise 实例
            if(typeof then === 'function'){
                // 执行 then 方法，将 this 指向 x（x是promise实例）
                then.call(x, y => {
                    if(called)  return;
                    called = true;
                    // y 也有可能是一个 promise，递归调用，直到获得普通值或抛出异常
                    // 这里之所以判断 called 是因为，called 可能会变成 true（至少递归了一次），
                    // called === true，说明 reject 执行了，就终止递归
                    resolvePromise(y, promise2, resolve, reject);
                }, err => {
                    if(called)  return;
                    called = true;
                    reject(err);
                });
            }else{      // 对象或者函数中没有 then 方法，就按照普通值处理
                resolve(x);
            }
        }catch(e){
            if(called)  return;
            called = true;
            reject(e);
        }
    }else{      // x 是一个普通值
        resolve(x);
    }
}
```

在上面代码中，多次对 `called` 变量做判断，这是为了防止多次调用，一旦失败就 reject 出去，不再递归调用。

以上就实现了一个 Promise，如果要检测符不符合 Promise/A+ 规范，可以使用 npm 下载 promises-aplus-tests 这个包，写入下面代码：  

```js
// Promise 就是你自己封装的 Promise 类
Promise.defer = Promise.deferred = function () {
    let dfd = {};
    dfd.promise = new Promise((resolve, reject) => {
        dfd.resolve = resolve;
        dfd.reject = reject;
    });
    return dfd;
}
```

然后运行下面的代码：  

```
npx promises-aplus-tests promise.js
```

如果又不符合规范的地方，终端会输出错误信息。关于 Promise/A+ 规范可以参考这个网站：  [Promise/A+](https://promisesaplus.com/)  

###### 下一篇：[实现Promise A+(二)：实现promsie中的其它方法](/posts/manual-implementation-promise-2)
