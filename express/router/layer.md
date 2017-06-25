# layer.md

修饰中间件的类



## 1 引入模块

```javascript
/**
 * Module dependencies.
 * @private
 */

var pathRegexp = require('path-to-regexp');
var debug = require('debug')('express:router:layer');

/**
 * Module variables.
 * @private
 */

var hasOwnProperty = Object.prototype.hasOwnProperty;
```

模块`path-to-regexp`是将路径转换为正则表达式 例如:

``````javascript
var pathToRegexp = require('path-to-regexp');

var keys = [];
var re = pathToRegexp('/foo/:bar', keys);
// re = /^\/foo\/([^\/]+?)\/?$/i
// keys = [{
//  name: 'bar',
//  prefix: '/',
//  delimiter: '/',
//  optional: false,
//  repeat: false,
//  pattern: '[^\\/]+?'
// }]

re.exec('/foo/test');
// => ['/foo/test', 'test']
``````



##  2 导出对象

```javascript
module.exports = Layer;

function Layer(path, options, fn) {
  if (!(this instanceof Layer)) {
    return new Layer(path, options, fn);
  }

  debug('new %o', path)
  var opts = options || {};

  this.handle = fn;
  this.name = fn.name || '<anonymous>';
  this.params = undefined;
  this.path = undefined;
  this.regexp = pathRegexp(path, this.keys = [], opts);

  // set fast path flags
  this.regexp.fast_star = path === '*'
  this.regexp.fast_slash = path === '/' && opts.end === false
}
```

1.创建`layer`的构造方法

其中`this.handle`是所有use或者verb方法调用的回调 `this.name`是函数名 `this.keys`就是在`pathRegexp()`函数下获取的

2.`this.regexp.fast_star`假如`path`为'*' 即任意路径 `this.regexp.fast_slash`假如`path`为'/'且`opts.end`等于false

3.在`route`的`all`或者`其他VERB`下创建的layer 是`Layer('/', {}, fn) `  因为调用这些方法的路径有可能会请求参数 没必要省略 

4.在`router`的use调用 创建的layer是`options.end`为`false` 因为很多都可能是在路由前要设置的中间件 属于通用中间件 所以假如他是 则只要判断路径是否为'/'即可

5.在`router`的route调用 创建的layer是`options.end`为`true`  跟第3个是差不多的道理



## 3 layer.match()

```javascript
/**
 * Check if this route matches `path`, if so
 * populate `.params`.
 *
 * @param {String} path
 * @return {Boolean}
 * @api private
 */

Layer.prototype.match = function match(path) {
  var match

  if (path != null) {
    // fast path non-ending match for / (any path matches)
    if (this.regexp.fast_slash) {
      this.params = {}
      this.path = ''
      return true
    }

    // fast path for * (everything matched in a param)
    if (this.regexp.fast_star) {
      this.params = {'0': decode_param(path)}
      this.path = path
      return true
    }

    // match the path
    match = this.regexp.exec(path)
  }

  if (!match) {
    this.params = undefined;
    this.path = undefined;
    return false;
  }

  // store values
  this.params = {};
  this.path = match[0]

  var keys = this.keys;
  var params = this.params;

  for (var i = 1; i < match.length; i++) {
    var key = keys[i - 1];
    var prop = key.name;
    var val = decode_param(match[i])

    if (val !== undefined || !(hasOwnProperty.call(params, prop))) {
      params[prop] = val;
    }
  }

  return true;
};
```

1.在`router.handle()`下的`while (match !== true && idx < stack.length)`使用 判断路径是否一致

2.`this.regexp.fast_slash`为true 则是没有请求参数的'/' 可以直接return true

3.`this.regexp.fast_star`为true 则是路径为'*',则算是任意路径(我想) 不过在请求参数的设置不是很懂 懂了的话会补充

4.对参数`path`正则判断 如果是false 返回return false 如果是true 则进行第5步

5.通过正则获取到的请求参数 遍历 获取layer的key值 对请求参数进行编码转换 然后将key 和 value 放在`layer.params`中



## 4.layer.handle_request()  

```javascript
/**
 * Handle the request for the layer.
 *
 * @param {Request} req
 * @param {Response} res
 * @param {function} next
 * @api private
 */

Layer.prototype.handle_request = function handle(req, res, next) {
  var fn = this.handle;

  if (fn.length > 3) {
    // not a standard request handler
    return next();
  }

  try {
    fn(req, res, next);
  } catch (err) {
    next(err);
  }
};

```

1.假如回调的函数参数有大于3个就不是标准的请求回调  则调用`next()`进行下一个中间件的调用

2.然后调用回调方法

layer.handle_error 差不多的道理