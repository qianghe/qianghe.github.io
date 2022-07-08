---
layout: post
title:  "@redux-tooltik是如何使用immer的？"
date:   2022-07-08 15:20:54 +0800
categories: redux redux-tooltik immer
---

## 一些碎碎念

使用redux，是为了做复杂的数据管理；但对于很多使用redux的人来说，配置繁琐而重复的模版文件，简直是一场永无止尽的噩梦。
 * 我们需要reducer；
 * 我们需要action;
 * 我们需要异步action；
 * 我们需要确认，在reducer中修改嵌套过深的state是否有效，否则会导致react-redux中判断shouldComponentUpdate失效，从而无法进行组件re-render；
    
    ...

写到这里我已经感到痛心疾首了，我们为了解决一个问题，又引入了另一问题；但是还好这个世界总是有“聪明的人”存在，开发出省时省力的工具，帮助开发人员们降低理解及使用成本（羡慕不已）。


## 自身角度
从我自己使用RTK来说，我个人感觉很好用的就是其文档中描述的：
  1. 降低了写reducer及action的成本，通过createSlice即可返回需要的reducers、actions；
  
  2. 降低了对异步action的处理成本，通过createAsyncThunk返回一个创建promise请求的createAction，这个promise可以被中途取消；createAction同样返回了pending\fufilled\rejected的同步action，便于用户直接在exteralReducers中针对三种情况进行state的更新：

```javascript
extraReducers: (builder) => {
    builder.addCase(fetchPreviewImage.pending, (state) => {
      state.url = '';
      state.loading = true;
    });
    builder.addCase(fetchPreviewImage.fulfilled, (state, action) => {
      state.url = action.payload?.imageUrl;
      state.loading = false;
    });
    builder.addCase(fetchPreviewImage.rejected, (state) => {
      state.url = '';
      state.loading = false;
    });
  },
```

  3. 在createSlice/createReducer中自动引入了immer，我们可以终于可以从“解构构造新对象返回state的方式”，直接跳到“直接修改state”的直观防止，即：

  ```javascript
  // 解构重组方式
  function updateVeryNestedField(state, action) {
    return {
      ...state,
      first: {
        ...state.first,
        second: {
          ...state.first.second,
          [action.someId]: {
            ...state.first.second[action.someId],
            fourth: action.someValue
          }
        }
      }
    }
  }
  ```

  ```javascript
  // immer直接修改的方式
  function updateNestedState(state, action) {
    let nestedState = state.nestedState
    // ERROR: this directly modifies the existing object reference - don't do this!
    nestedState.nestedField = action.data

    return {
      ...state,
      nestedState
    }
  }
  ```

