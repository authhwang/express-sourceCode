# application.js

## 1 模块依赖

```javascript
var finalhandler = require('finalhandler');
var Router = require('./router');
var methods = require('methods');
var middleware = require('./middleware/init');
var query = require('./middleware/query');
var debug = require('debug')('express:application');
var View = require('./view');
var http = require('http');
var compileETag = require('./utils').compileETag;
var compileQueryParser = require('./utils').compileQueryParser;
var compileTrust = require('./utils').compileTrust;
var deprecate = require('depd')('express');
var flatten = require('array-flatten');
var merge = require('utils-merge');
var resolve = require('path').resolve;
var setPrototypeOf = require('setprototypeof')
var slice = Array.prototype.slice;
```
## 2 导出对象
```javascript
var app = exports = module.exports = {};
```
整个文件暴露就是这个app对象 初始化时里面没有东西的 后面会补充。

## 3 app.init()
```javascript
app.init = function init() {
  this.cache = {};
  this.engines = {};
  this.settings = {};

  this.defaultConfiguration();
};
```
添加三个空对象属性 cache engines setting 后面会补充内容 

## 4 app.defaultConfiguration()
```javascript
app.defaultConfiguration = function defaultConfiguration() {
  var env = process.env.NODE_ENV || 'development';

  // default settings
  this.enable('x-powered-by');
  this.set('etag', 'weak');
  this.set('env', env);
  this.set('query parser', 'extended');
  this.set('subdomain offset', 2);
  this.set('trust proxy', false);

  // trust proxy inherit back-compat
  Object.defineProperty(this.settings, trustProxyDefaultSymbol, {
    configurable: true,
    value: true
  });

  debug('booting in %s mode', env);

  this.on('mount', function onmount(parent) {
    // inherit trust proxy
    if (this.settings[trustProxyDefaultSymbol] === true
      && typeof parent.settings['trust proxy fn'] === 'function') {
      delete this.settings['trust proxy'];
      delete this.settings['trust proxy fn'];
    }

    // inherit protos
    setPrototypeOf(this.request, parent.request)
    setPrototypeOf(this.response, parent.response)
    setPrototypeOf(this.engines, parent.engines)
    setPrototypeOf(this.settings, parent.settings)
  });

  // setup locals
  this.locals = Object.create(null);

  // top-most app is mounted at /
  this.mountpath = '/';

  // default locals
  this.locals.settings = this.settings;

  // default configuration
  this.set('view', View);
  this.set('views', resolve('views'));
  this.set('jsonp callback name', 'callback');

  if (env === 'production') {
    this.enable('view cache');
  }

  Object.defineProperty(this, 'router', {
    get: function() {
      throw new Error('\'app.router\' is deprecated!\nPlease see the 3.x to 4.x migration guide for details on how to update your app.');
    }
  });
};
```
1.获取环境 
```javascrpit
 var env = process.env.NODE_ENV || 'development';
 ```
2.设置网络有关的配置
- [tag]
- [x-powered-by]
- [query parser]
- [subdomain offset]
- [trust proxy]
<br>
我所了解的只有x-powered-by 是隐藏响应头的重要信息 
query parser 是查询字符串解析
trust proxy 使用代理服务器
subdomain offset 所忽略的字符偏移量(不过不知道有什么用。。。)
以后了解后会补上

3.在setting设置为```trustProxyDefaultSymbol```的属性<br>
4.设置mount事件 当使用代理服务器或者子应用时候才会响应事件(主要用于子域名时使用)<br>
5.设置local属性<br>
6.设置项目根目录<br>
7.将app的setting属性给local属性中的setting<br>
8.用```app.set()```方法在```app.setting```属性上做设置<br>
9.如果环境是production 就开启```view cache```(页面缓存的作用)<br>
10.设置无法直接在```app.router```上获取路由<br>

初始化app的属性就是这两个函数进行的

## 5 app.lazyrouter()

```javascript
app.lazyrouter = function lazyrouter() {
  if (!this._router) {
    this._router = new Router({
      caseSensitive: this.enabled('case sensitive routing'),
      strict: this.enabled('strict routing')
    });

    this._router.use(query(this.get('query parser fn')));
    this._router.use(middleware.init(this));
  }
};
```
对```app._router```的属性懒加载

