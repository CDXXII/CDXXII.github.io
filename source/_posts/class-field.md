---
title: 关于 class fields proposal 
date: 2018-11-03
---
最近关于社区关于 [class fields](https://github.com/tc39/proposal-class-fields/issues) 提案的讨论如火如荼，两方阵营都吵的不可开交。我们就来简单总结一下该提案有争议的内容。

## private fields 

简单讲，`#` is the new `_`。还记得我们之前是怎么实现 private fields 的吗？

````js
"use strict";

class Counter {
  constructor() {
    this._count = 0;
  }
  get count() {
    return this._count;
  }
  inc() {
    ++this._count;
  }
}

const c = new Counter();
c.inc();
c.count; // 1
c.count = 10; // (only in strict mode) Uncaught TypeError: Cannot set property count of #<Counter> which has only a getter
c._count = 10;
````

其实这样并不是真的 private，我们还是可以修改私有属性。我们接着改进：

````js
const count = Symbol();
class Counter {
  constructor() {
    this[count] = 0;
  }
  get count() {
    return this[count];
  }
  inc() {
    ++this[count];
  }
}
const c = new Counter();
c.inc();
c.count; // 1
// reflection
c[Object.getOwnPropertySymbols(c)[0]] = 10;
````

如上代码，我们虽然仍可以得到私有属性。但是这样的写法已经「相对安全」了。那最新的[提案](https://tc39.github.io/proposal-private-fields/)是怎么解决这个问题的呢？

````js
class Counter {
  #count = 0;
  get count() {
    return this.#count;
  }
  inc() {
    ++this.#count;
  }
}
````

虽然很丑陋，而且[争议](https://github.com/tc39/proposal-class-fields/issues/100)很大，但是终于，私有属性 `count` 已经「绝对安全」了。

## `[[Define]]` or `[[set]]` 

还有一点值得注意的是就是类字段从 `[[set]]` 到 `[[Define]]` 语义的[转变](https://github.com/tc39/proposal-class-fields#public-fields-created-with-objectdefineproperty)。还是从刚才那个例子说起：

````js
class Counter {
  #count = 0;
  set count(val) {
    this.#count = val;
  }
  get count() {
    return this.#count;
  }
  inc() {
    ++this.#count;
  }
}

class CounterStartFrom100 extends Counter {
  count = 100;
}

let c = new CounterStartFrom100()
````

在 Babel 7 以及 Chrome 72 以上，当 `new CounterStartFrom100()` 被调用的时候，按照 `[[Define]]` 的语义，并不会触发基类上的 `getter/setter` 。

## 参考资料

- [关于 class fields 的神秘话题](https://johnhax.net/2019/class-fields/slide?netease#1)


























