# 初探事件绑定

在原生组件的章节，我们跳过了事件绑定相关的介绍，现在回过头来看看原生组件的事件绑定是如何实现的。完整的React事件机制很庞大，本章先简单介绍与原生组件相关的事件绑定部分， putListener。

通过跟踪源代码，可以发现，定义在ReactEvent中的putListener，其实源自EventPluginHub，而EventPluginHub也并非真正的源头，真正的源头是CallbackRegister.putListener。而CallbackRegister中的putListener非常的简单：

```javascript
  putListener: function(id, registrationName, listener) {
    var bankForRegistrationName =
      listenerBank[registrationName] || (listenerBank[registrationName] = {});
    bankForRegistrationName[id] = listener;
  }
```

三个传参分别是domId，事件名称，和对应的回调函数。CallbackRegister将所有的回调函数都存储在了其闭包变量listenerBank中。listenerBank按照事件类型、domId将回调函数存储下来。putListener的过程就是将listener按照刚刚介绍的索引方式存储在listenerBank中。

到这里就结束了，React中原生组件的HTML代码段中并不会出现事件绑定的代码，但事件响应却能够正常触发，这颠覆了之前前端代码的事件绑定机制。

那么，React中到底是如何做到这一点的呢？这就说来话长了。React事件机制对开发者透明，但背后非常庞大，在src/core/ReactEvent.js源码中，附有一份字符图，来阐述整个React事件机制的过程，如下：

```javascript
/**
 * Overview of React and the event system:
 *
 *                    .
 * +-------------+    .
 * |    DOM      |    .
 * +-------------+    .                          +-----------+
 *       +            .                +--------+|SimpleEvent|
 *       |            .                |         |Plugin     |
 * +-----|-------+    .                v         +-----------+
 * |     |       |    .     +--------------+                    +------------+
 * |     +------------.---->|EventPluginHub|                    |    Event   |
 * |             |    .     |              |     +-----------+  | Propagators|
 * | ReactEvent  |    .     |              |     |TapEvent   |  |------------|
 * |             |    .     |              |<---+|Plugin     |  |other plugin|
 * |     +------------.---------+          |     +-----------+  |  utilities |
 * |     |       |    .     |   |          |                    +------------+
 * |     |       |    .     +---|----------+
 * |     |       |    .         |       ^        +-----------+
 * |     |       |    .         |       |        |Enter/Leave|
 * +-----| ------+    .         |       +-------+|Plugin     |
 *       |            .         v                +-----------+
 *       +            .      +--------+
 * +-------------+    .      |callback|
 * | application |    .      |registry|
 * |-------------|    .      +--------+
 * |             |    .
 * |             |    .
 * |             |    .
 * |             |    .
 * +-------------+    .
 *                    .
 *    React Core      .  General Purpose Event Plugin System
 */
```

React并没有将事件回调函数直接绑定到对应的dom节点上，而是在document上绑定了全局事件的顶层响应回调(在ReactDefaultInjection.inject()中绑定)，如此一来，除了window相关的事件，其他绝大部分事件都直接由document上注册的顶层回调直接处理。为了统一浏览器差异，React框架会将NativeEvent统一规范化处理，然后将规范化后的事件、事件类型信息、目标节点信息等数据传递给EventPluginHub，由事件插件中心中注册的各种插件负责根据这些基础信息生成对应的AbstractEvent队列，而真正的回调函数，都是绑定在这些AbstractEvent对象上的，最后再由EventPluginHub集中处理AbstractEvent队列。

上述过程提炼了整个事件机制的关键过程，忽略了诸多实现细节。更多讲解将随着进一步深入React框架展开。
