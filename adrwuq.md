# 创建自定义组件

在调用React.createClass时，其实就是调用的ReactCompositeComponent.createClass方法，这个方法之前已经介绍过了，这次跳出代码层面，从其实际应用场景来重新看看这个方法。

当我们调用createClass之后，拿到的实际上是一个Wrapper函数，这个wrapper函数在被调用时，会新建一个空的对象，而这个空对象的原型，是ReactCompositeComponentBase，因此它具备了所有ReactCompositeComponentBase的方法和属性；同时，由于空的Constructor类执行了mixSpecIntoComponent，所以在调用createClass时，传入的spec也被合并到了Constructor类中。

随后再执行construct方法，执行构造操作，并返回实例。

通过wrapper函数，保证了新类的调用方式一致性。同时，通过新建空类Constructor，再更新其prototype为ReactCompositeComponentBase实例，既使得Constructor类继承了ReactCompositeComponentBase的属性及方法，也避免了可能对Constructor.prototype的修改影响到其他自定义的组件。
