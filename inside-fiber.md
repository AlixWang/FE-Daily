# 译：深入react-fiber(react新调度算法的深入理解) #

![head-picture](http://ww1.sinaimg.cn/large/006tNc79gy1g5o1la5tiej30j708caa5.jpg)

## 说明 ##

本文是对国外大神对于fiber架构的深入讲解文章的翻译，因为自身水平所限难免出现偏差建议结合文末原文链接进行阅读。

我们都知道React是基于追踪组件状态更新并同步至屏幕上用来构建用户界面的前端库。在react中我们知道这是基于一个名叫reconciliation（调解）的过程，我们调用`setState`方法而框架内部则比较state和props的变化去重绘组件更新UI界面。

React的官方文档对此提供了一个很好地高级概述：React元素、生命周期函数、`render`函数的规则以及运用于子组件的diff算法。从render方法返回的不可变React元素树通常称为“虚拟DOM”。这个术语很早就帮助人们解释了React，但是同样引入了令人迷惑的事情，而且并没有在react的官方文档中有任何提及。在本文中我将坚称其为react元素树。
除了React元素树，框架内部也总是有内部示例树（比如组件、DOM节点等）用来保持内部状态。从版本16开始，React推出了该内部实例树的新实现以及管理代号为Fiber的算法。为了了解Fiber架构带来的优势，[了解React在Fiber中使用链表的方式和原因](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)。

> 这篇文章如果没有[Dan Abramov](https://medium.com/@dan_abramov)的帮助的话一定会花费更多的时间，而且没有如此全面。

这是一系列教你理解React内部原理的文章的第一篇。在本文中，我想提供与算法相关的重要概念和数据结构的深入概述，一旦我们了解了足够的背景知识，我们将会探索用于递归和解析fiber树的算法和主要函数。此系列的下一篇文章将演示React如何使用该算法执行初始渲染和处理state以及props更新。从那里我们将继续讨论调度程序的细节，子调解(reconciliation)进程，以及建立Effect list的机制。

## 继续 ##

我将带你一起了解一些React的高级知识🧙‍。我鼓励你阅读它以了解Concurrent React内部工作背后的魔力。而且这一系列的文章也能作为向React开源项目贡献的指南。我非常相信逆向工程，因此本文中会有很多React16.6.0中的源代码链接。

本文包含很多内容，所以如果你不能马上理解的话不要感到有压力。这些内容值得你花时间去了解。注意如果你只是为了能熟练使用React，那么你没必要了解这些内容。这篇文章的目标是为了帮助你深入理解React内部的运行原理。

## 背景 ##

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

你可以在[此处](https://stackblitz.com/edit/react-t4rdmh)进行在线查看，正如你所见，这只是一个简单包含`span`和`button`元素的组件。单击该按钮后，组件的状态将在`handleClick`函数内更新。然后这会导致span元素的文本更新。

这里有几种React会在(reconciliation)调解过程中会执行的操作。例如，以下是React在我们的示例中的第一次渲染和状态更新之后执行的高级操作：

+ 在`ClickCounter`组件`state`中更新`count`属性
+ 检索和比较`ClickCounter`组件的`Children`和`Props`
+ 更新`span`元素的`props`

在(reconciliation)调解的过程中还有一些其他操作被执行例如调用生命周期函数及更新`ref`，所有这些活动在`Fiber`架构中统称为“`work`”。`work`的类型通常都取决于React组件的类型。列如，对于Class组件，React需要去创建实例，但对于函数式组件来说却并不会。正如你所知，在React里面有多种组件类型。例如：class和函数式组件，原生组件(DOM)、`portals`等。React组件类型是由`createElement`函数的第一个参数决定的。这个函数通常被用于在`render`方法中创建组件元素。

在我们探索`Fiber`架构之前，先让我们熟悉一下React内部的数据结构。

## 从React Elements 到Fiber 节点 ##

React中的每个组件都有一个UI表示，我们可以调用一个视图或一个从render方法返回的模板。下面是我们`ClickCounter`组件的模板代码。

```javascript
<button key="1" onClick={this.onClick}>Update counter</button>
<span key="2">{this.state.count}</span>
```

## React 元素 ##

当模板通过JSX的解释器后，你将会得到一系列的React元素。这才是react组件的`render`方法真正返回的格式，而不是你看见的`HTML`模板。如果不使用JSX的话，那么我们的组件可以写成这样。

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

我们可以看到React为这些对象添加了`$$typeof`属性将他们标识为唯一的React元素。用`type`、`key`和`props`描述此对象。而他们的值就是你通过`React.createElement`函数传过来的参数值。请注意React如何将文本内容表示为`span`和按钮节点的子项。和`click`的监听函数是如何作为`button`元素的`props`。React其他方面比如`ref`字段超出了本文的讨论范围所以在此不做赘述。

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

在调解(reconciliation)期间React组件的`render`方法返回的数据会被合并进入fiber节点树中。与React组件不一样，fiber不会在每个渲染周期内重新创建。他们是包含组件状态和DOM节点的可变数据结构。

前文说过React根据组件的不同类型，框架会执行不同的动作。在我们的简单示例中，对于`class`组件`ClickCounter`来说会调用生命周期函数和`render`方法，而对于`span`原生类型(DOM Node)的组件来说其会响应DOM的改变。因此，每个React元素都会转换为相应类型的Fibre节点，用于描述需要完成的工作。

您可以将Fiber视为代表某些工作要做的数据结构，换句话说，就是一个工作单元，Fiber架构也提供了简便的方法去追踪、调度、暂停和取消工作进程。

当一个React组件第一次转变为Fiber节点的时候，React使用元素中的数据在createFiberFromTypeAndProps函数中创建Fiber。在随后的更新中，React重用Fiber节点，并使用来自相应React元素的数据更新必要的属性。如果不再从render方法返回相应的React元素，React可能还需要根据key标记移动或删除相应的节点。

根据[**ChildReconciler**](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactChildFiber.js#L239)函数查看`React`为现有Fieber节点执行的所有活动和函数。

因为React为每个元素创建了一个Fiber节点，因为这些这些元素组成了一个树，所以我们得到了一个Fiber节点组成的树。在`ClickCounter`示例中fiber树形如下图：

![image-1.png](https://tupp.xyz/2019/08/09/15653219085d4ceab46c3c7.png)

所有的Fiber Nodes 都通过 `child sibling return`链接成一个链表。可以通过阅读我的这篇文章[The how and why on React’s usage of linked list in Fiber](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)来了解为什么要使用链表。

## Current 树和workinprogress树 ##

在首次渲染过后，React最终得到了一个Fiber树，它反映当前界面UI的状态。这种Fiber树通常被简称为Current树。当React开始进行状态更新时，产生的树叫做workInProgress树，反映了将来要被渲染到屏幕上的React应用状态。

所有的工作都在`workInProgress`的`Fiber`树上执行，当`React`遍历`Current`树时，对于每个现有Fiber节点，它会创建一个构成`workInProgress`树的备用节点。这个节点是通过`Render`方法返回的数据创建的。当更新被执行且所有相关的工作的完成的时候，React会产生一个备用树(`workInProgress`树)去刷新当前屏幕，而当`workInProgress`树的状态更新到屏幕上之后，它就成了`Current`树。(可以看出`Current`树会被`workInPorgress`树取代)。

`React`的中心思想之一就是一致性，`React`总是一次性的更新完所有的DOM节点，它不会部分的更新。`workInProgree`树就像是一个草稿对于用户来说其是不可见的，所以`React`可以先对所有的组件进行处理（状态更新、执行生命周期函数、操作DOM元素等等）然后一次性的刷新屏幕。

在源代码中你可以看见有很多函数会从`Current`树和`workInProgress`树中获取`Fiber`节点。下面展示的是其中一个函数:

```javascript
function updateHostComponent(current, workInProgress, renderExpirationTime) {...}
```

每个`Fiber`节点都通过`alternate`字段保持对其他树中对应节点的引用。`Current`树中的节点指向`workInProgree`树中的节点，反之亦然。

## 副作用 ##

我们可以把React中的组件想象成一个用`state`和`props`来计算和表示UI的函数。剩下来的其他功能比如操作DOM或者执行生命周期函数都可以认为是副作用。在React的[官方文档](https://reactjs.org/docs/hooks-overview.html#%EF%B8%8F-effect-hook)中解释了副作用是怎么一回事.

你一定在React组件中执行过获取数据、订阅事件、或者手动操作DOM等操作。我们把这些操作称之为副作用是因为他们将影响其他组件，并且不能再渲染阶段完成。

我们知道大多数的`state`和`props`的改变都会导致副作用产生，由于副作用也是程序工作的一部分，`Fiber`节点是一种方便的机制，可以跟踪除更新之外的副作用。每个`Fiber`节点都可以关联相关的副作用。在`Fiber`节点中用`effectTag`表示副作用的类型。

副作用在`Fiber`中表示的意思就是在组件实例完成更新后需要执行的一系列[工作](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js)。对于DOM类型的元素来说就是增、删、更新。对于class组件来说就是执行一系列的生命周期函数（`componentDidMount,componentDidUpdate`）以及更新refs。当然也还存在一些其他类型的Fiber节点所对应的的其他副作用。

## 副作用链 ##

React处理组件更新非常的快速，为了达到这种性能要求其应用了一些非常有意思的技术。其中之一就是实现了一个能快速迭代的副作用Fiber节点的线性链表。迭代这样一个链表的速度比迭代树的速度快多了，而且能排除那些没有副作用的节点。

这个链表的主要目的就是标记那些具有DOM更新和其他副作用的节点。这个链表是`finishedWork`树的一个子集并且使用`nextEffect`属性替代`child`属性。在`current`和`workInProgress`树中都用到了副作用链。

[Dan Abramov](https://medium.com/@dan_abramov)对于副作用链表有这样的描述：他将其比作为一颗圣诞树，并且用圣诞灯将所有的副作用节点绑定在一起。为了便于理解，让我们想象一下下面的Fiber树中那些高亮的节点表示有一些副作用要执行。例如，我们的某个操作导致`c2`被插入DOM，`d2`和`c1`改变某些属性，`c2`执行了一些某些生命周期函数。副作用链表将会把这些节点链接到一起所以在进行迭代的时候能排除其他的节点。

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
它是在内部创建的，并充当最顶层组件的父级。`HostRoot Fiber`节点通过`stateNode`属性指向回`FiberRoot`：

```javascript
fiberRoot.current.stateNode === fiberRoot; // true
```

你可以通过`fiber root`最顶部的`HostFiber`节点来探索整个`fiber`树。或者你可以通过一个像下面这样的组件实例来查看其本身的`fiber`节点。

```javascript
compInstance._reactInternalFiber
```

## Fiber节点的结构 ##

让我们看看`ClickCounter`组件所创建的`Fiber`节点长啥样。

```javascript
{
    stateNode: new ClickCounter,
    type: ClickCounter,
    alternate: null,
    key: null,
    updateQueue: null,
    memoizedState: {count: 0},
    pendingProps: {},
    memoizedProps: {},
    tag: 1,
    effectTag: 0,
    nextEffect: null
}
```

`span`元素

```javascript
{
    stateNode: new HTMLSpanElement,
    type: "span",
    alternate: null,
    key: "2",
    updateQueue: null,
    memoizedState: null,
    pendingProps: {children: 0},
    memoizedProps: {children: 0},
    tag: 5,
    effectTag: 0,
    nextEffect: null
}
```

`Fiber`节点中有许多的字段，我在前面的章节中已经介绍了`alternate effectTag nextEffect`分别有什么作用，接下来我将其他字段都有啥作用。

### sateNode ###

保存对组件的类实例，DOM节点或与`Fiber`节点关联的其他类型React元素的引用。总的来说，我们可以说这个属性是用来将元素状态和`Fiber`节点进行关联。

### type ###

定义与此`Fiber`节点相关联的函数组件或者class组件，对于class组件来说指向其构造函数，对于DOM元素来说指向其`HTML`标签。我通常根据这个字段来判断当前`Fiber`组件与什么类型的React元素相关。

### tag ###

定义了`Fiber`节点的[类型](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/shared/ReactWorkTags.js),其用于在调解（reconciliation）算法中决定何种行为将被执行。在前面我们提到过，执行何种行为取决于React元素的类型。函数[createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414)将`React`元素映射为对应的`Fiber`类型。在我们的例子中，`ClickCounter`的`tag`属性的值为1表示是一个`ClassComponent`对于`span`元素来说其`tag`属性为5表示其是一个`HostComponent`。

### updateQueue ###

状态更新，回调和DOM更新的队列。

### memoizedState ###

对于`fiber`来说`state`用于创建屏幕输出，处理更新时，它会反映当前在屏幕上呈现的状态。

### memoizedProps ###

对于`fiber`来说`props`用于创建前一次渲染。

### pendingProps ###

已从React元素中的新数据更新并且需要应用于子组件或DOM元素。

### key ###

具有一组子项的唯一标识符可帮助React确定哪些项已更改，已添加或从列表中删除。它与[此处](https://reactjs.org/docs/lists-and-keys.html#keys)描述的React的“列表和键”功能有关。

你可以在[这里](https://github.com/facebook/react/blob/6e4f7c788603dac7fccd227a4852c110b072fe16/packages/react-reconciler/src/ReactFiber.js#L78)找到`fiber node`的所有结构。在上面我省略了许多其他的字段的说明。比如，我没有列出`child,sibling,return`这些指针属性，这些属性构成了`fiber`的树状结构。在我的[这](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)篇文章中有详细的介绍。以及特定作用于`Scheduler`阶段的`expirationTime`，`childExpirationTime`和`mode`等字段。

## General algorithm ##

React的渲染更新主要分为两个阶段：`render`和`commit`。

在第一个`render`阶段`React`通过`setState`和`React.render`将更新应用于组件并确定需要在UI中更新的内容。如果是首次渲染，React会通过`render`方法为每一个元素返回新的`fiber`节点。在接下来的更新中，`fibers`将会对那些已经存在的节点进行更新和重用。这一阶段执行的结果就是返回一个对副作用进行标记的`fiber`节点树。这些副作用代表了在接下来的`commit`阶段需要做的工作。在`commit`阶段，React得到了一个标记有副作用的fiber树并将其应用于组件实例。然后遍历`effect list`并执行DOM更新和用户可见的其他更改。

重要的是要理解`render`阶段的工作可以异步执行的。React可以根据可用时间处理一个或多个`fiber`节点，在可用时间内一项工作如果没有完成那么React将暂存此项工作并转移到下一项工作，下一项工作完成之后再回到停止的地方继续。但有时候，它可能需要丢弃前一项暂存的部分并头开始。由于在此阶段执行的工作不会导致任何用户可见的更改（如DOM更新），因此可以使这些暂停成为可能。相反，`commit`阶段始终是同步的。这是因为在此阶段执行的工作导致用户可见的变化，例如:DOM更新。所以React需要一次完成所有的操作。

调用生命周期方法是React执行的一种`work`。其中一些会在`render`阶段执行，另一些会在`commit`阶段执行。下面是在第一个`render`阶段时调用的生命周期列表：

+ [UNSAFE_]componentWillMount (deprecated)
+ [UNSAFE_]componentWillReceiveProps (deprecated)
+ getDerivedStateFromProps
+ shouldComponentUpdate
+ [UNSAFE_]componentWillUpdate (deprecated)
+ render

如您所见，在`render`阶段执行的一些遗留生命周期方法在版本16.3中标记为UNSAFE。它们现在在文档中称为遗留生命周期。它们将在未来的16.x版本中弃用，将在17.0中删除。您可以在[此处](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html)详细了解这些更改以及建议的迁移路径。

你是否对为什么是这样子的原因感到好奇？

我们知道React在`render`阶段不会执行像DOM更新这种副作用行为，React可以为异步组件处理异步的更新（甚至可能在多个线程中执行）。然而，标有UNSAFE的生命周期方法经常被开发者误用。开发人员倾向于将带有副作用的代码放在这些方法中，这可能会导致新的异步呈现方法出现问题。虽然带有UNSAFE前缀的生命周期方法在将来会被删除，但它们仍然可能在即将出现的并发模式（可选的）中引起问题。

下面是一些会在`commit`阶段执行的生命周期函数：

+ getSnapshotBeforeUpdate
+ componentDidMount
+ componentDidUpdate
+ componentWillUnmount

因为这些生命周期函数是在同步的`commit`阶段执行的，他们可以执行具有副作用的代码，并且能访问到DOM。

现在我们对于`fiber`树的大致流程和任务执行有了大致了解，让我们继续深入看看。

## Render phase ##

调解(reconciliation)算法总是通过在最顶级的`HostRoot`fiber节点上调用[renderRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1132)函数作为开始。但是React并不是每个节点都会遍历，对于那些已经执行过的`fiber`节点会直接跳过直到找到没有完成的节点。例如：你在一个底层组件中调用了`setState`，那个么React会从最顶级的组件开始找起，但是会忽略掉那些无关的组件，直到找到调用的组件。

## 工作循环的主要步骤 ##

所有的`fiber`节点都在[工作循环](https://github.com/facebook/react/blob/f765f022534958bcf49120bf23bc1aa665e8f651/packages/react-reconciler/src/ReactFiberScheduler.js#L1136)中进行处理。下面是工作循环的同步代码的实现。

```javascript
function workLoop(isYieldy) {
  if (!isYieldy) {
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {...}
}
```

在上面的代码中`nextUnitOfWork`取得了`workInProgress`树中具有未完成工作的`fiber`节点的引用。当React对`fiber`树进行遍历时，会用`nextUnitOfWork`标记从而知道哪些节点有未完成的工作。处理当前Fiber节点后，变量将包含对树中下一个Fiber节点的引用，如果没有的话就是`null`。之后，React退出工作循环并准备提交更改。

有4个主要函数用于遍历树并启动或完成工作：

+ [performUnitOfWork](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1056)
+ [beginWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489)
+ [completeUnitOfWork](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L879)
+ [completeWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberCompleteWork.js#L532)

根据下面的遍历树动画，可以搞清楚内部的工作原理。我已经在演示中使用了这些函数的简化实现。每个函数都对一个Fiber节点进行了处理，当相关函数运行结束时，您可以看到当前活动的Fiber节点发生了变化。您可以在视频中清楚地看到算法如何从一个分支转到另一个分支。它首先完成子节点工作，然后转移到父级元素上。

![animation](http://ww4.sinaimg.cn/large/006tNc79gy1g5vumowonqg30lo0botn4.gif)

注意，直线连接表示节点为兄弟关系，折线连接表示父子关系，例如b1没有子节点，而b2以一个子节点c1。

[这是](https://vimeo.com/302222454)视频的链接，您可以在其中暂停播放并检查当前节点和功能状态。从概念上讲，您可以将“开始”视为“进入”某个组件，并将“完成”视为“离开”它。您还可以点击[此处](https://stackblitz.com/edit/js-ntqfil?file=index.js)的模拟代码实现，我将解释这些函数的作用。

让我们从前两个函数`performUnitOfWork`和`beginWork`开始：

```javascript
function performUnitOfWork(workInProgress) {
    let next = beginWork(workInProgress);
    if (next === null) {
        next = completeUnitOfWork(workInProgress);
    }
    return next;
}

function beginWork(workInProgress) {
    console.log('work performed for ' + workInProgress.name);
    return workInProgress.child;
}
```

函数`performUnitOfWork`接受`workInProgress`fiber树，并调用`beginWork`函数。这个函数将会对所有需要执行任务的`fiber`节点启动相应任务。出于演示的目的，我们只需记录`fiber`的名称即可表示已完成工作。函数`beginWork`始终返回指向要在循环中处理的下一个子节点的指针或`null`。

如果有子节点，它将被分配给`workLoop`函数中的变量`nextUnitOfWork`。如果没有子节点，`React`知道当前节点下的分支已经遍历结束，所以当前节点遍历完成。**一旦当前节点遍历完成，它将需要为兄弟节点执行`workLoop`函数并在此之后回溯到父节点。**这是在`completeUnitOfWork`函数中完成的：

```javascript
function completeUnitOfWork(workInProgress) {
    while (true) {
        let returnFiber = workInProgress.return;
        let siblingFiber = workInProgress.sibling;

        nextUnitOfWork = completeWork(workInProgress);

        if (siblingFiber !== null) {
            // If there is a sibling, return it
            // to perform work for this sibling
            return siblingFiber;
        } else if (returnFiber !== null) {
            // If there's no more work in this returnFiber,
            // continue the loop to complete the parent.
            workInProgress = returnFiber;
            continue;
        } else {
            // We've reached the root.
            return null;
        }
    }
}

function completeWork(workInProgress) {
    console.log('work completed for ' + workInProgress.name);
    return null;
}
```

你可以看到函数的主体是一个大的`while`循环。React在`workInProgree`节点没有子节点的时候调用这个函数。当结束当前节点的工作后，他会检测当前节点是否有兄弟节点。如果找到，React退出该函数并返回指向兄弟的指针。它将被指向给`nextUnitOfWork`变量，React将从这个兄弟开始执行分支的工作。重要的是要理解，在此时，React只完成了前面兄弟姐妹的工作。它尚未完成父节点的工作。只有在完成以子节点开始的所有分支后，才能完成父节点和回溯的工作。

从实现中可以看出，`performUnitOfWork`和`completeUnitOfWork`主要用于迭代目的，而主要功能则在`beginWork`和`completeWork`函数中进行。在本系列的以下文章中，我们将了解当`React`调用入`beginWork`和`completeWork`函数时`ClickCounter`组件和`span`节点会发生什么。

## Commit 阶段 ##

本执行阶段以函数[completeRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L2306)开始。这是React更新DOM并调用生命周期方法的地方。

当React执行到这个阶段的时候，其已经生成了两棵`fiber`树（`current`和`workInProgress`）以及副作用链。第一个树（current tree）代表的是当前屏幕所显示的UI状态。替代树(workInProgress tree)在`render`阶段生成。它在源代码中称为`finishedWork`或`workInProgress`，表示将来在屏幕上更新的状态。替代树通过`child`和`sibling`指针与当前树类似地链接。

然后，有一个副作用列表 - 其是`finishedWorktree`的节点子集然后通过`nextEffect`指针相互链接形成一个链表。副作用链是在`render`阶段产生的。副作用链的主要作用就是确定需要插入，更新或删除哪些节点，以及哪些组件需要调用其生命周期方法。**而这正是在提交阶段迭代的节点集。**

为了调试需要，`current`树能够通过`fiber root`的`current`属性访问。`finishedWork`树能够通过`current`树中的`HostFiber`节点的`alternate`属性访问到。

在`commit`阶段运行的主要功能是[commitRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L523)。其主要功能如下：

+ 在使用快照效果标记的节点上调用`getSnapshotBeforeUpdate`生命周期方法
+ 在使用`Deletion`效果标记的节点上调用`componentWillUnmount`生命周期方法
+ 执行所有DOM插入，更新和删除
+ 将`finishedWork`树设置为`current`树
+ 在使用`Placement`效果标记的节点上调用`componentDidMount`生命周期方法
+ 在使用`Update`效果标记的节点上调用`componentDidUpdate`生命周期方法

在调用预处理方法`getSnapshotBeforeUpdate`后，`React`会将`fiber`树上的所有副作用进行提交。这里主要分为两步。第一步会执行所有DOM的插入、更新、删除和`ref`卸载。然后`React`将`finishWork`树赋值给`FiberRoot`将`workInProgress`树标记为`current`树。这些都是在`commit`阶段的第一步完成的，所以在`componentWillUnmonut`的生命周期方法执行的时侯`current`树还没有被`workInProgress`树替换，但是第二步开始之前，比如`componentDidMount/Update`这些生命周期函数执行的时候，`current`树就已经被`workInProgress`树替换了。在第二步的时候React会执行剩下的所有生命周期函数和ref的回调函数。因为整个fiber树中的添加、更新和删除行为都被调用过了，所以这些方法能作为单独的传递执行。（这句话没明白啥意思。。。。。）

下面是这些函数执行步骤的代码描述：

```javascript
function commitRoot(root, finishedWork) {
    commitBeforeMutationLifecycles()
    commitAllHostEffects();
    root.current = finishedWork;
    commitAllLifeCycles();
}
```

这些子函数中的每一个都实现了一个循环，该循环遍历副作用链表并检查副作用的类型。当它找到与函数目的相关的副作用时，就会执行相关的副作用。

## Pre-mutation lifecycle methods ##

例如，这是在副作用树上迭代并检查节点是否具有`SnapsShot`效果的代码：

```javascript
function commitBeforeMutationLifecycles() {
    while (nextEffect !== null) {
        const effectTag = nextEffect.effectTag;
        if (effectTag & Snapshot) {
            const current = nextEffect.alternate;
            commitBeforeMutationLifeCycles(current, nextEffect);
        }
        nextEffect = nextEffect.nextEffect;
    }
}
```

对于`class`组件，此副作用意味着调用`getSnapshotBeforeUpdatelifecycle`方法。

## DOM updates ##

[commitAllHostEffects](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L376)是`React`执行`DOM`更新的函数。该函数基本上定义了需要为节点完成的操作类型并执行它：

```javascript
function commitAllHostEffects() {
    switch (primaryEffectTag) {
        case Placement: {
            commitPlacement(nextEffect);
            ...
        }
        case PlacementAndUpdate: {
            commitPlacement(nextEffect);
            commitWork(current, nextEffect);
            ...
        }
        case Update: {
            commitWork(current, nextEffect);
            ...
        }
        case Deletion: {
            commitDeletion(nextEffect);
            ...
        }
    }
}
```

有趣的是，`React`调用`componentWillUnmount`方法作为`commitDeletion`函数中删除过程的一部分。

## Post-mutation lifecycle methods ##

[commitAllLifecycles](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L465)是`React`调用所有剩余生命周期方法`componentDidUpdate`和`componentDidMount`的函数。

这就是这篇文章的所有内容了，如果有疑问可以去原文下面进行提问，然后大家可以去看看作者的另一篇文章[In-depth explanation of state and props update in React.](https://medium.com/react-in-depth/in-depth-explanation-of-state-and-props-update-in-react-51ab94563311)

[原文链接](https://blog.ag-grid.com/inside-fiber-an-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)
