title: 用ELK处理日志时碰到的几个问题
date: 2016-08-16 11:50:55
tags: [日志, ELK, elasticsearch, logstash, kibana, 大数据]
---
![elk](http://nnblog-storage.b0.upaiyun.com/img/elk.png)
常说的ELK是指包括 elasticsearch、logstash 和 kibana 的一套技术栈，常用来处理日志等数据。最近在用这一套 stack 处理 nginx 的日志，而且还用到了 elastic 的另一个产品 beats，来作为客户端 shipper，形成了ELKB Stack。
<!-- more -->
<br />
在使用这一套技术栈的时候碰到了一些其实并不高深的小问题，在这里写下来作为一个记录。

## 日志格式
常见的 nginx 的 access_log 日志都是用竖线`|`作为分隔，但我们的是用逗号`,`作为分隔，就像下面这样：
```
04/Aug/2016:15:37:09 +0800,112.124.127.44,"-","Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36",GET,/,0.003,0.003,146,199,23025,1,200,200,14,"-",TLSv1,AES256-SHA,127.0.0.1:8384
```
一开始我并没意识到这会存在什么问题，就在 logstash 的配置文件中用 filter 里的 ruby 来解析，如下：
```
ruby {
    init => "@kname = ['time_local', 'remote_addr', 'remote_user', 'http_user_agent', 'request_method', 'uri', 'request_time', 'upstream_response_time', 'request_length', 'bytes_sent', 'connection', 'connection_request', 'status', 'upstream_status', 'body_bytes_sent', 'http_referer', 'ssl_protocol', 'ssl_cipher', 'upstream_addr']"
    code => "
      new_event = LogStash::Event.new(Hash[@kname.zip(event['message'].split(','))])
      new_event.remove('@timestamp')
      event.append(new_event)
    "
    remove_field => ["message"]
  }
```
无非就是把`split('|')`换成`split(',')`嘛，很简单！但是当我在终端观察输出的时候，发现却是这样的。
![elk-1](http://nnblog-storage.b0.upaiyun.com/img/elk-1.png!watermark1.0)
User-Agent 被分割成了两个字段的内容，导致从`request_method`字段开始，字段名与其对应的值都错位了，原来在日志内容中的浏览器的User-Agent `"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36"`也包含逗号`,`导致它也被分割了，以致后面的内容都错位了，很明显这种情况不应该被分割。<br />
由于对 ruby 不是很熟悉，就没有继续在ruby filter中想办法来解决这个问题，我就继续查阅了 Logstash 其他 filter 的用法，发现用 grok filter 似乎可以解决这个问题，于是就用 grok 替换了 ruby，配置如下：
```
grok {
        match=>{"message" => "%{GREEDYDATA:time_local},%{IP:remote_addr},\"%{GREEDYDATA:remote_user}\",\"%{GREEDYDATA:http_user_agent}\",
        %{WORD:request_method},%{URIPATHPARAM:uri},%{GREEDYDATA:request_time},%{GREEDYDATA:upstream_response_time},%{NUMBER:request_length},%{NUMBER:bytes_sent},
        %{NUMBER:connection},%{NUMBER:connection_request},%{NUMBER:status},%{NUMBER:upstream_status},%{NUMBER:body_bytes_sent},
        \"%{GREEDYDATA:http_referer}\",%{GREEDYDATA:ssl_protocol},%{GREEDYDATA:ssl_cipher},%{GREEDYDATA:upstream_addr}"}
    }
```
现在再重新解析观察一节结果。
![elk-2](http://nnblog-storage.b0.upaiyun.com/img/elk-2.png!watermark1.0)
嗯，问题解决，得到了正确的解析结果。但是，总觉得这样太不优雅了，看着无比的臃肿，而且这种正则匹配效率也不是很好。后来经同事提醒，csv filter 可以很好的解决这个问题，在查阅了csv filter的用法后，将 grok 替换成如下配置：
```
csv {
		source => "message"
		columns => ["time_local", "remote_addr", "remote_user", "http_user_agent", "request_method", "uri", "request_time", "upstream_response_time", "request_length", "bytes_sent", "connection", "connection_request", "status", "upstream_status", "body_bytes_sent", "http_referer", "ssl_protocol", "ssl_cipher", "upstream_addr"]
		convert => {
			"request_time" => "float"
			"upstream_response_time" => "float"
			"connection" => "integer"
			"connection_request" => "integer"
			"bytes_sent" => "integer"
			"body_bytes_sent" => "integer"
			"request_length" => "integer"
		}
	}
```
OK，试了一下，完美解决，既优雅又正确。

## geo_point
geo_point 是 elasticsearch 中的一种数据类型，Kibana 根据类型为 geo_point 的字段生成 Geo Coordinates。但是当我点Geo Coordinates得到的确实如下图的提示信息：
![elk-3](http://nnblog-storage.b0.upaiyun.com/img/elk-3.png)
在我的nginx-test1_log（nginx-test1_log为我用logstash将ngix日志导入到 elasticsearch 中取的索引名。）索引中不能找到类型geo_point类型的字段。<br />
Logstash 中有一个geoip filter 是专门来处理将 ip 解析成地理坐标的，解析完后会得到location字段，我的配置如下：
```
geoip {
		source => remote_addr
		fields => ["city_name", "country_name", "latitude", "longitude", "real_region_name", "location"]
		remove_field => ["[geoip][latitude]", "[geoip][longitude]"]
	}
```
Kibana 通常用那个location字段来生成 Geo Coordinates，于是我就去查看 nginx-test1_log 的mapping（通过 http://localhost:9200/nginx-test1_log/_mapping 查看。）发现location的类型是double，并不是geo_point，于是就想着把location字段的类型改为geo_point，然而发现并不能或者特别麻烦，就放弃了这个想法，Google 了一圈后，发现是 elasticsearch （以下简称『es』） 的 template 造成的（可通过 http://localhost:9200/_template 查看），es 会对 logstash 导入的数据采用默认的 logstash template，这个模板就通过 mapping 指定了location的类型为geo_point，但是如果在 Logstash 的 output 中为 es 指定的index的名字不是以logstash开头话，就不会采用这个模板，相应的location的类型就不会是geo_point了，由于我这里就是以nginx开头，所以就出现了这个问题。解决办法就是自己创建一个 template，然后在输出时指定，我直接把 logstash template 复制过来然后调整了下名字，调整后的 logstash 输出配置如下：
```
output {
  elasticsearch {
      hosts => ["0.0.0.0:9200"]
      index => "nginx-%{source_tag}"
      template => "/home/liuxin/logstash/templates/nginx-template.json"
      template_name => "nginx-*"
      template_overwrite => true
  }
}
```
至此，问题就解决了。<br />
入门ELK不久，感觉ELK高效、简单、易用和生态丰富，以后肯定还会碰到很多问题，再陆陆续续补充。
