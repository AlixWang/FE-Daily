
# 广泛存在于现代浏览器中的内存泄露问题

本文来自与shopify工程师Jake,详细叙述了工作中发现JS垃圾收集引擎并不总是能按照预期工作，我看了之后感觉还是比较有意思的，在这里翻译分享给大家。


```javascript

function demo() {
  const bigArrayBuffer = new ArrayBuffer(100_000_000);
  const id = setTimeout(() => {
    console.log(bigArrayBuffer.byteLength);
  }, 1000);

  return () => clearTimeout(id);
}

globalThis.cancelDemo = demo();


```
如上面代码所示，`bigArrayBuffer`在`demo`执行完毕后是不会被GC回收的，虽然返回的`cancelDemo`函数没有直接引用`bigArrayBuffer`。想要知道为何会如此我们先看看以下几个不会造成内存泄露的例子。

## 聪明的JS垃圾收集器

```javascript

function demo() {
  const bigArrayBuffer = new ArrayBuffer(100_000_000);
  console.log(bigArrayBuffer.byteLength);
}

demo();

```

函数执行完成后，`bigArrayBuffer`就被回收了，因为很明显看到不存在任何对`bigArrayBuffer`的引用了。


```javascript

function demo() {
  const bigArrayBuffer = new ArrayBuffer(100_000_000);

  setTimeout(() => {
    console.log(bigArrayBuffer.byteLength);
  }, 1000);
}

demo();

```
这个例子里面虽然添加了一个`setTimeout`定时器但是也已一样当定时器执行完毕后也不存在对`bigArrayBuffer`的引用了。

```javascript

function demo() {
  const bigArrayBuffer = new ArrayBuffer(100_000_000);

  const id = setTimeout(() => {
    console.log('hello');
  }, 1000);

  return () => clearTimeout(id);
}

globalThis.cancelDemo = demo();

```
这里也一样，虽然`demo`函数也返回了一个清楚定时器函数，但是定时器本身并没有`bigArrayBuffer`的引用。

## 泄露的根源

再回头看看开头的那个会造成内存泄露的例子，跟上面这三个不会造成内存泄露的例子做对比我们会发现，主要有以下几个原因：

1. `setTimeout`内部函数引用了`bigArrayBuffer`，当`demo()`函数调用后内部作用域就被创建了。
2. 过了一秒后虽然`setTimeout`内部函数不会再次被调用了，但是`demo()`创建的作用域还是会被继续保存，因为`setTimeout`的清理方法还是保存着。
3. 因为`bigArrayBuffer`在这个作用域中，所以`bigArrayBuffer`不会被垃圾回收。

所以这里的`bigArrayBuffer`泄露虽然不如预期，但是在我看来还是有一定的逻辑合理性的。如果想要GC回收掉`bigArrayBuffer`只需要去掉`cancelDemo`的引用即可。

```javascript

 globalThis.cancelDemo = null;

```

## 其他泄露的情况（除定时器以外）

```
  function demo() {
    const bigArrayBuffer = new ArrayBuffer(100_000_000);

    globalThis.innerFunc1 = () => {
      console.log(bigArrayBuffer.byteLength);
    };

    globalThis.innerFunc2 = () => {
      console.log('hello');
    };
  }

  demo();
  // bigArrayBuffer 没有被回收, 符合预期.

  globalThis.innerFunc1 = undefined;
  // bigArrayBuffer 没有被回收, 不符合预期.

  globalThis.innerFunc2 = undefined;
  // bigArrayBuffer 现在被回收.

```

这个例子乍看之下有点反直觉，明明`innerFunc2`已经手动置空了为什么`bigArrayBuffer`还是没有被回收。
结合下面这张图我们来分析一下

[leak]("./leak.png")

当Demo函数调用后，因为在全局GlobalThis上有属性 对Demo内部函数有引用，因此Demo的作用域会保留。 又由于
在Demo内部被GlobalThis属性引用的函数内有 BigArrayBuffer的引用，BigArrayBuffer在V8内部也会 被加入到
Demo的作用域对象里面。 当我们手动解除innerFunction1的对Demo内部函数的 引用时，虽然这时候从逻辑上来看
此时Demo作用域内部 不存在对BigArrayBuffer大对象的直接引用。GC应该能 正常回收了，但是由于Demo作用域是
在之前Demo函数 调用时就已经创建了，其内部BigArrayBuffer对象还是不会 动态回收掉，直到我们将innerFunction2
也解除引用。Demo 作用域整体被GC回收掉。








> ![原文地址](https://jakearchibald.com/2024/garbage-collection-and-closures/)
