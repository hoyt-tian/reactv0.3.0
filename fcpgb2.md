# createClass与复合组件

ReactNativeComponent只能提供基于单个HTML元素的原生组件，如果有更复杂的组件需要，就只能借助复合组件ReactCompositeComponent来实现。复合组件不仅包含属性(props)，还包含状态(state)，它可以将原生组件或者其他复合组件作为其子元素，组合成更加复杂、强大的新组件。

React.createClass调用得到的就是复合组件，我们再看一下React的官方示例
```javascript
var ExampleApplication = React.createClass({
        render: function() {
          var elapsed = Math.round(this.props.elapsed  / 100);
          var seconds = elapsed / 10 + (elapsed % 10 ? '' : '.0' );
          var message =
            'React has been successfully running for ' + seconds + ' seconds.';

          return React.DOM.p(null, message);
        }
});
```

创建复合组件必须要提供一个render函数，render函数执行后会返回复合组件的渲染结果，示例代码就是渲染了一个文本段落。

# 复合组件的继承关系
复合组件的继承关系较为复杂，createClass的函数回调是另一个函数，这个函数为复合组件提供了统一的构造调用ConvenienceConstructor(props, children)，这个构造调用内部做的事情很少，它首先生成一个对象，随后调用了这个对象的construct函数。construct函数就是之前介绍过的ReactComponent.Mixin中定义的初始化函数。以官方示例的代码为例，其实ExampleApplication在调用React.createClass之后得到的就是
```javascript
var ExampleApplication = function(props, children) {
   var instance = new Constructor();
   instance.construct.apply(instance, arguments);
    return instance;
}
```

而这个Constructor又是什么呢？它一开始是一个空函数，随后其原型被设置为ReactCompositeComponentBase。ReactCompositeComponentBase这个类继承了ReactComponent、ReactOwner、ReactPropTransfer和ReactCompositeComponent这四个类定义的Mixin。综上，当调用createClass时，创建了一个空类Constructor，并通过原型链的形式继承自ReactCompositeComponentBase。这一关系如下图所示。


![继承关系.png | center | 830x539](https://gw.alipayobjects.com/zos/skylark/77ade756-4a97-4965-95d3-745ed2ac6c1b/2018/png/09a05342-10ae-4330-8d27-1770f192edee.png "")


为什么不直接new一个ReactCompositeComponentBase对象，而要多此一举的增加一个Constructor呢？试想一下 ，假设按照下面的代码执行，是否可行？
```javascript
var ExampleApplication = function(props, children) {
   var instance = new ReactCompositeComponentBase();
   instance.construct.apply(instance, arguments);
    return instance;
}
```
这样做有隐患，var a = React.createClass({})，此时如果a修改了自己的prototype，就会导致ReactCompositeComponentBase的原型发生变化。若还有var b = React.createClass({})。由于a、b的原型是一致的，就会发生一方修改了自己的prototype，可能导致另一方无法按照预期运行的情况。正是出于这个原因，在React.createClass时才会多处一层类Constructor。这样一来无论var a = React.createClass()获得复合组件类后，如何修改其prototype，都完全不会影响到var b = React.createClass()。
