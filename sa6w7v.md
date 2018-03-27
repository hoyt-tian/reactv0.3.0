# 事件响应

# 事件绑定
React中事件都是代理处理，从jsx中解析出来的回调函数，并不会直接绑定到对应的dom节点，而是会根据事件的类型，存储到CallbackRegistey中定义的listenerBank中。listenerBank中存储了所有的回调函数，依次按照事件类型和容器id作为索引。

# 全局代理
在启用事件监听后，React会给documen注册几乎所有的事件监听处理函数。根据监听事件类型的不同，ReactEvent.TopLevelCallbackCreator会创建不同的TopLevelCallback作为事件监听回调函数，而在执行该回调函数之前，还会再执行NormalizedEventListener中定义的normalizeEvent方法，这个方法将原生的浏览器事件进行规范化处理，处理后的规范化事件对象才会被传递给TopLevelCallbackCreator创建的回调函数。相关代码如下：
```javascript
function trapBubbledEvent(topLevelType, handlerBaseName, onWhat) {
  listen(
    onWhat,
    handlerBaseName,
    ReactEvent.TopLevelCallbackCreator.createTopLevelCallback(topLevelType)
  );
}

function listen(el, handlerBaseName, cb) {
    EventListener.listen(el, handlerBaseName, createNormalizedCallback(cb));
 }

function createNormalizedCallback(cb) {
  return function(unfixedNativeEvent) {
    cb(normalizeEvent(unfixedNativeEvent));
  };
}

function createTopLevelCallback(topLevelType) { 
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

在TopLevelCallback回调时，会根据事件相关的target，查询相应的targetId和dom节点，并调用handleTopLevel进行处理。而在handleTopLevel中，EventPluginHub会根据事件和相关节点的信息，生成若干个abstractEvent组成的队列，随后处理该队列并清空。

EventPluginHub也不会负责处理生成AbstractEvent的细节，它通过调用所有注册的插件来生成AbstractEvent队列。
