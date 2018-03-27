# ReactCompositeComponent改动

# 概述

ReactComponent定义了抽象的React对象，其挂载、卸载的主要属性和方法。而ReactCompositeComponent中，还包含了状态更新、属性设置、组件渲染，声明周期声明等。此外还包含了对组件拥有者（refs）的处理方法和父组件属性传递给子组件的工具函数（ReactPropTransferer）等。

ReactCompositeComponent与React对象一样，是一个实例（单例），其源码如下:

```javascript
var ReactCompositeComponent = {
    LifeCycle: CompositeLifeCycle,
    Base: ReactCompositeComponentBase, // ???，没找到任何调用地方
    createClass: function(){},
    autobind: function(){}
}

module.exports = ReactCompositeComponent;
```


# SpecPolicy
SpecPolicy是一个枚举对象，它描述了ReactCompositeComponentInterface中，各个属性是否支持在Mixin多重继承时被重复定义的策略，它包含三种有效值：DEFINE\_ONCE, DEFINE\_MANAY和DEFINE\_BASE。

DEFINE\_ONCE，只会被定义一次，比如props, getInitialState, render。  
DEFINE\_MANY, 可以被重复定义，这些被重复定义的方法都会被链式调用，比如mixins，componentWillMount，componentDidUpdate等。React实现了一套链式调用的机制。  
DEFINE\_BASE, ReactCompositeComponentInterface中定义的该方法会被子类中定义的同名方法覆盖，比如updateComponent。
# ReactCompositeComponentInterface
React中通过这个接口，定义了ReactCompositeComponent包含的方法和这些方法在子类中被重复定义时的行为（SpecPolicy)，代码如下。
```javascript
var ReactCompositeComponentInterface = {
  mixins: SpecPolicy.DEFINE_MANY,
  props: SpecPolicy.DEFINE_ONCE,
  getInitialState: SpecPolicy.DEFINE_ONCE,
  render: SpecPolicy.DEFINE_ONCE,
  componentWillMount: SpecPolicy.DEFINE_MANY,
  componentDidMount: SpecPolicy.DEFINE_MANY,
  componentWillReceiveProps: SpecPolicy.DEFINE_MANY,
  shouldComponentUpdate: SpecPolicy.DEFINE_ONCE,
  componentWillUpdate: SpecPolicy.DEFINE_MANY,
  componentDidUpdate: SpecPolicy.DEFINE_MANY,
  componentWillUnmount: SpecPolicy.DEFINE_MANY,
  updateComponent: SpecPolicy.OVERRIDE_BASE
};
```
像componentWillUpdate等方法，有可能在子类、父类、mixins中都有定义，这些被定义的方法，都应该被调用，如何实现这个需求呢？源代码中给出了一个简单而又高效的实现方式，那就是createChainedFunction。
# createChainedFunction
它是一个工具函数，接受两个函数作为参数，同时返回一个wrapper的函数，在wrapper函数中，按照顺序调用之前传入的两个函数，即完成了2个函数的链式调用，其源码如下:
```javascript
function createChainedFunction(one, two) {
  return function chainedFunction(a, b, c, d, e, tooMany) {
    invariant(
      typeof tooMany === 'undefined',
      'Chained function can only take a maximum of 5 arguments.'
    );
    one.call(this, a, b, c, d, e);
    two.call(this, a, b, c, d, e);
  };
}
```

而如果有3个函数需要链式调用时，只需要先将前两个函数作为参数传给createChainedFunction，并将返回的结果作为其中一个参数，剩下的另一个函数作为第二个参数，再调用一次createChainedFunction即可，非常典型的迭代思想运用。

# mixSpecIntoComponent
类似mixInto方法，不过它处理的不是mixin，是什么呢？是执行React.createClass时传递过去的对象，也就是创建新类的描述信息。React会遍历传递对象的ownProperty，并尝试获取该property在ReactCompositeComponentInterface中定义的SpecPolicy类型。

如果对应的属性/方法在ReactCompositeComponentInterface中已经被定义，并且其SpecPolicy不为OVERRIDE\_BASE，就会抛出异常。

如果该属性存在SpecPolicy，对应的实际值为空或者不是React.autoBind创建的方法，则没有问题，否则将会抛出异常。

如果原型对象中包含了同名方法，此时需要确保SpecPolicy必须为DEFINE\_MANY，否则将抛出异常（OVERRIDE-BASE已经判断过了）。

如果该属性是displayName, mixins或者props，则调用RESERVED-SPEC-KEYS中对应的方法进行特殊处理。否则，如果该属性包含\_\_reactAutoBind，则在原型对象中的reactAutoBindMap也添加同名的处理函数（如果原型对象不存在reactAutoBind就新增该属性）。

