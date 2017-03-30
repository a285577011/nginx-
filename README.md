nginx负载均衡配置
144  作者 LittleOne丶 关注
2016.05.21 12:12* 字数 1094 阅读 731评论 0喜欢 10
负载均衡，单从字面上的意思来理解就可以解释N台服务器平均分担负载，不会因为某台服务器负载高宕机和某台服务器闲置的情况。那么负载均衡的前提就是要2台以上服务器才能实现。

由于没有服务器，所以本次测试直接host指定域名，服务器不够，我们用nodejs监听了三个端口(8881,8882,8888)来模拟多台服务器。nginx监听80端口作为主服务器。

var http = require('http');

http.createServer(function (request, response) {

    response.writeHead(200, {'Content-Type': 'text/plain'});
    response.end('server 8881');

}).listen(8881);

// 终端打印如下信息
console.log('Server running at http://127.0.0.1:8881/');
var http = require('http');

http.createServer(function (request, response) {

    response.writeHead(200, {'Content-Type': 'text/plain'});
    response.end('server 8882');

}).listen(8882);

// 终端打印如下信息
console.log('Server running at http://127.0.0.1:8882');
var http = require('http');

http.createServer(function (request, response) {

    response.writeHead(200, {'Content-Type': 'text/plain'});
    response.end('server 8888');

}).listen(8888);

// 终端打印如下信息
console.log('Server running at http://127.0.0.1:8888/');
测试域名yongle.com
A服务器监听80端口（主）
B服务器监听8881端口（从）
C服务器监听8882端口（从）
D服务器监听8888端口（从）

A服务器做为主服务器，域名直接解析到A服务器（ 127.0.0.1:80）上，由A服务器负载均衡到B服务器（ 127.0.0.1:8881）、C服务器（ 127.0.0.1:8882）和D服务器（ 127.0.0.1:8888）上。

A服务器nginx.conf设置打开nginx.conf，文件位置在nginx安装目录的conf目录下。在http段加入以下代码

upstream yongle.com {
      server 127.0.0.1:8881;
      server 127.0.0.1:8882;
      server 127.0.0.1:8888;
}

server{ 
    listen 80; 
    server_name yongle.com; 
    location / { 
        proxy_pass         http://yongle.com; 
        proxy_set_header   Host             $host; 
        proxy_set_header   X-Real-IP        $remote_addr; 
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for; 
    } 
}
保存重启nginx即可完成负载均衡

我们把域名解析到A服务器，然后由A服务器转发到B服务器C服务器与D服务器，那么A服务器只做一个转发功能，现在我们让A服务器也提供站点服务。
如果添加主服务器到upstream中，那么可能会有以下两种情况发生：
1、主服务器转发到了其它IP上，其它IP服务器正常处理；
2、主服务器转发到了自己IP上，然后又进到主服务器分配IP那里，假如一直分配到本机，则会造成一个死循环。
怎么解决这个问题呢？因为80端口已经用来监听负载均衡的处理，那么本服务器上就不能再使用80端口来处理yongle.com的访问请求，必须重新监听一个新的端口。于是我们把主服务器的nginx.conf加入以下一段代码：

server {
        listen       8000;
        server_name  yongle.com;
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
把主服务器添加到upstream中

upstream yongle.com {
        server 127.0.0.1:8881;
        server 127.0.0.1:8882;
        server 127.0.0.1:8888;
        server 127.0.0.1:8000;
    }
到这里我们就完成了把主服务器也加入到了负载均衡中。

Nginx负载均衡有4种方案配置
1、轮询
轮询即Round Robin，根据Nginx配置文件中的顺序，依次把客户端的Web请求分发到不同的后端服务器上
2、最少连接 least_conn;
Web请求会被转发到连接数最少的服务器上。
3、IP地址哈希 ip_hash;
前述的两种负载均衡方案中，同一客户端连续的Web请求可能会被分发到不同的后端服务器进行处理，因此如果涉及到会话Session，那么会话会比较复杂。常见的是基于数据库的会话持久化。要克服上面的难题，可以使用基于IP地址哈希的负载均衡方案。这样的话，同一客户端连续的Web请求都会被分发到同一服务器进行处理。
4、基于权重 weight
基于权重的负载均衡即Weighted Load Balancing，这种方式下，我们可以配置Nginx把请求更多地分发到高配置的后端服务器上，把相对较少的请求分发到低配服务器。
