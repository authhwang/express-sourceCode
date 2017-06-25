# index.js

这章有点难 感觉要看多几次才十分懂

## 1 依赖对象
```javascript
var Route = require('./route');
var Layer = require('./layer');
var methods = require('methods');
var mixin = require('utils-merge');
var debug = require('debug')('express:router');
var deprecate = require('depd')('express');
var flatten = require('array-flatten');
var parseUrl = require('parseurl');
var setPrototypeOf = require('setprototypeof')

var objectRegExp = /^\[object (\S+)\]$/;
var slice = Array.prototype.slice;
var toString = Object.prototype.toString;

```

## 2 导出对象与初始化配置
```javascript
/**
 * Initialize a new `Router` with the given `options`.
 *
 * @param {Object} options
 * @return {Router} which is an callable function
 * @public
 */

var proto = module.exports = function(options) {
  var opts = options || {};

  function router(req, res, next) {
    router.handle(req, res, next);
  }

  // mixin Router class functions
  setPrototypeOf(router, proto)

  router.params = {};
  router._params = [];
  router.caseSensitive = opts.caseSensitive;
  router.mergeParams = opts.mergeParams;
  router.strict = opts.strict;
  router.stack = [];

  return router;
};
```
1.option的内容有之前在```app.lazyRouter```上看到的```javascript
this._router = new Router({
      caseSensitive: this.enabled('case sensitive routing'),
      strict: this.enabled('strict routing')
    });
``` 还有一个```mergeParams```

2.
```javascript
function router(req, res, next) {
    router.handle(req, res, next);
  }
```
其实我觉得没必要将router写成函数 因为在```application.js```中的app.handle上也是执行```router.hanlde```方法 整个源码都不会直接执行```router(req,res,next)``` 所以感觉没必要(知道为啥需要写成函数 因为可能会直接用```app.use(router)```来使用路由中间件)

3.将router函数与本身的```route```做原型绑定 那可以通过router获取route里暴露的方法

4.初始化要保存信息的配置 并将router返回
- [params]
- [_params]
- [caseSensitive]
- [mergeParams]
- [strict]
- [stack]

## 3 router.use()
```javascript
/**
 * Use the given middleware function, with optional path, defaulting to "/".
 *
 * Use (like `.all`) will run for any http METHOD, but it will not add
 * handlers for those methods so OPTIONS requests will not consider `.use`
 * functions even if they could respond.
 *
 * The other difference is that _route_ path is stripped and not visible
 * to the handler function. The main effect of this feature is that mounted
 * handlers can operate without any code changes regardless of the "prefix"
 * pathname.
 *
 * @public
 */

proto.use = function use(fn) {
  var offset = 0;
  var path = '/';

  // default path to '/'
  // disambiguate router.use([fn])
  if (typeof fn !== 'function') {
    var arg = fn;

    while (Array.isArray(arg) && arg.length !== 0) {
      arg = arg[0];
    }

    // first arg is the path
    if (typeof arg !== 'function') {
      offset = 1;
      path = fn;
    }
  }

  var callbacks = flatten(slice.call(arguments, offset));

  if (callbacks.length === 0) {
    throw new TypeError('Router.use() requires middleware functions');
  }

  for (var i = 0; i < callbacks.length; i++) {
    var fn = callbacks[i];

    if (typeof fn !== 'function') {
      throw new TypeError('Router.use() requires middleware function but got a ' + gettype(fn));
    }

    // add the middleware
    debug('use %o %s', path, fn.name || '<anonymous>')

    var layer = new Layer(path, {
      sensitive: this.caseSensitive,
      strict: false,
      end: false
    }, fn);

    layer.route = undefined;

    this.stack.push(layer);
  }

  return this;
};
```
1.设置默认的路径```／```以及参数偏移值<br>
2.如果第一个参数不是方法 则遍历数组直到数组或者复合数组中的第一个值不是数组并保存为变量 <br>
3.如果该变量不是方法 则把该变量保存为路径 并设置参数偏移为1<br>
4.获取经过参数偏移后的参数数组并将其扁平化<br>
5.如果数组为空 则报错
6.遍历偏移后的参数数组 如果该参数不是方法则报错 然后创建一个```Layer```对象(在源码里中间件都叫```layer```) 将layer.route置为```undefined``` (稍后会说有啥用)
7.将该中间件加到```stack```数组中
8.返回router

