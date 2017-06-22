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
1.设置默认的路径```\```以及参数偏移值<br>
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

