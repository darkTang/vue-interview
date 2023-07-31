# 1. Vue的理解
Vue是一套用于构建用户界面的**渐进式框架**，Vue的核心库只关注图层。
- 渐进式框架：用什么加什么。声明式渲染 -> 组件系统 -> 客户端路由 -> 状态管理 -> 构建工具
  
## 1.1 声明式框架
- 命令式：关注过程。
- 声明式：关注结果。命令式逻辑已经封装好了。

## 1.2 MVVM模式
- MVC：Model、View、Controller -> Service
- MVVM：Model、View、ViewModel

## 1.3 虚拟DOM

## 1.4 组件化
高内聚、低耦合、单向数据流。
- 提高应用复用性。
- 降低更新范围，只重新渲染变化的组件。
- Vue中的每个组件都有一个渲染watcher（Vue2）、effect（Vue3）。
- 数据是响应的，数据变化后会执行watcher或者effect。
- 组件要合理拆分，如果不拆分组件，那更新的时候整个页面都要重新渲染。
- 如果过分拆分组件，会导致watcher、effect产生过多造成性能浪费。

# 2. SPA理解

# 3. Vue为什么需要虚拟DOM
- 虚拟DOM就是一个JS对象，通过操作对象而非真实DOM减少对真实DOM的操作，提高渲染性能。
- VDOM可以进行diff操作：
  - 挂载结束后，会记录第一次生成的VDOM-oldVnode
  - 当响应式数据发生变化，会引起组件重新render，此时会生成新的VDOM-newVnode
  - 使用oldVnode与newVnode做diff操作，将更改的部分应到真实DOM，从而实现最小量的DOM操作

# 4. 谈谈对响应式数据的理解
## 4.1 Vue2的响应式
- 内部采用`defineReactive`方法，使用`Object.defineProperty`将属性进行劫持(只能劫持已经存在的属性)，数组则是通过重写数组方法来实现。多层对象通过递归实现。
- 对每个属性需要重写`getter`和`setter`，性能差。
- 新增和删除属性无法监测到，需要通过`$set`和`$delete`实现。
- 数组不采用`Object.defineProperty`劫持，浪费性能，需要对数组单独处理。
- 对ES6新产生的Map和Set这些数据结构不支持。 
- `Object.defineProperty`只能对对象进行数据劫持。

## 4.2 Vue2如何监测数组的变化
```js
const obj = {
  name: "zs",
  age: 12,
  a: { b: 1 },
  arr: [1, 2, 3, { a: 2 }],
}
const methods = [
  "push",
  "pop",
  "shift",
  "unshift",
  "splice",
  "reverse",
  "sort",
];
const oldArrayPrototype = Array.prototype;
const newArrayPrototype = Object.create(oldArrayPrototype);
methods.forEach((method) => {
  newArrayPrototype[method] = function (...args) {
    // 执行对应的watcher，重新渲染
    console.log("触发更新");
    oldArrayPrototype[method].call(this, ...args);
  };
}
function defineReactive(target, key, value) {
  observer(value);
  Object.defineProperty(target, key, {
    get() {
      console.log(`${key}读取更新`);
      // 记录对应的watcher
      return value;
    },
    set(newVal) {
      if (value !== newVal) {
        // 执行对应的watcher，重新渲染
        console.log(`${key}修改更新`);
        value = newVal;
        observer(newVal);
      }
    },
  });
}
function observer(data) {
  if (typeof data !== "object" || data === null) return;
  if (Array.isArray(data)) {
    data.__proto__ = newArrayPrototype;
    for (let value of data) {
      observer(value)
    }
  } else {
    for (let key in data) {
      data.hasOwnProperty(key) && defineReactive(data, key, data[key]);
    }
  }
}
observer(obj);
console.log(obj);
```
- 数组的索引和长度变化无法监控。

## 4.3 Vue3的响应式
Vue3的响应式分别是由`Object.defineProperty`和`Proxy`实现的，对于简单数据类型，是由`Object.defineProperty`实现的，对于复杂数据类型，是由`Proxy`实现的。
`Proxy`对象用于创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）。
```js
const obj = { name: "zs", age: 12, a: { b: 1 }, arr: [1, 2, 3] }
let handle = {
  get(target, propName) {
    console.log(`${propName}读取更新`);
    // 收集effect
    if (typeof target[propName] !== "object" || target[propName] === null)
      return;
    return new Proxy(target[propName], handle);
    return target[propName];
  },
  set(target, propName, newVal) {
    console.log(`${propName}修改更新`)
    // 触发effer更新
    target[propName] = newVal;
  },
  deleteProperty(target, propName) {
    // 触发effer更新
    return delete target[propName];
  },
}
function reactive(target) {
  return new Proxy(target, handle);
}
const newObj = reactive(obj);
console.log(newObj);
```
- Proxy 可以代理对象和数组。
- 数组的索引和长度变化可以监控。

# 5. Vue如何进行依赖收集
- 每个属性都有自己的dep属性，存放在所依赖的watcher中，当属性变化后通知自己对应的watcher去更新。
- 默认在初始化时会调用render函数，此时会触发属性依赖收集(getter调用)。
- 当属性发生修改会触发watcher更新(setter调用)。
- dep和watcher是一个多对多的关系
- 一个属性可以在多个组件中使用 dep -> 多个watcher
- 一个组件中由多个属性组成 watcher -> 多个dep

# 6. watch和computed的区别
Vue2中有三种watcher（渲染watcher、计算属性watcher、用户自定义的watcher）。
Vue3中有三种effect（渲染effect、计算属性effect、用户自定义的effect）。

