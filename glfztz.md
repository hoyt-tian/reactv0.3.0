# 连续状态更新

如果连续的执行setState()时，React并不会进行多次重新渲染，而是只会渲染一次，避免了多余的DOM操作，这样的高效连续更新是怎么做到的呢？这就要从setState的实现说起。

# pendingState
React复合组件除了state，还包含一个内部属性，叫做\_pendingState。当复合组件处在MOUNTING状态或者RECEIVING\_PROPS状态时，setState的结果就会被写入到\_pendingState中；而当复合组件既不处在MOUNTING中，也不处在RECEIVING\_PROPS中，就会发起一个新事务，在事务中完成组件的状态更新。这样一来，无论调用多少次setState，只要当前尚处在MOUNTING或者RECEIVING\_PROPS状态时，setState就会将结果赋值给\_pendingState，而不会额外发起新的事务。

pendingState会优先于state作为nextState传递给\_receiveProsAndState，完成状态更新。

# 连续setState
当首次调用setState后，React会启用一个事务，在事务中进行状态更新。然而，setState有可能会被连续调用，当react已经处在事务中时，第二次调用setState就不会再发起新的事务，而是直接将状态更新写入到pendingState中。等到调用receiveProps时，\_pendingState被赋值给nextState，并将状态更新为RECEIVING\_STATE，进一步完成渲染。换言之，如果组件的状态已经变更成为RECEIVING\_STATE，再调用setState，此时就会产生一个新的事务。
```javascript
replaceState: function(completeState) {
    var compositeLifeCycleState = this._compositeLifeCycleState;

    this._pendingState = completeState;

    // 只有既不在挂载中、也不在接受属性中，才发起新事务，否则都直接更新pendingState
    if (compositeLifeCycleState !== CompositeLifeCycle.MOUNTING &&
        compositeLifeCycleState !== CompositeLifeCycle.RECEIVING_PROPS) {
      this._compositeLifeCycleState = CompositeLifeCycle.RECEIVING_STATE;

      var nextState = this._pendingState;
      this._pendingState = null;

      var transaction = ReactComponent.ReactReconcileTransaction.getPooled();
      // _receivePropsAndState会使用pendingState作为待渲染数据，
      // 同时将复合组件状态更新为RECEIVING_STATE，此时若调用setState，就会发起一个新事务
      transaction.perform(
        this._receivePropsAndState,
        this,
        this.props,
        nextState,
        transaction
      );
      ReactComponent.ReactReconcileTransaction.release(transaction);

      this._compositeLifeCycleState = null;
    }
  },
```


多数情况下，连续的setState只会产生一个事务。当一个事务已经启用后，调用setState时，结果会被缓存在pendingState中，由于setState会触发调用receiveProps，复合组件状态会被设置为RECEIVING\_PROPS，直至\_receivePropsAndState执行完毕，在执行\_receivePropsAndState时，会优先取pendingState作为nextState，同时将复合组件状态更新为RECEIVING\_STATE。因此，即便事务开启后，没有新的事务诞生，但由于setState的结果都写入到了pendingState中，在实际的渲染开始前，仍然会将最新改变的状态作为待渲染的数据。

当然，React只是尽可能的减少发起新事务，但不能彻底避免。如果在\_receivePropsAndState执行中，当前复合组件状态已经更新为RECEIVING\_STATE，而此刻又调用了setState，就会导致一个新事务被创建。



