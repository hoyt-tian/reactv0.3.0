# 复合组件的更新流程

在组件的挂载流程中讲到了，组件的渲染结果保存在\_renderedComponent中。当组件执行更新操作时，首先会将之前的renderedComponent保存下来，作为currentComponent；然后再执行\_renderValidatedComponent，将其结果记为nextComponent。

如果currentComponent和nextComponent两者类型相同，那么只需要更新子组件的属性值即可；两者类型不同，则需要将原油组件卸载，同时将新组件挂载到对应的位置。相关的源代码如下所示：
```javascript
updateComponent: function(transaction) {
    var currentComponent = this._renderedComponent;
    var nextComponent = this._renderValidatedComponent();
    if (currentComponent.constructor === nextComponent.constructor) {
      if (!nextComponent.props.isStatic) {
        currentComponent.receiveProps(nextComponent.props, transaction);
      }
    } else {
      // These two IDs are actually the same! But nothing should rely on that.
      var thisID = this._rootNodeID;
      var currentComponentID = currentComponent._rootNodeID;
      currentComponent.unmountComponent();
      var nextMarkup = nextComponent.mountComponent(thisID, transaction);
      ReactComponent.DOMIDOperations.dangerouslyReplaceNodeWithMarkupByID(
        currentComponentID,
        nextMarkup
      );
      this._renderedComponent = nextComponent;
    }
  },
```

