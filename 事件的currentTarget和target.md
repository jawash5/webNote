首先 [MDN](https://link.zhihu.com/?target=https%3A//developer.mozilla.org/zh-CN/docs/Web/API/Event/currentTarget) 对其解释如下：

`Event.currentTarget`：Event 接口的只读属性 `currentTarget` 表示的是当事件沿着 DOM 触发时事件的当前目标。它总是指向事件绑定的元素，而` Event.target` 则是事件触发的元素。

`Event.target`：触发事件的对象 (某个DOM元素) 的引用。当事件处理程序在事件的冒泡或捕获阶段被调用时，它与 `event.currentTarget` 不同。



例如：

```html
<ul>
    <li> li 1 </li>
    <li> li 2 </li>
</ul>
```

我们给 ul 添加点击事件 A，给 li 2 添加点击事件 B。

当点击 li 2 时，此时 `currentTarget` 为 ul，target 为 li 2。

当点击 li 1 时，此时 `currentTarget` 为 li 1，target 为 li 1。

即：

`Event.target` 事件触发的源头，这个元素有可能是子元素，有可能是父元素，主要是看执行事件的实际触发者。

`Event.currentTarget` 事件所绑定的元素。