例子:
```javascript
var express = require('express');
var app = express();

app.use(function foo() {});
app.use('/users', function bar() {}, function test() {});

console.log(app._router.stack);
```
结果:<br>
```
[{
	handle: [Function: query],
	name: 'query',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: {
		/^\/?(?=\/|$)/i
		fast_slash: true
	},
	route: undefined
}, {
	handle: [Function: expressInit],
	name: 'expressInit',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: {
		/^\/?(?=\/|$)/i
		fast_slash: true
	},
	route: undefined
}, {
	handle: [Function: foo],
	name: 'foo',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: {
		/^\/?(?=\/|$)/i
		fast_slash: true
	},
	route: undefined
}, {
	handle: [Function: bar],
	name: 'bar',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: /^\/users\/?(?=\/|$)/i,
	route: undefined
}, {
	handle: [Function: test],
	name: 'test',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: /^\/users\/?(?=\/|$)/i,
	route: undefined
}]
```

其中  第一第二个是前面```application.js```中的```app.lazyrouter```方法调用时加上去的
```javascript
this._router.use(query(this.get('query parser fn')));
this._router.use(middleware.init(this));
```

## 4 router.route()

```javascript
/**
 * Create a new Route for the given path.
 *
 * Each route contains a separate middleware stack and VERB handlers.
 *
 * See the Route api documentation for details on adding handlers
 * and middleware to routes.
 *
 * @param {String} path
 * @return {Route}
 * @public
 */

proto.route = function route(path) {
  var route = new Route(path);

  var layer = new Layer(path, {
    sensitive: this.caseSensitive,
    strict: this.strict,
    end: true
  }, route.dispatch.bind(route));

  layer.route = route;

  this.stack.push(layer);
  return route;
};
```

通过给定的路径创建新的路由中间件 每个路由都有自己的中间件队列和VERB handler 新的路由又有自己要走的function队列 变成每个可以独立起来

1.创建新的路由中间件(我觉得算是一个中间件)<br>
2.创建一个```Layer ```对象 然后用了一个```route.dispatch()```暂时不知道有啥用。。<br>
3.将```Layer```对象放进新的```router```的```stack```队列上<br>
4.返回新的```router```<br>

## 5 VERB方法
```javascript
methods.concat('all').forEach(function(method){
  proto[method] = function(path){
    var route = this.route(path)
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```
将all方法也加上 遍历VERB数组给router[VERB]设置函数： 根据路径创建一个新```route``` 并调用```route```的VERB方法

## 6 router.param()
```javascript
/**
 * Map the given param placeholder `name`(s) to the given callback.
 *
 * Parameter mapping is used to provide pre-conditions to routes
 * which use normalized placeholders. For example a _:user_id_ parameter
 * could automatically load a user's information from the database without
 * any additional code,
 *
 * The callback uses the same signature as middleware, the only difference
 * being that the value of the placeholder is passed, in this case the _id_
 * of the user. Once the `next()` function is invoked, just like middleware
 * it will continue on to execute the route, or subsequent parameter functions.
 *
 * Just like in middleware, you must either respond to the request or call next
 * to avoid stalling the request.
 *
 *  app.param('user_id', function(req, res, next, id){
 *    User.find(id, function(err, user){
 *      if (err) {
 *        return next(err);
 *      } else if (!user) {
 *        return next(new Error('failed to load user'));
 *      }
 *      req.user = user;
 *      next();
 *    });
 *  });
 *
 * @param {String} name
 * @param {Function} fn
 * @return {app} for chaining
 * @public
 */

proto.param = function param(name, fn) {
  // param logic
  if (typeof name === 'function') {
    deprecate('router.param(fn): Refactor to use path params');
    this._params.push(name);
    return;
  }

  // apply param functions
  var params = this._params;
  var len = params.length;
  var ret;

  if (name[0] === ':') {
    deprecate('router.param(' + JSON.stringify(name) + ', fn): Use router.param(' + JSON.stringify(name.substr(1)) + ', fn) instead');
    name = name.substr(1);
  }

  for (var i = 0; i < len; ++i) {
    if (ret = params[i](name, fn)) {
      fn = ret;
    }
  }

  // ensure we end up with a
  // middleware function
  if ('function' !== typeof fn) {
    throw new Error('invalid param() call for ' + name + ', got ' + fn);
  }

  (this.params[name] = this.params[name] || []).push(fn);
  return this;
};

```

