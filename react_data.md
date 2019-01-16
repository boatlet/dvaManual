# 数据的存放
##Redux store 的state(以后简称store)
### 全局数据
store是一个存在于全局范围内的对象，并且其数据几乎可以在任何地方被获取或发起修改，所以store适合保存全局需要使用的数据。
### 当有任意两个组件需要通讯时
react是一个单向数据流的框架，只有父级向子级通过props传递数据是比较方便的，父组件想向孙组件传递数据传统的做法需要传两次以上，当然也可以使用[context](https://react.docschina.org/docs/context.html) 来传递，但比较复杂。实际上redux的数据能够跨组件传递正是基于这一原理。
最方便的做法还是通过store来进行通讯。

## 组件自身的state
### 和组件自身高度相关的数据
只在本组件或其子组件用到的数据就可以存在组件自身的state里，其实并不是不能放在store只是，如果store太大，modal太长，对于管理，维护和代码的查找来说都是很不好的体验，所以和组件高度相关的数据建议放在组件自身的state里。

### 组件状态
组件独有的状态数据，比如菜单里面的activeKey,或是具有展开收起的展开收起状态等

# 数据的获取
## effects
effects的本质是dva集成的[redux-saga](conception.md#redux-saga),通过generator来将异步的方法改造成同步的方法，同时可以根据状态来调用多个reducer，
所以需要修改

### 可能会被重复调用的行为
比如注销，有些应用可能不止一个注销入口，但是注销流程基本一致，这种情况下，写在effects里，有利于行为的重复使用。

### 需要把数据存到store里面的接口调用
当然在组件里面请求接口之后，在通过dispatch直接传给store也是可行的

### 复杂的行为管理
利用generator的同步编程机制来处理很多复杂的行为是比较方便的。

## 写在组件里
当你的需要数据不是存在store里，而是在组件自身的state里时，请求也只能写在组件里了，如果是那种需要一开始就请求的数据，一般都是写在componentDidMount，而不是constructor里面，一个是为了提高首次响应的时间，
二是万一请求的数据返回了，而首次render都还没开始，（虽然不太可能），可能在回调里有些操作就会有问题了。

# redux的reducer的职责
reducer是redux规定的唯一可以修改state的地方，并且要求reducer要是没有副作用的纯函数，所以大部分情况下reducer就只是简单的把数据merge到state中。
因为每次reducer执行完state被修改之后都会触发订阅这个数据的组件的更新（diff）,如果一个动作，需要修改三个state,那么最好把这个动作作为这个reducer的name,然后再这个reducer中同时更新这三个state。
而不是调用3个reducer来执行这一个动作，所以reducer的粒度应该是行为。

# 数据的计算
## 状态数据
### props计算state
react 官方说把props的数据传到state的行为是不被建议的"反模式"（anti pattern），原因之一是如果把props直接传给state，如果类型是对象的话，他们会公用同一段内存，那么state的改变，便会引发props，不应该发生的改变。
但是实际使用中，偶尔也会有需要props传到state的时候，比如菜单里面的activeKey，有时候是需要在props里指定，同时在菜单组件内部的state中管理的。这个时候使用immutable的数据结构，或许是能解决问题的办法。当然如果仅仅是个string/number
应该没什么问题。
一般用props计算state最好只计算一次，放在constructor中会比较合适，react 16.4后，新增了[getDerivedStateFromProps](https://zhuanlan.zhihu.com/p/38030418),从名字就可以看出来，这就是专门通过props计算state的地方了。
## 衍生数据
### 与state相关
### 与state无关，props相关
    
       