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

当一个React组件第一次转变为Fiber节点的时候，React使用元素中的数据在createFiberFromTypeAndProps函数中创建Fiber。在随后的更新中，React重用Fiber节点，并使用来自相应React元素的数据更新必要的属性。如果不再从render方法返回相应的React元素，React可能还需要根据key prop移动层次结构中的节点或删除它。

查看[**ChildReconciler**](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactChildFiber.js#L239)函数查看`React`为现有Fieber节点执行的所有活动和函数。

因为React为每个元素创建了一个Fiber节点，因为这些这些元素组成了一个树，所以我们得到了一个Fiber节点组成的树。在`ClickCounter`示例中fiber树形如下图：

![image-1.png](https://tupp.xyz/2019/08/09/15653219085d4ceab46c3c7.png)

所有的Fiber Nodes 都通过 `child sibling return`链接成一个链表。可以通过阅读我的这篇文章[The how and why on React’s usage of linked list in Fiber](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)来了解为什么要使用链表的原因。

## Current 树和workinprogress树 ##

在首次渲染过后，React最终得到了一个Fiber树，它反映了用于呈现UI的应用状态。这种Fiber树通常被简称为Current树。当React开始进行状态更新时，它产生的树叫做workInProgress树其反映了将来要被渲染到屏幕上的React应用状态。

所有的工作都在`workInProgress`的`Fiber`树上执行，当`React`遍历`Current`树时，对于每个现有Fiber节点，它会创建一个构成`workInProgress`树的备用节点。这个节点是通过`Render`方法返回的数据创建的。当更新被执行且所有相关的工作的完成的时候，React会产生一个备用树去刷新当前屏幕，而当`workInProgress`树的状态更新到屏幕上之后，它就成了`Current`树。(可以看出`Current`树会被`workInPorgress`树取代)。

`React`的中心思想之一就是一致性，`React`总是一次性的更新完所有的DOM节点，它不会部分的更新。`workInProgree`树就像是一个草稿对于用户来说其是不可见的，所一`React`可以先对所有的组件进行处理（状态更新、执行生命周期函数、操作DOM元素等等）然后一次性的刷新屏幕。

在源代码中你可以看见有很多函数从`Current`树和`workInProgress`树种获取`Fiber`节点。下面展示的是其中一个函数:

```javascript
function updateHostComponent(current, workInProgress, renderExpirationTime) {...}
```

每个`Fiber`节点都保持对备用字段中其他树的对应的引用。`Current`树中的节点指向`workInProgree`树种的节点，反之亦然。

## 副作用 ##

我们可以把React中的组件想象成一个用`state`和`props`来计算和表示UI的函数。剩下来的其他功能比如操作DOM或者执行生命周期函数都可以认为是副作用。在React的[官方文档](https://reactjs.org/docs/hooks-overview.html#%EF%B8%8F-effect-hook)中解释了副作用.

你原来一定在React组件中执行过获取数据、订阅事件、或者手动操作DOM等。我们把这些操作称之为副作用是因为他们将影响其他组件，并且不能再渲染阶段完成。

我们知道大多数的`state`和`props`的改变都会导致副作用产生，由于副作用也是程序工作的一部分，`Fiber`节点是一种方便的机制，可以跟踪除更新之外的副作用。每个`Fiber`节点都可以关联相关的副作用。在`Fiber`节点中用`effectTag`表示。

所以副作用在`Fiber`中表示的意思就是在组件实例完成更新后需要执行的一系列[工作](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js)。对于淡出的DOM类型的元素来说就是增、删、更新。对于class组件来说就是执行一系列的生命周期函数（`componentDidMount,componentDidUpdate`）以及更新refs。也还存在一些其他类型的Fiber节点所对应的的其他副作用。

## 副作用链 ##

React处理组件更新非常的快速，为了达到这种性能要求其应用了一些非常有意思的技术。其中之一就是实现了一个能快速迭代的副作用Fiber节点的线性链表。迭代这样一个链表的速度比迭代树的速度快多了，而且能排除那些没有副作用的节点。

这个链表的主要目的就是标记那些具有DOM更新和其他副作用的节点。这个链表是`finishedWork`树的一个子集并且使用`nextEffect`属性替代`child`属性。在`current`和`workInProgress`树中都用到了副作用链。

[Dan Abramov](https://medium.com/@dan_abramov)对于副作用链表有这样的描述：他将其比作为一颗圣诞树，并且用圣诞灯将所有的副作用节点绑定在一起。为了便于理解，让我们想象一下下面的Fiber树中那些高亮的节点有一些副作用要执行。例如，我们的某个操作导致`c2`被插入DOM，`d2`和`c1`改变某些属性，`c2`执行了一些某些生命周期函数。副作用链表将会把这些节点链接到一起所以在进行迭代的时候能排除其他的节点。

![effct tree](http://ww3.sinaimg.cn/large/006tNc79ly1g5upyxehnnj30d709xmxf.jpg)

你可以很直观的看见这些具有副作用的节点是怎么样被链接起来的。当进行遍历的时候React使用`firstEffect`来指向链表的首个节点。所以上面的链表可以表示为下图这样的结构：

![effect list](http://ww3.sinaimg.cn/large/006tNc79ly1g5uq2tqxchj30js02vdfn.jpg)

从上面的图我们可以看出来，React将会按照从子节点到父节点的顺序执行副作用。

## Fiber树的根元素 ##

每个React应用都有一个或多个DOM元素来作为React组件的容器。在我们的示例中是一个ID为`container`的`div`元素。

```javascript
const domContainer = document.querySelector('#container');
ReactDOM.render(React.createElement(ClickCounter), domContainer);
```

React将会为每一个容器创建一个[fiber root](https://github.com/facebook/react/blob/0dc0ddc1ef5f90fe48b58f1a1ba753757961fc74/packages/react-reconciler/src/ReactFiberRoot.js#L31)对象,你可以通过DOM元素的引用来方位它。

```javascript
const fiberRoot = query('#container')._reactRootContainer._internalRoot
```

这个`fiber root`保存有对fiber树的引用，它存储在`fiber root`的`current`属性中。

```javascript
const hostRootFiberNode = fiberRoot.current
```

这个Fiber树以一个名为`HostRoot`的[特殊类型](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/shared/ReactWorkTags.js#L34)的fiber节点作为起点。
它是在内部创建的，并充当最顶层组件的父级。

[原文链接](https://blog.ag-grid.com/inside-fiber-an-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)
