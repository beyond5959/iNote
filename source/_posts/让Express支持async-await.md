---
title: 让Express支持async/await
date: 2017-10-07 16:25:20
tags: [Node.js, Express]
---
随着 Node.js v8 的发布，Node.js 已原生支持 async/await 函数，Web 框架 Koa 也随之发布了 Koa 2 正式版，支持 async/await 中间件，为处理异步回调带来了极大的方便。
<!-- more -->
既然 Koa 2 已经支持 async/await 中间件了，为什么不直接用 Koa，而还要去改造 Express 让其支持 async/await 中间件呢？因为 Koa 2 正式版发布才不久，而很多老项目用的都还是 Express，不可能将其推倒用 Koa 重写，这样成本太高，但又想用到新语法带来的便利，那就只能对 Express 进行改造了，而且这种改造必须是对业务无侵入的，不然会带来很多的麻烦。

### 直接使用 async/await
让我们先来看下在 Express 中直接使用 async/await 函数的情况。
```javascript
const express = require('express');
const app = express();
const { promisify } = require('util');
const { readFile } = require('fs');
const readFileAsync = promisify(readFile);
   
app.get('/', async function (req, res, next) {
  const data = await readFileAsync('./package.json');
  res.send(data.toString());
});
// Error Handler
app.use(function (err, req, res, next) {
  console.error('Error:', err);
  res.status(500).send('Service Error');
});
   
app.listen(3000, '127.0.0.1', function () {
  console.log(`Server running at http://${ this.address().address }:${ this.address().port }/`);
});
```
上面是没有对 Express 进行改造，直接使用 async/await 函数来处理请求，当请求`http://127.0.0.1:3000/`时，发现请求能正常请求，响应也能正常响应。这样似乎不对 Express 做任何改造也能直接使用 async/await 函数，但如果 async/await 函数里发生了错误能不能被我们的错误处理中间件处理呢？现在我们去读取一个不存在文件，例如将之前读取的`package.json`换成`age.json`。
```javascript
app.get('/', async function (req, res, next) {
  const data = await readFileAsync('./age.json');
  res.send(data.toString());
});
```
现在我们去请求`http://127.0.0.1:3000/`时，发现请求迟迟不能响应，最终会超时。而在终端报了如下的错误：
![UnhandlerRejectionError](http://nnblog-storage.b0.upaiyun.com/img/unhandlererror.png)
发现错误并没有被错误处理中间件处理，而是抛出了一个`unhandledRejection`异常，现在如果我们用 try/catch 来手动捕获错误会是什么情况呢？
```javascript
app.get('/', async function (req, res, next) {
  try {
    const data = await readFileAsync('./age.json');
    res.send(datas.toString());
  } catch(e) {
    next(e);
  }
});
```
发现请求被错误处理中间件处理了，说明我们手动显式的来捕获错误是可以的，但是如果在每个中间件或请求处理函数里面加一个 try/catch 也太不优雅了，对业务代码有一定的侵入性，代码也显得难看。所以通过直接使用 async/await 函数的实验，我们发现对 Express 改造的方向就是能够接收 async/await 函数里面抛出的错误，又对业务代码没有侵入性。

### 改造 Express
在 Express 中有两种方式来处理路由和中间件，一种是通过 Express 创建的 app，直接在 app 上添加中间件和处理路由，像下面这样：
```javascript
const express = require('express');
const app = express();
   
app.use(function (req, res, next) {
  next();
});
app.get('/', function (req, res, next) {
  res.send('hello, world');
});
app.post('/', function (req, res, next) {
  res.send('hello, world');
});
   
app.listen(3000, '127.0.0.1', function () {
  console.log(`Server running at http://${ this.address().address }:${ this.address().port }/`);
});
```
另外一种是通过 Express 的 Router 创建的路由实例，直接在路由实例上添加中间件和处理路由，像下面这样：
```javascript
const express = require('express');
const app = express();
const router = new express.Router();
app.use(router);
   