源码解释为啥这段代码不放在初始化的方法上 因为担心```app.setting```属性会有改变 所以等他初始化后再创建

## 6 app.handle()
```javascript
app.handle = function handle(req, res, callback) {
  var router = this._router;

  // final handler
  var done = callback || finalhandler(req, res, {
    env: this.get('env'),
    onerror: logerror.bind(this)
  });

  // no routes
  if (!router) {
    debug('no routes defined on app');
    done();
    return;
  }

  router.handle(req, res, done);
};
```
1.获取路由属性```_router```<br>
2.判断是否有callback 假如没有就```finalhandler``` (这finalhandler是一个只进行错误处理的callback)<br>
3.判断是否没有router (不过这是express开发者的调试判断 正常是不会进去的)<br>
4.然后交由路由处理

## 7 app.use()
```javascript
app.use = function use(fn) {
  var offset = 0;
  var path = '/';

  // default path to '/'
  // disambiguate app.use([fn])
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

  var fns = flatten(slice.call(arguments, offset));

  if (fns.length === 0) {
    throw new TypeError('app.use() requires middleware functions');
  }

  // setup router
  this.lazyrouter();
  var router = this._router;

  fns.forEach(function (fn) {
    // non-express app
    if (!fn || !fn.handle || !fn.set) {
      return router.use(path, fn);
    }

    debug('.use app under %s', path);
    fn.mountpath = path;
    fn.parent = this;

    // restore .app property on req and res
    router.use(path, function mounted_app(req, res, next) {
      var orig = req.app;
      fn.handle(req, res, function (err) {
        setPrototypeOf(req, orig.request)
        setPrototypeOf(res, orig.response)
        next(err);
      });
    });

    // mounted an app
    fn.emit('mount', this);
  }, this);

  return this;
};
```

1.设置路径变量```path``` 和偏移变量```offset ```<br>
2.进行判断：
- [如果参数不是函数]
	1.把参数的值传给变量
	2.假如该变量是复合数组 则一直循环直到第一个值不是数组
	3.如果该变量还不是函数 则偏移变量为1 路径变量为该变量的值

3.通过偏移变量获取下一个参数，并将其扁平化(即将复合数组变成为一个数组)
4.假如下一个参数都没有callback 则报错
5.懒加载获取路由
6.对callback数组进行循环处理
- [假如该callback不含handle和set时]
		则使用```router.use```方法
- [假如有]
		1.设置子应用的```mountpath```为路径变量
		2.设置子应用的父应用
		3.重新保存子应用的```req```属性和```res```属性
		4.触发```emit```事件

## 8 app.route()
```javascript
app.route = function(path){
  this.lazyrouter();
  return this._router.route(path);
};
```
```Router.route```的代理方法

## 9 app.engine()
```javascript
app.engine = function engine(ext, fn) {
  if (typeof fn !== 'function') {
    throw new Error('callback function required');
  }

  // get file extension
  var extension = ext[0] !== '.'
    ? '.' + ext
    : ext;

  // store engine
  this.engines[extension] = fn;

  return this;
};

```

假如所设置的引擎没加后缀符号```.``` 则补充上并将其设置在```engines```属性上

## 10 app.param()
```javascript
app.param = function param(name, fn) {
  this.lazyrouter();

  if (Array.isArray(name)) {
    for (var i = 0; i < name.length; i++) {
      this.param(name[i], fn);
    }

    return this;
  }

  this._router.param(name, fn);

  return this;
};

```
```Router.param()```的代理方法 假如name是数组 则遍历调用```Router.param()```

## 11 app.set()
```javascript
app.set = function set(setting, val) {
  if (arguments.length === 1) {
    // app.get(setting)
    return this.settings[setting];
  }

  debug('set "%s" to %o', setting, val);

  // set value
  this.settings[setting] = val;

  // trigger matched settings
  switch (setting) {
    case 'etag':
      this.set('etag fn', compileETag(val));
      break;
    case 'query parser':
      this.set('query parser fn', compileQueryParser(val));
      break;
    case 'trust proxy':
      this.set('trust proxy fn', compileTrust(val));

      // trust proxy inherit back-compat
      Object.defineProperty(this.settings, trustProxyDefaultSymbol, {
        configurable: true,
        value: false
      });

      break;
  }

  return this;
};
```
将想要保存的配置信息保存在```app```的```setting```属性<br>
1.假如只有一个参数 则是getter方法<br>
2.在```setting```属性设置键值<br>
3.假如要设置的键名是```etag``` ```query parser``` ```trust proxy```其中一个 则另做处理<br>

