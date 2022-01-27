### vue 执行流程和双向响应式原理

vue 核心

```js
|-- core
    |-- config.js
    |-- index.js
    |-- components
    |   |-- index.js
    |   |-- keep-alive.js
    |-- global-api
    |   |-- assets.js
    |   |-- extend.js
    |   |-- index.js
    |   |-- mixin.js
    |   |-- use.js
    |-- instance
    |   |-- events.js
    |   |-- index.js
    |   |-- init.js
    |   |-- inject.js
    |   |-- lifecycle.js
    |   |-- proxy.js
    |   |-- render.js
    |   |-- state.js
    |   |-- render-helpers
    |       |-- bind-dynamic-keys.js
    |       |-- bind-object-listeners.js
    |       |-- bind-object-props.js
    |       |-- check-keycodes.js
    |       |-- index.js
    |       |-- render-list.js
    |       |-- render-slot.js
    |       |-- render-static.js
    |       |-- resolve-filter.js
    |       |-- resolve-scoped-slots.js
    |       |-- resolve-slots.js
    |-- observer
    |   |-- array.js
    |   |-- dep.js
    |   |-- index.js
    |   |-- scheduler.js
    |   |-- traverse.js
    |   |-- watcher.js
    |-- util
    |   |-- debug.js
    |   |-- env.js
    |   |-- error.js
    |   |-- index.js
    |   |-- lang.js
    |   |-- next-tick.js
    |   |-- options.js
    |   |-- perf.js
    |   |-- props.js
    |-- vdom
        |-- create-component.js
        |-- create-element.js
        |-- create-functional-component.js
        |-- patch.js
        |-- vnode.js
        |-- helpers
        |   |-- extract-props.js
        |   |-- get-first-component-child.js
        |   |-- index.js
        |   |-- is-async-placeholder.js
        |   |-- merge-hook.js
        |   |-- normalize-children.js
        |   |-- normalize-scoped-slots.js
        |   |-- resolve-async-component.js
        |   |-- update-listeners.js
        |-- modules
            |-- directives.js
            |-- index.js
            |-- ref.js
```

1. `new Vue()`，执行`this.init()`,--》state.js 的`initState方法`，--》observer/index.js 的`observe(data, true /* asRootData */){ob = new Observer(value)}`,
   --》`ob = new Observer(value)`---》`defineReactive(obj, keys[i]);`
2. 定义响应式时候 初始化属性对应 dep,dep 是闭包，每个数据属性都对应一个 dep。
3. `vm.$mount(vm.$options.el)`，如果有 el，没有`渲染render函数，则根据template进行编译成ast和生成render函数`

```js
// const mount = Vue.prototype.$mount;
// 带编译器 重写 Vue.prototype.$mount
Vue.prototype.$mount = function () {
  // ...
  const { render, staticRenderFns } = compileToFunctions(
    template,
    {
      outputSourceRange: process.env.NODE_ENV !== "production",
      shouldDecodeNewlines,
      shouldDecodeNewlinesForHref,
      delimiters: options.delimiters,
      comments: options.comments,
    },
    this
  );
  options.render = render;
  return mount.call(this, el, hydrating);
};
```

4.  mount-》`runtime\index.js的Vue.prototype.$mount`--》调用 `lifecycle.js 中的 mountComponent`
    重点在于 mountComponent，这里进行了 vm 实例和 watcher 绑定以及初始化依赖存放参数。
    `mountComponent`主要做了：
    定义一个更新函数: `updateComponent = () => {vm._update(vm._render(), hydrating)}`
    调用 new Watcher()，
    ```js
    /* isRenderWatcher */
    new Watcher(
      vm,
      updateComponent,
      noop,
      {
        before() {
          if (vm._isMounted && !vm._isDestroyed) {
            callHook(vm, "beforeUpdate");
          }
        },
      },
      true
    );
    ```
