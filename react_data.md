# 数据的存放
##Redux store 的state(以后简称store)
### 全局数据
store是一个存在于全局范围内的对象，并且其数据几乎可以在任何地方被获取或发起修改，所以store适合保存全局需要使用的数据。
### 当有任意两个组件需要通讯时
react是一个单向数据流的框架，只有父级向子级通过props传递数据是比较方便的，父组件想向孙组件传递数据传统的做法需要传两次以上，当然也可以使用[context](https://react.docschina.org/docs/context.html) 来传递，但比较复杂。实际上redux的数据能够跨组件传递正是基于这一原理。
最方便的做法还是通过store来进行通讯。

    /**
     * 购物车组件
     * 用于购物车商品的展示
     */
    //从store中拿到购物列表数据
    @connect(({cashier})=>({cartPros:cashier.cartPros}))
    class Cart extends React.Component{
      render(){
        //展示购物列表
        const {cartPros} = this.props
        return (
            <View>
              {
                cartPros.map(v=><View><Text>{v.name}</Text></View>)
              }
            </View>
        )
      }
    }
    
    /**
     * 商品列表
     */
    @connect()
    class CommodityList extends React.Component{
      render(){
        const {list} = this.props
        //展示商品列表 并在商品被点击时 发送一个dispatch 把被点击的商品加到store的cartPros 里面
        return (
            <View>
              {
                list.map(v=> <Commodity data={v} onPress={v=>this.props.dispatch({type:"cashier/addCartPros",payload:v})} />)
              }
            </View>
        )
      }
    }
    
    /**
     * 主页面 左右布局 购物车在左 商品列表在右 他们是兄弟关系
     */
    class Index extends React.Component{
      render(){
        return (
            <View style={{flexDirection:"row"}}>
              <Cart></Cart>
              <CommodityList></CommodityList>
            </View>
        )
      }
    }

## 组件自身的state
### 和组件自身高度相关的数据
只在本组件或其子组件用到的数据就可以存在组件自身的state里，其实并不是不能放在store，只是如果store太大，modal太长，对于管理，维护和代码的查找来说都是很不好的体验，所以和组件高度相关的数据建议放在组件自身的state里。

    class User extend React.Component{
        state = {userInfo:{}}
        componentDidMount(){
            fetch(httpConfig.getUserInfo).then(res=>this.setState({userInfo:res.data}))
        }
        render(){
            return (
                userInfo.avatar 
                ...
            )
        }
    }

### 组件状态
组件独有的状态数据，比如选项卡组件里面的activeKey,或是具有展开收起功能的展开收起状态等
    
    /**
     * 选项卡组件
     */
    export default class TabsCard extends React.PureComponent{
      state = {
        activeKey:""
      }
    
      handleTabClick(data,k){
        this.setState({activeKey:k})
        if( typeof this.props.onChange === "function"){
          this.props.onChange(data)
        }
      }
    
      render(){
        const {data = [],page} = this.props
        const activeKey = typeof page === "undefined" ? this.state.activeKey : page
        return(
            <View>
              <View>
                {
                  data.map((v,k)=>{
                    const active = activeKey === k
                    return (
                        <TouchableNativeFeedback key={k} onPress={()=>{this.handleTabClick(v,k)}}>
                          <View style={[styles.item,active ? styles.activeItem : {}]} >
                            <Text color={active ? "blue" : "grey"}>{v.title}</Text>
                          </View>
                        </TouchableNativeFeedback>
                    )
                  })
                }
              </View>
              <View>
                {this.props.children[activeKey]}
              </View>
            </View>
        )
      }
    }
    
    

# 数据的获取
## effects
effects的本质是dva集成的[redux-saga](conception.md#redux-saga),通过generator来将异步的方法改造成同步的方法，同时可以根据状态来调用多个reducer，
effects的获取到的数据一般通过put方法调用reducer来保存再store里。

    /**
     * 选择店铺的页面的数据初始化
     * @param payload
     * @param call
     * @param put
     * @param select
     * @returns {IterableIterator<*>}
     */
    *shopManageInit({payload},{call,put,select}){
      //从store中拿到当前的登录的用户数据
      const user = yield select(state=>state.app.get("currentUser"))
      //获取该用户下绑定的店铺列表和被邀请的列表
      const shops = yield call(fetch,httpConfig.getShopByUserId,{data:{userId:global.userId}})
      const inviteShops = yield call(fetch,httpConfig.getInviteList,{data:{inviteMobile:user.mobile}})
      //获取该收银机绑定的店铺
      const devicesOfShopData = yield call(fetch,httpConfig.getDeviceData, {data:{deviceName:global.deviceName}})
      //把以上数据保存到store中
      yield put({type:"setShopManageData",payload:{shops,inviteShops:inviteShops.list,devicesOfShop:devicesOfShopData.list}})
    },

### 可能会被重复调用的行为
比如注销，有些应用可能不止一个注销入口，但是注销流程基本一致，这种情况下，写在effects里，有利于行为的重复使用。

### 需要把数据存到store里面的接口调用
当然在组件里面请求接口之后，在通过dispatch直接传给store也是可行的

    fetch(httpconfig.getUser,{data:{userId:9527}}).then(res=>this.props.dispatch({type:"app/setUser",payload:res.data}) ) 

### 复杂的行为管理
利用generator的同步编程机制来处理很多复杂的行为是比较方便的。

     /**
     * 这是一段 带下拉加载 上拉更新 参数修改自动更新 分页等等功能 的 infinity list的数据修改逻辑
     * 每次调用时分三种状态 每种状态参数和策略不一样 先刷新页面 展现loading等 然后获取数据 数据到后 展示数据 取消loading
     *
     * isLoading:数据初始化
     * refresh:下拉刷新
     * pull:向下滑动 分页加载数据
     * @param payload
     * @param call
     * @param put
     * @param select
     */
    *fetchData({payload},{call,put,select}){
      const isHasMore = data=>{
        const haveMore = data.pages > data.pageNum
        return haveMore
      }
      const {key,url,param={},shouldUpdateData} = payload;
      const state = yield select(state=>state.longList.toJS() );
      const {parameter={}} = state;
      const {pageNum} = parameter
      let newPageNum = pageNum;
      switch (key){
        case "isLoading":
          newPageNum = 1
          yield put({type:'setState',payload:{key:'isLoading',value:true}})
          break;
        case "refresh":
          newPageNum = 1
          yield put({type:'setState',payload:{key:'refresh',value:true}})
          break;
        case "pull":
          newPageNum++
          yield put({type:'setState',payload:{key:'footerLoading',value:true}})
          break;
      }
      // console.log("fetchData longList url",url)
      if(!url){
        return
      }
      const {sort = {}} = param
      let requestData = {}
      if(sort.index && sort.method){
        const newSort = {orderByName:sort.index,desc:sort.method === "desc" ? "descend" : "ascend"}
        requestData = Object.assign({},parameter,param,{pageNum:newPageNum,...newSort},{sort:null});
      }else{
        requestData = Object.assign({},parameter,{pageNum:newPageNum},param,{sort:null});
      }
    
      // const requestData = Object.assign({},parameter,{pageNum:newPageNum},param);
      let haveMore = false
      let data = {list:[]}
      let responseData = {}
      try{
        responseData = yield call(fetch,url,{data:requestData},true)
        data = responseData.data || requestData
        haveMore = isHasMore(data);
      }catch (e) {
        haveMore = false
      }
      let extProps = {data:data.list}
      if(typeof shouldUpdateData === "function"){
        try{
          if(!shouldUpdateData(responseData,key)){
            return
          }
        }catch (e) {}
      }
      switch (key){
        case "isLoading":
          yield put({type:'setData',payload:{key:'isLoading',value:false,haveMore,...extProps}})
          break;
        case "refresh":
          yield put({type:'setData',payload:{key:'refresh',value:false,...extProps,haveMore}})
          break;
        case "pull":
          yield put({type:'pushData',payload:{...extProps,haveMore}})
          break;
        default:
          break;
      }
      yield put({type:"setPageNum",payload:newPageNum})
    }


## 写在组件里
当你的需要数据不是存在store里，而是在组件自身的state里时，请求也只能写在组件里了，如果是那种需要一开始就请求的数据，一般都是写在componentDidMount，而不是constructor里面，一个是为了提高首次响应的时间，
二是万一请求的数据返回了，而首次render都还没开始，（虽然不太可能），可能在回调里有些操作就会有问题了。

    class User extend React.Component{
            state = {userInfo:{}}
            componentDidMount(){
                fetch(httpConfig.getUserInfo).then(res=>this.setState({userInfo:res.data}))
            }
            render(){
                return (
                    userInfo.avatar 
                    ...
                )
            }
        }

# redux的reducer的职责
reducer是redux规定的唯一可以修改state的地方，并且要求reducer要是没有副作用的纯函数，所以大部分情况下reducer就只是简单的把数据merge到state中。
因为每次reducer执行完state被修改之后都会触发订阅这个数据的组件的更新（diff）,如果一个动作，需要修改三个state,那么最好把这个动作作为这个reducer的name,然后再这个reducer中同时更新这三个state。
而不是调用3个reducer来执行这一个动作，所以reducer的粒度应该是行为。
eg:会员登录

    {
      namespace:"cashier",
      state:STATE,
      reducers:{
        //一个会员登录动作 需要修改三个数据 所以这三个数据应该同时修改 尽管有单独修改的接口
        memberLogin(state,{payload}){
          const {memberInfo，isMember} = payload
          return state.set("memberInfo",memberInfo).set("pageKey","cash").set("isMember",isMember)
        },
      },
      effects:{
        *memberMobileChange({payload},{call,put}){
          const mobile = payload;
            //拿手机号去登录会员 把得到的信息保存
            const res = yield call(fetch,httpConfig.memberLogin,{data:{mobile}},true)
            ... 
            yield put({type:"memberLogin",payload:{isMember:false,memberInfo:res.data}})
        },
      }
    }


# 数据的计算
## 状态数据
### props计算state
react 官方说把props的数据传到state的行为是不被建议的"反模式"（anti pattern），原因之一是如果把props直接传给state，如果类型是对象的话，他们会公用同一段内存，那么state的改变，便会引发props，不应该发生的改变。
但是实际使用中，偶尔也会有需要props传到state的时候，比如菜单里面的activeKey，有时候是需要在props里指定，同时在菜单组件内部的state中管理的。这个时候使用immutable的数据结构，或许是能解决问题的办法。当然如果仅仅是个string/number
应该没什么问题。
一般用props计算state最好只计算一次，放在constructor中会比较合适，react 16.4后，新增了[getDerivedStateFromProps](https://zhuanlan.zhihu.com/p/38030418),从名字就可以看出来，这就是专门通过props计算state的地方了。
## 衍生数据
衍生数据是指由state props或是store 里面的state 计算而来的数据。
之所以这些数据不保存，是因为他们一般都是由两个以上的state或props 等组合而成，每次修改其中一个state都要跟着修改其衍生数据，开发上比较麻烦，还容易出错，尤其如果组合的计算公式比较复杂，还会造成很多不必要的计算开销。
根据衍生数据与直接数据之间的相关关系，我把衍生数据分为以下几类
### 与state相关
这种数据我一般放在组件类里面，然后通过类的 [getter](http://es6.ruanyifeng.com/#docs/class#%E5%8F%96%E5%80%BC%E5%87%BD%E6%95%B0%EF%BC%88getter%EF%BC%89%E5%92%8C%E5%AD%98%E5%80%BC%E5%87%BD%E6%95%B0%EF%BC%88setter%EF%BC%89) 来在被使用时才计算数据，并且统一了数据的读取方式，方便修改和防止忘记生成数据。如果数据的计算比较耗时，不妨考虑在计算时使用 [reselect](conception.md#reselect)

    class SettleAcount extends React.PureComponent{
        this.state = {
              receiveMoney:0.0, //客户支付的钱
        }
        change = 0.0    //找零
        get change(){
            let {receiveMoney} = this.state;//客户支付的钱 从input读入
            let realReceiveMoney = parseFloat(this.realReceiveMoney) ; //客户实际要支付的钱 这也是个衍生数据
            //当客户支付的钱大于实际要支付的钱时 找零 = 客户支付的钱 - 实际要支付的钱
            if(receiveMoney > realReceiveMoney){
              return receiveMoney - realReceiveMoney
            }else{
              return 0;
            }
        }
        render(){
            //每次访问change的时候 都会执行 change的getter    
            return <View>{this.change}</View>
        }
    }



### 与state无关，但与store传过来的数据有关
也就是对在store里的数据进行衍生计算，这种计算一般结合[reselect](conception.md#reselect)放在 [connect](https://www.redux.org.cn/docs/react-redux/api.html)的mapStateToProps里面
    
    //reselect 可以保证相同数据只计算一次
    import {createSelector} from "reselect"
    
    //计算结算页面的商品小计
    //这边从store的state中读取 cartPros 这是一个用来存放用户购买的商品的数组
    //但是商品的数据结构里只有价格 数量 折扣 没有小计，并且以上数据是可以变化的
    //所以要利用以上数据计算小计
    const createCartProsSelectorOfSettler = createSelector(state=>state.get("cartPros").toJS(),state=>state.get("isMember"),(cartPros,isMember)=>{
      return cartPros.map(v=>({...v,subtotal:tools.getCommoditySubtotalPrice(v,"",isMember),finalPrice:tools.getFinallyPrice(v,isMember)}))
    })
    
    @connect(({cashier,app}) => {
      return {
        app,
        cartPros:createCartProsSelectorOfSettler(cashier),// cashier里面有购物车数据cartPros
      }
    } )
    @toJS
    export default class SettleAcount extends React.Component{
    render(){
        const {cartPros} = this.props;
        return (
          <View>
          {
            cartPros.map(v=>{
                return (
                 <Text>{v.name}</Text>
                 <Text>{v.count}</Text>
                 <Text>{v.price}</Text>
                 <Text>{v.discount}</Text>
                 <Text>{v.subtitle}</Text>
                )
            })
          }
           
          </View>
        )
      }
    }

       