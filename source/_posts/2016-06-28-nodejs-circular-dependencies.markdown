---
layout: post
title: "Nodejs Circular Dependencies"
date: 2016-06-28 13:23:14 +0800
comments: true
categories:
---

## 模块循环依赖问题背景
在Node.js中，模块间的循环依赖(Circular Dependencies)引用如果处理不好，往往会导致很难调试的问题。

常见的问题情形是：新加入一个模块后，先前的某个模块就在开始载入时成为部分加载状态，导致依赖该它的其他模块找不到本应导出的变量或方法。导致模块链接在运行时出错。

### Node.js的module系统支持部分加载
代码实例如下：
```javascript auth.js
'use strict';
const user = require('./user');

console.log('# in auth: user is');
console.log(user);

function authenticate(name) {
    return user.find(name);
}

function enabled() {
    return true;
}

module.exports = {
    authenticate: authenticate,
    enabled: enabled
};

console.log('# auth loaded');
```
```javascript user.js
'use strict';
const message = require('./message');

console.log('# in user: message is');
console.log(message);

function find(name) {
    console.log('found user: ' + name);
    message.hello(name);
    return name;
}

module.exports = {
    find: find
};

console.log('# user loaded');
```
```javascript message.js
'use strict';
const auth = require('./auth');

console.log('# in message: auth is');
console.log(auth);

function hello(name) {
    console.log('hello, ' + name);
    // if (auth.enabled(name)) console.log('hello, ' + name);
}

module.exports = {
    hello: hello
};

console.log('# message loaded');
```
```javascript main.js
'use strict';
console.log('# main starting');
const auth = require('./auth');
console.log('# in main, auth is');
console.log(auth);
console.log('start running...');
auth.authenticate('Alice');
```

运行结果
```bash result
# main starting
# in message: auth is
{}
# message loaded
# in user: message is
{ hello: [Function: hello] }
# user loaded
# in auth: user is
{ find: [Function: find] }
# auth loaded
# in main, auth is
{ authenticate: [Function: authenticate],
      enabled: [Function: enabled] }
start running...
found user: Alice
hello, Alice
```
三个模块的关系为：auth模块引用user模块，并调用了其导出的find的方法；user模块引用message模块，并调用其导出的hello方法；message模块引用auth模块，但没有调用其任何导出的方法。
main模块引用了auth模块，并调用了其导出的authenticate方法。即程序存在模块循环依赖，但不存在循环模块引用（方法间的循环调用）。

从结果我们可以看出：main函数应用auth模块时，vm根据require规则相继递归加载了user和message模块，message模块先完成全部加载（接着相继是user、auth、main）。注意到，在message加载时，由于auth模块尚未完成加载，在message模块内表现为一个临时的空对象，即部分加载。之后的main加载时，auth模块已经全部完成初始化加载过程，因此可以正常使用了。

## 问题引入
改变message中的hello方法实现（第8、9行），引入循环依赖。
```javascript message.js
'use strict';
const auth = require('./auth');

console.log('# in message: auth is');
console.log(auth);

function hello(name) {
    // console.log('hello, ' + name);
    if (auth.enabled(name)) console.log('hello, ' + name);
}

module.exports = {
    hello: hello
};

console.log('# message loaded');
```
再次运行，结果如下
```bash result
# main starting
# in message: auth is
{}
# message loaded
# in user: message is
{ hello: [Function: hello] }
# user loaded
# in auth: user is
{ find: [Function: find] }
# auth loaded
# in main, auth is
{ authenticate: [Function: authenticate],
  enabled: [Function: enabled] }
start running...
found user: Alice
/xxxx/xxxxx/xxxxx/xxxx/message.js:8
    if (auth.enabled(name)) console.log('hello, ' + name);
             ^

TypeError: auth.enabled is not a function
```
有了之前的分析，错误的原因也就不难分析了：message模块加载时，auth模块还未初始化完成，即此时还未用已经初始化完成的正确对象来覆盖module.exports对象，从临时的空对象里当然就找不到enabled方法。

## 循环依赖解决
那么可不可以“毕其功于一役”，让每个模块既能在加载时就导出正确的对象，同时将模块中的方法定义和实现推移至运行时再动态改变呢？
###前置模块导出
前置模块导出的方法，可以做到将模块方法的定义和挂载延迟到运行时，而非在模块第一次加载完成后再用一个变量统一覆盖掉module.exports对象(像举例中的传统模块声明方法)。
因此更高效地利用了部分加载的特性，让模块间彼此更加独立、协作更好。对于刚才的问题，具体的优化实现如下。
```javascript auth.js
'use strict';
const Module = module.exports;

const user = require('./user');

console.log('# in auth: user is');
console.log(user);

function authenticate(name) {
    return user.find(name);
}

function enabled() {
    return true;
}

Module.authenticate = authenticate;
Module.enabled = enabled;

console.log('# auth loaded');
```
```javascript user.js
'use strict';
const Module = module.exports;

const message = require('./message');

console.log('# in user: message is');
console.log(message);

function find(name) {
    console.log('found user: ' + name);
    message.hello(name);
    return name;
}

Module.find = find;

console.log('# user loaded');
```
```javascript message.js
'use strict';
const Module = module.exports;

const auth = require('./auth');

console.log('# in message: auth is');
console.log(auth);

function hello(name) {
    //console.log('hello, ' + name);
    if (auth.enabled(name)) console.log('hello, ' + name);
}

Module.hello = hello;

console.log('# message loaded');
```
``` bash result
# main starting
# in message: auth is
{}
# message loaded
# in user: message is
{ hello: [Function: hello] }
# user loaded
# in auth: user is
{ find: [Function: find] }
# auth loaded
# in main, auth is
{ authenticate: [Function: authenticate],
  enabled: [Function: enabled] }
start running...
found user: Alice
hello, Alice
```
引用的错误解决，运行正确。

基于这种设计的模块前置导出技术，即“加载时先导出模块、运行时后实现方法”的模块实现思路，可应用于存在多模块间循环依赖引用的情况，从而方便实现逻辑关系更复杂的模块。
