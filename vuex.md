# Vuex

## 概述

* 状态管理模式：采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。

* 状态
  * state: 驱动应用的数据源
  * view: 以声明方式将 state 映射到视图
  * actions: 响应 view 上的用户输入导致的状态变化

* 单向数据流：actions => state => view => actions

* 解决的痛点：
  * 多个视图依赖于同一状态
  * 来着不同视图的行为需要变更同一状态

## Store

* 核心是 Store（仓库），包含应用的状态（state）
* 与全局对象的区别：
  * Vuex 的状态存储是响应式；store 状态的改变组件将会得到响应更新。
  * 不能直接改变 store 中的状态，唯一途径是显式地提交 commit mutation。这个约定的意图是更明确地追踪状态的变化。

## State

* 单一状态树：用一个对象包含全部的应用层级状态
* 使用：
  * 在计算属性获取；
  * 每当数据发生改变时都会重新计算属性
  * 通过 store 选项将状态从根组件注入到每个子组件，子组件通过 this.$store 访问

```js
const store = Vue.use(Vuex);

const app = new Vue({
  el: '#app',
  store,
});
```

* mapState 辅助函数：
  * 解决一个组件同时获取多个状态
  * `...mapState({})` 与局部计算属性混合使用

```js
import { mapState } from 'vuex'

export default {
  computed: mapState({
    count: state => state.count,
    // 传字符串参数 'count' 等同于 `state => state.count`
    countAlias: 'count',
    // 为了能够使用 `this` 获取局部状态，必须使用常规函数
    countPlusLocalState(state) {
      return state.count + this.localCount
    }
  })
}

//当映射的计算属性的名称与 state 的子节点名称同名时
computed: mapState(['count']);
```

* 组件应有自己的局部状态，仅应用于多层组件间的状态才放到 Vuex

## Getter

* Store 的计算属性：只有当它的依赖值发生了改变才会被重新计算
* 接受 state 作为第一个参数，接受其他的 getter 作为第二个参数
* 通过 this.$store.getters.doneTodosCount 访问
* 通过方法访问：每次都会进行调用，而不会缓存结果
* mapGetters 辅助函数：将 getter 映射到局部计算属性

```js
const store = new Vuex.Store({
  state: {
    todos: [
      { id: 1, text: '', done: true },
      { id: 1, text: '', done: false },
    ]
  },
  getters: {
    doneTodos: state => {
      return state.todos.filter(todo => todo.done);
    }
  }
});

//store.getters.getTodoById(2);
getter: {
  getTodoById: (state) => (id) => {
    return state.todos.find(todo => todo.id === id);
  }
}

//mapGetters
computed: {
  //对象展开运算符
  ...mapGetters([
    'doneTodosCount',
    'anotherGetter',
  ])
}

//getter 别名
//this.doneCount 映射 this.$store.getters.doneTodosCount
mapGetters({
  doneCount: 'doneTodosCount',
});
```

## Mutation

* 更改 Vuex 的 store 中的状态唯一方法是提交 mutation
* 每个 mutation 都有一个字符串的事件类型（type）和一个回调函数（handler）
* 回调函数接受 state 作为第一个参数
* 调用方式：store.commit('increment')，函数接受额外参数，即 mutation 的载荷 payload；在大多数情况下，payload 为对象

```js
const store = new Vuex.Store({
  state:{ count: 1 },
  mutations: {
    increment(state) {
      state.count++;
    }
  }
})

//推荐风格
store.commit({
  type: 'increment',
  amount: 10,
});
```

* 规则：
  * 最好提前在 store 中初始化好所有属性
  * 给对象添加新属性时：
    * Vue.set(obj, 'newProp', 123);
    * 新对象替换老对象 `state.obj = { ...state.obj, newProp: 123 }`
* 使用常量替代事件类型，同时将常量放到单独的文件中易于维护（在多个协作的大项目尤为重要）

```js
//mutation-types.js
export const SOME_MUTATION = 'SOME_MUTATION';

//store.js
import Vuex from 'vuex'
import { SOME_MUTATION } from './mutation-types'

const store = new Vuex.Store({
  state: {...},
  mutations: {
    //ES6 风格的计算属性命名
    [SOME_MUTATION] (state) {}
  }
})
```

* **Mutation 必须是同步函数**，任何的回调函数中进行的状态改变都是不可跟踪的
* 辅助函数：mapMutations，支持 payload

```js
import { mapMutations } from 'vuex'

export default {
  methods: {
    ...mapMutations([
      'increment',  // 将 `this.increment()` 映射为 `this.$store.commit('increment')`
      'incrementBy', // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
    ]),
    ...mapMutations({
      add: 'increment', // 将 `this.add()` 映射为 `this.$store.commit('increment')`
    })
  }
}
```

## Action

* 包含任何的异步操作
* 提交 mutation，而非直接变更状态
* 参数：第一个参数：接受同 store 实例具有相同的方法和属性的 context 对象

```js
const store = new Vuex.store({
  state: {
    count: 0,
  },
  mutations: {
    increment(state) {
      state.count++;
    }
  },
  actions: {
    increment(context) { //参数解构 { commit }
      context.commit('increment'); // commit('increment')
    }
  }
})
```

* 分发 action：store.dispatch 触发

