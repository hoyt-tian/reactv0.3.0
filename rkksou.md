# 事件插件机制

React中一个规范化之后的事件，并不是由EventPluginHub直接处理的，而是由EventPluginHub按照某种顺序调用插件进行处理，根据规范化的原生事件对象，生成若干个抽象事件对象(AbstractEvent)。
injection.plugins中存放着EventPluginHub中已经注册的所有插件，通过调用这些插件的extractAbstractEvents方法来生成抽象事件对象，所有插件生成的抽象事件对象都会被汇集在一起作为结果返回。
EventPluginHub的设计形式，使得具体插件只需要关注于它感兴趣的事件处理，中间的其他流程都交由EventPluginHub来处理。
在ReactDefaultInjection中，可以看到默认的插件插入代码，其中SimpleEventPlugin用来处理绝大多数的默认浏览器事件的。
```javascript
  EventPluginHub.injection.injectEventPluginsByName({
    'SimpleEventPlugin': SimpleEventPlugin,
    'EnterLeaveEventPlugin': EnterLeaveEventPlugin
  });
```
# ReactEventTopLevelCallback.createTopLevelCallback
这个函数会根据事件类型创建对应的回调函数，回调函数在执行时，会根据当前的event.target，获取到对应的React组件DOM节点和该节点的ID，然后转交给handleTopLevel进行处理。
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
topLevelCallback只处理规范化之后的原生事件对象，规范话的工作交由NormalizedEventListener中的函数进行处理，处理时主要就是将target属性进行了修正。严格模式下，原生事件的target属性是只读的，浏览器可能会报错。
# ReactEvent.handleTopLevel
normalized后的事件会被EventPluginHub处理，更准确的说，是相关的插件进行处理，得到若干个抽象事件对象（abstractEvent），这些抽象事件对象都会被添加到EventPluginHub的事件队列中，然后立即开始处理。
```javascript
function handleTopLevel(
    topLevelType,
    nativeEvent,
    renderedTargetID,
    renderedTarget) {
  var abstractEvents = EventPluginHub.extractAbstractEvents(
    topLevelType,
    nativeEvent,
    renderedTargetID,
    renderedTarget
  );

  // The event queue being processed in the same cycle allows preventDefault.
  EventPluginHub.enqueueAbstractEvents(abstractEvents);
  EventPluginHub.processAbstractEventQueue();
}
```
# EventPluginHub.extractAbstractEvents
每一个插件也会有extractAbstractEvents方法，每一个normalized事件，都会被EventPluginHub中已经注册的插件进行处理以生成AbstractEvent，所有插件生成的AbstractEvent，最后会一并作为函数返回。
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
# SimpleEventPlugin
SimpleEventPlugin负责处理一般的浏览器事件，它会根据顶层事件的类型，生成对应的事件数据，最后根据事件数据和其他信息，生成一个抽象事件对象（AbstractEvent）。
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
# AbstractEvent
抽象事件对象定义在src/event/AbstractEvent.js中，由于事件生成非常频繁，因此它也是一个具备缓冲池功能的类。抽象事件对象记录了事件的类型，目标id，顶层事件类型，native事件对象（normalized后的），事件数据和事件的target，此外还包括该事件要处理的回调函数队列。
```javascript
function AbstractEvent(
    abstractEventType,
    abstractTargetID,  // Allows the abstract target to differ from native.
    originatingTopLevelEventType,
    nativeEvent,
    data) {
  this.type = abstractEventType;
  this.abstractTargetID = abstractTargetID || '';
  this.originatingTopLevelEventType = originatingTopLevelEventType;
  this.nativeEvent = nativeEvent;
  this.data = data;
  // TODO: Deprecate storing target - doesn't always make sense for some types
  this.target = nativeEvent && nativeEvent.target;
  this._dispatchListeners = null;
  this._dispatchIDs = null;

  this.isPropagationStopped = false;
}
```
在抽象事件对象的type属性中，会根据当前事件的阶段（捕获或者冒泡），将对应的事件回调监听返回，将之赋值给\_dispatchListeners。
# 冒泡阶段和捕获阶段
一般来说，事件插件在生成了abstractEvent之后，都会通过EventPropagator将捕获和冒泡阶段的事件回调关联到抽象事件对象上面。
插件中会定义abstractEventTypes，它定义了事件在不同阶段回调函数的标准名称，以SimpleEventPlugin为例，截取其部分事件回调函数名称定义如下：
```javascript
abstractEventTypes: {
    mouseDown: {
      phasedRegistrationNames: {
        bubbled: keyOf({onMouseDown: true}),
        captured: keyOf({onMouseDownCapture: true})
      }
    },
    mouseUp: {
      phasedRegistrationNames: {
        bubbled: keyOf({onMouseUp: true}),
        captured: keyOf({onMouseUpCapture: true})
      }
    },
   //... 
}
```
mounseDown、mouseUp这些键就是事件类型，而phasedRegistrationNames中则固定使用bubbled和captured分别代表冒泡阶段和捕获阶段，onMouseDown、onMouseUpCapture就是在jsx中的事件回调函数标准名称。

这里还用到了一个工具函数——keyOf(src/vendor/keyOf.js)，这个函数非常巧妙。它做的事情非常简单，就是返回一个只包含一个key的对象所包含的key。比如keyOf({onMouseUp:true})，返回值就是onMouseUp。为什么不直接使用字符串，而何必多此一举呢？因为它比字符串更加灵活，如果直接写成字符串，那么编译阶段，该字符串是不能被优化处理的，而以键的形式，编译器就可以用别名来替换它，从而进行优化，而代码中的语义书写却并不受到影响。因此React源代码中，不少字符串值都被尽可能的用keyOf进行了替换。

