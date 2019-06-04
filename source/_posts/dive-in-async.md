---
title: 深入 async functions
date: 2018-11-11
---

本文主要阐述了在 Javascript 中 `async` 函数的相关特性和运行机制，及 V8 引擎如何对异步函数进行优化。

## async function 的实现

`async` 函数本质是一个自带 runner 的 `Generator` 函数。我们可以使用 `Generator` 和 `Promise` 的特性简单实现一个 `async` 函数。

```js
function spawn(generatorFunc) {
  return new Promise(function(resolve, reject) {
    const iterator = generatorFunc();

    function step(nextFunc) {
      let next;

      try {
        next = nextFunc();
      } catch (err) {
        return reject(err);
      }

      if (next.done) {
        return resolve(next.value);
      }

      Promise.resolve(next.value).then(
        function(v) {
          step(function() {
            return iterator.next(v);
          });
        },
        function(e) {
          step(function() {
            return iterator.throw(e);
          });
        }
      );
    }

    step(function() {
      return iterator.next(undefined);
    });
  });
}

const sleep = (ms = 1000) => new Promise(resolve => setTimeout(resolve, ms));

function* init() {
  yield sleep();
  console.log(1);

  yield sleep();
  console.log(2);

  yield sleep();
  console.log(3);
}

spawn(init);
```

## async function constructor

除了传统的函数声明和函数表达式，也可以通过异步函数构造器来创建 `async` 函数。但是需要注意的是 `AsyncFunction` 并不是一个全局对象。只能通过 `Object.getPrototypeOf()` 获得。同样，可以通过修改 `AsyncFunction` 的原型对象对所有 `AsyncFunction` 的实例提供自定义方法。

```js
const AsyncFunction = Object.getPrototypeOf(async function() {}).constructor;

AsyncFunction.prototype.say = function() {
  console.log('Don\'t Panic!');
};

const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));

let foo = new AsyncFunction(
  'name',
  'await sleep(1000); console.log("Hi " + name)'
);

foo(42); // Hi 42

foo.say(); // Don't Panic!
```

## 使用 async functions 的注意事项

首先，需要注意的是 `async` 函数永远都会返回一个 `Promise` 对象，如果没有显式的指定返回值则会返回一个状态为 resolved 且值为 `undefined` 的 `Promise` 对象（抛出异常且未捕获除外）。

通常 `await` 会用来等待 `Promise` 对象。如果该值不是一个 `Promise` 对象(或 thenable 对象)，`await` 会把该值转换成状态为 resolved 的 `Promise` 对象，然后等待其处理结果。使用 `await` 等待一个被调用的方法，则会执行该方法并将其返回值转换为 `Promise` 对象，然后同样等待其处理结果。执行过程遇到的任何同步异常都会导致 `Promise` 对象的状态变成 rejected。

```js
async function someAsyncThing() {
  throw 'oops';
  await Promise.resolve('something happens'); // expression is unreachable!
}; // Uncaught (in promise) oops

let foo = someAsyncThing();

console.log(foo) // Promise {<rejected>: "oops"}

const someOthersThings = function() {
  console.log('You again!'); // You again!
};

someOthersThings();
```

如上述代码，函数体内部同步抛出未捕获的异常，会导致 `async` 函数中断执行，并会立即返回一个状态为 rejected 的 `Promise` 对象。但并不会使得脚本挂起而导致后续代码无法执行。

````js
async function someAsyncThing() {
  await Promise.reject('oops');
  await Promise.resolve('something happens'); // expression is unreachable!
} // Uncaught (in promise) oops

let foo = someAsyncThing();

console.log(foo); // Promise {<pending>}

foo.catch(err => console.log(err)); // oops
````

如果任何一个 `await` 操作符后面的 `Promise` 对象变为 rejected 状态， `async` 函数同样会中断执行，立即返回一个状态为 pending 的 `Promise` 对象，并异步的将状态变为 rejected。

大多数情况下，从结果来看`return` 和 `return await` 没有什么明显的区别。虽然，根据各个 runtime  间的差异性，`return await` 可能会因为多创建一个中间对象（`Promise` 对象）而浪费少量的内存。但它们之间的差异性主要表现在嵌套于 `try...catch` 代码块中。下面是一个 `await`、`return`、`return await` 之前区别的例子。

````js
// without return or await
async function rejectionWithoutReturnOrAwait() {
  try {
    Promise.reject(new Error('oops')); // Uncaught (in promise) Error: oops
  } catch (e) {
    return 'Saved!';
  }
}

let resultA = rejectionWithoutReturnOrAwait();

console.log(resultA); // Promise {<resolved>: undefined}

resultA.catch(err => console.log(err)); // unreachable code

