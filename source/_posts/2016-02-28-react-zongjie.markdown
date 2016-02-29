---
layout: post
title: "React 总结"
date: 2016-02-28 21:20:01 +0800
comments: true
categories: 
---

## 前言
啊，React 的项目完结到现在已经有一个月了，之前一直唠叨着说昨晚项目要写篇总结，无奈一直拖拖拖拖，拖到现在，还是写了它吧。

## 项目
这个前端项目用的是： 

1. React
2. Redux
3. React Router
4. ES6
5. Babel


### React
React是Facebook开发的一款JS库, 目的是为了构建那些数据会随着时间改变的大型应用。

主要原理则是`虚拟 DOM` 以及 `Component 组件`

React有个diff算法，更新Virtual DOM并不保证马上影响真实的DOM，React会等到事件循环结束，然后利用这个diff算法，通过当前新的dom表述与之前的作比较，计算出最小的步骤更新真实的DOM。

### Redux

Redux 是应用状态管理服务，用来管理 React 中的 state，应用中所有的 state 都以一个对象树的形式储存在一个单一的 store 中。

React 算是传统 MVVM 意义上的 View 层，负责来渲染页面，数据管理方面则需要搭配其他的库，比如出名的是 Facebook 出的 flux 以及 Redux, 但目前来说，Redux 使用的人要多于 flux


### React Router

`React Router` 是完整的 React 路由解决方案，可以根据不同的`URL`来加载不同的`component`

## 要点
基本知识在这里就不啰嗦了，在线看文档就够了。在这里写下之前所记的一些要点吧。

###1、action
`action` 用来表达发生的事件，如果要改变`state`，必须统一通过发`action`

###2、reducer
`reducer`用来接收之前的 `state` 以及 `action`, 从而返回新的`state`
```ruby
function todoApp(state = initialState, action) {
  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return Object.assign({}, state, {
        visibilityFilter: action.filter
      });
    case COMPLETE_TODO:
      return Object.assign({}, state, {
        todos: todos(state.todos, action)
      });
    default:
      return state;
  }
}
```
默认应该返回`旧state`,注意在返回`新state`时，不要对`旧state`做出改变。
###3、component
 创建组件，就相当于 class xx extends Component
```ruby
# 创建组件
React.createClass()

# 相当于
class ChainSelectBox extends Component {

}
```
###4、connect()
任何一个 connect() 的组件，都可以得到 dispatch 方法作为组件的 props, 以及得到全局的 state.一次我们一般会在较顶级的组件进行`connect()`操作，并且将数据传入子组件进行使用。

```ruby
class ApprovalChainManage extends Component {

    render() {
        return <ChildComponent />
    }
}

# 当 state 有更新的时候，就会被调用
const mapStateToProps = ( state ) => {
  return {
    approvalChainEntities:       state.ApprovalChainEntity
  }
};

# 将 actions 传入 props
const mapDispatchToProps = ( dispatch ) => ( {
  approvalChainActions:       bindActionCreators( approvalChainActions, dispatch )
} );


export default connect( mapStateToProps, mapDispatchToProps )( ApprovalChainManage );
```

如果父组件以及子组件都有`connect()`的操作，那么当`state`有变化时，两者的`mapStateToProps()`方法都会被调用。具体的说应该是，当`state`有变化时，所有正在被加载的`connect()`过的组件，内的`mapStateToProps()`方法都会被调用

```ruby
# 子组件
class ChildComponent extends Component {

    render() {
        # JSX 语法
        return (
            <div>
                <span>Yo yo</span>
            </div>
        )
    }
}

const mapStateToProps = ( state ) => {
  return {
    approvalChainEntities:       state.ApprovalChainEntity
  }
};

const mapDispatchToProps = ( dispatch ) => ( {
   studentActions:       bindActionCreators( StudentActions, dispatch )
} );


export default connect( mapStateToProps, mapDispatchToProps )( ApprovalChainManage );


# 父组件
class ApprovalChainManage extends Component {

    render() {
        return <ChildComponent />
    }
}

# 当 state 有更新的时候，就会被调用
const mapStateToProps = ( state ) => {
  return {
    approvalChainEntities:       state.ApprovalChainEntity
  }
};

# 将 actions 传入 props
const mapDispatchToProps = ( dispatch ) => ( {
  approvalChainActions:       bindActionCreators( approvalChainActions, dispatch )
} );


export default connect( mapStateToProps, mapDispatchToProps )( ApprovalChainManage );
```

