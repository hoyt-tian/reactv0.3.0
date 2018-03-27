# 创建顶层事件回调

每次执行React.renderComponent时，React框架都会尝试创建顶层事件回调。如果此时尚未创建顶层事件回调，React框架就是调用listenAtTopLevel方法，开启顶层事件监听。

几乎所有的顶层监听都注册在document上，通过trapBubbledEvent依次注册下表中的事件
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
相关代码如下
```javascript
function listenAtTopLevel(touchNotMouse) {
  invariant(
    !_isListening,
    'listenAtTopLevel(...): Cannot setup top-level listener more than once.'
  );
  var mountAt = document;

  registerDocumentScrollListener();
  registerDocumentResizeListener();
  trapBubbledEvent(topLevelTypes.topMouseOver, 'mouseover', mountAt);
  trapBubbledEvent(topLevelTypes.topMouseDown, 'mousedown', mountAt);
  trapBubbledEvent(topLevelTypes.topMouseUp, 'mouseup', mountAt);
  trapBubbledEvent(topLevelTypes.topMouseMove, 'mousemove', mountAt);
  trapBubbledEvent(topLevelTypes.topMouseOut, 'mouseout', mountAt);
  trapBubbledEvent(topLevelTypes.topClick, 'click', mountAt);
  trapBubbledEvent(topLevelTypes.topDoubleClick, 'dblclick', mountAt);
  trapBubbledEvent(topLevelTypes.topMouseWheel, 'mousewheel', mountAt);
  if (touchNotMouse) {
    trapBubbledEvent(topLevelTypes.topTouchStart, 'touchstart', mountAt);
    trapBubbledEvent(topLevelTypes.topTouchEnd, 'touchend', mountAt);
    trapBubbledEvent(topLevelTypes.topTouchMove, 'touchmove', mountAt);
    trapBubbledEvent(topLevelTypes.topTouchCancel, 'touchcancel', mountAt);
  }
  trapBubbledEvent(topLevelTypes.topKeyUp, 'keyup', mountAt);
  trapBubbledEvent(topLevelTypes.topKeyPress, 'keypress', mountAt);
  trapBubbledEvent(topLevelTypes.topKeyDown, 'keydown', mountAt);
  trapBubbledEvent(topLevelTypes.topChange, 'change', mountAt);
  trapBubbledEvent(
    topLevelTypes.topDOMCharacterDataModified,
    'DOMCharacterDataModified',
    mountAt
  );

  // Firefox needs to capture a different mouse scroll event.
  // @see http://www.quirksmode.org/dom/events/tests/scroll.html
  trapBubbledEvent(topLevelTypes.topMouseWheel, 'DOMMouseScroll', mountAt);
  // IE < 9 doesn't support capturing so just trap the bubbled event there.
  if (isEventSupported('scroll', true)) {
    trapCapturedEvent(topLevelTypes.topScroll, 'scroll', mountAt);
  } else {
    trapBubbledEvent(topLevelTypes.topScroll, 'scroll', window);
  }

  if (isEventSupported('focus', true)) {
    trapCapturedEvent(topLevelTypes.topFocus, 'focus', mountAt);
    trapCapturedEvent(topLevelTypes.topBlur, 'blur', mountAt);
  } else if (isEventSupported('focusin')) {
    // IE has `focusin` and `focusout` events which bubble.
    // @see http://www.quirksmode.org/blog/archives/2008/04/delegating_the.html
    trapBubbledEvent(topLevelTypes.topFocus, 'focusin', mountAt);
    trapBubbledEvent(topLevelTypes.topBlur, 'focusout', mountAt);
  }
}
```
而trapBubbledEvent的内容则十分简单
```javascript
function trapBubbledEvent(topLevelType, handlerBaseName, onWhat) {
  listen(
    onWhat,
    handlerBaseName,
    ReactEvent.TopLevelCallbackCreator.createTopLevelCallback(topLevelType)
  );
}
```
Listen函数由src/event/NormalizedEventListener.js提供，这个函数所做的，就是将TopLevelCallbackCreator生成的回调函数外面再包上一层规范化的处理回调，以保证传递给顶层函数回调的事件是跨浏览器兼容的
```javascript
  listen: function(el, handlerBaseName, cb) {
    EventListener.listen(el, handlerBaseName, createNormalizedCallback(cb));
  },
```
createNormalizedCallback其实也不复杂，其实现如下。
```javascript
function normalizeEvent(eventParam) {
  var normalized = eventParam || window.event;
  var hasTargetProperty = 'target' in normalized;
  var eventTarget = normalized.target || normalized.srcElement || window;
  var textNodeNormalizedTarget =
    (eventTarget.nodeType === 3) ? eventTarget.parentNode : eventTarget;
  if (!hasTargetProperty || normalized.target !== textNodeNormalizedTarget) {
    normalized = Object.create(normalized);
    normalized.target = textNodeNormalizedTarget;
  }
  return normalized;
}

function createNormalizedCallback(cb) {
  return function(unfixedNativeEvent) {
    cb(normalizeEvent(unfixedNativeEvent));
  };
}
```

而在NormalizedEventListener中调用的EventListener.listen函数是由src/vendor/stubs/EventListener.js提供的跨浏览器事件绑定功能，一般都是绑定在document上，而事件的相应函数，则是由TopLevelCallbackCreator创建。

ReactEventTopLevelCallback.js中定义了creatTopLevelCallback函数，它以一个函数作为其返回值。这个实际的回调函数，负责根据规范化后的事件，根据事件的target查找到对应的React组件实例，并调用ReatEvent.handleTopLevel进行实际的事件处理。
```javascript
  createTopLevelCallback: function(topLevelType) {
    return function(fixedNativeEvent) {
      if (!_topLevelListenersEnabled) {
        return;
      }
      var renderedTarget = ReactInstanceHandles.getFirstReactDOM(
        fixedNativeEvent.target
      ) || ExecutionEnvironment.global;
      var renderedTargetID = getDOMNodeID(renderedTarget);
      var event = fixedNativeEvent;
      var target = renderedTarget;
      ReactEvent.handleTopLevel(topLevelType, event, renderedTargetID, target);
    };
  }
```

