### axios 取消请求用法

```js
const CancelToken = axios.CancelToken;
let cancel;

axios.get("/user/12345", {
  cancelToken: new CancelToken(function executor(c) {
    // An executor function receives a cancel function as a parameter
    cancel = c;
  }),
});

// cancel the request
cancel();
```

### 取消依据

```
基于ajax请求的 xhr对象方法 xhr.abort(),我们只需要在适当时机暴露接口给外面执行这个xhr对象的obort方法即可执行。
```

### 实现

```
1. 配置axios请求时候，axios配置参数提供一个cancelToken属性，该属性值是CancelToken实例的对象，执行new CancelToken的时候传入一个执行器，执行器参数暴露接口。
2. CancelToken()内部在this上添加promise对象。 待执行取消请求可以改变promise状态。
3. promise状态更改成resolved会执行then方法。
4. xhr 打开请求后，调用config.cancelToken.subscribe(onCanceled)，config.cancelToken就是new CancelToken的实例，此时将取消请求添加进this实例的_listeners数组上。
5  cancel()请求将resolved promise的状态，执行then，then内部执行onCanceled即可取消请求。

```

#### CancelToken 核心代码

```js
function CancelToken(executor) {
  if (typeof executor !== "function") {
    throw new TypeError("executor must be a function.");
  }
  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });
  var token = this;
  // eslint-disable-next-line func-names
  this.promise.then(function (cancel) {
    if (!token._listeners) return;
    var i;
    var l = token._listeners.length;
    for (i = 0; i < l; i++) {
      token._listeners[i](cancel);
    }
    token._listeners = null;
  });

  // eslint-disable-next-line func-names
  this.promise.then = function (onfulfilled) {
    var _resolve;
    // eslint-disable-next-line func-names
    var promise = new Promise(function (resolve) {
      token.subscribe(resolve);
      _resolve = resolve;
    }).then(onfulfilled);

    promise.cancel = function reject() {
      token.unsubscribe(_resolve);
    };

    return promise;
  };

  executor(function cancel(message) {
    if (token.reason) {
      // Cancellation has already been requested
      return;
    }
    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}
/**
 * Throws a `Cancel` if cancellation has been requested.
 */
CancelToken.prototype.throwIfRequested = function throwIfRequested() {
  if (this.reason) {
    throw this.reason;
  }
};
/**
 * Subscribe to the cancel signal
 */
CancelToken.prototype.subscribe = function subscribe(listener) {
  if (this.reason) {
    listener(this.reason);
    return;
  }
  if (this._listeners) {
    this._listeners.push(listener);
  } else {
    this._listeners = [listener];
  }
};
```

#### xhr.js 核心代码

```js
if (config.cancelToken || config.signal) {
  // Handle cancellation
  // eslint-disable-next-line func-names
  onCanceled = function (cancel) {
    if (!request) {
      return;
    }
    reject(
      !cancel || (cancel && cancel.type) ? new Cancel("canceled") : cancel
    );
    request.abort();
    request = null;
  };

  config.cancelToken && config.cancelToken.subscribe(onCanceled);
  if (config.signal) {
    config.signal.aborted
      ? onCanceled()
      : config.signal.addEventListener("abort", onCanceled);
  }
}
```