router.get('/', function (req, res, next) {
  res.send('hello, world');
});
router.post('/', function (req, res, next) {
  res.send('hello, world');
});
   
app.listen(3000, '127.0.0.1', function () {
  console.log(`Server running at http://${ this.address().address }:${ this.address().port }/`);
});
```
这两种方法可以混合起来用，现在我们思考一下怎样才能让一个形如`app.get('/', async function(req, res, next){})`的函数，让里面的 async 函数抛出的错误能被统一处理呢？要让错误被统一的处理当然要调用 `next(err)` 来让错误被传递到错误处理中间件，又由于 async 函数返回的是 Promise，所以肯定是形如这样的`asyncFn().then().catch(function(err){ next(err) })`，所以按这样改造一下就有如下的代码：
```javascript
app.get = function (...data) {
  const params = [];
  for (let item of data) {
    if (Object.prototype.toString.call(item) !== '[object AsyncFunction]') {
      params.push(item);
      continue;
    }
    const handle = function (...data) {
      const [ req, res, next ] = data;
      item(req, res, next).then(next).catch(next);
    };
    params.push(handle);
  }
  app.get(...params)
}
```
上面的这段代码中，我们判断`app.get()`这个函数的参数中，若有 async 函数，就采用`item(req, res, next).then(next).catch(next);`来处理，这样就能捕获函数内抛出的错误，并传到错误处理中间件里面去。但是这段代码有一个明显的错误就是最后调用 app.get()，这样就递归了，破坏了 app.get 的功能，也根本处理不了请求，因此还需要继续改造。
我们之前说 Express 两种处理路由和中间件的方式可以混用，那么我们就混用这两种方式来避免递归，代码如下：
```javascript
const express = require('express');
const app = express();
const router = new express.Router();
app.use(router);
    
app.get = function (...data) {
  const params = [];
  for (let item of data) {
    if (Object.prototype.toString.call(item) !== '[object AsyncFunction]') {
      params.push(item);
      continue;
    }
    const handle = function (...data) {
      const [ req, res, next ] = data;
      item(req, res, next).then(next).catch(next);
    };
    params.push(handle);
  }
  router.get(...params)
}
```
像上面这样改造之后似乎一切都能正常工作了，能正常处理请求了。但通过查看 Express 的源码，发现这样破坏了 app.get() 这个方法，因为 app.get() 不仅能用来处理路由，而且还能用来获取应用的配置，在 Express 中对应的源码如下：
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
所以在改造时，我们也需要对 app.get 做特殊处理。在实际的应用中我们不仅有 get 请求，还有 post、put 和 delete 等请求，所以我们最终改造的代码如下：
```javascript
const { promisify } = require('util');
const { readFile } = require('fs');
const readFileAsync = promisify(readFile);
const express = require('express');
const app = express();
const router = new express.Router();
const methods = [ 'get', 'post', 'put', 'delete' ];
app.use(router);
    
for (let method of methods) {
  app[method] = function (...data) {
    if (method === 'get' && data.length === 1) return app.set(data[0]);

    const params = [];
    for (let item of data) {
      if (Object.prototype.toString.call(item) !== '[object AsyncFunction]') {
        params.push(item);
        continue;
      }
      const handle = function (...data) {
        const [ req, res, next ] = data;
        item(req, res, next).then(next).catch(next);
      };
      params.push(handle);
    }
    router[method](...params);
  };
}
      
app.get('/', async function (req, res, next) {
  const data = await readFileAsync('./package.json');
  res.send(data.toString());
});
      
app.post('/', async function (req, res, next) {
  const data = await readFileAsync('./age.json');
  res.send(data.toString());
});
    
router.use(function (err, req, res, next) {
  console.error('Error:', err);
  res.status(500).send('Service Error');
}); 
     
app.listen(3000, '127.0.0.1', function () {
  console.log(`Server running at http://${ this.address().address }:${ this.address().port }/`);
});
```
现在就改造完了，我们只需要加一小段代码，就可以直接用 async function 作为 handler 处理请求，对业务也毫无侵入性，抛出的错误也能传递到错误处理中间件。
