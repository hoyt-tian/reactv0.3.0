# DOM缓存机制

在进行各种操作时，时常涉及到DOM树的查询操作，为了加速这个过程，React设计了一套节点缓存机制，以额外的存储空间换取更高的查询效率。
# ReactDOMNodeCache
nodeCache中存储了id和node的键值对。
```javascript
var nodeCache = {};

/**
 * DOM node cache only intended for use by React. Placed into a shared module so
 * that both read and write utilities may benefit from a shared cache.
 */
var ReactDOMNodeCache = {
  /**
   * Releases fast id lookups (node/style cache). This implementation is
   * aggressive with purging because the bookkeeping associated with doing fine
   * grained deleted from the cache may outweight the benefits of the cache. The
   * heuristic that should be used to purge is 'any time anything is deleted'.
   * Typically this means that a large amount of content is being replaced and
   * several elements would need purging regardless. It's also a time when an
   * application is likely not in the middle of a "smooth operation" (such as
   * animating/scrolling).
   */
  purgeEntireCache: function() {
    nodeCache = {};
    return nodeCache;
  },
  getCachedNodeByID: function(id) {
    return nodeCache[id] ||
      (nodeCache[id] =
        document.getElementById(id) ||
        ReactMount.findReactRenderedDOMNodeSlow(id));
  }
};
```
当试图根据id查找某个节点时，首先从nodeCahce中查找，如果没有，则根据document.getElementById查找，如果仍然没有，则根据# ReactMount.findReactRenderedDOMNodeSlow(id)
# ReactMount.findReactRenderedDOMNodeSlow
如果nodeCache和documen中都不存在对应id的节点，说明有可能是一个React渲染的节点。此时，先通过ReactMount.findReactContainerForID找到包含这个id的React容器，然后再通过ReactInstanceHandles.findComponentRoot在React容器中查找包含id的react组件。
```javascript
findReactRenderedDOMNodeSlow: function(id) {
    var reactRoot = ReactMount.findReactContainerForID(id);
    return ReactInstanceHandles.findComponentRoot(reactRoot, id);
  }
```
findReactContainerForID是根据某个给定的id，上溯找到这个id对应的react实例。所有React渲染的节点，其id都遵循特定的格式，ReactRootID的生成代码如下，其中mountPointCount是一个全局计数器：
```javascript
 getReactRootID: function(mountPointCount) {
    return '.reactRoot[' + mountPointCount + ']';
  },
 findReactContainerForID: function(id) {
    var reatRootID = ReactInstanceHandles.getReactRootIDFromNodeID(id);
    // TODO: Consider throwing if `id` is not a valid React element ID.
    return containersByReactRootID[reatRootID];
  }
```
containersByReactRootID是一个map，它以id为key，DOM节点为值，可以根据给定的id查询其对应的dom节点，在节点被渲染时，其信息会被注册到containersByReactRootID中，相关代码如下：
```javascript
registerContainer: function(container) {
    var reactRootID = getReactRootID(container);
    if (reactRootID) {
      // If one exists, make sure it is a valid "reactRoot" ID.
      reactRootID = ReactInstanceHandles.getReactRootIDFromNodeID(reactRootID);
    }
    if (!reactRootID) {
      // No valid "reactRoot" ID found, create one.
      reactRootID = ReactInstanceHandles.getReactRootID(
        globalMountPointCounter++
      );
    }
    containersByReactRootID[reactRootID] = container;
    return reactRootID;
  }
```
而在找到了根容器节点和带查询的id时，就可以开始在根容器节点中查找这个id对应的内容了，其查询代码如下：
```javascript
findComponentRoot: function(ancestorNode, id) {
    var child = ancestorNode.firstChild;
    while (child) {
      if (id === child.id) {
        return child;
      } else if (id.indexOf(child.id) === 0) {
        return ReactInstanceHandles.findComponentRoot(child, id);
      }
      child = child.nextSibling;
    }
    // Effectively: return null;
  }
```
如果根节点的某个子节点id与目标id相等，则返回该节点；如果子节点的id比目标节点更长，但其id的起始部分与目标节点一致，同时还包含了其他后缀，根据React的节点命名规范，该子节点必然是待查询id的根容器节点，此时递归调用findComonentRoot即可。