```js
actions: {
  checkout({ commit, state }, products ) {
    // 把当前购物车的物品备份起来
    const savedCartItems = [...state.cart.added];
    // 发出结账请求，然后乐观地清空购物车
    commit(types.CHECKOUT_REQUEST);
    // 购物 API 接受一个成功回调和一个失败回调
    shop.buyProducts(
      products,
      //成功操作
      () => commit(types.CHECKOUT_SUCCESS),
      //失败操作
      () => commit(types.CHECKOUT_FAILURE, savedCartItems)
    )
  }
}
```

* 辅助函数 mapActions，支持载荷

```js
methods: {
  ...mapActins([
    'increment', // 将 `this.increment()` 映射为 `this.$store.dispatch('increment')`
  ]),
  ...mapActions({
    add: 'increment', // 将 `this.add()` 映射为 `this.$store.dispatch('increment')`
    })
  })
}
```

* store.dispatch 返回 Promise
* 一个 store.dispatch 在不同模块中可以触发多个 action 函数。只有当所有触发函数完成后，返回的 Promise 才会执行。

```js
actions: {
  async actionA({ commit }) {
    commit('gotData', awaitData())
  },
  async actionB({ dispatch, commit }) {
    await dispatch('actionA') // 等待 actionA 完成
    commit('gotOtherData', await getOtherData());
  }
}
```

## Module

* 解决应该变得非常复杂时，store 对象变得臃肿。建议将 store 分割成模块（module）。每个模块都有自己的 state，mutation，action，getter 甚至嵌套子模块。

```js
const moduleA = {
  state: {},
  mutations: {},
  actions: {},
  getters: {},
};

const moduleB = {
  state: {},
  mutations: {},
  actions: {},
};

const store = new Vuex.Store({
  modules: {
    a: modulesA,
    b: modulesB,
  }
});

store.state.a // moduleA 的状态
store.state.b // moduleB 的状态
```

* 模块内的局部状态

```js
const moduleA = {
  actions: {
    //`state` 对象是模块的局部状态
    incrementIfOddOnRootSum({ state, commit, rootState }) {
      if ((state.count + rootState.count) % 2 === 1) {
        commit('increment');
      }
    }
  },
  getters: {
    sumWithRootCount(state, getters, rootState) {
      return state.count + rootState.count;
    }
  }
}
```

* 命名空间
  * 默认情况下，模块内部的 action、mutation 和 getter 是注册在全局命名空间的
  * 应用带命名空间的模块：`namespaced: true`

```js
const store = new Vuex.Store({
  modules: {
    account: {
      namespaced: true,
      state: {},
      getters: {},
      actions: {},
      mutations: {},
      modules: {
        // 继承父模块的命名空间
        myPage: {
          state: {},
          getters: {},
        },
        // 进一步嵌套命名空间
        posts: {
          namespaced: true,
          state: {},
          getters: {}
        }
      }
    }
  }
})
```

## 项目结构

* 对于大型应用，将 Vuex 相关代码分割到模块中

```js
store
  ├── index.js
  ├── actions.js
  ├── mutations.js
  ├── modules
      ├── cart.js //购物车模块
      ├── products.js //产品模块
```

## 插件

* plugins 选项，这个选项暴露出每次 mutation 的钩子。
* Vuex 插件是一个函数，接受 store 作为唯一参数

```js
const myPlugin = store => {
  //当 store 初始化后调用
  store.subscribe((mutation, state) => {
    // 每次 mutation 之后调用
    // mutation 的格式为 { type, payload }
  })
};

const store = new Vuex.Store({
  plugins: [myPlugin],
});
```

* 在插件中不允许直接修改状态值，只能通过 mutation 触发。通过提交 mutation 后，插件可同步数据源到 store 上。
* 插件生成 state 的快照，比较改变的前后状态；需要对 state 对象进行深拷贝。

```js
const myPluginWithSnapshot = store => {
  let prevState = _.cloneDeep(store.state)
  store.subscribe((mutation, state) => {
    let nextState = _.cloneDeep(state)
    // 比较 prevState 和 nextState...

    // 保存状态，用于下一次 mutation
    prevState = nextState
  })
};

//生成状态快照的插件应该只在开发阶段使用
const store = new Vuex.Store({
  plugins: process.env.NODE_ENV !== 'production' ? [myPluginWithSnapshot] : [];
})
```

## 严格模式

* 开启严格摸索，在创建 store 中传入 strict: true
* 在严格模式下，无论何时发生了状态变更且不是由 mutation 函数引起的，将会抛出错误
* 不要在发布环境下启用严格模式，以避免性能损失

## 表单处理

* 在严格模式下使用 Vuex 时，在属于 Vuex 的 state 上使用 v-model 会比较棘手。由于修改不是通过 mutation 函数执行，将抛出错误。
* 解决办法：
  * 不用 v-model 改用 'Vuex 方式'：给 input 中绑定 value, 监听 input/change 事件，在事件回调中调用 action
  * 使用 v-model, 使用带有 setter 的双向绑定计算属性

```js
<input v-model="message">

computed: {
  message: {
    get() {
      reutrn this.$store.state.obj.message;
    },
    set(value) {
      this.$store.commit('updateMessage', value);
    }
  }
}
```
