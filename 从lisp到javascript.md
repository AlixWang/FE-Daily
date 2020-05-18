# 用`Lisp`的思维来写`Javascript` #

> Lisp语言是最古老的计算机语言之一，也是函数式语言的鼻祖，现今主要的方言有`Common Lisp`,`Scheme`,`Clojure`。而最广为人知的`emacs`编辑器也是用`lisp`作为其扩展语言。

## Lisp的基本语法 ##

`Lisp`中最具特色的应该就是其独特的前缀表示法和数量众多的括号，比如在`Javascript`中进行加减乘除运算是这样：

```javascript
((2 + 2) * 3 / 4) // 3
```

但是在`Lisp`中写作这样：

```lisp
(/ (* 3 (+ 2 2)) 4) // 3
```

因为`Lisp`是函数式的编程语言，联想到函数调用的写法不难看出此处的前缀表示法就可以看做是`+ - * /`的函数调用。可以用`js`来进行如下表达。

```javascript

function add (...args) { 
    // do  add
}

function sub (...args) {
    // do  subtraction 
}

function multiplication (...args) {
    // do  multiplication
}

function division (...args) {
    // do division
}

(division(multiplication(3, (add(2, 2)))), 4)

```

可以看到当我们把`js`中的加减乘除进行函数包装然后进行调用完全是和`lisp`中的前缀表示法的形式是一致的。所以lisp中的运算符完全可以看做是函数的调用。这样一来就不难理解为什么`Lisp`中未出现如此奇怪的前缀表示法而不是大多数语言采用的中缀表示法。

 

