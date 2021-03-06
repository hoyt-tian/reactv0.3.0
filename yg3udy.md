# Mixin和多重继承

ReactComponent中还有一个非常重要的成员，Mixin。Mixin本身也是一个简单对象，当某个类需要继承其他另外几个类的方法时，只要这些类都把自己的公共方法放在mixin里，就可以利用mixInto函数快速的将这些类的方法继承下来。简言之，Mixin就是用来实现多重继承的，可以用如下的伪代码进行描述

```javascript
var A = {
   Mixin: { ... }
}

var B = {
   Mixin: {...}
}

var C = function() {}

mixInto(C, A.Mixin)
mixInto(C, B.Mixin)
```

# 为何引入多继承
多重继承比单一继承更复杂，一般都建议慎用，React框架中为何要设计这样一种多重继承机制呢？这是因为React框架尽量将逻辑相关的代码放到了同一文件中，这导致了像ReactCompositeComponent这些类必须同时从很多模块中继承其方法才行；同时。下图给出了React核心类之间的多重继承关系：

![image.png | center | 830x539](https://gw.alipayobjects.com/zos/skylark/cd19ca08-e5ea-40d7-87fc-f26dfed306d9/2018/png/a07f1d8d-6f6a-40ee-a8c8-88308c4d2938.png "")
稍微解释一下上面的图片，有很多类还没涉及到，ReactComponent定义的组件公共方法都放在了它的Mixin中，同时被ReactCompositeComponentBase（复合组件基类）和ReactNativeComponent（原生组件类）继承了。ReactNativeComponent从ReactMultiChild继承了子节点操作相关的方法。而ReactCompositeComponentBase不仅继承了ReactComponent，还从ReactOwner继承了ref相关的方法，从ReactPropTransfer继承了属性操作相关的方法。
# mixInto具体实现

mixInto方法定义在src/utils/mixInto.js中，它负责将一个mixin对象中的成员添加到给定的类上面，其源码如下：
```javascript
var mixInto = function(constructor, mixin) {
  var methodName;
  for (methodName in mixin) {
    if (!mixin.hasOwnProperty(methodName)) {
      continue;
    }
    constructor.prototype[methodName] = mixin[methodName];
  }
};
```
在进行添加操作时，必须是mixin的自有属性（hasOwnProperty)才会被添加到目标类的原型对象上面。这段代码也告诉我们，mixInto在实现多重继承的时候，有一个潜在风险，那就是，如果同时继承自多个mixin，如果这些mixin中存在同名的成员，那么就会被后来的mixin覆盖掉。
