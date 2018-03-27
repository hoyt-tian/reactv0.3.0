# 事件机制

React采用了一种新型的事件代理机制，出于灵活性考虑，同时还提出了一套插件机制，使得事件处理过程非常的灵活，整个事件机制非常庞大，本章将针对事件机制展开详细的描述。

React在ReactEvent的源代码中给出了整个事件机制的关系图，我们先来看看这张图
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
整个事件体系交汇在ReactEvent和EventPluginHub两个节点，所有的DOM事件都会被ReactEvent.js文件中定义的几个方法处理，并呈递给EventPluginHub，EventPluginHub不负责处理具体的事件，它负责调度事件处理流程，具体的事件会由对应的事件插件进行处理。React已经定义好了几个事件插件，理论上讲，我们也可以开发自己的事件插件以扩展React的事件机制。

# ensureListening和listenAtTopLevel

ensureListening方法用来启用React的事件顶层监听。如果尚未开启顶层监听，此时就会调用listenAtTopLevel开启该功能，其代码如下
```javascript
function ensureListening(touchNotMouse, TopLevelCallbackCreator) {
  invariant(
    ExecutionEnvironment.canUseDOM,
    'ensureListening(...): Cannot toggle event listening in a Worker thread. ' +
    'This is likely a bug in the framework. Please report immediately.'
  );
  if (!_isListening) {
    ReactEvent.TopLevelCallbackCreator = TopLevelCallbackCreator;
    listenAtTopLevel(touchNotMouse);
    _isListening = true;
  }
}
```

listenAtTopLevel接受一个参数，该参数表明touch事件是否可用。React默认会将全局代理事件统一注册到document上。当resize或者scroll事件触发后，React会更新当前滚动条的记录值当浏览器触发scroll或者resize事件后，会调用src/core/BrowserEnv.js中的BrowserEnv.refreshAuthoritativeScrollValues()对BrowserEnv的水平、垂直方向滚动偏移进行更新。

随后，React还会通过trapBubbledEvent依次注册下表中的事件
* mouseover
* mousedown
* mouseup
* mousemove
* mouseout
* click
* dbclick
* mousewheel
* keyup
* keypress
* keydown
* change
* DOMCharacterDataModified
* DOMMouseScroll
* scroll，如果document支持就注册在document上，否则注册在window上
* focus/blur或者focusin/focusout

若当前环境支持触摸(touch)事件，还会额外注册4个跟touch相关的监听
* touchstart
* touchend
* touchmove
* touchcancel

在进行事件绑定时，实际的TopLevelCallback是通过传递给ReactEvent的TopLevelCallbackCreator生成的。绑定时尝试查询里目标最近的React实例(通过ReactInstanceHandles.getFirstReactDOM，具体分析后续补充，此处聚焦在事件处理上)，在找到React实例后，查询其ID信息，然后交给ReactEvent.handleTopLevel处理。

# trapBubbledEvent
在listenAtTopLevel中，所有的顶层事件注册都是通过调用trapBubbledEvent完成的，这个函数内部其实也非常简单，它直接调用了NormalizedEventListener中的listen方法完成了事件监听注册，事件回调则通过ReactEventTopLevelCallbackCreator进行创建。
# normalizedEvent
NormalizedEventListener仍然只是做了一层额外封装，真正的事件注册还是交给了EventListener的listen方法实际完成的。NormalizedEventListener封装的目的，是为了将本地的事件对象进行预处理，通过调用normalizedEvent，统一事件对象的target属性，同时处理了文本节点的特殊情况，其代码如下
```javascript
function normalizeEvent(eventParam) {
  var normalized = eventParam || window.event;
  var hasTargetProperty = 'target' in normalized;
  var eventTarget = normalized.target || normalized.srcElement || window;
  // Safari may fire events on text nodes (Node.TEXT_NODE is 3)
  // @see http://www.quirksmode.org/js/events_properties.html
  var textNodeNormalizedTarget =
    (eventTarget.nodeType === 3) ? eventTarget.parentNode : eventTarget;
  if (!hasTargetProperty || normalized.target !== textNodeNormalizedTarget) {
    // Create an object that inherits from the native event so that we can set
    // `target` on it. (It is read-only and setting it throws in strict mode).
    normalized = Object.create(normalized);
    normalized.target = textNodeNormalizedTarget;
  }
  return normalized;
}
```
上述代码在浏览器中执行可能会抛出异常，这是因为严格模式下，normalized对象的target属性是只读的，因此执行到normalized.target = textNodeNormalizedTarget时就会抛出异常。
# EventListener
事件的冒泡和捕获监听，真正的代码是在src/vendor/stubs/EventListener.js中，该对象包含两个方法，listen和capture，提供了跨浏览器的监听支持，代码很简洁。
```javascript
var EventListener = {
  /**
   * Listens to bubbled events on a DOM node.
   *
   * @param {Element} el DOM element to register listener on.
   * @param {string} handlerBaseName 'click'/'mouseover'
   * @param {Function!} cb Callback function
   */
  listen: function(el, handlerBaseName, cb) {
    if (el.addEventListener) {
      el.addEventListener(handlerBaseName, cb, false);
    } else if (el.attachEvent) {
      el.attachEvent('on' + handlerBaseName, cb);
    }
  },

  /**
   * Listens to captured events on a DOM node.
   *
   * @see `EventListener.listen` for params.
   * @throws Exception if addEventListener is not supported.
   */
  capture: function(el, handlerBaseName, cb) {
    if (!el.addEventListener) {
      console.error(
        'You are attempting to use addEventlistener ' +
        'in a browser that does not support it support it.' +
        'This likely means that you will not receive events that ' +
        'your application relies on (such as scroll).');
      return;
    } else {
      el.addEventListener(handlerBaseName, cb, true);
    }
  }
};
```

