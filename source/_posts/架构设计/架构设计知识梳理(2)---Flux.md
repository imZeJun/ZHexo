---
title: 架构设计知识梳理(2) - Flux
date: 2017-02-21 00:11
categories : 架构设计知识梳理
---

# 一、概述
![Flux.png](http://upload-images.jianshu.io/upload_images/1949836-630d26c3c8a6e456.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`Flux`的特点：
- 数据是单向的
- `App`被分为三个部分
`View`：`App`界面，它根据用户交互创造相应的响应`Action`。
`Dispatcher`：处理中心，接收各种`Action`并路由到对应的`Store`。
`Store`：维护`App`各个模块的数据状态，他们会根据当前的动作`Action`处理不同的业务状态，会产生一个`change`事件来通知`View`更新状态。
- `Action`是一个简单的`Java`对象，包含`type`和`data`。

# 二、`Action`
`Type`：`String`类型标示事件类型。
`Data`：`Map`类型，携带这个动作需要的数据。
```
Bundler data = new Bundle();
data.put("USER_ID", id);
Action action = new ViewAction("SHOW_USER", data);
```
# 三、`Store`
`Store`模块包含了`App`状态和业务逻辑，它能管理各种对象的状态而不是一个。
`Store`必须暴露一个接口来获取`App`状态，这样`View`模块就可以访问`Store`模块来更新`UI`。
![Flux Store.png](http://upload-images.jianshu.io/upload_images/1949836-da7caf666f3815cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`Store`的职责不是用来从外部获取数据，而是用来跟踪通过`Action`获取的数据。
# 四、网络请求和异步回调
异步网络请求是通过`Action Creator`触发的，`API Adapter`模块负责请求`API`并返回结果，最后`Action Creator`会把相应的`Action`和数据发送出去。
`Store`完全是同步模型，这样使`Store`的内部逻辑更简单。
所有的`Action`都是通过`Action Creator`触发的，把所有的`Action`都集中在一个地方，容易排查`bug`。
# 五、`Todo App`源码
`1.1 actions/Action`：一个`Action`对应一个事件，并且包含了该事件相关联的数据。
`1.2 actions/ActionCreator`：这是一个单例模式，他构造时需要传入一个`Dispatcher`对象，封装一些方法，并通过`Dispatcher`分发出去。
```
public static ActionCreator get(Dispatcher dispatcher) { //ActionCreator和Dispatcher类似，都是单例模式。
    if (instance == null) {
        instance = new ActionCreator(dispatcher);
    }
    return instance;
}

public void create(String text) {
    dispatcher.dispatch(TodoActions.TODO_CREATE, TodoActions.KET_TEXT, text);
}
```
`1.3 actions/TodoActions`：封装`Action`相关的常量。

`dispatcher/Dispatcher`：这也是一个单例模式，他构造时需要传入一个`Bus`对象，它提供了`register`和`unRegister`，并通过`bus`发送出去。
```
//构造时和Bus相关联。
public static Dispatcher get(Bus bus) {
    if (instance == null) {
        instance = new Dispatcher(bus);
     }
     return instance;
}
```
`model/Todo`：表示一个事件。
`stores/Store`：它是一个接口，同样和`Dispatcher`相关联，他的子类是`TodoStore`。
整个数据流的流程是：
```
Activity发起事件 -> ActionCreator#xxx -> Dispatcher#dispatch -> Bus#post -> TodoStore#onAction -> TodoStore#emitChange -> Bus#post -> Activity#onTodoStoreChange
```
`Activity`接收数据的流程
```
Activity#onTodoStoreChange -> updateUi -> TodoStore#getTodos()
```
