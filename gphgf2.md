# 选区保存与还原

在ReactReconcileTransaction中，定义了一个处理选区信息的拦截器。如果没有针对选区信息的处理，被选中内容被重新渲染后，选区、光标、焦点的信息就会丢失。因此，在事务开始前，相关信息会被获取并保存下来，而当事务结束阶段，选区信息又会被还原到原来的位置。

要支持选区信息，必须是textare或者input（input的type为text），或者contentEditable属性为true。事务执行前，通过window.getSelection即可获取到当前选区的信息，选区的起始、结束索引会保存下来，当前获取焦点的节点也会保存下来，作为拦截器初始化的返回值。
```javascript
getSelectionInformation: function() {
    var focusedElem = getActiveElement();
    return {
      focusedElem: focusedElem,
      selectionRange:
          ReactInputSelection.hasSelectionCapabilities(focusedElem) ?
          ReactInputSelection.getSelection(focusedElem) :
          null
    };
  }
```

等到事务结束阶段，拦截器的close方法会被调用，之前保存的焦点信息和选区信息，此时都会作为参数传递给close方法，而close方法也就是ReactInputSelection中的restoreSelection方法。该方法执行时，会比较当前的焦点元素和之前的焦点元素，若两者不一样，那么让之前的元素重新获取焦点。如果之前的元素支持选区，还会通过setSelection方法更新选区信息。

```javascript
 restoreSelection: function(priorSelectionInformation) {
    var curFocusedElem = getActiveElement();
    var priorFocusedElem = priorSelectionInformation.focusedElem;
    var priorSelectionRange = priorSelectionInformation.selectionRange;
    if (curFocusedElem !== priorFocusedElem &&
        document.getElementById(priorFocusedElem.id)) {
      if (ReactInputSelection.hasSelectionCapabilities(priorFocusedElem)) {
        ReactInputSelection.setSelection(
          priorFocusedElem,
          priorSelectionRange
        );
      }
      priorFocusedElem.focus();
    }
  }
```

# 选区操作
有了选区的起始、结束信息，要还原它就比较简单了。通过createTextRange或者createRange接口，就可以创建一段选区，随后通过moveStart/moveEnd或者setStart/setEnd设置选区的开始、结束位置，相关代码如下
```javascript
 setSelection: function(input, rangeObj) {
    var range;
    var start = rangeObj.start;
    var end = rangeObj.end;
    if (typeof end === 'undefined') {
      end = start;
    }
    if (document.selection) {
      // IE is inconsistent about character offsets when it comes to carriage
      // returns, so we need to manually take them into account
      if (input.tagName === 'TEXTAREA') {
        var cr_before =
          (input.value.slice(0, start).match(/\r/g) || []).length;
        var cr_inside =
          (input.value.slice(start, end).match(/\r/g) || []).length;
        start -= cr_before;
        end -= cr_before + cr_inside;
      }
      range = input.createTextRange();
      range.collapse(true);
      range.moveStart('character', start);
      range.moveEnd('character', end - start);
      range.select();
    } else {
      if (input.contentEditable === 'true') {
        if (input.childNodes.length === 1) {
          range = document.createRange();
          range.setStart(input.childNodes[0], start);
          range.setEnd(input.childNodes[0], end);
          var sel = window.getSelection();
          sel.removeAllRanges();
          sel.addRange(range);
        }
      } else {
        input.selectionStart = start;
        input.selectionEnd = Math.min(end, input.value.length);
        input.focus();
      }
    }
  }
```

