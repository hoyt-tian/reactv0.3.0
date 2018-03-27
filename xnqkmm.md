# 多重继承和Mixin

React在实现核心类时，大量的使用了Mixin，这是为了高效便捷的实现“多继承”。Mixin本身就是一个简单的对象，当某个类需要继承其他另外几个类的方法时，只要这些类都把自己的公共方法放在mixin里，就可以利用mixInto快速的将这些类的方法继承下来。通过mixInto这个函数，可以将给定的mixin添加到目标类中。下图给出了React部分关键类的mixin关系。以ReactComponent为例，该类的Mixin中定义了ReactComponent的基本方法，并将之以mixin的形式添加到了ReactNativeComponent类和ReactCompositeComponentBase，**而ReactCompositeComponentBase则以ReactCompositeComponent.Base的形式暴露出来。**

![Mixin关系图.png | center | 832x541](https://gw.alipayobjects.com/zos/skylark/63f3f5ef-5cd6-4291-aad4-ac0818dc8576/2018/png/5745863d-044f-464e-bafe-c57f640241f1.png "")

# mixInto的实现
mixInto方法定义在src/utils/mixInto.js中，其源码如下：
```javascript
var mixInto = function(constructor, methodBag) {
  var methodName;
  for (methodName in methodBag) {
    if (!methodBag.hasOwnProperty(methodName)) {
      continue;
    }
    constructor.prototype[methodName] = methodBag[methodName];
  }
};

```
其实mixin的过程，就是将mixin对象的自有方法添加到给定类的原型对象上。关于原型(prototype)，更多内容可以参考https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\_Objects/Object/prototype