## 12 app.path()
```javascript
app.path = function path() {
  return this.parent
    ? this.parent.path() + this.mountpath
    : '';
};
```
返回```app```的绝对路径 假如是子应用则在父应用的绝对路径加子应用的绝对路径


## 13 app.enabled()
```javascript
app.enabled = function enabled(setting) {
  return Boolean(this.set(setting));
};
```
检查该```setting```是不是可用(enable)

## 14 app.disabled()
```javascript
app.disabled = function disabled(setting) {
  return !this.set(setting);
};
```
检查该```setting```是不是不可用(disable)

## 15 app.enable()
```javascript
app.enable = function enable(setting) {
  return this.set(setting, true);
};
```
设置该```setting```为可用

## 16 app.disable()
```javascript
app.disable = function disable(setting) {
  return this.set(setting, false);
};

```
设置该```setting```为不可用

## 17 methods.foreach()
```javascript
methods.forEach(function(method){
  app[method] = function(path){
    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter();

    var route = this._router.route(path);
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});

```
```methods```是模块```methods```所暴露的方法<br>
是为了获取http所有VERB <br>
1.将VERB数组进行遍历<br>
2.给```app```添加所有VERB的方法 <br>
3.该方法是 假如```get```方法 而且参数只有一个 则判断为是调用```app.set```方法<br>
4.假如不是 则调用```route.VERB```方法进行处理<br>

## 18 app.all()
```javascript
app.all = function all(path) {
  this.lazyrouter();

  var route = this._router.route(path);
  var args = slice.call(arguments, 1);

  for (var i = 0; i < methods.length; i++) {
    route[methods[i]].apply(route, args);
  }

  return this;
};
```
调用```app```中的每一个VERB方法

## 19 app.listen()
```javascript
app.listen = function listen() {
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```
可以跟前面的```express.js```中的代码:
```javascript
var app = function(req, res, next) {
    app.handle(req, res, next);
  };
```
进行呼应 为什么app一开始是一个函数 因为会放在```http.createServer(req,res,next)```的回调函数中 因此整个```express```可以说是一个函数 也是在这里做文章 后面的调用也是交给了```app.handle()```处理

## 20 app.render()
```javascript
app.render = function render(name, options, callback) {
  var cache = this.cache;
  var done = callback;
  var engines = this.engines;
  var opts = options;
  var renderOptions = {};
  var view;

  // support callback function as second arg
  if (typeof options === 'function') {
    done = options;
    opts = {};
  }

  // merge app.locals
  merge(renderOptions, this.locals);

  // merge options._locals
  if (opts._locals) {
    merge(renderOptions, opts._locals);
  }

  // merge options
  merge(renderOptions, opts);

  // set .cache unless explicitly provided
  if (renderOptions.cache == null) {
    renderOptions.cache = this.enabled('view cache');
  }

  // primed cache
  if (renderOptions.cache) {
    view = cache[name];
  }

  // view
  if (!view) {
    var View = this.get('view');

    view = new View(name, {
      defaultEngine: this.get('view engine'),
      root: this.get('views'),
      engines: engines
    });

    if (!view.path) {
      var dirs = Array.isArray(view.root) && view.root.length > 1
        ? 'directories "' + view.root.slice(0, -1).join('", "') + '" or "' + view.root[view.root.length - 1] + '"'
        : 'directory "' + view.root + '"'
      var err = new Error('Failed to lookup view "' + name + '" in views ' + dirs);
      err.view = view;
      return done(err);
    }

    // prime the cache
    if (renderOptions.cache) {
      cache[name] = view;
    }
  }

  // render
  tryRender(view, renderOptions, done);
};

```
这个东西不是今晚要聊的板子 所以以后要研究的时候再看 接下来再研究```router.js```
