# 子节点更新和ReactTextComponent

无论是原生组件还是复合组件，在更新时，都有可能需要更新其子节点。当然，对于复合组件来说，归根结底也是原生组件的子节点更新，因此，当原生组件的属性发生更新时，就会执行\_updateDOMChildren。

在\_updateDOMChildren时，会将props.children进行预处理。预处理的过程中，还会将文本内容转换为ReactTextComponent，这是一个特殊的抽象组件子类，专门负责显示文字信息，ReactTextComponent的挂载过程非常简单
```javascript
mountComponent: function(rootID) {
    ReactComponent.Mixin.mountComponent.call(this, rootID);
    return (
      '<span id="' + rootID + '">' +
        escapeTextForBrowser(this.props.text) +
      '</span>'
    );
  },
```

ReactTextComponent会将文本内容进行escape处理，内部是通过正则表达式进行的字符替换，其实现细节可以在src/utils/escapeTextForBrowser中查看。

预处理后的children信息存储到一个对象中，记为nextChildren，当前的子节点信息则存储在\_renderedChildren中，然后调用ReactMultiChildren中的updateMultiChild方法进行更新。

在执行updateMultiChild时，会进行两轮判断：

1. 出现在nextChildren且同时出现在\_renderedComponent中，说明该子节点是需要保留的节点，但有可能需要根据情况调整位置
2. 只出现在\_renderedComponent但没有出现在nextChildren中，说明该节点可以直接卸载。
3. 只出现在了nextChildren中，但没有出现在\_renderedComponent中，说明该节点是新插入的。

所有的DOM操作都会先被添加到不同类型的队列之中，这三个队列分别是：移动队列、卸载队列和插入队列。

为了防止\_renderedChildren中有需要被卸载但没有被处理的元素，在遍历了nextChlidren之后，还会在遍历一遍\_renderedChildren。如果某个元素只出现在了\_renderedChildren中而没有出现在nextChildren中，就说明需要卸载该节点。

