# 深入react-fiber-(react新调度算法的深入理解) #

![head-picture](http://ww1.sinaimg.cn/large/006tNc79gy1g5o1la5tiej30j708caa5.jpg)

我们都知道React是基于追踪组件状态更新并同步至屏幕上用来构建用户界面的前端库。在react中我们知道这是基于一个名叫reconciliation（调解）的过程，我们调用`setState`方法而框架内部则比较state和props的变化去重绘组件更新UI界面。

React的官方文档对此提供了一个很好地高级概述：React元素、生命周期函数、`render`函数的规则以及运用于子组件的diff算法。从render方法返回的不可变React元素树通常称为“虚拟DOM”。这个术语很早就帮助人们解释了React，但是同样引入了令人迷惑的事情，而且并没有在react的官方文档中有任何提及。在本文中我将坚称其为react元素树。
除了React元素树，框架内部也总是有内部示例树（比如组件、DOM节点等）用来保持内部状态。从版本16开始，React推出了该内部实例树的新实现以及管理代号为Fiber的算法。为了了解Fiber架构带来的优势，[了解React在Fiber中使用链表的方式和原因](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)。

[原文链接](https://blog.ag-grid.com/inside-fiber-an-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)
