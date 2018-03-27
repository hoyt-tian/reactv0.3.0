# ReactNativeComponent

ReactNativeComponent是React对HTML标签节点的封装处理，由于render的返回结果中很可能包含了HTML标签，涉及到事件绑定、ref处理等，都需要将它们转化成React对象来处理，ReactNativeComponent就是负责处理HTML原生标签的。

# 构造函数
ReactNativeComponent的构造函数接受两个参数，tag和omitClose，其中tag是对应的html标签名称，omitClose则表示该元素是否单闭合，比如hr就是一个单闭合标签，还有img等。

构造函数只是根据传参生成了三个属性，\_tagOpen, tagCloase和tagName，很好理解。

# mountComponent
依然是先调用了ReactComponent.Mixin中的mountComponent方法，随后对props进行了验证，最后返回了该标签的html字符串。私有方法createOpenTagMarkup，createContentMarkup分别用来辅助生成开标签、样式和属性内容。

由于标签可能包含多个子元素，为了处理这些子元素，React将相关操作封装到了ReactMultiChild中。在执行createContentMarkup时，如果存在多个child，就会调用到。

# ReactMultiChild
ReactNativeComponent除了从ReactComponent.Mixin和ReactNativeComponent.Mixin继承方法和属性，还继承了来自ReactMultiChild的Mixin。

# mountMultiChild
如果有多个child时，React逐一的给每个child执行mountComponent，并将这些child的执行结果累积到同一个字符串中，最后返回该字符串，即完成了多个子元素的挂载。

React 会给这些子元素添加\_domIndex的索引，并把传入的children保存在renderedChildren中。

# createDOMComponentClass
该方法是定义在ReactDOM.js中的辅助方法，用来创建DOMComponent的wrapper类，其源码如下：
```plain
function createDOMComponentClass(tag, omitClose) {
  var Constructor = function() {};

  Constructor.prototype = new ReactNativeComponent(tag, omitClose);
  Constructor.prototype.constructor = Constructor;

  return function(props, children) {
    var instance = new Constructor();
    instance.construct.apply(instance, arguments);
    return instance;
  };
}
```
类似于React.createClass的处理，返回了统一的wrapper。

objMapKeyVal是一个工具函数，它会遍历给定对象所有hasOwnProperty的键值对，执行回调函数。而通过下面的代码，创建了所有HTML元素对应的ReactNativeComponent。
```plain
var ReactDOM = objMapKeyVal({
  a: false,
  abbr: false,
  address: false,
  audio: false,
  b: false,
  body: false,
  br: true,
  button: false,
  code: false,
  col: true,
  colgroup: false,
  dd: false,
  div: false,
  section: false,
  dl: false,
  dt: false,
  em: false,
  embed: true,
  fieldset: false,
  footer: false,
  // Danger: this gets monkeypatched! See ReactDOMForm for more info.
  form: false,
  h1: false,
  h2: false,
  h3: false,
  h4: false,
  h5: false,
  h6: false,
  header: false,
  hr: true,
  i: false,
  iframe: false,
  img: true,
  input: true,
  label: false,
  legend: false,
  li: false,
  line: false,
  nav: false,
  object: false,
  ol: false,
  optgroup: false,
  option: false,
  p: false,
  param: true,
  pre: false,
  select: false,
  small: false,
  source: false,
  span: false,
  sub: false,
  sup: false,
  strong: false,
  table: false,
  tbody: false,
  td: false,
  textarea: false,
  tfoot: false,
  th: false,
  thead: false,
  time: false,
  title: false,
  tr: false,
  u: false,
  ul: false,
  video: false,
  wbr: false,

  // SVG
  circle: false,
  g: false,
  path: false,
  polyline: false,
  rect: false,
  svg: false,
  text: false
}, createDOMComponentClass);
```
# Form表单的特殊处理
由于form表单的submit事件在某些浏览器中不会触发冒泡事件，为了保证事件处理的统一形式，React对form进行了特殊处理，实际使用中，form将作为ReactCompositeComponent，而不是ReactNativeComponent，之前生成的ReactNativeComponent对象会被覆盖掉。
