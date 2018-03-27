# React

React对象（src/core/React.js），代码不过32行，如下所示，但通过对该文件的分析，能够推导出整个React框架的整体结构和框架初始化流程，并可以帮助我们按图索骥，找到其他基础类。

```javascript
"use strict";

var ReactCompositeComponent = require('ReactCompositeComponent');
var ReactComponent = require('ReactComponent');
var ReactDOM = require('ReactDOM');
var ReactMount = require('ReactMount');

var ReactDefaultInjection = require('ReactDefaultInjection');

ReactDefaultInjection.inject();

var React = {
  DOM: ReactDOM,
  initializeTouchEvents: function(shouldUseTouch) {
    ReactMount.useTouchEvents = shouldUseTouch;
  },
  autoBind: ReactCompositeComponent.autoBind,
  createClass: ReactCompositeComponent.createClass,
  createComponentRenderer: ReactMount.createComponentRenderer,
  constructAndRenderComponent: ReactMount.constructAndRenderComponent,
  constructAndRenderComponentByID: ReactMount.constructAndRenderComponentByID,
  renderComponent: ReactMount.renderComponent,
  unmountAndReleaseReactRootNode: ReactMount.unmountAndReleaseReactRootNode,
  isValidComponent: ReactComponent.isValidComponent
};

module.exports = React;
```


React对象是典型的单例设计模式，React对象中聚合了React框架常用的方法及属性，多数情况下，这些方法或属性都是源自其他核心对象。

在React单例中创建了别名，例如常用的createClass，它源自ReactCompositeComponent，renderComponent则是源自ReactMount，React.DOM其实就是ReactDOM。

ReactDefaultInjection.inject主要完成了事件相关的初始化工作，后续在介绍事件机制时再细说。