###5、Store
当用户进行`刷新`操作时，`Store`内的数据是会被清空的，所有组件重新加载，相当于第一次打开页面，这点要切记。

Store 内的值是所有人共享的，如果你在组件 A 执行了 action, 更新了一些数据，并且跳往其他页面，此时这些数据也是一直存在的。

###6、mapStateToProps

该函数返回的值会植入该组件的`props`，并且当`state`有更新时，会调用该方法。

不过细心的人会发现，第一次打开页面，`mapStateToProps()`会被执行两次

```ruby
# 第一次是因为初始化，第二次是因为connect
const mapStateToProps = ( state ) => {
  console.log("开始 state cashview")
  return {
    entities: state.cashEntity
  }
};
```

###7、通过路由来统一管理加载页面

```ruby
 <Route path='/' component={Root}>
    <IndexRoute component={LoginView} />
      <Route path='cash' component={CashView}>
        <Route path='product/:id' component={CashProductDetail}></Route>
        <Route path='index' component={CashIndex}></Route>
      </Route>
 </Route>

```
一般来说几个类似的页面会统一挂在某个`view`下，作为子组件，比如`登陆页`有自己的`view`,`商品列表``商品详情``商品购买`会挂载某个`view`下，`订单列表``订单详情`会挂载某个`view`下。

并且我们会在`view`上`connect()`获取数据，接着将数据传入子组件。这样整个结构就比较清晰。

上述例子，除去`顶级组件`会`connect()`外，我们还会在`cash`这个`view`上进行`connect()`操作。

此时`URL`为`localhost:3000/cash/index`时，便会加载`Root``CashView``CashIndex`组件

此时`URL`为`localhost:3000/cash/product/1`时，便会加载`Root``CashView``CashProductDetail`组件

`详情页`以及`首页列表`在进行切换时，仅仅只是替换了各自的组件，不会影响到`CashView`层。

```ruby
class CashView extends Component {

  render () {
    const {
      entities, actions, children, publicEntities, publicActions
    } = this.props;

    return (
      <div>
        <div className="er-header">
          <Header navBarIndex={1}/>
        </div>
        # 这里来根据路由加载不同的子组件
        # 使用 cloneElement 的原因是为了将数据传入子组件
        # cloneElement 并非是真正的 clone, 只是将我们传入的数据进行合并罢了
        {cloneElement(children, { entities, actions, publicEntities, publicActions })}
        <Footer/>
      </div>
    );
  }
}
```

###8、重新渲染页面
现在有这样的一个需求:

>点击按钮后，该按钮需要变色

1. 组件加载完成，页面渲染完毕
2. 单击按钮，发送action修改state内的值，此时重新render(),按钮成功变色 A
3. 单击第二个按钮，发送action,修改state,重新render()后,此时第二个按钮变色B，第一个按钮的颜色还是 A

就是说，渲染是基于最新的 dom 元素来进行渲染的，而不是重新完全渲染

###9、Redux 的 store 以及 React 的 state
由于 Store 就是来管理 React 的数据的，所以很多人会有这样的误解：

`Store是在管理操作React内的State`

其实不是，`Redux`的`store`跟`React`的`state`并没有什么关系。只是我们用前者来代替后者而已。

所以，`React`内的`state`我们也是可以正常使用的。
```ruby
class Teacher extends Component {
    construtor() {
        # 由于使用了 ES6 语法，因此不能用 defaultState 属性来进行初始化
        this.state = {
            name: 'fancy'
        }
    }
    
    componentWillMount(){ 
        if(this.props.rightLinkTxt){
          # 更改该组件的 state, 此时该组件会重新 render, 父级组件不受影响
          this.setState({
            name: 'nancy'
          });
        }
    }
}
```