之所以要内置immer，作者在[文档](https://redux-toolkit.js.org/usage/immer-reducers#why-immer-is-built-in)中也解释的很清楚了：immutable的数据结构在reducer修改中更为可读，并且其直接修改的方式降低了解构生成对象引入bug的问题。

文档中推荐的RTK自带的query功能，帮助fetch及缓存的相关功能，我是用的项目上暂时还未涉猎。

（因为没有很细致的看文档）在开发调试期间，出现了一个让我很困惑的问题，createSlice中reducer函数打印出的state竟然是Proxy对象，如下：

<img src="https://github.com/qianghe/blogs/blob/main/imgs/immer-proxy.png?raw=true" height=80 />

调试到这里的时候，我整个人都惊呆了；当时还没看createSlice的源码（且没有细读文档...），并不知道其使用immer对象中proxy的神奇之处。当时黑人问号缀满了我的扁平的额头：redux中reducer需要返回的state应该是纯js对象呀，现在返回proxy是什么情况，那middleware在dispatch的过程中又是这么处理这个proxy的呢？

带着这个愚蠢的疑问（是我想的复杂了...）去看了源码实现。


### createSlice做了什么？

<img src="https://github.com/qianghe/blogs/blob/main/imgs/createSlice-demo.png?raw=true" width=280 />

createSlice通过用户定义的reducers及name等，返回了一个slice对象，这个对象包含了actions、reducer等我们业务需要的对象：
  * 多个reducer可以通过redux提供的combineReducers方法进行整合；
  * actions用于我们出发dispatch的调用（不需要我们额外的通过createAction创建了）；

在createSlice函数中返回的reducer如下：

```javascript
reducer(state, action) {
   if (!_reducer) _reducer = buildReducer()

   return _reducer(state, action)
 },
```

其调用了buildReducer：

```javascript
function buildReducer() {
  const [
    extraReducers = {},
    actionMatchers = [],
    defaultCaseReducer = undefined,
  ] =
    typeof options.extraReducers === 'function'
      ? executeReducerBuilderCallback(options.extraReducers)
      : [options.extraReducers]

  const finalCaseReducers = { ...extraReducers, ...sliceCaseReducersByType }
  return createReducer(
    initialState,
    finalCaseReducers as any,
    actionMatchers,
    defaultCaseReducer
  )
}
```

buildReducer中整合了当前slice 的所有reducers，用户自定义的reducer及exterReducers；并返回了createReducer函数执行的结果。我们大概可以预估下这个返回结果，它其实是一个大的reducer处理器，它获取了当前的state及action并进行处理，执行符合触发的action.type的reducer：

```javascript
// 模拟返回
reducer(state, action) => {
  // filterReducers 复合action的
  // 顺序执行filterReducers
  filterReducers.reduce((preState, filterReducer) => {
    //...
    const nextState = filterReducer(preState, action)
    return nextState
  }, state)
}
```

在真实的createReducer中，直接看其返回的对象：

```javascript
return function reducer(state, action) {
  // ...
  return caseReducers.reduce((previousState, caseReducer): S => {
    if (caseReducer) {
      if (isDraft(previousState)) {
          //...
      } else if (!isDraftable(previousState)) {
        //...
      } else {
        return createNextState(previousState, (draft: Draft<S>) => {
          return caseReducer(draft, action)
        })
      }
    }

    return previousState
  }, state)
}
```

其处理的方式和预想的差不多，其中isDraft和isDraftable是对immer结构的判断；直接看下是原生js对象的情况，其else的处理，返回了一个createNextState函数执行的结果，这个函数同样是immer提供的。


### createSlice中如何使用了immer？

接着上面说，createNextState函数一定返回的是纯js结构，这样就解答我上面提出的愚蠢疑问...

```javascript
return createNextState(previousState, (draft: Draft<S>) => {
  return caseReducer(draft, action)
})
```

函数的第二参数，即一个回调函数，其入参是draft（即immer提供的proxy对象），其作为reducer的如参，代替了之前的state，因此我们可以在createSlice的reducer中直接修改state，因为这时的state已经是immer对象创建的proxy代理对象了，其内置了set\get等方法，可以让我们直接对对象、数组、Map、Set进行直接操作。

**这个proxy的结构：**

```typescript
const state: ProxyState = {
    type_: isArray ? ProxyType.ProxyArray : (ProxyType.ProxyObject as any),
    // Track which produce call this is associated with.
    scope_: parent ? parent.scope_ : getCurrentScope()!,
    // True for both shallow and deep changes.
    modified_: false,
    // Used during finalization.
    finalized_: false,
    // Track which properties have been assigned (true) or deleted (false).
    assigned_: {},
    // The parent draft state.
    parent_: parent,
    // The base state.
    base_: base,
    // The base proxy.
    draft_: null as any, // set below
    // The base copy with any updated values.
    copy_: null,
    // Called by the `produce` function.
    revoke_: null as any,
    isManual_: false
}
```

createNextState函数即是immer提供的produce对象方法（核心流程）：

```javascript
if (isDraftable(base)) {
  const scope = enterScope(this)
  // 通过原始的base，即state，构造了一个proxy对象；
  const proxy = createProxy(this, base, undefined)
  let hasError = true
  try {
    // recipe在此处即使我们的reducer
    result = recipe(proxy)
    hasError = false
  } finally {
    // finally instead of catch + rethrow better preserves original stack
    if (hasError) revokeScope(scope)
    else leaveScope(scope)
  }
  // ...
  usePatchesInScope(scope, patchListener)
  // 返回处理的结果
  return processResult(result, scope)
}
```

在执行reducer的过程中，对state的修改，都会通过proxy对象的traps进行记录和更新，比如set方法，如果对属性进行了更新，会同步到proxy.copy_解构中，这是修改的新的satate的内容，是一个纯js对象。

```javascript
const objectTraps = {
  set (state, prop, value){
    //...
    // 改变会同步到state.copy_上
    state.copy_![prop] = value
		state.assigned_[prop] = true
		return true
  }
}
```

因此可以想到processResult返回的就是state.copy_（有新的修改）或者sate.base_（无修改）。


### 最后的巴啦啦
到此，上面关于本人提出的疑惑终于解答了：mmer解构只在createSlice/createReducer的reducer中使用了，而其执行后返回的nextState仍然是一个纯js对象。

阅读代码的过程中，是一个能更快速了解作者设计意图和实现方式的过程（其实文档中有很多处写到了，但是实在没有好好的阅读），也是自己学习他人设计和实现的便捷路径。我很喜欢作者面对网友质疑的态度，一个开源包的设计是要满足大部分人的场景需求的，但是也不能一味的迎合所有使用者的要求，比如“在createSlice中是否可以关闭immer的使用”这个问题上也有诸多使用者提出了对这个功能的渴求，但作者还是坚持自己的设计理念，并贯彻始终。

感恩～～～

<img src="https://github.com/qianghe/blogs/blob/main/imgs/3q-duck.png?raw=true" width=100 height=100/>
