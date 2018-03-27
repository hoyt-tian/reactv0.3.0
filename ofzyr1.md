# 初窥React

# 基本示例
在React源代码目录下，examples文件中包含了各种React实例代码。其中basic文件夹下，给出了一个React最简单的示例。核心代码非常简单，如下所示：
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

这段代码实现了一个自动更新React执行时间的计时器，可以算是React框架的Hello Demo。在这个示例代码中，引出了两个非常非常重要的函数，分别是React.createClass和React.renderComponent。
# 组件和复合组件
组件是React中的重要概念，组件有自己的属性(props)。

React.createClass的返回值是复合组件。除了属性，复合组件还包含状态(state)。在实际的React项目中，通常会创建各种复合组件，来实现各种视图。

在创建复合组件时，必须为其指定render方法。该方法负责将组件的属性和状态按照需要进行渲染。
# renderComponent
React.renderComponent通常是整个react程序初始化的入口（对应到15.0.0以及往后的版本，就是ReactDOM.render)。renderComponent接口引出了React框架的渲染机制及其详细过程，我们先粗略的过一遍这个接口的具体实现，在浏览的过程中，事件相关的代码会先忽略掉。

renderComponent(component, container)负责将一个component实例渲染到给定的container中。React框架在进行渲染时，会尽可能地复用现有的DOM节点，因此，React会先判断，当前的container是否存在一个与之对应的、已经渲染过的component，具体的查找过程暂且略过。

如果React并没找到某个已有的组件实例与container有关联，它就会将component与container的关联信息保存下来，以待后续查询，同时调用mountComonentIntoNode，将component挂载到container上。相反，如果React找到了与container关联的组件实例，则会执行一个更新流程，使用component的属性信息，来更新查找到的react组件实例。这一过程的简图如下所示：

![renderComponent.png | center | 515x774](https://gw.alipayobjects.com/zos/skylark/182ca13d-7467-40dc-881f-7f467e3894a2/2018/png/17513c4d-c622-47f9-af26-91a8fa17e114.png "")

