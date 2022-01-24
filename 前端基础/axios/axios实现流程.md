### axios 代码结构

axios
├── CHANGELOG.md
├── dist
│ ├── axios.js
│ ├── axios.map
│ ├── axios.min.js
│ └── axios.min.map
├── index.d.ts
├── index.js
├── lib
│ ├── adapters
│ │ ├── http.js
│ │ ├── README.md
│ │ └── xhr.js
│ ├── axios.js
│ ├── cancel
│ │ ├── Cancel.js
│ │ ├── CancelToken.js
│ │ └── isCancel.js
│ ├── core
│ │ ├── Axios.js
│ │ ├── buildFullPath.js
│ │ ├── createError.js
│ │ ├── dispatchRequest.js
│ │ ├── enhanceError.js
│ │ ├── InterceptorManager.js
│ │ ├── mergeConfig.js
│ │ ├── README.md
│ │ ├── settle.js
│ │ └── transformData.js
│ ├── defaults.js
│ ├── env
│ │ ├── data.js
│ │ └── README.md
│ ├── helpers
│ │ ├── bind.js
│ │ ├── buildURL.js
│ │ ├── combineURLs.js
│ │ ├── cookies.js
│ │ ├── deprecatedMethod.js
│ │ ├── isAbsoluteURL.js
│ │ ├── isAxiosError.js
│ │ ├── isURLSameOrigin.js
│ │ ├── normalizeHeaderName.js
│ │ ├── parseHeaders.js
│ │ ├── README.md
│ │ ├── spread.js
│ │ └── validator.js
│ ├── README.md
│ └── utils.js
├── LICENSE
├── package.json
├── README.md
├── SECURITY.md
├── tree.md
├── tsconfig.json
├── tslint.json
└── UPGRADE_GUIDE.md

###

引入 axios
import axios from 'axios'
lib|axios.js

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);
  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);
  // Copy context to instance
  utils.extend(instance, context);
  // Factory for creating new instances
  instance.create = function create(instanceConfig) {
    return createInstance(mergeConfig(defaultConfig, instanceConfig));
  };
  return instance;
}
// Create the default instance to be exported
var axios = createInstance(defaults);

module.exports = axios;
// Allow use of default import syntax in TypeScript
module.exports.default = axios;
```

1. new Axios 主要在在实例上挂载了拦截器对象，用来存储请求拦截和响应拦截。interceptors 对象具有 request，response
   的 InterceptorManager 实例。
   function Axios(instanceConfig) {
   this.defaults = instanceConfig;
   this.interceptors = {
   request: new InterceptorManager(),
   response: new InterceptorManager()
   };
   }
   InterceptorManager 定义：
   function InterceptorManager() {
   this.handlers = [];
   }
2. 其实我们用的 axios 就是 axios 原型上的 Axios.prototype.request 方法。方法用来处理配置数据和发起请求，最后处理结果
   和返回请求结果。

```js
/**
 * Dispatch a request
 *
 * @param {Object} config The config specific for this request (merged with this.defaults)
 */
