# Hello React和renderComponent

在examples目录下，我们可以找到官方给出的使用示例。其中，basic目录下给出了React组件的示例，当然，React并没有俗套的显示Hello World，而是显示了一个不停更新变化的计时器。其源码如下
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
var start = new Date().getTime();
setInterval(function() {
    React.renderComponent(
      ExampleApplication({elapsed: new Date().getTime() - start}),
      document.getElementById('container')
    );
}, 50);
```

上面这段代码，首先创建了一个自定义的ReactCompositeComponent类型，随后创建了一个定时器，每隔50ms执行一次React.renderComponent。

打开浏览器，可以发现，页面上的计时器不断的进行着更新。React.createClass这里不展开叙述，先重点看一下renderComponent。renderComponent是React框架提供的一个非常重要的接口，该方法接收两个参数，component和container，其中component是一个react组件，container是DOM容器，它将给定的component渲染到目标container上面。

那么renderComponent具体做了些什么呢？让我们一探究竟。

renderComponent的具体实现在ReactMount.js中。每当执行renderComponet时，React都会先判断，这次的renderComponent是应该执行组件更新，还是应该挂载渲染。具体来说，当一个组件需要被渲染到指定容器中时，React首先会通过container，查找是否存在与之关联的component实例。如果没找到，说明需要执行一个新的挂载渲染流程，React会将当前的container注册下来，并将其与传入的component关联映射，随后调用component.mountComponentIntoNode方法，完成后续操作。而如果通过container找到了关联的component，说明container已经挂载过该组件，此时需要执行更新的流程，调用replaceProps来更新已挂载好的组件，其流程如下图所示：

![renderComponent.png | center | 515x774](https://gw.alipayobjects.com/zos/skylark/543ffe0a-b66a-436c-af0d-d6dc6b4b139b/2018/png/204bf188-c481-4e1e-bf57-58ff0e5eae76.png "")


