# 框架初始化

# 初始化入口
在React对象被定义之前，还存在这两行重要的代码
```plain
var ReactDefaultInjection = require('ReactDefaultInjection');

ReactDefaultInjection.inject();
```

ReactDefaultInjection.inject方法主要是为了处理事件相关的预处理工作。由于涉及到事件及其插件机制，并且涉及到部分核心类，暂时只简单介绍一下，下文会详细展开叙述。

inejct方法主要做了4件事情，分别是

设置插件调用顺序
设置React实例处理
注入默认事件处理插件
将form表单从ReactNativeComponent替换为ReactCompositeComponent

DefaultEventPluginOrder定义在src/eventPlugins/DefaultEventPluginOrder.js中，它是一个数组，默认按照"ResponderEventPlugin","SimpleEventPlugin","TapEventPlugin","EnterLeaveEventPlugin","AnalyticsEventPlugin"的顺序执行事件处理插件，需要说明的是，这里设置的是执行顺序，而不是注入插件，真正的插件注入是在EventPluginHub.inject中调用injectEventPluginByName实现的。

在DefaultEventPluginOrder数组定义时，并没有直接使用字符串，而是使用了一个工具函数，keyOf(oneKeyObj)。keyOf函数定义在src/vendor/core/keyOf.js中，阅读其源代码可以发现，这个函数就是返回了oneKeyObj的第一个ownProperty。该函数的注释解释了为什么要使用这种方式取代字符串常量的原因：这样做可以允许对key进行压缩，同时不影响程序的正确性和源代码的可读性，而若使用字符串常量，在压缩混淆阶段，并不会被动态替换。

在插件注入完成之后，React会根据设置的EventPluginOrder重新计算插件调用的顺序，通过调用\_recomputePluginsList完成更新。插件注入完成后，会被添加到EventPlugin.injection.pluginsByName上，而重新计算得出的顺序则会保存在plugins数组中。
