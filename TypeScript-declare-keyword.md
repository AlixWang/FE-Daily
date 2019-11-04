# TypesScript中的declare关键字 #

## declare关键字的作用 ##

declare关键字的作用是声明一个全局的类型定义或者变量列如：

```typescript

declare jQyery

```

当我们进行了上述的声明过后我们就可以在项目的任意地方使用jQuery这个变量了，即使我们并没有引入jQuery的库声明文件。

## 什么时候使用declare关键字 ##

当某个第三方库没有声明文件而我们又要其不报错的时候