5.  `Watcher类`：
    将当前 Watcher 实例 this 赋值给`vm._watcher`
    将 this.vm 赋值 vm `this.vm=vm`
    初始化依赖存放参数,

    ```js
    //放置watcher依赖的dep和dep依赖的watcher
    this.deps = [];
    this.newDeps = [];
    this.depIds = new Set();
    this.newDepIds = new Set();
    ```

    ```js
    get() {
      pushTarget(this); // 设置Dep.target=watcher
      let value;
      const vm = this.vm;
      value = this.getter.call(vm, vm); // 调用更新函数，执行ast包装的render函数返回虚拟dom和进行依赖收集
      popTarget();
      this.cleanupDeps();
      return value;
    }
    ```

    Watcher 实例 this,调用内部 get 方法，内部 get 方法将全局变量`Dep.target设置为Watcher实例的this`,
    接着调用`vm._render()`生成 vdom
    在解析通过 ast 的包装的 render(vm)---》vdom 过程中，会读取 vm 的数据属性，触发`observer/index.js`的
    Object.defineProperty 对应的 get 方法，get 方法内部做了依赖收集:

    init 的响应式定义

    ```js
    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,
      get: function reactiveGetter() {
        const value = getter ? getter.call(obj) : val;
        if (Dep.target) {
          // 这里进行双向依赖收集，重点是dep收集watcher，为了数据改变派发更新指定watcher
          dep.depend(); // dep 是什么？ dep是闭包，初始化响应式时创建，对应数据的属性
          if (childOb) {
            childOb.dep.depend();
            if (Array.isArray(value)) {
              dependArray(value);
            }
          }
        }
        return value;
      },
      set: function reactiveSetter(newVal) {
        const value = getter ? getter.call(obj) : val;
        if (getter && !setter) return;
        if (setter) {
          setter.call(obj, newVal);
        } else {
          val = newVal;
        }
        childOb = !shallow && observe(newVal);
        dep.notify();
      },
    });
    ```

    读取 defineProperty 内部 get 时，--》执行`dep.js 的 depend 方法`
    Dep 类内部方法 depend

    ```js
    depend() {
    //Dep.target 就是当前wachter
    // this 要观察的数据属性对应创建的dep dep跟数据属性一一对应
      if (Dep.target) {
        // 调用Watcher类内部方法addDep
        Dep.target.addDep(this);
      }
    }
    ```

    再调用 Watcher 类内部方法 addDep

    ```js
      addDep(dep: Dep) {
        const id = dep.id;
        if (!this.newDepIds.has(id)) {
          this.newDepIds.add(id);
          this.newDeps.push(dep);// watcher收集依赖的dep
          if (!this.depIds.has(id)) {
            dep.addSub(this); // 属性收集要派发更新的watcher
          }
        }
      }
    ```

6.  收集完数据后，再通过编译后，虚拟 dom 也生成了，此时在`lifecycle.js`调用`vm._update(vnode, hydrating);`生成真正 dom

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this;
  const prevEl = vm.$el;
  const prevVnode = vm._vnode;
  const restoreActiveInstance = setActiveInstance(vm);
  vm._vnode = vnode;
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode);
  }
  restoreActiveInstance();
  if (prevEl) {
    prevEl.__vue__ = null;
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm;
  }
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el;
  }
};
```

7. view 输入数据--》v-model 语法糖对数据重新赋值，触发 set 方法，`dep.notify()`派发更新订阅的 watcher，

```js
// Dep
 notify() {
   const subs = this.subs.slice();
   for (let i = 0, l = subs.length; i < l; i++) {
     subs[i].update();
   }
 }
 // Watcher
   update() {
   if (this.lazy) {
     this.dirty = true;
   } else if (this.sync) {
     this.run();
   } else {
     queueWatcher(this);
   }
 }
 // 放入队列
 export function queueWatcher (watcher: Watcher) {
  // ...
  nextTick(flushSchedulerQueue)
}
// 执行flushSchedulerQueue
// 模板重新获取渲染
flushSchedulerQueue(){
 watcher.run()
}
run(){
  const value = this.get();
}
get(){
// 再次调用this.get 取值和更新组件
 value = this.getter.call(vm, vm); // 调用更新函数，执行ast包装的render函数返回虚拟dom和进行依赖收集
}

```
