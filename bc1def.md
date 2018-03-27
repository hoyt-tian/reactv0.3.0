# Immutable Object

由于javascript本身是动态语言，对象天生就是mutable（除非适用Object.freeze处理过），某些情况下，为了保证数据不被修改，就需要immutable object，任何对immutable object的改动，都会重新生成一个immutable对象，而原来的对象不受影响。

v0.3.0中的Immutable Object提供了几个简易的方法实现Immutable。在源代码中，\_\_DEV\_\_运行环境下Immutable的实现与正是环境略有差异，主要是增加了一些日志、警告输出。

# 构造函数
Immutable Object的构造函数接受一个map，通过mergeInto方法，将map中的所有属性添加到新的immutable对象中。

# set方法
Immutable对象同时包含一个set方法，该方法接受一个immutable对象，和要修改的数据(map类型)，它首先根据immutable对象新建另一个immutable对象，然后再将map中的数据合并到新的immutable对象中。

# setField
对set方法进行了封装，允许调用时传递字段名称和对应的值，底层仍然是封装成map，然后掉set方法。

# setDeep
对immutable对象进行深度更新，逻辑类似于deepCopy，其实现如下：
```javascript
function _setDeep(object, put) {
  checkMergeObjectArgs(object, put);
  var totalNewFields = {};

  // To maintain the order of the keys, copy the base object's entries first.
  var keys = Object.keys(object);
  for (var ii = 0; ii < keys.length; ii++) {
    var key = keys[ii];
    if (!put.hasOwnProperty(key)) {
      totalNewFields[key] = object[key];
    } else if (isTerminal(object[key]) || isTerminal(put[key])) {
      totalNewFields[key] = put[key];
    } else {
      totalNewFields[key] = _setDeep(object[key], put[key]);
    }
  }

  // Apply any new keys that the base object didn't have.
  var newKeys = Object.keys(put);
  for (ii = 0; ii < newKeys.length; ii++) {
    var newKey = newKeys[ii];
    if (object.hasOwnProperty(newKey)) {
      continue;
    }
    totalNewFields[newKey] = put[newKey];
  }

  return (object instanceof ImmutableObject || put instanceof ImmutableObject) ?
    new ImmutableObject(totalNewFields) :
    totalNewFields;
}
```

在React v0.3.0的源代码中，并没有用到Immutable Object的实例。