这个方法做的是对路由参数的预处理 假设你先对user_id 和 user_name 做了预处理 当你在处理有这两个参数之一的路径请求的回调时 使用的路由参数值就是你已经做了预处理的值 而且同一个router下假如有几个相同路径的中间件 处理一次后则无须再进行处理 猛得一批<br>
```javascript
var express = require('express');
var app = express();

app.param('user_id', function foo() {});
app.param('user_id', function bar() {});
app.param('user_name', function test() {});
```
1.如果第一个参数是```function```则添加到全局属性```_params``` 中保存并返回<br>
2.假如第一个参数含有':' 则去掉':'<br>
3.遍历```this._params``` 将里面的所有方法通过```name```和```fn```都处理一遍 (不过this.params一开始是空对象 假如没走第一步就自然跳过)
4.判断```fn```是```function```还是```error``` 假如是错误则报错
5.向```this.params[name]```添加方法

结果:<br>
```
{
	user_id: [
		[Function: foo],
		[Function: bar]
	],
	user_name: [
		[Function: test]
	]
}
```
## 7 router.handle()
```javascript
/**
 * Dispatch a req, res into the router.
 * @private
 */

proto.handle = function handle(req, res, out) {
  var self = this;

  debug('dispatching %s %s', req.method, req.url);

  var idx = 0;
  var protohost = getProtohost(req.url) || ''
  var removed = '';
  var slashAdded = false;
  var paramcalled = {};

  // store options for OPTIONS request
  // only used if OPTIONS request
  var options = [];

  // middleware and routes
  var stack = self.stack;

  // manage inter-router variables
  var parentParams = req.params;
  var parentUrl = req.baseUrl || '';
  var done = restore(out, req, 'baseUrl', 'next', 'params');

  // setup next layer
  req.next = next;

  // for options requests, respond with a default if nothing else responds
  if (req.method === 'OPTIONS') {
    done = wrap(done, function(old, err) {
      if (err || options.length === 0) return old(err);
      sendOptionsResponse(res, options, old);
    });
  }

  // setup basic req values
  req.baseUrl = parentUrl;
  req.originalUrl = req.originalUrl || req.url;

  next();

  function next(err) {
    var layerError = err === 'route'
      ? null
      : err;

    // remove added slash
    if (slashAdded) {
      req.url = req.url.substr(1);
      slashAdded = false;
    }

    // restore altered req.url
    if (removed.length !== 0) {
      req.baseUrl = parentUrl;
      req.url = protohost + removed + req.url.substr(protohost.length);
      removed = '';
    }

    // signal to exit router
    if (layerError === 'router') {
      setImmediate(done, null)
      return
    }

    // no more matching layers
    if (idx >= stack.length) {
      setImmediate(done, layerError);
      return;
    }

    // get pathname of request
    var path = getPathname(req);

    if (path == null) {
      return done(layerError);
    }

    // find next matching layer
    var layer;
    var match;
    var route;

    while (match !== true && idx < stack.length) {
      layer = stack[idx++];
      match = matchLayer(layer, path);
      route = layer.route;

      if (typeof match !== 'boolean') {
        // hold on to layerError
        layerError = layerError || match;
      }

      if (match !== true) {
        continue;
      }

      if (!route) {
        // process non-route handlers normally
        continue;
      }

      if (layerError) {
        // routes do not match with a pending error
        match = false;
        continue;
      }

      var method = req.method;
      var has_method = route._handles_method(method);

      // build up automatic options response
      if (!has_method && method === 'OPTIONS') {
        appendMethods(options, route._options());
      }

      // don't even bother matching route
      if (!has_method && method !== 'HEAD') {
        match = false;
        continue;
      }
    }

    // no match
    if (match !== true) {
      return done(layerError);
    }

    // store route for dispatch on change
    if (route) {
      req.route = route;
    }

    // Capture one-time layer values
    req.params = self.mergeParams
      ? mergeParams(layer.params, parentParams)
      : layer.params;
    var layerPath = layer.path;

    // this should be done for the layer
    self.process_params(layer, paramcalled, req, res, function (err) {
      if (err) {
        return next(layerError || err);
      }

      if (route) {
        return layer.handle_request(req, res, next);
      }

      trim_prefix(layer, layerError, layerPath, path);
    });
  }
```
贼长 感觉要分几部分来看才行 
1.设置几个变量
- [idx -中间件队列的index]
- [protohost -假如直接使用'/'则'' 不然则获取主机名]
- [remove]
- [slashAdded -去除第一个'/' 默认为false]
- [paramcalled]
- [options -在VERB为option时使用]
- [stack -获取该```router```的stack]
- [parentParams -req的请求参数]
- [parentUrl -req的请求路径的根路径 假如没有则'']
- [done -保存调用时的req中```baseurl```,```next```,```params```属性的值，并返回一个函数 该函数包含可以当时保存的值并修改到req 然后调用```out```函数]<br>
  2.设置```next```函数到```req.next```<br>
  3.假如是```option```请求 将```done```变量修改成有默认响应假如没有东西响应<br>
  4.设置```req.baseUrl```为parentUrl<br>
  5.设置```req.originalUrl``` 自身有就自身 没有就为''(req.originalUrl 就是路径加请求参数)<br>
  6.调用```next()```<br>
  7. `next(err)`的实现
    1.设置变量```layerError ``` 判断```err```参数是否等于'route'如果是则为null 否则为err<br>
    2.如果`slashAdded`为true 则去除第一个'/' <br>
    3.判断`removed` 假如有则修改`req.url`<br>
    4.如果`layerError`等于'router' 则异步调用`done` 并return<br>
    5.如果`idx`>=`stack.length`则异步调用
    `setImmediate(done, layerError);`并return
    6.获取`req.pathname`设置到`path`
    7.如果`path`为null则调用`done(laterError)`
    8.设置三个变量
    - [layer -中间件]
    - [match]
    - [route]
      9.进入循环 （这个循环是路径符合的就会跳过循环 不符合的则继续循环)<br>
       1.获取index为`idx`的中间件 设置给`layer` 然后`idx`加一<br>
       2.执行`layer.match(path)` 判断是否相同<br>
       3.获取`layer.route` 设置给`route`<br>
       4.如果`match`不为boolean 则是err 那么`layerError = layerError || match;`<br>
       5.如果`match`不为true 则路径不匹配 就会跳过这轮进入下一轮<br>
       6.如果`layer`不含`route` 则直接跳过循环 执行循环下面的代码<br>
       7. 如果`layerError`有值 则一直循环 <br>
          8.获取`req.method` 设置给`method`<br>
          9.判断`route`是否含有该VERB方法 有则返回`ture`<br>
          10.如果`has_method`为false 并`method`为`option` 则创建默认的响应<br>
          11.如果`has_method`为false 并`method`不为`HEAD ` 则将`match`设置为false 并跳过该循环<br>
          10.假如`match`不为true 则调用`done(layerError)`<br>
          11.将`layer.route`保存在`req.route`<br>
          12.设置`req.params` 并与`layer.params` 相添加(不过layer.params为undefined)
          13.设置`layer.path` 为变量`laterPath`
          14.调用`router.process_params()`进行参数预处理

