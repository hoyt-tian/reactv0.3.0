# 缓冲池机制

对于一些创建、销毁开销比较大的对象，或者出于减少频繁申请内存的操作，一般都会设置一个对象缓冲池，尽可能的复用已经有的对象。常见的如数据库缓冲池，游戏中一些反复创建的对象，一般也会设计一个缓冲池。缓冲池技术的本质就是使用空间换取时间，尽可能的复用已经申请好的，并且可以被修改值再次利用的对象。

在react中，也用到了这种技术，比如React的事件处理时，抽象事件对象（AbstractEvent）一般会频繁创建，为了提升性能，就使用了缓冲池。当一个抽象事件对象用完之后，它就会被放置回缓冲池，等待下一次使用，当然，下一次使用时，会更新它的属性信息，更新内存比申请新内存要快，因此缓冲池会有效的提升性能。

React对缓冲池进行了封装，其相关代码都存放在了src/utils/PooledClass中，PooledClass对外提供了4个方法，其中最最重要的就是addPoolingTo方法。

# addPoolingTo
该方法接受两个参数，第一个参数是要添加缓冲池支持的类/构造函数，第二个参数可选，是实例化缓冲池存储类型的wrapper函数，默认支持一个参数(oneArgumentPooler)，同时PooledClass中还提供了针对两个参数(twoArgumentPooler)、五个参数(fiveArgumentPooler)的辅助函数。

在执行addPoolingTo方法时，会给该类增加一个属性instancePool，它是一个数组，用来保存可用的实例对象，如果没有设置缓冲池大小，React默认会设置poolSize为10，然后，为该类添加释放方法。

```javascript
var addPoolingTo = function(CopyConstructor, pooler) {
  var NewKlass = CopyConstructor;
  NewKlass.instancePool = [];
  NewKlass.getPooled = pooler || DEFAULT_POOLER;
  if (!NewKlass.poolSize) {
    NewKlass.poolSize = DEFAULT_POOL_SIZE;
  }
  NewKlass.release = standardReleaser;
  return NewKlass;
};
```

当某个类支持了缓冲池功能时，需要一个新的实例，就直接调用Klass.getPooled()即可，例如ReactComponent.js文件中第394行，获取一个新的事务对象。
```javascript
var transaction = ReactComponent.ReactReconcileTransaction.getPooled();
```
# oneArgumentPooler
在执行pooler函数时，会先判断当前缓冲池(instancePool)中是否存在可用的对象，如果存在，直接从缓冲池中取出一个实例，并重新初始化该对象。如果缓冲池中没有可用的对象，此时就需要new一个新的对象。
```javascript
var oneArgumentPooler = function(copyFieldsFrom) {
  var Klass = this;
  if (Klass.instancePool.length) {
    var instance = Klass.instancePool.pop();
    Klass.call(instance, copyFieldsFrom);
    return instance;
  } else {
    return new Klass(copyFieldsFrom);
  }
};
```

twoArgumentPooler和fiveArgumentPooler执行的内容一样，唯一的区别就是构造新实例的参数数量不同。

# standardReleaser
从缓冲池中取出的对象，使用完毕后需要调用release方法，以将该实例返回缓冲池。

如果实例对象存在destructor方法，就调用该方法；

在当前缓冲池大小没有达到最大值时，实例对象就会被push到缓冲池中。
```javascript
var standardReleaser = function(instance) {
  var Klass = this;
  if (instance.destructor) {
    instance.destructor();
  }
  if (Klass.instancePool.length < Klass.poolSize) {
    Klass.instancePool.push(instance);
  }
};
```

# 性能抖动

理论上讲，缓冲池是空间换时间的策略，但缓冲池并不一定总是能够提升性能，反而有可能导致性能下降。这与js的执行环境有着密切关联。

缓冲池能够提升性能的前提条件，是通过缓冲池避免频繁的内存申请和释放。如果缓冲池大小不合理、内存空间本身有限制或者其他问题，导致无法避免内存申请、释放或者不会导致频繁的内存申请及释放，那么缓冲池机制就未必会有提升效果，甚至会导致性能下降。

现代的js执行引擎都会做内存相关的优化，避免内存申请、释放带来的性能下降，因此，缓冲池机制在Chrome等浏览器中，数量级较小时，难以体现出优势；数量级过大时，超过某个临界值时，有可能引起抖动，也无法提升性能。缓冲池能提升性能的那个阈值区间，还未必会出现在日常的运行情景中。

综合这几条原因，其实在现代浏览器的js执行引擎中，的确可以不需要考虑缓冲池相关的机制了，执行引擎的优化工作已经可以屏蔽掉这些差异。


