# 事件机制初始化

React框架在初始化时，与事件机制相关的处理代码，在src/core/ReactDefaultInjection.js中，在执行inject时，主要做了四件事情。

第一件，初始化事件插件的执行顺序。在src/eventPlugins/DefaultEventPluginOrder.js中，定义了事件插件的执行顺序，它是一个数组。每一次插入新的插件时，都会根据这个插件顺序的数组重新计算生成插件列表。这一过程的核心代码在src/eventPlugin/EventPluginHub.js中，如下所示。
```javascript
_recomputePluginsList: function() {
    var injectPluginByName = function(name, PluginModule) {
      var pluginIndex = injection.EventPluginOrder.indexOf(name);
      throwIf(pluginIndex === -1, ERRORS.DEPENDENCY_ERROR + name);
      if (!injection.plugins[pluginIndex]) {
        injection.plugins[pluginIndex] = PluginModule;
        for (var eventName in PluginModule.abstractEventTypes) {
          var eventType = PluginModule.abstractEventTypes[eventName];
          recordAllRegistrationNames(eventType, PluginModule);
        }
      }
    };
    if (injection.EventPluginOrder) {  // Else, do when plugin order injected
      var injectedPluginsByName = injection.pluginsByName;
      for (var name in injectedPluginsByName) {
        injectPluginByName(name, injectedPluginsByName[name]);
      }
    }
  }
```

第二件，事件插件中心注入了ReactInstanceHandles，该模块用于查询React组件实例。

第三件，事件插件中心插入了两个常用的事件插件，SimpleEventPlugin和EnterLeaveEventPlugin。其中SimpleEventPlugin用于处理各种常见的事件，例如单击、双击、焦点变化、滚动等常见事件；EnterLeaveEventPlugin则是专门处理mouse in/out相关的事件。

第四件，则是在ReactDOM中注入了一个ReactDOMForm的组件。Form作为HTML标签元素，React会创建一个与之对应的原生组件。实际测试一下，会发现，ReactDOM中之前创建的原生的Form原生组件，已经被ReactDOMForm替换掉了，这也是ReactDOM中仅有的一个非NativeComponent。为何又会另行注入一个ReactDOMForm呢？这是因为在IE8里面，表单的onSubmit事件并不会冒泡或者被顶层事件相应捕获，因此React框架新建了一个复合组件，在这个复合组件的componentDidMount回调中，添加了顶层的submit事件处理。