## 6.1 computed
- 计算属性仅当用户取值时才会执行相对应的方法。
- 计算属性具备缓存，依赖的值不发生变化，计算属性的方法不会重新执行。
- 计算属性不支持异步逻辑。

1. 每一个计算属性内部维护一个dirty属性，默认为true。
2. 当取值为true时，执行计算属性的方法，将值缓存起来，并且将dirty: false。
3. 再次取值时，dirty: false，直接返回缓存的值。
4. 当依赖的值发生变化时，触发更新，页面重新渲染，重新将dirty: true，重复2步骤。

1. 计算属性会创建一个计算属性watcher。这个watcher不会立即执行。
2. 通过`Object.defineProperty`将计算属性定义到实例上。
3. 当用户取值触发getter，拿到计算属性对应的watcher，看dirty是否为true，为true则求值。
4. 让计算属性依赖的属性收集最外层的渲染watcher和计算属性watcher，可以做到依赖的属性变化，触发计算属性watcher更新dirty重新为true，并且可以触发渲染watcher更新页面。
5. 如果依赖的属性没有发生变化，则采用缓存。

## 6.2 watch
- 监控值的变化，当值发生变化时调用对应的回调函数。（支持异步逻辑，异步要注意竞态问题）
> Vue3提供了onCleanup函数，让用户更加方便使用，也解决了清理问题。

# 7. Vue中的key的作用和原理
- key主要用于Vue的虚拟DOM算法，在新旧虚拟DOM对比时做出判断。如果不使用key，Vue会最大限度减少动态元素并且尽可能的尝试**就地修改/复用**相同类型元素的算法。
- Vue在patch过程中通过key可以判断两个虚拟节点是否是相同节点。（可以复用老节点）
- 无key会导致更新的时候出现问题。
- 尽量不要采用索引作为key。

# 8. Vue3中CompositionAPI的优势
- 在Vue2中采用的是OptionsAPI，用户提供的data、props、methods、computed、watch等属性。（用户编写复杂业务逻辑会出现反复横跳问题）
- Vue中所有的属性都是通过this访问，this存在指向明确问题。
- `CompositionAPI`对`Tree-Shaking`更友好，因为逻辑都会封装在函数中，通过导入导出进行`Tree-Shaking`。
- 组件逻辑共享问题，Vue2采用mixins实现组件之间的逻辑共享；但是会有数据来源不明确，命名冲突等问题。Vue3采用`CompositionAPI`提取公共逻辑非常方便。

# 9. Vue的性能优化
- 数据层级不易过深，合理设置响应式数据。
- 使用数据时缓存值的结果，不频繁取值。
- 合理设置key。
- v-show和v-if的选取。
- 控制组件粒度，合理拆分组件。
- 采用异步组件 -> 借助webpack的分包能力。
- 使用keep-alive缓存组件。
- 分页、滚动等策略。

# 10. SPA首屏加载速度慢的解决
- 使用路由懒加载、异步组件、组件拆分等，减少入口文件体积大小（优化体验骨架屏）
- 抽离公共代码，采用splitChunks进行代码分割。
- 组件通过按需加载方式。
- 静态资源缓存（http缓存、localStorage缓存）
- 图片资源压缩。
- 静态资源CDN（就近访问+缓存）加速。
- 使用SSR对首屏进行服务端渲染。

# 11. 异步组件的作用及原理
## 11.1 Vue2的写法
- 回调函数写法
```js
Vue.component('async-example', function (resolve, reject) {
  setTimeout(function () {
    // 向 `resolve` 回调传递组件定义
    resolve({        // 这个对象就是一个Vue配置项
      template: '<div>I am async!</div>'
    })
  }, 1000)
})
```
- 将异步组件和 webpack 的 code-splitting 功能一起配合使用：
```js
Vue.component('async-webpack-example', function (resolve) {
  // 这个特殊的 `require` 语法将会告诉 webpack
  // 自动将你的构建代码切割成多个包，这些包
  // 会通过 Ajax 请求加载
  require(['./my-async-component'], resolve)
})
```
- promise写法
```js
new Vue({
  // ...
  components: {
    'my-component': () => import('./my-async-component')
  }
})
```
- 高阶组件
```js
const AsyncComponent = () => ({
  // 需要加载的组件 (应该是一个 `Promise` 对象)
  component: import('./MyComponent.vue'),
  // 异步组件加载时使用的组件
  loading: LoadingComponent,
  // 加载失败时使用的组件
  error: ErrorComponent,
  // 展示加载时组件的延时时间。默认值是 200 (毫秒)
  delay: 200,
  // 如果提供了超时时间且组件加载也超时了，
  // 则使用加载失败时使用的组件。默认值是：`Infinity`
  timeout: 3000
})
// 最后将 AsyncComponent 即可
```

# 12. nextTick的理解
- Vue视图更新是异步的，底层本身就有一个nextTick方法，维护一个异步队列，帮助我们收集watcher，并去重。
- 自己定义的nextTick方法会与底层的nextTick合并，按顺序依次执行。
```js
handleClick() {
  this.count = 12     // [底层的nextTick, 定义的nextTick]  先执行底层的nextTick，帮我们进行页面更新，再执行定义的nextTick，获取页面最新数据
  this.$nextTick(() => {
    console.log(document.querySelector('div').innerHTML)
  })
}
---------------------------------------
this.$nextTick(() => {
  console.log(document.querySelector('div').innerHTML)
})
this.count = 12       // [定义的nextTick, 底层的nextTick1, 底层的nextTick2...]  先执行定义的nextTick，获取的页面数据还是老数据，再执行 底层的nextTick，进行页面更新
```

# 13. 递归组件的理解
