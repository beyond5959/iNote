title: 利用redis将log4js产生的日志送到logstash
date: 2016-08-23 16:43:51
tags: [redis, logstash]
---
![log4js-logstash-redis](http://nnblog-storage.b0.upaiyun.com/img/log4js-logstash-redis.png!watermark1.0)

在用Node.js开发项目的时候，我们常用 [log4js](https://www.npmjs.com/package/log4js) 模块来进行日志的记录，可以通过配置 log4js 的 [Appenders](https://github.com/nomiddlename/log4js-node/wiki/Appenders) 将日志输出到Console、File和GELF等不同的地方。
<!-- more -->
## logstash
logstash 是 [elastic](https://www.elastic.co/) 技术栈的其中一员，常被用来收集和解析日志。一种比较常见的做法就是在项目工程中把日志写入到日志文件中，然后用 logstash 读取和解析日志文件。比如在 Node.js 项目中，若要将日志记录到文件中，只需对 log4js 做如下配置即可：
```javascript
const log4js = require('log4js');
log4js.configure({
  appenders: [{
    type: 'file',
    filename: './example.log'
  }]
});
        
const logger = log4js.getLogger();
logger.info('hello');
```
这样日志就会被记录到 `example.log` 文件中，然后再对 logstash 的 input 做如下配置：
```
file {
    path => "YourLogPath/example.log"
    start_position => "beginning"
  }
}
```
这样 logstash 就可以读取这个日志文件。可是这样总感觉用一个文件作为中转略显麻烦，如果能将 log4js 产生的日志直接送到 logstash 就更好了。
## logstashUDP
log4js 内置了 logstashUDP 来直接将日志输出到 logstash。配置如下：
```javascript
log4js.configure({
  appenders: [{
    type: "logstashUDP",
    host: "localhost",
    port: 12345
	}]
});
```
然后将 logstash 的配置改成下面这样：
```
input {
	udp {
    host => "127.0.0.1"
		port => 12345
	}
}
```
嗯，很简单嘛！现在 log4js 产生的日志就能直接送到 logstash 了，而不再需要用一个文件作为中转。但是，当我在使用的时候发现一个问题，就是如果 logstash 服务挂了，这时候 log4js 仍然在不断的产生日志数据，这时候首先想到的就是把 logstash 重新启动起来，但重启后却发现 logstash 没能获取到在 logstash 挂掉的时候 log4js 产生的数据，也就是说如果 logstash 挂掉了的话，那么 log4js 产生的数据就会丢失，不会被处理。<br />
这时候就想着用一个代理来暂存 log4js 产生的数据，log4js 将数据输出到代理，logstash 从代理那里读取数据，logstash 读取一条数据，代理就丢弃掉那条数据。对，也就是队列。这样就不会有数据丢失的问题了。
## log4js-logstash-redis
[log4js-logstash-redis](https://www.npmjs.com/package/log4js-logstash-redis) 就是为了解决上述问题而开发的一个 log4js 的 Appender。<br />
他的原理是：log4js 将产生的日志输出到 redis，然后让 logstash 从 redis 读取数据。但是直接叫 log4js-redis 就好了嘛，为什么叫 log4js-logstash-redis 呢？这是因为将日志格式针对 logstash 做过一些更友好的定制。😊 <br />
### 安装
```
npm install log4js-logstash-redis --save
```
### 使用
```javascript
log4js.configure({
  appenders: [{
    type: "log4js-logstash-redis",
    key: "test",
    redis: {
      db: 1
    }
	}]
});
```
redis 配置可选，没有的话 redis 连接就采用默认值。logstash 的配置如下：
```
redis {
		data_type => "list"
		key => "test"
		codec => json
}
```
其中 data_type 的值一定要是 list。