## 8 route.process_params()
```javascript

/**
* Process any parameters for the layer.
* @private
   */


proto.process_params = function process_params(layer, called, req, res, done) {
  var params = this.params;

  // captured parameters from the layer, keys and values
  var keys = layer.keys;

  // fast track
  if (!keys || keys.length === 0) {
    return done();
  }

  var i = 0;
  var name;
  var paramIndex = 0;
  var key;
  var paramVal;
  var paramCallbacks;
  var paramCalled;

  // process params in order
  // param callbacks can be async
  function param(err) {
    if (err) {
      return done(err);
    }
    
    if (i >= keys.length ) {
      return done();
    }
    
    paramIndex = 0;
    key = keys[i++];
    name = key.name;
    paramVal = req.params[name];
    paramCallbacks = params[name];
    paramCalled = called[name];
    
    if (paramVal === undefined || !paramCallbacks) {
      return param();
    }
    
    // param previously called with same value or error occurred
    if (paramCalled && (paramCalled.match === paramVal
      || (paramCalled.error && paramCalled.error !== 'route'))) {
      // restore value
      req.params[name] = paramCalled.value;
    
      // next param
      return param(paramCalled.error);
    }
    
    called[name] = paramCalled = {
      error: null,
      match: paramVal,
      value: paramVal
    };
    
    paramCallback();
  }

  // single param callbacks
  function paramCallback(err) {
    var fn = paramCallbacks[paramIndex++];

    // store updated value
    paramCalled.value = req.params[key.name];
    
    if (err) {
      // store error
      paramCalled.error = err;
      param(err);
      return;
    }
    
    if (!fn) return param();
    
    try {
      fn(req, res, paramCallback, paramVal, key.name);
    } catch (e) {
      paramCallback(e);
    }
  }

  param();
};
```
进行参数的预处理 
1.获取注册过的`this.params`<br>
2.获取`layer`中的请求参数 假如该中间件没有请求参数 则调用`done`<br>
3.设置变量
- [i]
   - [key -keys[i++]]
   - [name -key.name]
   - [paramVal -req.params[name]]
   - [paramCallbacks -params[name]]
   - [paramCalled -called[name]]
   - [paramIndex -可能一个请求参数有几个预处理函数 当下一个请求参数会重新置0]