言归正传，继续回到事件的冒泡、捕获事件回调收集流程，下面是进行两个阶段的绑定代码
```javascript
function accumulateTwoPhaseDispatchesSingle(abstractEvent) {
  if (abstractEvent && abstractEvent.type.phasedRegistrationNames) {
    injection.InstanceHandle.traverseTwoPhase(
      abstractEvent.abstractTargetID,
      accumulateDirectionalDispatches,
      abstractEvent
    );
  }
}
```
traverseTwoPhase函数，其实就是调用了两次traverseParentPath方法，traverseParentPath方法可以指定遍历的起始和结束的节点ID，该方法会根据两个节点之间的关系，决定其遍历的顺序应该向上或者向下。

怎么做的呢？首先，React会去获取start和stop的两个节点最近的公共祖先节点，如果公共祖先节点的就是stop节点，显然是从下往上遍历，否则就从上往下遍历。traverse函数会根据遍历的方向，实际调用paretnID或者nextDescendantID作为查找下一个待遍历节点的函数。

每一个被遍历过的节点，都会执行函数accumulateDirectionalDispatches。该函数就是具体从CallbackRegistry中取出冒泡或者捕获阶段所有关联的回调函数并将其添加到abstractEvent.\_dispatchListeners上。代码如下：
```javascript
function accumulateDirectionalDispatches(domID, upwards, abstractEvent) {
  if (__DEV__) {
    if (!domID) {
      throw new Error('Dispatching id must not be null');
    }
    injection.validate();
  }
  var phase = upwards ? PropagationPhases.bubbled : PropagationPhases.captured;
  var listener = listenerAtPhase(domID, abstractEvent, phase);
  if (listener) {
    abstractEvent._dispatchListeners =
      accumulate(abstractEvent._dispatchListeners, listener);
    abstractEvent._dispatchIDs = accumulate(abstractEvent._dispatchIDs, domID);
  }
}
```
查询关联回调通过listenerAtPhase来查询，通过抽象事件的类型和阶段，就能查询到注册事件回调的标准回调名称，也就是在事件处理插件中abstractEventTypes里面定义的phasedRegistrationNames中的值。所有的回调函数都是存储在CallbackRegistery中的。

CallbackRegister将相同类型的事件回调函数贮存在同一个bank中，再通过组件的id加以区分。在查询时，就只需要标准事件名称，就能找到对应的回调函数。主要的代码如下：
```javascript
function accumulateDirectionalDispatches(domID, upwards, abstractEvent) {
  var phase = upwards ? PropagationPhases.bubbled : PropagationPhases.captured;
  var listener = listenerAtPhase(domID, abstractEvent, phase);
  if (listener) {
    abstractEvent._dispatchListeners =
      accumulate(abstractEvent._dispatchListeners, listener);
    abstractEvent._dispatchIDs = accumulate(abstractEvent._dispatchIDs, domID);
  }
}
```
冒泡、捕获阶段的事件回调函数都会集中在\_dispatchListeners中，等待后续统一执行。
# 事件响应
当EventPluginHub在调用所有插件生成完毕abstractEvent之后，会先将所有的抽象事件对象入队到EventPluginHub的等待队列中，旋即执行processAbstractEventQueue。EventPluginUtils提供了一个工具函数executeDispatchesInOrder来依次执行事件回调。
```javascript
  var abstractEvents = EventPluginHub.extractAbstractEvents(
    topLevelType,
    nativeEvent,
    renderedTargetID,
    renderedTarget
  );

  // The event queue being processed in the same cycle allows preventDefault.
  EventPluginHub.enqueueAbstractEvents(abstractEvents);
  EventPluginHub.processAbstractEventQueue();
```
在processAbstractEventQueue中，，则是通过executeDispatchesAndRelease来实际调度队列中的每一个abstractEvent对象。
```javascript
var executeDispatchesAndRelease = function(abstractEvent) {
  if (abstractEvent) {
    var PluginModule = getPluginModuleForAbstractEvent(abstractEvent);
    var pluginExecuteDispatch = PluginModule && PluginModule.executeDispatch;
    EventPluginUtils.executeDispatchesInOrder(
      abstractEvent,
      pluginExecuteDispatch || EventPluginUtils.executeDispatch
    );
    AbstractEvent.release(abstractEvent);
  }
};
```
事件回调函数的执行，默认由EventPluginUtil中提供的executeDispatch完成，但如果相应的插件模块，并且该插件定义了executeDispatch方法，则优先使用该模块中定义的dispatch方法。EventPluginUtil中定义的executeDispatch定义如下
```javascript
function executeDispatch(abstractEvent, listener, domID) {
  listener(abstractEvent, domID);
}
```
所有抽象事件对象中的回调函数都会被依次执行，全部执行完毕后，抽象事件对象的\_dispatchListeners会被清空。
```javascript
function forEachEventDispatch(abstractEvent, cb) {
  var dispatchListeners = abstractEvent._dispatchListeners;
  var dispatchIDs = abstractEvent._dispatchIDs;
  if (__DEV__) {
    validateEventDispatches(abstractEvent);
  }
  if (Array.isArray(dispatchListeners)) {
    var i;
    for (
      i = 0;
      i < dispatchListeners.length && !abstractEvent.isPropagationStopped;
      i++) {
      // Listeners and IDs are two parallel arrays that are always in sync.
      cb(abstractEvent, dispatchListeners[i], dispatchIDs[i]);
    }
  } else if (dispatchListeners) {
    cb(abstractEvent, dispatchListeners, dispatchIDs);
  }
}
```