如果原型对象也存在同名方法，此时已经确定SpecPolicy是DEFINE\_MANY，因此需要创建链式调用，以保证原型链上定义的方法和传入对象上定义的同名方法都会被调用到。而如果原型对象上并不存在同名函数，此时只要将它添加到原型对象上即可。
# CompositeComponentLifeCycle
不同于ReactComponent中定义的ComponentLifeCycle，CompositeComponentLifeCycle主要用来标志一些进行时态，包括MOUNTING、UNMOUNTING、RECEIVING\_PROPS和RECEIVING\_STATE。
# ReactCompositeComponentMixin
## construct
“构造“函数（因为不是真正的构造函数、但实际执行构造过程，因此打上引号以示区别）中做的事情非常少，首先调用了父类中的同名方法，将实例构造成一个ReactComponent，然后将state、\_pendingState和compositeLifeCycleState置为null。

\_pendingState是一个私有属性，当当前组件尚未完成前一轮的状态更新时，过渡的状态值就会存储在其中。

注意：compositeLifeCycleState区别于ComponentLifeCycle。

## mountComponent
对于组件挂载，同样也先调用ReactComponent.Mixin中的同名方法，完成基础的挂载操作，在ReactCompsiteComponent中，有更多关于挂载的处理。

组件的lifeCycleState和compositeLifeCycleState分别更新为UNMOUNTED和MOUNTING。

如果该类定义了propDeclarations，那么此时会调用\_assertValidProps进行属性类型校验。

如果存在\_reactAutoBindMap，就调用bindAutoBindMethods进行自动绑定。（详细内容后续会讲解）。

随后，如果定义了getInitialState方法，那么state的值就会被赋为getInitialState的返回值，否则为null；pendingState也会赋值为null。

如果定义了componentWillMount，此时就会被立即调用。调用componentWillMount结束后，如果pendingState中有值，就会覆盖当前的state。

而如果定义了componentDidMount，它就会被添加到事务对象的ReactOnDOMReady队列中，等待挂载完成后被调用。这部分涉及到事务机制的内容，会在事务相关章节展开讲解。

接着通过调用\_renderValidatedComponent，得到要被挂载的组件对象，通过调用该对象的mountComponent方法完成挂载。

## unmountComponent
组件卸载流程也是类似，先更新compositeLifeCycleState为UNMOUNTING，随后触发componentWillMount，如果定义了的话。然后通过调用ReactComponent.Mixin中的unmountComponent方法和\_renderdComponent.unmountComponent方法完成卸载，最后清空refs。

## receiveProps
这里也会进行属性类型验证，如果定义了的话。同时会将compositeLifeCycleState更新为RECEIVING\_PROPS，并调用cmponentWillReceiveProps方法，执行完毕后，将compositeLifeCycleState更新为RECEIVING-STATE，并再次调用receivePropsAndState执行真正的更新操作。

## setState
setState的执行类似setProps，先把要更新的部分merge当前的pendingState或state，再调用replace。代码很简单
```javascript
setState: function(partialState) {
    // Merge with `_pendingState` if it exists, otherwise with existing state.
    this.replaceState(merge(this._pendingState || this.state, partialState));
  }
```

## replaceState
进行状态替换时，会先将要更新的状态值保存在pendingState重，再判断当前组件的状态是否可更新。如果组件处在可以执行更新操作的状态时，就会从事务缓冲池中取出一个可用事务，在事务中调用\_receivePropsAndState完成更新操作。

## \_receivePropsAndState
若定义了shouldComponentUpdate方法，会在此时被调用，并且如果返回值为true，就会调用\_performComponentUpdate执行更新操作；否则就只是更新props和state属性。

## \_performComponentUpdate
componentWillUpdate方法若定义了会被立即执行，随后更新props和state属性，并调用updateComponent进行更新。如果还定义了componentDidUpdate方法，会被添加到事务的ReactOnDOMReady的队列中，等待更新完成后被调用。

## updateComponent
该方法实际进行了组件更新，调用\_renderValidateComponent会得到新渲染出来的组件nextComponent，然后比较renderedComponent和nextComponent两者类型是否一致。

若一致，且nextComponent的props中isStatic未定义或不为true，则通过调用renderedComponent对象的receiveProps方法执行更新。

若两者类型不同，则先将renderedComponent执行卸载，然后将nextComponent挂载上去。

## forceUpdate
强制组件执行更新，使用事务执行\_performComponentUpdate。

## \_renderValidatedComponent
执行实际的渲染，这里会调用render方法，以获得要渲染的结果。render方法在创建ReactCompositeComponent时必须定义

## \_assertValidProps
属性类型校验（运行时态），不同于typescript的编译时态校验。

