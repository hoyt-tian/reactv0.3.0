# 组件

组件是React中最为重要的概念。组件可以进行如下操作：
* 挂载（mountComponent）。组件可以被挂载到指定的容器中，mountComponent将完成组件的初始化，组件的事件注册也是在挂载时完成。

* 更新属性(receiveProps)。当属性被挂载之后，可以通过setProps或者replaceProps进行更新。

* 卸载(unmountComponent)。


ReactComponent定义在src/core/ReactComponent.js中，组件的构造函数接受键值对对象作为初始属性，其子类包含复合组件(ReactCompositeComponent)和原生组件(ReactNativeComponent)，某种程度上，可以说ReactComponent是组件的抽象类。

# 生命周期

组件的生命周期**只有两种状态**: MOUNTED和UNMOUNTED，其定义如下

```javascript
var ComponentLifeCycle = keyMirror({
  MOUNTED: null,
  UNMOUNTED: null
});
```

这里用到了一个工具方法:keyMirror。这个方法被定义在src/utils/KeyMirror.js文件中，其作用是根据给定的对象，创建枚举对象，源码非常简单易懂。

```
var keyMirror = function(obj) {
  var ret = {};
  var key;

  throwIf(!(obj instanceof Object) || Array.isArray(obj), NOT_OBJECT_ERROR);

  for (key in obj) {
    if (!obj.hasOwnProperty(key)) {
      continue;
    }
    ret[key] = key;
  }
  return ret;
};
```

遍历参数对象，当某个属性不是从原型链上继承而来时，就将它添加到ret的结果中，并以key作为ret中对应值，最后返回ret作为一个枚举对象。就拿ComponentLifeCycle为例，调用keyMirror方法之后，得到的结果就相当于：

```
const ComponentLifeCycle = {
    MOUNTED: "MOUNTED",
    UNMOUNTED: "UNMOUNTED"
}
```

需要注意的是，复合组件中定义了CompositeLifeCycle，它区别于ComponentLifeCyle，不能将两者混淆。

# 依赖注入

出于高内聚、低耦合的考虑，ReactComponent中用到的外部功能依赖，都是可注入的，它们包括负责处理DOM相关操作的的辅助类和负责事务处理相关的辅助类，相关代码如下:

```javascript
ReactComponent = {
  
  setDOMOperations: function(DOMIDOperations) {
    ReactComponent.DOMIDOperations = DOMIDOperations;
  },
  setReactReconcileTransaction: function(ReactReconcileTransaction) {
    ReactComponent.ReactReconcileTransaction = ReactReconcileTransaction;
  },
}
```

通过依赖注入的形式，ReactComponent不必关注DOM操作、事务相关的具体实现，这些功能全部都交由相应的依赖模块专门处理。

# 组件校验

ReactComponent中定义了一个名为isValidComponent的方法，它接受一个对象作为参数，判断该对象是否为一个合法的ReactComponent。判断逻辑很简单，即传入的对象必须包含两个方法：mountComponentIntoNode和receiveProps，相关的判断代码如下：
```javascript
 isValidComponent: function(object) {
    return !!(
      object &&
      typeof object.mountComponentIntoNode === 'function' &&
      typeof object.receiveProps === 'function'
    );
  }
```

源代码实现中用到了一个连续惊叹号取布尔值的技巧，下表给出个不同类型的variable进行!和!!运算的结果。
| variable | !运算 | !!运算 |
| :--- | :--- | :--- |
| null | true | false |
| undefined | true | false |
| new Date() | false | true |
| /\w/i | false | true |
| "some val" | false | true |
| {} | false | true |
| function(){} | false | true |
| [ ] | false | true |



ReactComponent中还定义了一个非常重要的成员——Mixin，下一章我们将会详细介绍它
