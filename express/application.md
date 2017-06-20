# application.js

## 1.模块依赖

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
## 2.导出对象
```javascript
var app = exports = module.exports = {};
```
整个文件暴露就是这个app对象 初始化时里面没有东西的 后面会补充。

## 3.app.init()
```javascript
app.init = function init() {
  this.cache = {};
  this.engines = {};
  this.settings = {};

  this.defaultConfiguration();
};
```
添加三个空对象属性 cache engines setting 后面会补充内容 

## 4.app.defaultConfiguration()
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
我所了解的只有x-powered-by 是隐藏响应头的重要信息 
query parser 是查询字符串解析
trust proxy 使用代理服务器
subdomain offset 所忽略的字符偏移量(不过不知道有什么用。。。)
以后了解后会补上

3.在setting设置为```trustProxyDefaultSymbol```的属性
4.设置mount事件 当使用代理服务器或者子应用时候才会响应事件
5.设置local属性
6.设置项目根目录
7.将app的setting属性给local属性中的setting
8.用```app.set()```方法在```app.setting```属性上做设置
9.如果环境是production 就开启```view cache```(页面缓存的作用)
10.设置无法直接在```app.router```上获取路由

初始化app的属性就是这两个函数进行的