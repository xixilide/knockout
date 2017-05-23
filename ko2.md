# 自己动手写Knockoutjs - 实现计算属性和计算属性的依赖收集
### 1. 计算属性的定义

先来看下官方文档中计算属性如何使用的。
```js
var myViewModel = { };
myViewModel.firstName = ko.observable("Bob");
myViewModel.lastName = ko.observable("Smith");
myViewModel.fullName = ko.dependentObservable(function () {
return myViewModel.firstName() + " " + myViewModel.lastName();
            })
```
参照observable的定义，先写出一个基本的。
```js
ko.dependentObservable = function(evaluatorFunction, evaluatorFunctionTarget) {
    var _lastValue;

    function dependentObservable() {
        return _lastValue;
    }
    ko.subscribable.call(dependentObservable);
    return dependentObservable;
}
```
### 2. evaluate方法

evaluatorFunction是计算属性的计算方法，需要用它来给_lastValue重新赋值。通过call方法可以定义evaluatorFunction中的this。
evaluatorFunction是计算属性所依赖的属性变化时需要执行的方法，它的执行需要注册到其依赖属性的subscribtiones中，并且还要通知这个计算属性的变化到依赖于它的计算属性。这样，我们需要一个包装方法evaluate，它是observable对象发生变化时的需要执行的subscribtiones。
代码如下：
```js
ko.dependentObservable = function(evaluatorFunction, evaluatorFunctionTarget) {
    var _lastValue;
    function evaluate() {
        _lastValue = evaluatorFunctionTarget ? evaluatorFunction.call(evaluatorFunctionTarget) : evaluatorFunction();

        dependentObservable.notifySubscribers(_lastValue);
    }

    function dependentObservable() {
        return _lastValue;
    }
    ko.subscribable.call(dependentObservable);
    evaluate();

    return dependentObservable;
}
```
### 3. 依赖收集的基本原理

计算属性我们定义好了，接下来就是依赖收集了。
依赖收集就是给observable对象（计算属性也算observable对象）注册计算属性的evaluate方法。
收集的时机就是在执行evaluate方法时，当调用observable对象的get方法时，需要将这个observable对象收集暂存起来，这样我们就得到了计算属性所依赖的observable对象的列表，然后我们需要给列表中的每个observable对象注册这个计算属性的evaluate方法。当所有的计算属性的evaluate方法都执行完一遍时，每个observable对象就都具有了依赖于它的计算属性的evaluate方法。

### 4. 收集暂存箱的实现

收集暂存箱需要在开始的时候初始化一个数组，结束的时候返回这个数组。
```js
ko.dependencyDetection = (function () {
    var _detectedDependencies = [];
    return {
        begin: function () {
            _detectedDependencies.push([]);
        },

        end: function () {
            return _detectedDependencies.pop();
        },

        registerDependency: function (subscribable) {
            if (_detectedDependencies.length > 0) {
                _detectedDependencies[_detectedDependencies.length - 1].push(subscribable);
            }
        }
    };
})();
```
这里我们用一个闭包实现了变量的缓存，`_detectedDependencies`是一个长度为1的数组，用它的pop方法可以巧妙地实现在end的时候消除当前收集数组的依赖并返回这个数组。

### 5. 依赖收集的具体实现

首先在observable对象的get方法中用暂存器收集。
```js
function observable(newValue) {
    if (arguments.length > 0) {
        // set 方法
        _latestValue = newValue;
        observable.notifySubscribers(_latestValue);
    } else {
        // get 方法
        ko.dependencyDetection.registerDependency(observable);
    }
    return _latestValue;
}
function dependentObservable() {
    ko.dependencyDetection.registerDependency(dependentObservable);
    return _lastValue;
}
```
所有的计算属性的依赖收集都只发生一次，就是在定义计算属性时，我们加入_isFirstEvaluation来标识是否是第一次。
```js
var _lastValue,_isFirstEvaluation = true;
function evaluate() {
    _isFirstEvaluation && ko.dependencyDetection.begin();
    _lastValue = evaluatorFunctionTarget ? evaluatorFunction.call(evaluatorFunctionTarget) : evaluatorFunction();
    _isFirstEvaluation && replaceSubscriptionsToDependencies(ko.dependencyDetection.end());

    dependentObservable.notifySubscribers(_lastValue);
    _isFirstEvaluation = false;
}
```
给收集列表中的每个observable对象注册计算属性的evaluate方法。
```js
function replaceSubscriptionsToDependencies(dependencies) {
dependencies.forEach(function(dependence) {
        dependence.subscribe(evaluate);
    });
}
```
