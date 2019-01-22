
## dva
React不同于Angular，它是单向的数据流，所以当子组件要向父组件传值的时候，就会造成困难，
过去的做法是使用钩子函数从子组件的响应函数中获取数据来更新父组件的状态，但这样在开发上是非常繁琐的。
于是fb出了一个用于管理react数据流的工具叫flux,但在业界被接受的程度并不高，于是Redux诞生了。
但是如果直接去写Redux的话，会发现写法是极其繁琐的，有很多很机械和冗余的代码，阿里的一个大神由于受不了
这些繁琐的写法，发明了dva,这个大神也受不了webpack繁琐的配置发明了roadhog ，然后dva + redhog =》umi,
umi + antd pro => antd pro2。

所以简单来说，dva是为了简化redux的使用提供的框架，同时集成了用来管理应用副作用和同步解决方案的redux-saga。
    
## redux-saga
[redux-saga](https://zhuanlan.zhihu.com/p/23012870)是一个用于管理应用程序 Side Effect（副作用，例如异步获取数据，访问浏览器缓存等）的 library，它的目标是让副作用管理更容易，执行更高效，测试更简单，在处理故障时更容易。

在实际开发中，我们在view层触发一个事件，可能会需要调用多个接口，同时在接口的调用上有先后关系，比如需要现调A接口，拿到数据a,同时发起一个action把a保存，同时在利用a去调用接口B,获取数据b,再把数据b保存在state中。同时
这一系列的操作还可能在很多地方被调用，这个时候使用redux-saga就会很方便。
它类似于[redux-thunk](https://juejin.im/post/5b035c0c51882565bd258f12)可以将从发起请求到把数据存入Redex的state的整个过程，或者是多个action抽象成一个effects,以便于复用和管理。
同时利用[generator](http://es6.ruanyifeng.com/#docs/generator-async)来实现异步调用的同步写法，方便effects的编程。

## immutable 
immutable是指不可变的数据结构的意思，因为在react里diff算法判断一个组件是否需要触发render的一个很重要的因素是看这个组件的state或props是否发生变化，但是这种是否发生变化的判断只是一个简单的浅比较，如果一个对象它只是里面的某个数据改变了，
但是对象本身没有改变，那么这种改变是不能被react感知的，这个时候就是改变了数据也不会刷新，为了避免这个问题，建议在react中都使用immutable的数据结构。
要实现immutable，一般来说比较简单的方式是利用 [解构运算符](http://es6.ruanyifeng.com/#docs/destructuring)
例如： 
` let a = [1,2,3]; let c = [...a,4] `
然而这种写法在用到比较复杂的数据结构比较深的场景时，就不好用了。这个时候使用facebook官方出品的 [immutableJs](https://facebook.github.io/immutable-js/docs/#/) 是个不错的选项    


## reselect
[reselect](https://www.jianshu.com/p/1fcef4c892ba)是一个可以用来缓存计算的工具，只有当传入reseletor的参数变化是，才会触发reselector的重新计算。

       