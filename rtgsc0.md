# 组件挂载渲染

组件的渲染通过renderComponent完成，背后涉及到了一个相对复杂的流程。

在执行渲染时，React会先查看该实例是否有已经渲染过的节点存在。如果存在，那么应该执行更新流程，否则，执行挂载流程。

挂载流程稍微简单一些，我们先来看看挂载的过程。首先，新的container会调用ReactMount.registerContainer，以便将container和的reactRootID和container的映射关系存储在containersByReactRoot中。而后，通过调用mountComponentIntoNode，执行真正的挂载流程。挂载时，会引入事务机制，在事务中执行ReactComponent.Mixin.\_mountComponentIntoNode方法，该方法会记录下挂载的时间开销，并调用mountComponent得到待更新的HTML字符串，最后通过更新container的innerHTML完成挂载。

更新的流程稍有区别，其实更新时，就是把nextComponent的属性值，更新到preComponent上，然后让preComponent执行\_performComponentUpdate，完成更新操作。当然，更新的过程也是在事务中完成的。当然，这是preComponent与nextComponent两者类型一致时的处理流程，有的时候，有可能两者的类型并不一致，这时的处理就更加简单，首先卸载currentComponent，然后挂载nextComponent，即完成更新操作。

整个流程如下图所示：


![挂载渲染流程.png | center | 832x495](https://gw.alipayobjects.com/zos/skylark/c31c997f-d436-4d69-b130-8867d8c8a55c/2018/png/d29ce9b8-8586-4044-b861-2d428fcd5b13.png "")


