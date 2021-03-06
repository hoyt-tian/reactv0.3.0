# 设计思想总结

# Keep It Simple
高内聚、低耦合是所有系统设计的重要原则之一。保持函数功能简单聚焦，可以帮助我们下意识的降低代码耦合。React中的函数基本都比较小巧。

而且，React的函数基本上不超过5个形参，一来js的参数本身可以是复合类型，5个复合类型的形参已经可以传递大量数据了，二来参数列表过长，也不便于记忆。
# Don't Repeat Yourself
凡是会被两个不同的js文件同时引用的代码，在React中都写在了独立的文件模块中，以便其他地方调用，在util目录下有些工具函数其实很短，但会以独立的文件存储，来方便其他文件import。

千万不要用复制粘贴的方法来解决重复的需求，一旦要进行更改，很可能导致遗漏。
# Composition Over Inheritance
优先使用组合来解决问题，而非继承。组合意味着只要将现有的功能模块进行有机的整合即可，而继承则表示现有的解决方案无法搞定问题，需要新的过程。过度的继承只会导致复杂度上升。
# Dependency Injection
在React框架中，很多资源都是支持注入的，例如像DOMOperation等，这样做的好处就在于模块可以很方便的进行更换，但不影响整个框架。依赖注入在Java等后台开发语言中很早就有提倡，它从系统层面要求模块之间降低耦合度。
# Interface Segregation Principle
在设计接口的时候，需要考虑接口隔离原则。接口中定义的内容，不能包含与抽象过程无关的内容。以React中的事务接口为例，Transaction中定义的接口全部都是通用的事务处理方法及属性，而getTransactionWrapper则是需要被实现的功能，其他与接口过程无关的属性和方法，也不应当添加到接口定义中。
遵循接口隔离原则，也是尽可能的让接口保持简洁。
# Separation of Concerns
在React的源代码中，还经常能看到很多层级的封装调用，例如进行顶层事件绑定时的调用，相关代码如下
```javascript
function trapBubbledEvent(topLevelType, handlerBaseName, onWhat) {
  listen(
    onWhat,
    handlerBaseName,
    ReactEvent.TopLevelCallbackCreator.createTopLevelCallback(topLevelType)
  );
}

listen: function(el, handlerBaseName, cb) {
    EventListener.listen(el, handlerBaseName, createNormalizedCallback(cb));
}

function createNormalizedCallback(cb) {
  return function(unfixedNativeEvent) {
    cb(normalizeEvent(unfixedNativeEvent));
  };
}

 EventListener.listen: function(el, handlerBaseName, cb) {
    if (el.addEventListener) {
      el.addEventListener(handlerBaseName, cb, false);
    } else if (el.attachEvent) {
      el.attachEvent('on' + handlerBaseName, cb);
    }
  }
```
可以看到，很多函数的实现非常简单，就是调用了更底层的某个调用，然后某个参数使用了另外一个函数进行了处理。有没有必要进行代码合并处理呢？其实这种代码分离，是的确存在逻辑上的隔离的，应当这样做。在进行大规模的工程开发时，效率有时未必是优先级最高的需求了，但代码的可读性非常重要，它才能保证项目在人员流动的过程中少出问题。分层的考虑问题，可以帮助我们在特定层面聚焦特定需求，比起杂糅在一起的代码，逻辑更清晰，流程更清晰。