所以在较小的组件内，比如单选框，要通过`state`来改变样式的话，就可以直接使用`React`的`state`，更新状态只在该组件内，而不是使用`redux`的`store`，勾选后发送`action`, `reducer` 处理数据, 造成所有正在被加载的组件都重新 render。

这样结构就比较清晰，将小组件的影响分离了出来。

###10、直接在 URL 上修改参数
```ruby
# 以下按顺序发生
1. url
2. url?params=123   => 直接按enter => 刷新动作
3. url?params=12 => 直接按enter => update
4. url?params=112 => 直接按enter => update
5. url?params=122 => 直接按enter => update
6. url?params= => 直接按enter => 刷新动作
```

###11、Reducer 与 entites

1、我们把所有的 reducer都合并成了一个 rootRecuder
```javascript
export default combineReducers({
  router: routerStateReducer,
  loginEntity,
  homeEntity
})

// 实际上转换的是：

export default {
  router: routerStateReducer(state.router, action),
  // 注意这里的 state.loginEntity，这就是为什么我们在自己的 reducer 只能够拿到自己相关 entites 内容的原因。
  loginStzEntity: loginStzEntity(state.loginStzEntity, action),
  homeEntity: homeEntity(state.homeEntity, action),
}

// 根rootReducer 返回所有 state 的内容
```

2、将 rootRecuder 跟 store 进行绑定
```javascript
// 这样子以后发的action通通都会被 rootReducer 所接收到
const store = createStoreWithMiddleware( createStore )(
    rootReducer, initialState
);

// 也就是说一个 action 会被里面所有的子 reducer 所接收~
```

###12、render  的时机
```ruby
class Father extends Component(){
    
    render() {
        return ( <Children/> )
    }
}

class Children extends Component(){
    
    doSomething() {
        # setState 后, this.state 的值并不会立马更新！
        this.setState({
            name: "fancy"
        })
        # 这里会执行完，才进行重新 render
        console.log("yo");
    }
    
    render() {
        return ( <div> Yo </div> )
    }
}
```

###13、简易流程

1. 将 reducer 注册到 store
2. 用connect 将组件连接 redux
3. 利用redux从 根组件 注入的 dispatch 去触发 action 
4. 此时 state 和 action 就会传入 reducer

###14、优化
React 的机制是这样的，一个父组件下面一大堆子组件，如果父组件进行`re-render`，那么旗下的所有子组件都会`re-render`，但是子组件的`state``props`在未改变的情况下也会`re-render`，虽然 React 使用了 虚拟 DOM, 但`render`那么多次总会浪费性能不是么？

所以我们可以用 `Pure render decorator`，他能够在`props`未改变的情况下，阻止组件进行`re-render`,原理就是利用`shouldComponentUpdate`

```ruby
function shouldComponentUpdate(nextProps, nextState) {
  return shallowCompare(this, nextProps, nextState);
}

function pureRende(component) {
  component.prototype.shouldComponentUpdate = shouldComponentUpdate;
}
module.exports = pureRender;

```

但这种方案还是有缺憾：

当 props 有复杂类型时，我们会遇到两种情况

####情况一，我修改detail的内容，而不改detail的引用
这样就会引起一个bug，比如我修改detail.name,因为detail的引用没有改，所以
props.detail ===nextProps.detail 还是为true。。
所以我们为了安全起见必须修改detail的引用，（redux的reducer就是这么做的）

####情况二，我修改detail的引用
这种虽然没有bug，但是容易误杀，比如如果我新旧两个detail的内容是一样的，岂不是还要，render。。所以还是不完美，，你可能会说用 深比较就好了，，但是 深比较及其消耗性能，要用递归保证每个子元素一样,

解决方案是使用 
[immutable.js](https://github.com/camsong/blog/issues/3)


暂时先这样吧，有空再补上其他的。

