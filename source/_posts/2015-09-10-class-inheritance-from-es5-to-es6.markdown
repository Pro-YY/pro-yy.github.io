---
layout: post
title: "Class Inheritance from ES5 to ES6"
date: 2015-09-10 19:29:24 +0800
comments: true
categories:
---

## Test Case
For test-driven development, the test case always comes first. The **Phone** inherits the Node.js **EventEmitter**: a phone **is a** customized event emitter, only augmented with its own properties and methods.

A phone has its own *name*, can *powerOn* itself.
Since it's an event emitter, it can emit 'income' event, and can set the handler by *addListener* as event emitter.

{% codeblock lang:javascript test.js %}
var EventEmitter = require('events').EventEmitter;

//var Phone = require('./phone-es5');
//var Phone = require('./phone-util');
var Phone = require('./phone-es6');   // all implementation should have same interface

var p1 = new Phone('p1');             // class instantiation with parameter
p1.powerOn();                         // invoke instance methodl

console.log(p1 instanceof Phone);     // test ISA object of super class
console.log(p1 instanceof EventEmitter);
console.log(p1);                      // print instance at run-time

var incomeListener = function(number) {
    console.log('incomming call from: ' + number);
};

p1.addListener('income', incomeListener);
p1.income('+4478123456');             // invoke instance method
Phone.vendor('xiaomi');               // static/class method
{% endcodeblock %}

Expect the following desired output:
{% codeblock %}
[Phone instance method] p1 is powered on.
true
true
Phone {
  domain: null,
  _events: {},
  _eventsCount: 0,
  _maxListeners: 20,
  name: 'p1' }
incomming call from: +4478123456
[Phone static method] vendor: xiaomi
{% endcodeblock %}

## Prototype-Based Inheritance in ES5
In fact, javascript object is prototype-based, there isn't *class* at all, everything in javascript is object.
There's only *Object Extension* (extends an exists object to a new object), rather than *Class Inheritance* (create new subclass that inherits the parent class). The prototype-based inheritance is more flexible, and it's easy to emulate traditional textbook or java-like class or inheritance.

{% codeblock lang:javascript phone-es5.js %}
var EventEmitter = require('events').EventEmitter;
var Phone = function(name) {
  EventEmitter.call(this);    // call parent's constructor
  this.setMaxListeners(20);   // customize with parent's method
  this.name = name;           // self field
};
Phone.prototype = new EventEmitter();
Phone.prototype.contstructor = Phone;
Phone.prototype.powerOn = function() {
  console.log('[Phone instance method] ' + this.name + ' is powered on.');
};
Phone.prototype.income = function(number) {
  this.emit('income', number);
};
Phone.vendor = function(name) {
  console.log('[Phone static method] vendor: ' + name);
}
module.exports = Phone;
{% endcodeblock %}

## Using Node.js util.inherits
Node.js util module provides syntactical sugar for easily implemention of class-like inheritance.
{% codeblock lang:javascript phone-util.js %}
var EventEmitter = require('events').EventEmitter;
var util = require('util');
var Phone = function(name) {
  EventEmitter.call(this);    // call parent's constructor
  this.setMaxListeners(20);   // customize with parent's method
  this.name = name;           // self field
};
util.inherits(Phone, EventEmitter);
Phone.prototype.powerOn = function() {
  console.log('[Phone instance method] ' + this.name + ' is powered on.');
};
Phone.prototype.income = function(number) {
  this.emit('income', number);
};
Phone.vendor = function(name) {
  console.log('[Phone static method] vendor: ' + name);
}
module.exports = Phone;
{% endcodeblock %}

util.inherits levarages Object.create() of ES5, the inherits line equals to:

{% codeblock lang:javascript %}
Phone.super_ = EventEmitter;
Phone.prototype = Object.create(EventEmitter.prototype, {
  constructor: {
    value: Phone,
    enumerable: false,
    writable: true,
    configurable: true
  }
});
{% endcodeblock %}

## Class Syntax of ES6
From node-v4.0, lots of ES6 features are implemented by default(shipping). Such as *block scoping*, *collections*, *promises* and *Arrow Functions*. And *Class* syntax is also fully supported. But, note that *class* is just another syntactical sugar:

{% blockquote %}
JavaScript classes are introduced in ECMAScript 6 and are syntactical sugar over JavaScript's existing prototype-based inheritance.
The class syntax is not introducing a new object-oriented inheritance model to JavaScript.
JavaScript classes provide a much simpler and clearer syntax to create objects and deal with inheritance.
{% endblockquote %}

With *class* syntax, class/inheritance can be implemented in a more straightforward manner.

{% codeblock lang:javascript phone-es6.js %}
'use strict';
var EventEmitter = require('events').EventEmitter;

class Phone extends EventEmitter {
  constructor(name) {
    super();
    this.setMaxListeners(20);
    this.name = name;
  }
  powerOn() {
    console.log('[Phone instance method] ' + this.name + ' is powered on.');
  }
  income(number) {
    this.emit('income', number);
  }
  static vendor(name) {
    console.log(`[Phone static method] vendor: ${name}`);
  }
};

module.exports = Phone;
{% endcodeblock %}

Note: 'use strict' is required, and we use the template string syntax in static method (class method).
And *class* declaration (as well as *let*, *const*, and *function* in strict mode) cannot be hoisted, unlike *var* and *function*.

In summary, with ES6 support in Node.js v4.0, class inheritance has never been more easier. So, let's embrace ES6.