## \_bindAutoBindMethods
自动将当前方法绑定到组件。在createClass时，传入的spec中可能会定义了一些方法，如果该方法需要绑定到组件上执行，就可以使用React.autoBind方法进行绑定，如下所示
```plain
React.createClass({
        handleClick: React.autoBind(function() {
          this.setState({jumping: true});
        }),
        render: function() {
          return <a onClick={this.handleClick}>Jump</a>;
        }
 });
```
autoBind的实际执行代码如下:
```plain
  autoBind: function(method) {
    function unbound() {
      invariant(
        false,
        'React.autoBind(...): Attempted to invoke an auto-bound method that ' +
        'was not correctly defined on the class specification.'
      );
    }
    unbound.__reactAutoBind = method;
    return unbound;
  }
```
可以发现，autoBind方法唯一做的就是将要处理的函数以\_\_reactAutoBind的别名保存了下来。在mixSpecIntoComponent时，如果某个属性包含\_\_reactAutoBind，说明该属性就是被React.autoBind处理过的方法，这个方法就会被保存到\_\_reactAutoBindMap对象中(\_\_reactAutoBindMap则会保存在原型对象上)。

在组件挂载时（ReactCompositeComponent.mountComponent)，如果当前实例包含\_\_reactAutoBindMap，就会调用\_bindAutoBindMethods进行绑定。绑定的过程很简单，就是遍历\_\_reactAutbBindMap中的成员（必须是hasOwnProperty为true的成员），然后调用\_bindAutoBindMethod进行处理。
```javascript
 _bindAutoBindMethods: function() {
    for (var autoBindKey in this.__reactAutoBindMap) {
      if (!this.__reactAutoBindMap.hasOwnProperty(autoBindKey)) {
        continue;
      }
      var method = this.__reactAutoBindMap[autoBindKey];
      this[autoBindKey] = this._bindAutoBindMethod(method);
    }
  }
```
## \_bindAutoBindMethod
该方法返回一个闭包，将当前组件的this指针存放在component闭包变量中，在绑定方法实际调用执行时，就通过call将this指针设定为组件实例。其代码如下
```javascript
  _bindAutoBindMethod: function(method) {
    var component = this;
    var hasWarned = false;
    function autoBound(a, b, c, d, e, tooMany) {
      invariant(
        typeof tooMany === 'undefined',
        'React.autoBind(...): Methods can only take a maximum of 5 arguments.'
      );
      if (component._lifeCycleState === ReactComponent.LifeCycle.MOUNTED) {
        return method.call(component, a, b, c, d, e);
      } else if (!hasWarned) {
        hasWarned = true;
        if (__DEV__) {
          console.warn(
            'React.autoBind(...): Attempted to invoke an auto-bound method ' +
            'on an unmounted instance of `%s`. You either have a memory leak ' +
            'or an event handler that is being run after unmounting.',
            component.constructor.displayName || 'ReactCompositeComponent'
          );
        }
      }
    }
    return autoBound;
  }
```

# ReactCompositeComponentBase
ReactCompositeComponentBase是所有CompositeComponent类的基类，它本身是一个空类，同时，从ReactComponent、ReactOwner、ReactPropTransferer各自的mixin中继承了属性和方法，还包括ReactCompositeComponentMixin。

ReactOwner的mixin主要提供了ref相关的功能，ReactPropTransferer的mixin提供了transferPropsTo方法，用来将props在组件之间进行转移。

## createClass
复合组件是继承自ReactCompositeComponentBase，ReactCompositeComponent是一个对象，整合了复合组件创建相关的资源。其最重要的就是createClass方法，这个方法也同样存在于React对象中。它接受一个对象，React根据该对象的描述信息，创建一个新的ReactCompositeComponent子类，其源码如下:
```javascript
  createClass: function(spec) {
    var Constructor = function() {};
    Constructor.prototype = new ReactCompositeComponentBase();
    Constructor.prototype.constructor = Constructor;
    mixSpecIntoComponent(Constructor, spec);
    invariant(
      Constructor.prototype.render,
      'createClass(...): Class specification must implement a `render` method.'
    );

    var ConvenienceConstructor = function(props, children) {
      var instance = new Constructor();
      instance.construct.apply(instance, arguments);
      return instance;
    };
    ConvenienceConstructor.componentConstructor = Constructor;
    ConvenienceConstructor.originalSpec = spec;
    return ConvenienceConstructor;
  }
```

这段代码，首先定一个空函数Constructor，然后更新了它的prototype，这是一个非常经典的继承技巧。**通过这样的方式，既使得Constructor拥有了ReactCompositeComponentBase这个类的方法，又避免了Constructor在修改prototype时**，**将ReactCompositeComponentBase的内容也修改掉了**。

mixSpecIntoComponent在前文中已经有详细介绍，它把定义在spec中的方法和属性添加到新创建的类中，并且会根据ReactCompositeComponentInterface中定义的属性及方法的策略（SpecPolicy）进行不同的处理。

spec中必须出现render方法，在创建新类型时，React会额外校验这个方法，确保其存在。

为了统一调用，createClass会创建并返回一个wrapper函数，这个wrapper函数接受两个参数：props和children。当wrapper函数实际执行时，会首先创建一个新类型的对象，随后调用construct方法进行构建，最后返回这个实例。
