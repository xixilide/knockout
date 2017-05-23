# 自己动手写Knockoutjs - 可监控数组
### 1. 测试用例
```js
<select id="select1" multiple="multiple" style="width: 200px" data-bind="options:items"></select>
<button onclick="addItem()">添加</button>
<button onclick="delLastItem()">删除最后一项</button>
<button onclick="reverse()">反转</button>

<script src="knockout.js"></script>
<script>
    var myViewModel = { };
       myViewModel.items = ko.observableArray(["Alpha", "Beta", "Gamma"])
    ko.applyBindingsToNode(myViewModel, document.getElementById('select1'));
    function addItem() {
        myViewModel.items.push('Added');
    }
    function delLastItem() {
        myViewModel.items.pop();
    }
    function reverse() {
        myViewModel.items.reverse();
    }
</script>
```
### 2. 可监控数组的定义

不同于一般的observable对象，可监控数组需要有更多的操作api，如push、pop等。这样，我们可以通过继承observable，然后再加入扩展的方法。
```js
ko.observableArray = function(initialValues) {
    var result = new ko.observable(initialValues);
    return result;
}
```
### 3. 扩展方法的实现

通过result()可以得到原生的数组对象，在扩展方法里需要做两件事：操作原生数组；通知subscribtiones。
```js
["pop", "push", "reverse", "shift", "sort", "splice", "unshift"].forEach(function(methodName) {
    result[methodName] = function() {
        var underlyingArray = result();
        var methodCallResult = underlyingArray[methodName].apply(underlyingArray, arguments);
        result.valueHasMutated();
        return methodCallResult;
    }
});
valueHasMutated的实现放在observable中。

observable.valueHasMutated = function () {
    observable.notifySubscribers(_latestValue);
}
```
对于像slice这样的方法，当其被调用时是不需要发出通知的。
```js
['slice'].forEach(function(methodName) {
    result[methodName] = function() {
            var underlyingArray = result();
               return underlyingArray[methodName].apply(underlyingArray, arguments);
        }
});
```
