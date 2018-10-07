---
title: 通过单元测试为API自动生成文档
date: 2017-01-03 23:10:26
tags: [doc]
---
在开发中，为项目生成文档是很常见的需求，很多第三方库（如[jsdoc](https://www.npmjs.com/package/jsdoc)、[swagger](https://www.npmjs.com/package/swagger)等）的做法是为需要生成文档的函数编写相应的符合规范的注释，然后运行相应的命令，生成一个静态网页形式的文档。
<!-- more -->

用注释生成文档的好处是可以为无论是普通函数还是 API，只要编写了相应的注释都能生成相应的文档，然而这种做法总觉得有点繁琐，尤其是只需要为 API 生成文档的时候，需要手动编写大量的输入和输出作为使用示例。而且我只想需要 markdown 形式的文档，丢在内部 Gitlab 的 wiki 上供前端人员査阅，然后可以根据 commit 的 history 查阅不同的版本。

不想手动为 API 文档编写大量的输入输出，那哪里会有输入输出呢，就很容易的想到了单元测试会产生输入和输出。好，那就用单元测试来为 API 生成文档。

我在单元测试中主要用的库有 [mocha](https://www.npmjs.com/package/mocha)、[supertest](https://www.npmjs.com/package/supertest)和[power-assert](https://www.npmjs.com/package/power-assert)，Web 框架为 [express](https://www.npmjs.com/package/express)。


完整代码示例：
```javascript
// app.js
const express = require('express');
const bodyParser = require('body-parser');
const app = express();
     
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));
    
app.post('/user/create', function (req, res) {
  const name = req.body.name;
  res.json({
    status: 200,
    error_code: 0,
    message: 'success',
    data: {
      id: '123abc',
      name: 'node',
      gender: 'male',
      age: 23,
    },
  });
});
     
app.get('/user/search', function (req, res) {
  const name = req.query.name;
  res.json({
    status: 200,
    error_code: 0,
    message: 'success',
    data: {
      id: '123abc',
      name: 'node',
      gender: 'male',
      age: 23,
    },
  });
});
     
app.get('/user/:id', function (req, res) {
  const userId = req.params.id;
  res.json({
    status: 200,
    error_code: 0,
    message: 'success',
    data: {
      id: '123abc',
      name: 'node',
      gender: 'male',
      age: 23,
    },
  });
});
     
app.listen(3000, function () {
  console.log(`Server is listening on 3000`);
});
module.exports = app;
```
下面是测试文件。
```javascript
// test/uset.test.js
const assert = require('power-assert');
const supertest = require('supertest');
const U = require('../utils');
const app = require('../app');
const agent = supertest.agent(app);
      
describe('Test', function () {
  it('should create user success', function (done) {
    U.test({
      agent,
      file: 'user',
      group: '用户相关API',
      title: '创建用户',
      method: 'post',
      url: '/user/create'
      params: {
        name: { value: 'node', type: 'String', required: true, desc: '名称' },
        gender: { value: 'male', type: 'String', required: false, desc: '性别' },
        age: { value: 23, type: 'Int', required: false, desc: '' },
      },
      headers: { entrance: 'client' },
      expect: 200,
      callback (err, res) {
        if (err) return done(err);

        assert(res.body.data.name === 'node');
        assert(res.body.data.age === 23);
        done();
      },
    });
  });
      
  it('should search user success', function (done) {
    U.test({
      agent,
      file: 'user',
      group: '用户相关API',
      title: '搜索用户',
      method: 'get',
      url: '/user/search'
      params: {
        name: { value: 'node', type: 'String', required: true, desc: '名称' },
      },
      expect: 200,
      callback (err, res) {
        if (err) return done(err);
     
        assert(res.body.data.name === 'node');
        assert(res.body.data.age === 23);
        done();
      },
    });
  });
     
  it('should search user success', function (done) {
    U.test({
      agent,
      file: 'user',
      group: '用户相关API',
      title: '获取用户信息',
      method: 'get',
      url: '/user/:id'
      params: {
        id: { value: '123abc', type: 'String', required: true, desc: '' },
      },
      expect: 200,
      callback (err, res) {
        if (err) return done(err);
        
        assert(res.body.data.name === 'node');
        assert(res.body.data.age === 23);
        done();
      },
    });
  });
});
```
在测试文件中，测试用例的代码调用到了 utils.js 中的 test 方法，该方法的主要作用是接收单元测试的输入和输出并生成相应的文档，其中需要向 test 方法传入一个对象作为参数，对象中的字段解读如下：
> agent：调用 API 的代理。
> file：生成的文档的文件名称。
> group：某一组文档的名称。
> title：接口的名称。
> method：接口的方法。
> params：接口的参数，即输入。
> headers: 添加到请求头中的信息。
> expect：supertest 的expect。
> callback：supertest 方法的 end 的回调函数。

utils.js代码如下：
```javascript
// utils.js
const path = require('path');
const fs = require('fs');
      
const mdStr = {};
exports.test = function (obj) {
  if (!mdStr[obj.group]) {
    mdStr[obj.group] = '';
    mdStr[obj.group] += '## ' + obj.group + '\n\n';
  }
  const fields = {};
       
  mdStr[obj.group] += `### ${ obj.title } \`${ obj.method }\` ${ obj.url } \n\n#### 参数\n`;
  mdStr[obj.group] += '\n参数名 | 类型 | 是否必填 | 说明\n-----|-----|-----|-----\n';
  Object.keys(obj.params).forEach(function (param) {
    const paramVal = obj.params[param];
    fields[param] = paramVal['value'];
    mdStr[obj.group] += `${ param } | ${ paramVal['type'] } | ${ paramVal['required'] ? '是' : '否' } | ${ paramVal['desc'] } \n`;
  });
  mdStr[obj.group] += '\n#### 使用示例\n\n请求参数: \n\n';
      
  mdStr[obj.group] += '```json\n' + JSON.stringify(fields, null, 2) + '\n```\n';
  mdStr[obj.group] += '\n返回结果:\n\n';
       
  if (obj.url.indexOf(':') > -1) {
    obj.url = obj.url.replace(/:\w*/g, function (word) {
      return fields[word.substr(1)];
    });
  }
        
  obj.agent[obj.method](obj.url)
  .set(obj.header || {})
  .query(fields)
  .send(fields)
  .expect(obj.expect)
  .end(function (err, res) {
    mdStr[obj.group] += '```json\n' + JSON.stringify(res.body, null, 2) + '\n```\n';
    mdStr[obj.group] += '\n';
       
    if (process.env['GEN_DOC'] > 0) {
      fs.writeFileSync(path.resolve(__dirname, './docs/', obj.file + '.md'), mdStr[obj.group]);
    }
    obj.callback(err, res);
  });
}
```
这样，在根目录创建一个 docs 目录，运行 `npm run test:doc` 命令，就会在 docs 目录下生成文档。如果运行单元测试不想生成文档，直接用`npm test`就可以了，相应的package.json配置如下：
```json
"scripts": {
  "test": "export NODE_ENV='test' && mocha",
  "test:doc": "export NODE_ENV='test' && export GEN_DOC=1 && mocha"
}
```
如果不想为某个 API 生成文档，就不要调用 utils 的 test，直接按原生的写法就可以了。

若需要对参数进行签名，可在调用 test 方法时，增加形如`sign: true`的配置，然后在 test 方法中做相应的判断和实现相应的签名。
生成的文档内容形式如下：
![test_doc](http://nnblog-storage.b0.upaiyun.com/img/test_doc.jpg!watermark1.0)
