# 源码探索策略

如何高效地研究框架源代码呢？一般的经验是：

广度优先，摸清脉络；
深度重读，理解过程；
分而治之，各个击破；
合而审之，融会贯通。

首先，采取广度优先的策略，探索框架的组织结构，不必拘泥于特定函数的实现细节，而是搞清楚源代码的功能模块划分以及核心流程。随后，深度优先阅读一遍代码，这次的阅读将聚焦在框架的核心流程实现上，在具体研究某个流程时，可以略过与该流程无关的代码。运用分治法研究源码，一定程度上可以降低复杂度。分治的研究完各个部分的源码之后，还要将所有的流程和模块合并到一起理解，真正做到融会贯通。

以React v0.3.0为例，初窥了basic中的示例之后，我们不难发现两个重要的调用过程：createClass和renderComponent。这两个接口都被定义在了src/core/React.js中，React.js的源码很少，不到30行，如下所示
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
```
通过阅读src/core/React.js文件，就会发现，原来createClass和renderComponent都是其他模块提供的调用，React内部有包括ReactDOM、ReactComponent、ReactCompositeComponent、ReactMount、ReactDefaultInjection等模块；于此同时，我们还发现，React中定义的其他接口包括: autoBind、createComponentRender、isValidComponent等接口。顺着这些接口看下去，就能够逐步厘清React框架中各个源文件的接口信息。

在了解了这些模块的基本内容之后，就可以开始逐个的深度研读接口内部实现。当React中定义的函数都被全部研读一遍之后，基本上就完成了框架源码的学习过程。剩余的工作，就是查漏补缺，融会贯通了。

