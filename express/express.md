# express.js

1.入口文件`index.js`只有一句代码
`module.exports = require('./lib/express.js');`<br>
因此我们首先从`express.js`开始看.

```javascript

exports = module.exports = createApplication;


function createApplication() {
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };

  mixin(app, EventEmitter.prototype, false);
  mixin(app, proto, false);

  // expose the prototype that will get set on requests
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  // expose the prototype that will get set on responses
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  app.init();
  return app;
}
```
这部分代码是express.js主要内容,暴露了一个创建方法.方法里面的app也是一个函数。<br>
至于为啥用一个函数而不用对象暂时没想到。然后用混入`mixin`方法获取`EventEmitter.prototype` 和`application.js里面暴露的方法和属性`.
然后再给app两个属性`request`和`response` 这个两个属性是用`Object.create`函数生成 既继承了`response.js` `request.js`所暴露的东西,还设置文件标识符的值.最后再调用`application.js`的init方法并返回app。

```javascript
exports.application = proto;
exports.request = req;
exports.response = res;

exports.Route = Route;
exports.Router = Router;

exports.query = require('./middleware/query');
exports.static = require('serve-static');
```

这部分代码是`express.js`另一半内容，这里的作用是为`createApplication`方法添加属性虽然不是很懂这样暴露有什么意义.

由于`app.init`看上去做了好多事情，接下来去看`application.js`
