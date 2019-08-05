# 深入react-fiber-(react新调度算法的深入理解) #

![head-picture](http://ww1.sinaimg.cn/large/006tNc79gy1g5o1la5tiej30j708caa5.jpg)

我们都知道React是基于追踪组件状态更新并同步至屏幕上用来构建用户界面的前端库。在react中我们知道这是基于一个名叫reconciliation（调解）的过程，我们调用`setState`方法而框架内部则比较state和props的变化去重绘组件更新UI界面。

React的官方文档对此提供了一个很好地高级概述：React元素、生命周期函数、`render`函数的规则以及运用于子组件的diff算法。从render方法返回的不可变React元素树通常称为“虚拟DOM”。这个术语很早就帮助人们解释了React，但是同样引入了令人迷惑的事情，而且并没有在react的官方文档中有任何提及。在本文中我将坚称其为react元素树。
除了React元素树，框架内部也总是有内部示例树（比如组件、DOM节点等）用来保持内部状态。从版本16开始，React推出了该内部实例树的新实现以及管理代号为Fiber的算法。为了了解Fiber架构带来的优势，[了解React在Fiber中使用链表的方式和原因](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)。

> 这篇文章如果没有[Dan Abramov](https://medium.com/@dan_abramov)的帮助的话一定会花费更多的时间，而且没有如此全面。

这是一系列教你理解React内部原理的文章的第一篇。在本文中，我想提供与算法相关的重要概念和数据结构的深入概述，一旦我们了解了足够的背景知识，我们将会探索用于递归和解析fiber树的算法和主要函数。此系列的下一篇文章将演示React如何使用该算法执行初始渲染和处理state以及props更新。从那里我们将继续讨论调度程序的细节，子调解进程，以及建立Effect list的机制。

## 继续 ##

我将带你一起了解一些React的高级知识🧙‍。我鼓励你阅读它以了解Concurrent React内部工作背后的魔力。而且这一系列的文章也能作为想React开源项目贡献的指南。我非常相信逆向工程，因此最新版本16.6.0中的源代码会有很多链接。

这里包含很多内容，所有如果你不能马上理解的话不要感到有压力。这些内容值得你花时间去了解。注意如果只是为了能熟练使用React，那么你没必要了解这些内容。这篇文章是为了

## 设置背景 ##

这里有一个简单的例子我将贯穿这一系列文章。屏幕上为一个递增的按钮。

![picture](http://ww4.sinaimg.cn/large/006tNc79gy1g5p8b2lejng305u01wq3m.gif)

具体代码如下

```javascript
class ClickCounter extends React.Component {
    constructor(props) {
        super(props);
        this.state = {count: 0};
        this.handleClick = this.handleClick.bind(this);
    }

    handleClick() {
        this.setState((state) => {
            return {count: state.count + 1};
        });
    }


    render() {
        return [
            <button key="1" onClick={this.handleClick}>Update counter</button>,
            <span key="2">{this.state.count}</span>
        ]
    }
}
```

你可以在[此处](https://stackblitz.com/edit/react-t4rdmh)进行在线查看，正如你所见，这只是一个简单包含`span`和`button`元素的组件。单击该按钮后，组件的状态将在处理程序内更新。反过来，这会导致span元素的文本更新。

[原文链接](https://blog.ag-grid.com/inside-fiber-an-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)