Axios.prototype.request = function request(config) {
  if (typeof config === "string") {
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
    config = config || {};
  }
  config = mergeConfig(this.defaults, config);
  // Set config.method
  if (config.method) {
    config.method = config.method.toLowerCase();
  } else if (this.defaults.method) {
    config.method = this.defaults.method.toLowerCase();
  } else {
    config.method = "get";
  }
  var transitional = config.transitional;
  if (transitional !== undefined) {
    validator.assertOptions(
      transitional,
      {
        silentJSONParsing: validators.transitional(validators.boolean),
        forcedJSONParsing: validators.transitional(validators.boolean),
        clarifyTimeoutError: validators.transitional(validators.boolean),
      },
      false
    );
  }
  // filter out skipped interceptors
  var requestInterceptorChain = [];
  var synchronousRequestInterceptors = true;
  this.interceptors.request.forEach(function unshiftRequestInterceptors(
    interceptor
  ) {
    if (
      typeof interceptor.runWhen === "function" &&
      interceptor.runWhen(config) === false
    ) {
      return;
    }
    synchronousRequestInterceptors =
      synchronousRequestInterceptors && interceptor.synchronous;
    requestInterceptorChain.unshift(
      interceptor.fulfilled,
      interceptor.rejected
    );
  });
  var responseInterceptorChain = [];
  this.interceptors.response.forEach(function pushResponseInterceptors(
    interceptor
  ) {
    responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
  });
  var promise;
  if (!synchronousRequestInterceptors) {
    var chain = [dispatchRequest, undefined];
    Array.prototype.unshift.apply(chain, requestInterceptorChain);
    chain = chain.concat(responseInterceptorChain);
    promise = Promise.resolve(config);
    while (chain.length) {
      promise = promise.then(chain.shift(), chain.shift());
    }
    return promise;
  }
  var newConfig = config;
  while (requestInterceptorChain.length) {
    var onFulfilled = requestInterceptorChain.shift();
    var onRejected = requestInterceptorChain.shift();
    try {
      newConfig = onFulfilled(newConfig);
    } catch (error) {
      onRejected(error);
      break;
    }
  }
  try {
    promise = dispatchRequest(newConfig);
  } catch (error) {
    return Promise.reject(error);
  }
  while (responseInterceptorChain.length) {
    promise = promise.then(
      responseInterceptorChain.shift(),
      responseInterceptorChain.shift()
    );
  }
  return promise;
};
```

注册拦截器的用法

```js
InterceptorManager.prototype.use = function use(fulfilled, rejected, options) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected,
    synchronous: options ? options.synchronous : false,
    runWhen: options ? options.runWhen : null
  });
  return this.handlers.length - 1;
};
调用InterceptorManager原型use方法，往interceptors的request和respose对象的上handlers数组添加
成功和失败的处理。
// 添加请求拦截器
axios.interceptors.request.use(
  function (config) {
    // 在发送请求之前做些什么
    return config;
  },
  function (error) {
    // 对请求错误做些什么
    return Promise.reject(error);
  }
);
// 添加响应拦截器
axios.interceptors.response.use(
  function (response) {
    // 2xx 范围内的状态码都会触发该函数。
    // 对响应数据做点什么
    return response;
  },
  function (error) {
    // 超出 2xx 范围的状态码都会触发该函数。
    // 对响应错误做点什么
    return Promise.reject(error);
  }
);
```

3. promise 链的执行

```js
var chain = [dispatchRequest, undefined]; //undefined 占位符号。为了让promise成对取出执行
Array.prototype.unshift.apply(chain, requestInterceptorChain);
chain = chain.concat(responseInterceptorChain);
```

如果有请求拦截，则取出 promise 化的 requestInterceptorChain 上的处理函数执行，返回新的 config。
接着发起网络请求，执行 dispatchRequest 请求分发。这里 dispatchRequest 调用了 adapter 适配器，根据环境用 ajax 或 http。
再返回 promise 结果。

4. dispatchRequest 执行过程
   先做了转换请求数据处理。

```js
module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);
  // Ensure headers exist
  config.headers = config.headers || {};
  // Transform request data
  config.data = transformData.call(
    config,
    config.data,
    config.headers,
    config.transformRequest
  );
  // Flatten headers
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers
  );
  utils.forEach(
    ["delete", "get", "head", "post", "put", "patch", "common"],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );
  var adapter = config.adapter || defaults.adapter;
  return adapter(config).then(
    function onAdapterResolution(response) {
      throwIfCancellationRequested(config);
      // Transform response data
      response.data = transformData.call(
        config,
        response.data,
        response.headers,
        config.transformResponse
      );
      return response;
    },
    function onAdapterRejection(reason) {
      if (!isCancel(reason)) {
        throwIfCancellationRequested(config);
        // Transform response data
        if (reason && reason.response) {
          reason.response.data = transformData.call(
            config,
            reason.response.data,
            reason.response.headers,
            config.transformResponse
          );
        }
      }
      return Promise.reject(reason);
    }
  );
};
```

再调用 adapter（）,adapter 在默认配置文件中定义。

```js
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== "undefined") {
    // For browsers use XHR adapter
    adapter = require("./adapters/xhr");
  } else if (
    typeof process !== "undefined" &&
    Object.prototype.toString.call(process) === "[object process]"
  ) {
    // For node use HTTP adapter
    adapter = require("./adapters/http");
  }
  return adapter;
}
```

发起真正 XMLHttpRequest 请求，返回 promise 包装结果。

```js
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    var requestData = config.data;
    var requestHeaders = config.headers;
    var responseType = config.responseType;
    var onCanceled;
    function done() {
      if (config.cancelToken) {
        config.cancelToken.unsubscribe(onCanceled);
      }
      if (config.signal) {
        config.signal.removeEventListener("abort", onCanceled);
      }
    }
    if (utils.isFormData(requestData)) {
      delete requestHeaders["Content-Type"]; // Let the browser set it
    }
    var request = new XMLHttpRequest();
    // HTTP basic authentication
    if (config.auth) {
      var username = config.auth.username || "";
      var password = config.auth.password
        ? unescape(encodeURIComponent(config.auth.password))
        : "";
      requestHeaders.Authorization = "Basic " + btoa(username + ":" + password);
    }
    var fullPath = buildFullPath(config.baseURL, config.url);
    request.open(
      config.method.toUpperCase(),
      buildURL(fullPath, config.params, config.paramsSerializer),
      true
    );
    // Set the request timeout in MS
    request.timeout = config.timeout;
    function onloadend() {
      if (!request) {
        return;
      }
      // Prepare the response
      var responseHeaders =
        "getAllResponseHeaders" in request
          ? parseHeaders(request.getAllResponseHeaders())
          : null;
      var responseData =
        !responseType || responseType === "text" || responseType === "json"
          ? request.responseText
          : request.response;
      var response = {
        data: responseData,
        status: request.status,
        statusText: request.statusText,
        headers: responseHeaders,
        config: config,
        request: request,
      };
      settle(
        function _resolve(value) {
          resolve(value);
          done();
        },
        function _reject(err) {
          reject(err);
          done();
        },
        response
      );
      // Clean up request
      request = null;
    }
    if ("onloadend" in request) {
      // Use onloadend if available
      request.onloadend = onloadend;
    } else {
      // Listen for ready state to emulate onloadend
      request.onreadystatechange = function handleLoad() {
        if (!request || request.readyState !== 4) {
          return;
        }
        // The request errored out and we didn't get a response, this will be
        // handled by onerror instead
        // With one exception: request that using file: protocol, most browsers
        // will return status as 0 even though it's a successful request
        if (
          request.status === 0 &&
          !(request.responseURL && request.responseURL.indexOf("file:") === 0)
        ) {
          return;
        }
        // readystate handler is calling before onerror or ontimeout handlers,
        // so we should call onloadend on the next 'tick'
        setTimeout(onloadend);
      };
    }
    // Handle browser request cancellation (as opposed to a manual cancellation)
    request.onabort = function handleAbort() {
      if (!request) {
        return;
      }
      reject(createError("Request aborted", config, "ECONNABORTED", request));
      // Clean up request
      request = null;
    };
    // Handle low level network errors
    request.onerror = function handleError() {
      // Real errors are hidden from us by the browser
      // onerror should only fire if it's a network error
      reject(createError("Network Error", config, null, request));
      // Clean up request
      request = null;
    };
    // Handle timeout
    request.ontimeout = function handleTimeout() {
      var timeoutErrorMessage = config.timeout
        ? "timeout of " + config.timeout + "ms exceeded"
        : "timeout exceeded";
      var transitional = config.transitional || defaults.transitional;
      if (config.timeoutErrorMessage) {
        timeoutErrorMessage = config.timeoutErrorMessage;
      }
      reject(
        createError(
          timeoutErrorMessage,
          config,
          transitional.clarifyTimeoutError ? "ETIMEDOUT" : "ECONNABORTED",
          request
        )
      );

      // Clean up request
      request = null;
    };
    // Add xsrf header
    // This is only done if running in a standard browser environment.
    // Specifically not if we're in a web worker, or react-native.
    if (utils.isStandardBrowserEnv()) {
      // Add xsrf header
      var xsrfValue =
        (config.withCredentials || isURLSameOrigin(fullPath)) &&
        config.xsrfCookieName
          ? cookies.read(config.xsrfCookieName)
          : undefined;

      if (xsrfValue) {
        requestHeaders[config.xsrfHeaderName] = xsrfValue;
      }
    }
    // Add headers to the request
    if ("setRequestHeader" in request) {
      utils.forEach(requestHeaders, function setRequestHeader(val, key) {
        if (
          typeof requestData === "undefined" &&
          key.toLowerCase() === "content-type"
        ) {
          // Remove Content-Type if data is undefined
          delete requestHeaders[key];
        } else {
          // Otherwise add header to the request
          request.setRequestHeader(key, val);
        }
      });
    }
    // Add withCredentials to request if needed
    if (!utils.isUndefined(config.withCredentials)) {
      request.withCredentials = !!config.withCredentials;
    }
    // Add responseType to request if needed
    if (responseType && responseType !== "json") {
      request.responseType = config.responseType;
    }
    // Handle progress if needed
    if (typeof config.onDownloadProgress === "function") {
      request.addEventListener("progress", config.onDownloadProgress);
    }
    // Not all browsers support upload events
    if (typeof config.onUploadProgress === "function" && request.upload) {
      request.upload.addEventListener("progress", config.onUploadProgress);
    }
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
    if (!requestData) {
      requestData = null;
    }
    // Send the request
    request.send(requestData);
  });
};
```