// awaiting
async function rejectionWithAwait() {
  try {
    await Promise.reject(new Error('oops')); // Uncaught (in promise) Error: oops
  } catch (e) {
    return 'Saved!';
  }
}

let resultB = rejectionWithAwait();

console.log(resultB); // Promise {<pending>}

resultB.then(res => console.log(res)); // Saved!

// returning
async function rejectionWithReturn() {
  try {
    return Promise.reject(new Error('oops'));
  } catch (e) {
    return 'Saved!';
  }
}

let resultC = rejectionWithReturn();

console.log(resultC); // Promise {<pending>}

resultC.catch(err => console.log(err)); // Error: oops

// return-awaiting
async function rejectionWithReturnAndAwait() {
  try {
    return await Promise.reject(new Error('oops'));
  } catch (e) {
    return 'Saved!';
  }
}

let resultD = rejectionWithReturnAndAwait();

console.log(resultD); // Promise {<pending>}

resultD.then(res => console.log(res)); // Saved!
````

## V8 引擎对异步函数的优化

{% asset_img await-bug-node-8.svg %}

上图的代码分别展示了不同版本的 V8 引擎对 `await` 语句执行顺序的差异性。导致这一差异的主要原因是 `await p;` 这条语句。

简单的讲，Node.js 10 中的实现（V8 团队的 PR 被 merge 之前）是符合现行标准的，而 Node.js 8 中的实现并没有严格按照 [Promise Resolve Functions](https://tc39.github.io/ecma262/#sec-promise-resolve-functions) 中的第十三步执行。但是我们在 Chrome 72 以上的版本执行这个代码段，依然会得到和 Node.js 8 中一样的答案，是因为在最新版本的 V8 引擎，V8 团队为了提高 `async` 函数的执行效率，提出新的[优化建议](https://github.com/tc39/ecma262/pull/1250)并已经被 TC39 采纳。

它们之间到底有什么区别呢？我们将上图中的代码稍稍改进一下。

````js
const p = Promise.resolve();

(async () => {
  await p;
  console.log('after:await');
})();

p.then(() => {
  console.log('tick:a');
})
  .then(() => {
    console.log('tick:b');
  })
  .then(() => {
    console.log('tick:c');
  });

// Node.js 10:
// tick:a
// tick:b
// after:await
// tick:c

// Chrome 73:
// after:await
// tick:a
// tick:b
// tick:c
````

其中关键代码便是 `await p;`。而导致差异性的原因是因为 `p` 是一个 `Promise` 对象。`Promise` 构造器的 callback 内 `resolve(thenable)` 并不等于 `Promise.resolve(thenable)`。

依照现行（V8 团队的 PR 被 merge 之前，也就是 Node.js 10 里）规范，上述代码中的 IIFE 等价于：

````js
(() => {
  return new Promise(resolve => {
    new Promise(res => {
      res(p);
    }).then(() => {
      console.log('after:await');
      resolve();
    });
  });
})();
````

其中执行 `res(p);` 这条语句的时，因为 `p` 是一个 `Promise` 对象，而 `Promise` 对象都是 thenable 对象。所以在执行 `res()`  之前，JavaScript 引擎会创建一个 MicroTask 去处理这个 thenable 对象，  在[规范](https://tc39.github.io/ecma262/#sec-promiseresolvethenablejob)中这个 MicroTask 被定义为 PromiseResolveThenableJob。PromiseResolveThenableJob 的执行过程中会调用 thenable 对象的 `then` 方法，而 `Promise` 对象的 `then` 方法也是一个 MicroTask，所以在 loop 中，这个过程增加了两次 MicroTick。若 thenable 对象的 then 方法是一个同步方法，则这个过程只会增加一次 MicroTick。


而在 Chrome 73 及未来版本的 Node.js 中，`res(p);` 这条语句等价于 `Promise.resolve(p)` 。即：

````js
(() => {
  return new Promise(resolve => {
    Promise.resolve(p).then(() => {
      console.log('after:await');
      resolve();
    });
  });
})();
````

那这样做的好处是什么呢？显而易见，在代码执行过程中不需要再次创建用于包装的 `Promise` 对象，至少三次的 MicroTick 被优化到只有一个。如下图所示，这样做提高了 `async` 函数的性能。

{% asset_img await-benchmark-optimization.svg %}

## 参考资料

- [Async-Await ≈ Generators + Promises](https://hackernoon.com/async-await-generators-promises-51f1a6ceede2)
- [await vs return vs return await](https://jakearchibald.com/2017/await-vs-return-vs-return-await/)
- [Faster async functions and promises](https://v8.dev/blog/fast-async#await-under-the-hood)
- [Difference between resolve(thenable) and resolve('non-thenable-object')](https://stackoverflow.com/questions/53894038/whats-the-difference-between-resolvethenable-and-resolvenon-thenable-object)
