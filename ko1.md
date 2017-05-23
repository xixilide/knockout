# 自己动手写Knockoutjs - 实现基本的双向数据绑定

`如何实现一个简化版本的Knockout`， Knockout的源码现在已经有五千多行，这个版本的ko仅有七十行，实现了基本的双向数据绑定、两个绑定（value、text）。

### 1. 实现ko.observable

ko对象是一个function对象

ko.observable = function (initialValue) {
    function observable(newValue) {
    }
    return observable;
}
要想知道数据发生了变化，必须要保存之前的数据，在这里可以用闭包保存老数据。
```js

ko.observable = function (initialValue) {
var _latestValue = initialValue;
function observable(newValue) {
        if (arguments.length > 0) {
            _latestValue = newValue;
            observable.notifySubscribers(_latestValue);
        }
        return _latestValue;
    }
    return observable;
}
```
在ko中对于数据的set和get方法，可以通过参数的多少来判断，当有参数传入时，即是set方法。
数据变化的监听用观察者模式实现。
```js
ko.subscribable = function () {
    var _subscriptions = [];

    this.subscribe = function (callback) {
        _subscriptions.push(callback);
    };

    this.notifySubscribers = function (valueToNotify) {
        for(var i = 0; i < _subscriptions.length;i++) {
            _subscriptions[i](valueToNotify);
        }
    };
}
```
在ko.observable用call来继承ko.subscribable的属性，ko.subscribable.call(observable);。

### 2. 自定义绑定的实现

自定义绑定要定义两件事情：数据变化时操作dom来改变视图；监听dom的变化，定义变化后去改变数据的方法。
```js
ko.bindingHandlers.value = {
    init: function(element, value) {
        element.addEventListener("change", function () { value(this.value); }, false);
    },
    update: function (element, value) {
        element.value = value();
    }
};
```
### 3. ko.applyBindings 的实现

ko.applyBindings 是view和model 绑定的入口。它需要
 1. 解析html代码中的绑定
 2. 执行自定义绑定中的init
 3. 给model注册监听，监听的回调要做的事情是调用自定义绑定的update方法。
这里仅实现了单个dom node和model的绑定。
```js
ko.applyBindingsToNode = function (viewModel, node) {
    var isFirstEvaluation = true;
    function evaluate() {
        var parsedBindings = parseBindingAttribute(node.getAttribute(bindingAttributeName), viewModel);
        for (var bindingKey in parsedBindings) {
            if (ko.bindingHandlers[bindingKey]) {
                if (isFirstEvaluation && typeof ko.bindingHandlers[bindingKey].init == "function") {
                    ko.bindingHandlers[bindingKey].init(node, parsedBindings[bindingKey]);
                    parsedBindings[bindingKey].subscribe(evaluate);
                }
                if (typeof ko.bindingHandlers[bindingKey].update == "function") {
                    ko.bindingHandlers[bindingKey].update(node, parsedBindings[bindingKey]);
                }
                if (isFirstEvaluation) parsedBindings[bindingKey].subscribe(evaluate);
            }
        }
    }

    evaluate();
    isFirstEvaluation = false;
};
```
