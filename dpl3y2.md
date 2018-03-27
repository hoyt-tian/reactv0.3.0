# mixSpecIntoComponent详解

在创建复合组件调用React.createClass时，传递的参数被称为spec（应该是specification的缩写)，spec中可以定义render, componentWillMount等方法，这些方法都会被添加到创建的类上面。

但，并不能直接将spec中的方法添加到新类上就结束，为什么呢？因为有可能spec中定义的方法，与其他Mixin中定义的方法重名了，这个时候不能简单的进行覆盖，而是要不同的方法不同的处理。

ReactCompositeCompoentInterfac中定义了一些特殊方法/属性在进行mixin的处理策略，处理策略有3种，分别是:DEFINE\_ONCE，DEFINE\_MANY和DEFINE\_BASE。

DEFINE\_ONCE：意味着遵守该策略的方法或属性，只能出现一次，不允许多处定义，比如props属性和getInitialState方法，都只允许被定义一次，但被定义的属性或方法既可以出现在Mixin中，也可以出现在spec中。

DEFINE\_MANY：策略允许某属性或者方法被重复定义，比如componentWillMount方法，继承自多个mixin，spec中也可能会定义，这些被定义了的componentWillMount方法都会被执行。

OVERRIDE\_BASE：spec中出现的方法，将直接覆盖ReactCompositeComponent类中的方法

mixSpecIntoComponent函数负责将spec添加到给定的类上，其具体的执行流程，就是遍历spec中的自有元素，查询该属性名是否存在对应的specPolicy。

若ReactCompositeComponentMixin中定义了同名属性或方法，则其对应的策略应当为OVERRIDE\_BASE，否则抛出异常。

若构造函数的原型链上，存在同名的方法或属性，则其对应的specPolicy必须为DEFINE\_MANY，否则抛出异常。

DEFINE\_ONCE和OVERRIDE\_BASE的处理都相对更加简单，只需要在原型对象上添加或者覆盖某个属性或方法即可。而如果多个Mixin中都存在定义，并且spec中也定了，此时应当如何处理呢？这就要用到一个工具函数createChainedFunction。这个函数接受两个回调函数作为参数，当执行createChainedFunction时，回返回一个新函数，这个新函数接受最多5个传参，在新函数被执行时，先前传入的两个回调函数就会依次被执行，其源码如下
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

这样一来，借助createChainedFunction，就可以将Mixin中出现了多次的函数串联起来，替换原来原型对象上定义的同名函数，实现函数的链式调用。

而除了根据spec策略进行差异处理，还有几个属性也需要额外处理，它们是displayName、mixins和props，它们对应的处理函数被定义在了RESERVED\_SPEC\_KEYS对象中。处理displayName这个属性很简单，就是在传入的类上添加一个displayName属性；而如果传入的spec中包含mixins，则递归调用mixSpecIntoComponent，将这些mixin逐一的添加到新的类上；而如果传入的属性是props，因为props本身是被定义在了ReactComponent中，并且其spec策略是只允许定义一次，为何spec或者mixin中还可以出现props呢？因为mixin和spec中出现的props，并不是当作ReactComponent中定义的props来处理，而是会被认为是prosDeclarations，用来进行属性的类型校验。
