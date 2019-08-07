# 深入react-fiber-(react新调度算法的深入理解) #

![head-picture](http://ww1.sinaimg.cn/large/006tNc79gy1g5o1la5tiej30j708caa5.jpg)

我们都知道React是基于追踪组件状态更新并同步至屏幕上用来构建用户界面的前端库。在react中我们知道这是基于一个名叫reconciliation（调解）的过程，我们调用`setState`方法而框架内部则比较state和props的变化去重绘组件更新UI界面。

React的官方文档对此提供了一个很好地高级概述：React元素、生命周期函数、`render`函数的规则以及运用于子组件的diff算法。从render方法返回的不可变React元素树通常称为“虚拟DOM”。这个术语很早就帮助人们解释了React，但是同样引入了令人迷惑的事情，而且并没有在react的官方文档中有任何提及。在本文中我将坚称其为react元素树。
除了React元素树，框架内部也总是有内部示例树（比如组件、DOM节点等）用来保持内部状态。从版本16开始，React推出了该内部实例树的新实现以及管理代号为Fiber的算法。为了了解Fiber架构带来的优势，[了解React在Fiber中使用链表的方式和原因](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)。

> 这篇文章如果没有[Dan Abramov](https://medium.com/@dan_abramov)的帮助的话一定会花费更多的时间，而且没有如此全面。

这是一系列教你理解React内部原理的文章的第一篇。在本文中，我想提供与算法相关的重要概念和数据结构的深入概述，一旦我们了解了足够的背景知识，我们将会探索用于递归和解析fiber树的算法和主要函数。此系列的下一篇文章将演示React如何使用该算法执行初始渲染和处理state以及props更新。从那里我们将继续讨论调度程序的细节，子调解(reconciliation)进程，以及建立Effect list的机制。

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

这里有几种React会在(reconciliation)调解过程中会执行的操作。例如，以下是React在我们的简单应用程序中的第一次渲染和状态更新之后执行的高级操作：

+ 在`ClickCounter`组件`state`中更新`count`属性
+ 检索和比较`ClickCounter`组件的`Children`和`Props`
+ 更新`span`元素的`props`

在(reconciliation)调解的过程中还有一些其他操作被执行例如调用生命周期函数及更新`reef`，所有这些活动在`Fiber`架构中统称为“`work`”。`work`的类型通常都取决于React组件的类型。列如，对于Class组件，React需要去创建实例，但对于函数式组件来说却并不会。正如你所知，在React里面有多种组件类型。例如：class和函数式组件，原生组件(DOM)、`portals`等。React组件类型是由`createElement`函数的第一个参数决定的。这个函数通常被用于在`render`方法中创建组件元素。

在我们探索`Fiber`架构之前，先让我们熟悉一下React内部的数据结构。

## 从React Elements 到Fiber 节点 ##

React中的每个组件都有一个UI表示，我们可以调用一个视图或一个从render方法返回的模板。下面是我们`ClickCounter`组件的模板代码。

```javascript
<button key="1" onClick={this.onClick}>Update counter</button>
<span key="2">{this.state.count}</span>
```

## React 元素 ##

当模板通过JSX的解释器后，你将会得到一系列的React元素。这才是react组件的`render`方法真正返回的格式，而不是你看见的`HTML`模板。如果不使用JSX的话，那么我们的组件可以写成扎样。

```javascript
class ClickCounter {
    ...
    render() {
        return [
            React.createElement(
                'button',
                {
                    key: '1',
                    onClick: this.onClick
                },
                'Update counter'
            ),
            React.createElement(
                'span',
                {
                    key: '2'
                },
                this.state.count
            )
        ]
    }
}
```

`render`方法中的`React.createElement`调用结果将会返回下面的数据结构：

```javascript
[
    {
        $$typeof: Symbol(react.element),
        type: 'button',
        key: "1",
        props: {
            children: 'Update counter',
            onClick: () => { ... }
        }
    },
    {
        $$typeof: Symbol(react.element),
        type: 'span',
        key: "2",
        props: {
            children: 0
        }
    }
]
```

我们可以看到React为这些对象添加了`$$typeof`属性唯一的将他们标识为React元素。用`type`、`key`和`props`描述此对象。而他们的值就是你通过`React.createElement`传过来的值。请注意React如何将文本内容表示为`span`和按钮节点的子项。和`click`的监听函数是如何作为`button`元素的`props`。React其他方面比如`ref`字段就超出了本文的讨论范围。

`ClickCounter`的React元素并没有任何的`props`和`key`:

```javascript
{
    $$typeof: Symbol(react.element),
    key: null,
    props: {},
    ref: null,
    type: ClickCounter
}
```

## Fiber节点 ##

在调解(reconciliation)期间React组件的`render`方法返回的数据会被合并进入fiber节点树中。与React组件不一样，fiber不会再每个渲染周期内重新创建。他们是包含组件状态和DOM节点的可变数据结构。

前文说过React根据组件的不同类型，框架会执行不同的动作。在我们的简单示例中，对于`class`组件`ClickCounter`来说会调用生命周期函数和`render`方法，而对于`span`原生类型(DOM Node)的组件来说其会响应DOM的改变。因此，每个React元素都会转换为相应类型的Fibre节点，用于描述需要完成的工作。

您可以将Fiber视为代表某些工作要做的数据结构，换句话说，就是一个工作单元
，Fiber架构也提供了简便的方法去追踪、调度、暂停和取消工作进程。

当一个React组件第一次转变为Fiber节点的时候，React使用元素中的数据在createFiberFromTypeAndProps函数中创建Fiber。在随后的更新中，React重用Fiber节点，并使用来自相应React元素的数据更新必要的属性。

[原文链接](https://blog.ag-grid.com/inside-fiber-an-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)
