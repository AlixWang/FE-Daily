# 迭代器协议和可迭代协议 #

## 0x01 起源 ##

思考一下以下代码：

```javascript
var obj = {a:1,b:2}
[...obj]
```

如何使以上代码不报错?

## 0x02 思考 ##

这里就要基于迭代器协议和可迭代协议对obj对象进行一系列的改造。
首先扩展运算符起到的作用是啥？
我们知道`{...obj}`对于对象的扩展运算等价于Object.assign({},obj),即对obj进行了一次浅拷贝，而对于数组的扩展呢？

```javascript
var arr = [1,2,3,4]
[...arr]
```

这里其实也是对数组进行了一次扩展运算效果等价于 arr.slice(0)，而当我们将对象进行数组展开的时候会发生啥呢？
`[...obj]`会提示`TypeError: obj is not iterable`根据提示我们知道数组的扩展运算需要在一个实现了iterable的对象上进行，ok那什么叫实现了iterable呢？

## 0x03 原理 ##

根据MDN文档的描述，为了变成可迭代对象，一个对象必须实现@@iterator方法， @@iterator 方法, 意思是这个对象（或者它原型链 prototype chain 上的某个对象）必须有一个名字是 Symbol.iterator 的属性:
那么是否只要实现了可迭代协议了这个对象就是可迭代的呢？我们试一试

```javascript
  obj[Symble.iterator] = function() {
    return this
  }
  [...obj]
```

ok 经过测试我们发现只满足可迭代协议的情况下obj依然不是一个合法的可迭代对象，我们还需要实现obj的迭代器协议，迭代器协议的具体规范可以参展MDN上的讲解，简单来说就是对象要有一个next方法返回一个包含done和value的对象，

```javascript
  obj.next = function() {
    return {
      done:true,
      value: undefined
    }
  }
```

再来一次`[...obj]`试试看能不能成功，ok这次终于没有报错了，直接返回了空数组。

让我们来完整的实现一个返回对象value的迭代器：

```javascript
var obj = { a: 1, b: 2 }

obj[Symbol.iterator] = function () {
  return {
    next: function () {
      if (this._count < this._value.length) {
        this._count += 1
        return {
          done: false,
          value: this._value[this._count - 1]
        }
      } else {
        return {
          done: true
        }
      }

    },
    _value: Object.values(this),
    _count: 0
  }
}

[...obj] // [1,2]
```

## 0x04 扩展 ##

js内置对象里面具有迭代器的有哪些：

> String, Array, TypedArray, Map and Set

js内可以接受迭代器作为参数的有哪些

> Map([iterable]), WeakMap([iterable]), Set([iterable]) , WeakSet([iterable]), Promise.all(iterable), Promise.race(iterable) Array.from()

js内用于可迭代的语法

>for-of ,yeild, 扩展运算符，以及结构赋值

## 0x5 参考链接 ##

[MDN 迭代协议]('https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Iteration_protocols')