4.调用`param()`<br>
   -1.假如`paramVal === undefined`则表示没有该参数 `!paramCallbacks`表示没有该参数预处理方法<br>
   -2.假如之前有调用过或者有报错的则直接调用`param(paramCalled.error)`进入下一轮<br>
   -3.对`called`变量进行设置 这变量就是`router.hanlde`调用时创建的属性`paramcalled` 用于在该函数下全局保存预处理信息<br>
   -4.调用预处理函数`paramCallback ()`<br>
   -5.调用`paramCallback()`<br>
   ----1.在`paramCallbacks`数组下获取预处理函数<br>
   ----2.调用`fn`<br>
   ----3.老铁要注意 在预处理函数下第三个是next 在这里其实把`paramCallback`放了进去 让`paramIndex`自加 有一个就继续一个 没有就跳出函数<br>



 总结:

 写到这里其实大概了解整个express的流程 先用文字写下怕忘记
 刚创建一个express()时会有一个懒加载的router
 当假如是app.get这样的话 就会创建一个route 创建的时候会把一个新的layer和该route绑定 放在总的router的stack上 然后调用route的verb方法 创建一个新的layer并将回调方法放在route的stack上
 当执行的时候会先执行router上的layer layer的方法绑定 route 然后执行route的dispatch方法来调用route本身的stack里layer自身里面的方法


 假如回调里调用了next 就会回到router.handle里的next方法 这样就会回到在那个while循环里进行下一个中间件

 假如直接用express.router 创建新的路的router router.use方法就会把你想做的处理放在这个router的stack上 然后将这个中间件放到总的router的stack上 由于router本身就是个函数 当她被调用的时候 还是调回自身的router.handle上 然后再去处理里面的stack

 常用的两种路由加载原理就是这样 后面的只是补充

