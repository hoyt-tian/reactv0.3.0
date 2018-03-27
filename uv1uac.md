# 事件插件中心和抽象事件对象

所有的顶层事件回调都汇集到ReactEvent中的handleTopLevel中进行处理，在这里，原生事件及其相关的其他信息会被传递给事件插件中心，事件插件中心将根据不同的事件类型和其他相关信息，调度合适的事件插件来进行处理，最终生成抽象事件对象的队列。

抽象事件对象是React框架定义的一个内部类，用来管理事件相关的基本信息和回调信息，其源码位于src/event/AbstractEvent.js中。事件插件中心执行extractAbstractEvents来生成抽象事件对象的队列，如下所示：
```javascript
var extractAbstractEvents =
  function(topLevelType, nativeEvent, renderedTargetID, renderedTarget) {
    var abstractEvents;
    var plugins = injection.plugins;
    var len = plugins.length;
    for (var i = 0; i < len; i++) {
      // Not every plugin in the ordering may be loaded at runtime.
      var possiblePlugin = plugins[i];
      var extractedAbstractEvents =
        possiblePlugin &&
        possiblePlugin.extractAbstractEvents(
          topLevelType,
          nativeEvent,
          renderedTargetID,
          renderedTarget
        );
      if (extractedAbstractEvents) {
        abstractEvents = accumulate(abstractEvents, extractedAbstractEvents);
      }
    }
    return abstractEvents;
  };
```

插件中心会遍历所有的可用插件，这些插件按照某种特定的顺序进行了排序，每一个插件都存在一个extractAbstractEvents方法，这个方法可能产生若干个抽象事件对象。不妨以SimpleEventPlugin为例，其核心代码如下：
```javascript
extractAbstractEvents:
    function(topLevelType, nativeEvent, renderedTargetID, renderedTarget) {
      var data;
      var abstractEventType =
        SimpleEventPlugin.topLevelTypesToAbstract[topLevelType];
      if (!abstractEventType) {
        return null;
      }
      switch(topLevelType) {
        case topLevelTypes.topMouseWheel:
          data = AbstractEvent.normalizeMouseWheelData(nativeEvent);
          break;
        case topLevelTypes.topScroll:
          data = AbstractEvent.normalizeScrollDataFromTarget(renderedTarget);
          break;
        case topLevelTypes.topClick:
        case topLevelTypes.topDoubleClick:
        case topLevelTypes.topChange:
        case topLevelTypes.topDOMCharacterDataModified:
        case topLevelTypes.topMouseDown:
        case topLevelTypes.topMouseUp:
        case topLevelTypes.topMouseMove:
        case topLevelTypes.topTouchMove:
        case topLevelTypes.topTouchStart:
        case topLevelTypes.topTouchEnd:
          data = AbstractEvent.normalizePointerData(nativeEvent);
          break;
        default:
          data = null;
      }
      var abstractEvent = AbstractEvent.getPooled(
        abstractEventType,
        renderedTargetID,
        topLevelType,
        nativeEvent,
        data
      );
      EventPropagators.accumulateTwoPhaseDispatches(abstractEvent);
      return abstractEvent;
    }
```
SimpleEventPlugin将根据当前触发的事件类型，对原生事件的数据进行进一步处理，例如，如果是鼠标滚轮事件，那么就规范化滚轮相关的数据，如果是点击事件，就规范化点击的相关信息......最后根据这些信息，创建一个抽象事件对象，然后为这个对象添加冒泡和捕获阶段的回调，相关的详细代码可以自行阅读EventPropagators.accumulateTwoPhaseDispatches中的代码，这里不再赘述。

事件处理插件可以返回不止单个抽象事件对象，比如EnterLeaveEventPlugin中，就会返回一组抽象事件对象。accumulate函数既能够处理数组的情况，也能够处理单个对象，将结果合并到最终的abstractEvents中。
