# 组件的公共方法

组件的公共方法都被定义在了ReactComponent.Mixin中，这些公共方法可以分成如下几类：

<div class="bi-table">
 <table>
   <colgroup><col width="196px"><col width="632px"></colgroup>
   <tbody>
    <tr>
      <td><div data-type="p">分类</div></td>
      <td><div data-type="alignment" data-value="center" style="text-align:center;"><div data-type="p">方法名称</div></div>
</td>
    </tr>
    <tr>
      <td><div data-type="p">初始化方法</div></td>
      <td><div data-type="p">construct</div></td>
    </tr>
    <tr>
      <td><div data-type="p">&NegativeMediumSpace;</div><div data-type="p">组件挂载</div></td>
      <td><div data-type="p">mountComponent</div><div data-type="p">mountComponentIntoNode</div><div data-type="p">_mountComponentIntoNode</div></td>
    </tr>
    <tr>
      <td><div data-type="p">组件卸载</div></td>
      <td><div data-type="p">unmountComponent</div><div data-type="p">unmountComponentFromNode</div></td>
    </tr>
    <tr>
      <td><div data-type="p">属性更新</div></td>
      <td><div data-type="p">setProps</div><div data-type="p">receiveProps</div></td>
    </tr>
    <tr>
      <td><div data-type="p">容器相关</div></td>
      <td><div data-type="p">getDOMNode</div><div data-type="p">isOwnedBy</div></td>
    </tr>
   </tbody>
 </table>
</div>

表中列出的方法，存在于所有ReactComponent及其子类的实例中，接下来我们逐一看看这些方法。
# construct
注意不是constructor，construct方法负责完成对ReactComponent的初始化工作，它接受两个参数initialProps和children，分别代表该组件的初始属性和子元素集合。

## 记录Owner

初始化时，首先会记录下当前实例的Owner信息。Owner信息存储在ReactCurrentOwner.current中，所谓的Owner，就是负责渲染当前组件的那个组件，有点绕口，举个例子。
```javascript
var a = React.createClass({
   render: function(){
       return (<ComponentA>
          <ComponentB />
          <ComponentC></ComponentC>
       </ComponentA>)
   }
})
```
上述代码中，a就是ComponentA、ComponentB以及ComponentC的Owner。ComponentA等在执行construct方法时，首先会在props中增加一个key为'{owner}'的属性，其值为a。

## 初始状态

组件在初始化是，componentLifeCycle状态是UNMOUNTED。

## 子元素
当construct方法的实际传参为1个时，说明该组件不包含子元素。

当construct实际参数为2个时，先判断第2个参数的类型，若为null、字符串或者数字时，就会直接作为props.children的值；若第2个参数的类型为数组，只要数组中的每一个元素都不是null或者布尔值时，就将它直接付给props.children。

若第2个参数的数组中包含了null或者布尔值，或者实际传参数量超过2个时，这时会从第2个参数开始，将所有后续参数添加到一个数组中（标记为Target)，如果参数的类型是数组，那么这个数组中的每一个元素都会被添加到Target中。

# 组件挂载
与组件挂载相关的函数包括mountComponent、mountComponentIntoNode和\_mountComponentIntoNode。组件挂载的过程十分复杂，不同类型的组件，挂载行为也有不同，因此，React框架将挂载行为的公共代码抽出来，放在ReactComponent的Mixin中，同时，ReactCompositeComponent和ReactNativeComponent中也包含挂载的处理代码。

在ReactComponent中，主要负责处理组件挂载时的ref处理和ComponentLifeCycle更新（mountComponent函数），还包括挂载时的事务调度(mountComponentIntoNode函数)以及实际的HTML代码注入到特定DOM节点(\_mountComponentIntoNode函数)，先理解流程，实现细节以后再讨论

# 组件卸载
组件卸载的过程相较挂载更为简单一些，卸载时将ref移除，并更新了ComponentLifeCycle的状态，更新DOM时也没有引入事务机制。

# 属性更新
setProps方法内部也调用了replaceProps方法，而进行组件属性更新时，将启用事务进行管理。具体的更新过程将用专门的章节讲解。

# 容器相关
与容器相关的接口有两个，isOwnedBy和getDOMNode。isOwnedBy来判断当前组件是否属于给定实例，直接通过比较props中的'{owner}'属性是否全等于传入参数来确定，参见"记录Owner"。

getDOMNode用来返回当前组件实例对应的容器DOM节点，React会将组件的容器节点缓存在this.\_rootNode上。如果\_rootNode不为空，那么直接返回\_rootNode就是目标容器节点。

如果\_rootNode为空，则此时需要进行查询。组件在挂载时(ReactComponent.mountComponent)，会将目标容器的id存储在rootNodeID这个内部属性中，在getDOMNode时，直接调用doucment.getElementById(this.\_rootNodeID)进行查询，如果查到了，就将该节点存储在\_rootNode上，以便后续直接返回。如果仍然没有查到，说明该组件是被其他组件包含并渲染，此时需要向上查询对应的容器节点，通过调用ReactMount.findReactRenderedDOMNodeSlow来完成。

