# 一种快速将将10进制转换为二进制的方法

今天本来无意中发现`Number`的`toString`方法不是完全的继承自`Object`原型上，而是可以给其传参数，
这就引起了我比较大的兴趣。我就想知道这个传入的参数到底有啥用？还有`Number`上的`toString`方
法到底和`Object`上的`toString`方法有啥区别。

## 不同数据类型的toString方法分别返回啥

1. `Object` 对象的`toString` 返回的就是表示对象数据类型的字符串 `[object Object]`,进
而我们可以根据对象`toString`方法使用`Object.prototype.toString.call([arry | number | string | ...])` 来达到精确判断数据类型的功能比如：

    ```javascript
    const a = []
    const b = ''
    Object.prototype.toString.call(a) // [object Array]
    Object.prototype.toString.call(b) // [object String]

    ```

2. `Array`上的`toString`方法分为两种情况
   + 数组内元素是简单数据类型会将数组内的元素用`,`合并成字符串返回例如：

    ```javascript
    const a = [1,2,3,4]
    a.toString() // 1,2,3,4
    ```

   + 数组内元素是复杂数据类型这种情况下会调用子元素的`toString`方法

    ```javascript
    const a = [{},{}]
    a.toString() // [object Object],[object Object]
    const b = [()=>{},()=>{}]
    b.toString() // ()=>{},()=>{}
    const c = [[1,2],[3,4]]
    c.toString() // 1,2,3,4
    ```

    延申一下，所以我们完全可以通过数组的`toString`方法来实现`join`的简单功能

3. `Function`上的`toString`方法，这个就比较简单了就是纯粹的将函数体作为字符串返回

    ```javascript
        var a = () => {}
        a.toString() // ()=>{}
    ```

4. `String`的`toString`方法，就是纯粹的返回字符串

## 与众不同的`Number`类型的`toString`方法

`Number`类型的`toString`方法究竟是啥样的呢？让我们来看看

```javascript
var a = 1
a.toString() // 1
```

看起来好像也没有什么神奇的地方啊
