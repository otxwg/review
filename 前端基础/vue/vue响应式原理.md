### vue 执行原理和双向响应式实现流程

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
    将当前 Watcher 实例 this `vm._watcher`
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

        Watcher 实例 this,调用内部 get 方法，内部 get 方法将全局变量 Dep.target 设置为 Watcher 实例的 this,接着调用 vm.\_render()生成 vdom
        在解析 ast 的 render(vm) ---》vdom 过程中，会读取 vm 的数据属性，触发 `observer/index.js`的 Object.defineProperty 对应的 get 方法，get 方法内部做了依赖收集:

    响应式定义
    ```js

        Object.defineProperty(obj, key, {
          enumerable: true,
          configurable: true,
          get: function reactiveGetter() {
            const value = getter ? getter.call(obj) : val;
            if (Dep.target) {
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

        dep.js depend 方法

        ```js
        depend() {
        //Dep.target 就是当前wachter
        // this 要观察的数据属性dep
          if (Dep.target) {
            Dep.target.addDep(this);
          }
        }
        ```

        (1)watcher 收集 dep
        (2)dep 收集 watcher

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

```js
/**
 * Define a reactive property on an Object.
 */
export function defineReactive(
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep();
  console.log(dep, "dep");
  const property = Object.getOwnPropertyDescriptor(obj, key);
  if (property && property.configurable === false) {
    return;
  }

   cater for pre-defined getter/setters
  const getter = property && property.get;
  const setter = property && property.set;
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key];
  }

  let childOb = !shallow && observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val;
      if (Dep.target) {
        // 1. 定义响应式时候 初始化属性对应dep
        // 2. 首次渲染，mount-》调用lifecycle.js中的mountComponent--》new Watcher--》，
        // 将当前watcher赋值给vm._watcher
        // 当前Watcher实例this，初始化依赖存放参数
        //  this.deps = [];
        //  this.newDeps = [];
        //  this.depIds = new Set();
        //  this.newDepIds = new Set();
        //  3. Watcher实例this,调用内部get方法，内部get方法将全局变量Dep.target设置为Watcher实例的this，
        //  接着调用vm._render()生成vdom，
        //  4. 在解析ast ---》vdom过程中，会读取vm的数据属性，触发 Object.defineProperty对应的get方法，
        //  get方法内部做了依赖下列收集操作--》
        //  watcher 收集dep
        //  dep收集watcher
        dep.depend();  // dep 是什么？ dep是闭包，初始化响应式时创建，对应数据的属性
         depend () {
           if (Dep.target) {
             Dep.target.addDep(this)
           }
         }
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
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return;
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== "production" && customSetter) {
        customSetter();
      }
       #7981: for accessor properties without setter
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
}
```
