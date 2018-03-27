# ReactComponent

# 简介
ReactComponent类定义了React组件的基本属性和方法，包含了一些工具函数。

* 成员方法与对象方法

类方法直接被定义在json上面。而所有组件实例（this）的属性和方法，可能被多重继承，则被定义在mixin中。

* 生命周期

所有的React组件都都有自己的生命周期（ComponentLifeCycle），组件的生命周期取值只有两种，MOUNTED和UNMOUNTED，分别代表组件已经挂载到DOM树中与未被挂在到DOM树中。

我们熟知的componentWillMount、componentDidMount都是在生命周期发生变化时执行的钩子函数，定义在ReactCompositeComponent中。

* 依赖注入

值得一提的是，DOM操作（ReactDOMIDOperations）与事物处理（ReactReconcileTransaction）的两个模块作为两个属性挂在到ReactComponent中，并采用（setDOMOperations）与（setReactReconcileTransaction）两个方法，可以设置具体的实现，也就是说如果有特殊需要，可以不使用React自带的事务及DOM操作，而改用其他的实现。

在源码注释中，这种方法被成为“依赖注入”。熟悉设计模式的同学，可以联想下策略模式，这些属性是ReactComponent上的不同策略，消费属性前可以任意修改策略，贴个[设计模式-策略模式](https://github.com/antgod/refactor/blob/master/src/3.%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/2.%E7%AD%96%E7%95%A5%E6%A8%A1%E5%BC%8F/1.%E8%AE%A1%E7%AE%97%E5%A5%96%E9%87%91.html)的例子

# 分析
接下来我们按照源代码中的顺序，逐一地介绍ReactComponent的成员。

## 成员方法

### isValidComponent
这是一个基础工具方法，用来判断给定对象是否为一个React组件。

其判断依据是，该对象存在mountComponentIntoNode方法和receiveProps方法。

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

### LifeCycle
组件的生命周期只有两个取值：MOUNTED和UNMOUNTED，其值域定义在ReactComponent.js文件中。

我们熟知的componentWillMount等，其实都是回调函数，而非组件的生命周期状态值。

**LifeCycle会在文章结尾ComponentLifeCycle做详细介绍。**

### DOMOperations
所有跟dom操作相关的方法都封装在了ReactDOMOperations这个类中，React为了使DOM操作可以灵活替换，这个属性可以通过setDOMOperations进行设置，这样一来，DOM相关操作就可以封装成一个相对独立的模块，React也可以支持其他的DOM操作插件。

<span style="color:#8C8C8C;">_这里顺便展开一个话题，就是缩写词汇的大小写问题。一般的变量使用驼峰规则进行命名，都没有太大问题，但如果遇到了缩写词汇，该如何处理呢？常常能看到五花八门的处理方式，_</span><span style="color:#8C8C8C;">_用什么规则其实都可以，但最重要的就是规则要统一_</span><span style="color:#8C8C8C;">_，React给了一个很好的示例，所有的缩写词汇一律使用大写。_</span>

### ReactReconcileTransaction
类似于DOMOperation，它是用来处理事务机制的。同样有setReactReconcileTransaction方法支持设置。事务机制保证了React数据更新时的正确性，其详细内容也会在后续展开叙述。

## 实例方法（Mixin）
当某一个类需要同时从多个基类继承方法时，传统的单继承模式就不太方便了，为了实现多重继承，把公共的属性和方法都被放置到了Mixin中，任何一个类想要添加其他类的属性和方法，只需要通过mixInto方法即可，其详细实现将在Mixin与多继承章节中进行讲述。

### getDOMNode
该方法返回React组件对应的根DOM节点，只有被挂载后的React组件才能获取到DOM节点，否则会抛出异常，其实现如下

```javascript
getDOMNode: function() {
      invariant(
        ExecutionEnvironment.canUseDOM,
        'getDOMNode(): The DOM is not supported in the current environment.'
      );
      invariant(
        this._lifeCycleState === ComponentLifeCycle.MOUNTED,
        'getDOMNode(): A component must be mounted to have a DOM node.'
      );
      var rootNode = this._rootNode;
      if (!rootNode) {
        rootNode = document.getElementById(this._rootNodeID);
        if (!rootNode) {
          // TODO: Log the frequency that we reach this path.
          rootNode = ReactMount.findReactRenderedDOMNodeSlow(this._rootNodeID);
        }
        this._rootNode = rootNode;
      }
      return rootNode;
}
```

React在挂载组件时，会给节点分配一个自动生成的rootNodeID，并将这个ID信息存储在\_rootNodeID中。在查找DOM节点时，首先根据\_rootNodeID查找document中是否存在对应的节点，若有，并返回该节点；若无，则通过调用ReactMount.findReactRenderedDOMNodeSlow进行查找，这个方法相对复杂一些，后续再展开叙述。

无论通过哪种方法，只要找到了对应的DOM节点，就将它存储在this.\_rootNode中，后续的调用就直接返回。

### setProps
在调用setProps时，React会先调用merge方法，将当前的props和传入的参数进行合并，再将合并后的结果传递给replaceProps完成属性设置。

### replaceProps
该方法负责更新react组件的props。为了确保数据的正确性，React会启用一个事务，在事务保护中执行更新操作。实际的更新操作时在receiveProps中执行的。而相关的事务处理流程也会在后续详细阐述。事务机制是React源码中非常核心和关键的机制之一。

### construct
每当我们实例化一个组件时候（<ReactComponent />）,我们实际就调用的这个函数。怎么得出来这个结论的呢？好，并不是每位同学都了解浏览器的堆栈(调用栈与宜昌栈)，这里科普下基础经验（说常识怕打击到各位宝宝的自尊心^\_^）：

* 在construct里任意一行打debugger。
* 打开开发人员调试工具，刷新页面，观察调用栈（Call Stack）


![image.png | center | 830x197](https://gw.alipayobjects.com/zos/skylark/69c4e23f-b792-4959-ad31-9310d37692c6/2018/png/64bb86f4-125d-4d3e-accf-7fa8f3b4274d.png "")
* 依次点击construct到anonymous的所有执行方法，我们发现，每当实例化一个React时候，首先调用createClass的
  ConvenienceConstructor方法->调用Constructor方法->调用construct方法。实例化组件的属性与children分别传入construct的参数中。
 

construct承担的是构造函数的职责，负责对所有的React组件进行初始化构建

* 初始化了基本属性值，包括props = initialProps，\_lifeCycleState = UNMOUNTED
* 对children进行了预处理和验证
* 对props[OWNER]进行处理，确保组件已经初始化。防止未初始化的组件调用replaceProps。


#### 问题来了，**为什么不直接将构造函数声明成这样呢？？？**

这是为了将内存申请和内存初始化刻意分开，无参的构造函数及其类进行new操作时更加简洁，这样无论继承的类原本的构造函数是何种形式，只要正确调用了construct方法，就可以将实例构造使之包含ReactComponent的公共属性和方法，在多继承的模式下，比起构造函数更加灵活；更重要的是，construct函数本身还可能被动态更新，这一点后续会讲到，因此无法直接使用constructor。当然，对实际的构造函数调用处，也要稍加改进，这一点要等到后续讲到ReactCompositeComponent时再详细展开。

### mountComponent/unmountComponent
每当调用renderComponent时调用这个函数。
<span style="color:#8C8C8C;">_在renderComponent调用mountComponentIntoNode->调用\_mountComponentIntoNode->调用mountComponent（ReactCompositeCompoent）->调用mountComponent（ReactCompoent）_</span>

该方法接受两个参数：rootId和transaction，其中transaction是事务对象。React中将挂载的流程进行了分层处理，就ReactComponent层面来说，它只负责下面几个工作
* 判断组件状态是否为UNMOUNTED

* 为组件添加ref（dom元素指向，可以在mount后根据指向查找相应的dom元素）

* 更新rootId

* 更新\_lifeCycleState为MOUNTED


ReactCompositeComponent的mountComponent执行挂载时，还会有其他额外工作，ReactComponent中只做了组件层面的必要工作。

卸载组件流程类似于挂载流程。

这两个方法子类重载时，都需要显示调用，以保证处理流程被正确执行。

### receiveProps
receiveProps会在新属性被传递过来后执行，其执行过程也会被事务机制保护。ReactComponent层面主要针对ref的变化进行了更新操作。

### mountComponentIntoNode
该方法会被ReactMount.renderComponent调用，它接受两个参数：rootID和container。其中rootID是要挂载的根节点id，container是容器节点。

该方法做了一层事务封装，实际的更新操作则是在\_mountComponentIntoNode中。React会记录下执行\_mountComponentIntoNode的时间开销。通过调用mountComponent生成待挂载的DOM对象。
更新到document中就要分情况讨论了：

* 如果容器节点存在父节点，则尝试通过container.nextSibling来定位容器节点的更新位置（container的下一个元素）。先将container从父节点中删除，并将container的内容更新为待挂载的DOM信息。此时再判断之前保存下来的container.nextSibiling是否存在，若是，则通过parent.insertBefore将container重新插入到container原来的位置；若否，那么直接通过父节点执行appendChild添加到父节点的最后。


* 若container的父节点不存在（container为html元素，没有父节点，或者外面缺失body\html等标签），则直接通过innerHTML进行更新。

![React v0.3.0.png | center | 430x820](https://gw.alipayobjects.com/zos/skylark/7a7ce8ff-21bd-4980-9eea-f63e8fee979a/2018/png/f7211825-fe03-4635-a93b-e86469fcd86c.png "")

另外，每当需要发起事务时，都会先从缓冲池中取出一个事务对象，待执行完毕后释放该对象。事务机制的详细内容也是后续讲解的重点之一。

### unmountComponentFromNode
首先调用unmountComponent()对组件本身进行了卸载处理，然后从container中移除了全部的dom节点。在进行节点清空时，React使用了循环调用container.removeChild(container.lastChild)的方法，并通过注释解释说这种方式效率最高并给出了一个测试网址[http://jsperf.com/emptying-a-node](http://jsperf.com/emptying-a-node，这需要结合浏览器的源代码才能解释具体缘由了，我暂时并不清楚其根本原因。)，这需要结合浏览器的源代码才能解释具体缘由了，我暂时并不清楚其根本原因。

### isOwnedBy
React组件会将owner作为props的属性保存下来，isOwnedBy会将传入的owner与props中保存的值进行对比。

### update/updateAll
这是两个过时的方法，它们分别被setProps和replaceProps替代了，显然是在v0.3.0之前定义的方法。

### ComponentLifeCycle和keyMirror
这是在src/core/ReactComponent.js中定义的枚举对象，描述了React组件的生存周期取值。

```javascript
/**
 * Every React component is in one of these life cycles.
 */
var ComponentLifeCycle = keyMirror({
  /**
   * Mounted components have a DOM node representation and are capable of
   * receiving new props.
   */
  MOUNTED: null,
  /**
   * Unmounted components are inactive and cannot receive new props.
   */
  UNMOUNTED: null
});
```

这里用到了一个工具函数，keyMirror在src/utils/KeyMirror.js文件中定义，其作用是根据给定的对象，创建枚举对象，源码非常简单易懂。

```javascript
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

```javascript
const ComponentLifeCycle = {
    MOUNTED: "MOUNTED",
    UNMOUNTED: "UNMOUNTED"
}
```

