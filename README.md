# Nginx，Charles与Webpack配置前端API代理教程(超详细)

### 为什么前端需要配置API代理？
我们在开发一个项目的时候，如果服务采用的是分布式部署，也就是说按不同模块或功能部署于不同的服务器，如下图

![](https://user-gold-cdn.xitu.io/2018/10/2/16632a7aa3bbb53b?w=1116&h=870&f=png&s=89566)
客户端不同的请求会被主服务器转发到对应的服务器上去，如果在开发阶段，也有一个这样做转发的服务器，那么前端的开发是不需要配置代理的，我们今天要探讨的是开发阶段无转发服务器，需要前端配置代理的情况，如下图

![](https://user-gold-cdn.xitu.io/2018/10/2/16632b5f9cd44e40?w=1140&h=886&f=png&s=93350)
大家都知道，浏览器是存在跨域限制的，处于安全性考虑，服务器ABC不可能设置为允许任何请求都可以访问，So, **我们配置前端API的代理的目的其实就是为了解决跨域问题**，前端按照既定的规则配置好代理之后，就能保证开发阶段和线上部署服务的一致。

### 配置前端API代理的三种方式
本文以配置Dva项目的代理为示例，因为Dva项目的脚手架自带Mock功能，省去了自己写接口的麻烦，同时公司内部项目也采用此技术，能让团队人员也做一参考。

示例地址：https://github.com/ranshaw/frontEndProxy.git

 拉取代码，安装依赖之后，访问http://localhost:8000（默认是8000端口，如果被占用，会在其他端口），就能看到Welcome to dva的欢迎页面。项目中.roadhogrc.mock.js为Mock数据的配置文件，现为
 
![](https://user-gold-cdn.xitu.io/2018/10/2/16632fda1c8e3366?w=902&h=346&f=png&s=40371)
在浏览器中输入 http://localhost:8000/users/1 就能看到返回的是你在.roadhogrc.mock.js中配置的数据

![](https://user-gold-cdn.xitu.io/2018/10/2/16633a89d8b11a79?w=820&h=518&f=png&s=44953)
我们最终要达成的目的:
1. 使用www.frontproxy.com访问开发环境，即在浏览器输入www.frontproxy.com等同于输入localhost:8000。
2. 将users、todos、posts模块也就是请求路径中以/users/、/todos/、/posts/开始的请求接口，都代理到不同的服务器。
3. 最终实现请求如 www.frontproxy.com/users/1 返回的数据为请求
http://jsonplaceholder.typicode.com/users/1 
返回的数据

注：本文中将三个模块的请求都代理到 http://jsonplaceholder.typicode.com 上；为了测试方便都采用Get请求，其他请求方式同Get方式；下文中讲述的配置方法以Mac为例，Windows上原理一致，具体方法需自行google。

### 配置Nginx和Hosts实现

##### 配置Hosts
在mac终端输入 ```sudo vi /etc/hosts```，对Hosts文件进行编辑，添加如下配置
```127.0.0.1   www.frontendproxy.com```
![](https://user-gold-cdn.xitu.io/2018/10/2/16633f58120b3add?w=726&h=146&f=png&s=41963)
然后保存。如果对Vim命令不了解，点击[Vim命令详解](https://www.cnblogs.com/usergaojie/p/4583796.html)学习。

现在我们访问www.frontproxy.com:8000和访问localhost:8000的效果是一样的，
![](https://user-gold-cdn.xitu.io/2018/10/2/16633e168d0ca000?w=1000&h=436&f=png&s=69840)
因为Hosts配置的映射关系，不支持自定义端口，所以现在访问的时候还必须要加上端口，下面我们会通过Nginx配置，将url中端口去掉。
##### 配置Nginx
安装好Nginx之后，在Mac终端输入 
```cd /usr/local/etc/nginx                            ```
找到nginx.conf文件，可使用软件打开或者继续输入``` vi nginx.conf``` 添加如下配置
```
server {
        listen 80;
        server_name www.frontendproxy.com;
        index index.html;
        location / {
            proxy_pass http://127.0.0.1:8000/;
        }
    }
```
在终端输入 ```sudo nginx ```，启动Nginx,此时在浏览器输入 http://www.frontendproxy.com 就可以正常显示了，但是当你修改页面的内容你会发现，http://localhost:8000 可以自动刷新，更新你刚才修改的内容，http://www.frontendproxy.com 页面并没有自动刷新，手动刷新后修改的内容才显示出来，并且页面中有一个报错

![](https://user-gold-cdn.xitu.io/2018/10/2/16634048ab5c8b21?w=1368&h=126&f=png&s=51310)
webSocket通信出了问题，我们需要添加如下配置
```
location /sockjs-node/ {
    proxy_pass http://127.0.0.1:8000/sockjs-node/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";	
 }
```
在终端输入```sudo nginx -s reload ``` 重启nginx,然后发现报错消失，修改页面内容后，可以自动刷新页面了。

到这一步，我们可以使用域名访问本地项目，进行愉快的开发了，但是请求users模块的数据仍然是Mock中的数据，

![](https://user-gold-cdn.xitu.io/2018/10/2/166341285691ccd8?w=898&h=504&f=png&s=51916)

我们在Nginx中添加如下配置，对users模块的请求进行代理，
```
location /users/ {
    proxy_pass http://jsonplaceholder.typicode.com/users/;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```
在终端输入```sudo nginx -s reload ```重启Nginx，再次请求 http://www.frontendproxy.com/users/1 你会发现返回的数据已经不是Mock配置的数据了，而是 http://jsonplaceholder.typicode.com/users/1 返回的数据，说明代理已经生效

![](https://user-gold-cdn.xitu.io/2018/10/2/166341fc14249e36?w=1188&h=1152&f=png&s=190357)

同理对todos、posts模块进行配置，完整配置如下
```
server {
    listen 80;
    server_name www.frontendproxy.com;
    index index.html;
    location / {
        proxy_pass http://127.0.0.1:8000/;
    }
    location /sockjs-node/ {
        proxy_pass http://127.0.0.1:8000/sockjs-node/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";	
     }
    location /users/ {
        proxy_pass http://jsonplaceholder.typicode.com/users/;
        proxy_set_header X-Real-IP $remote_addr;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    location /todos/ {
        proxy_pass http://jsonplaceholder.typicode.com/todos/;
        proxy_set_header X-Real-IP $remote_addr;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    location /posts/ {
        proxy_pass http://jsonplaceholder.typicode.com/posts/;
        proxy_set_header X-Real-IP $remote_addr;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
 
OK,我们现在已经完成了对users、todos、posts模块请求接口的代理，不同模块对应的服务器的地址，都可以自定义设置。

### Charles配置代理
Charles为Mac上一个超级好用的抓包软件，特别是对于移动端开发接口的调试，简直不要太爽，在此不做延伸，后续会专门写文章介绍,下载地址为：
链接: https://pan.baidu.com/s/1UCJu9KaB3hgRXNG-UqYnDA 提取码: shfp。

如果你按照上文教程配置了Hosts和Nginx，请把相关配置都清除掉，然后我们开始Charles的配置。

Charles并不能直接对Chrome进行抓包，需要对Chrome设置代理，
我这使用的是Chrome的一个插件SwitchyOmega，在SwitchOmega中配置代理，8888为Charles默认的端口，如果你更改过，请填写你更改后的端口

![](https://user-gold-cdn.xitu.io/2018/10/2/166343b6f4e81978?w=2266&h=662&f=png&s=127653)
配置生效以后查看Chrome浏览器中代理的设置，

![](https://user-gold-cdn.xitu.io/2018/10/2/1663441390b8ed9f?w=1452&h=230&f=png&s=31378)
此时，通过Chrome浏览器发出的请求，就会被Charles抓住

![](https://user-gold-cdn.xitu.io/2018/10/2/1663443854a09c82?w=910&h=180&f=png&s=25804)
现在我们开始配置Charles的代理规则，如下图打开Charles中tools中的Map Remote

![](https://user-gold-cdn.xitu.io/2018/10/2/1663501dc84408e3?w=930&h=262&f=png&s=70924)

点击add按钮进行添加

![](https://user-gold-cdn.xitu.io/2018/10/2/16635030ece24a28?w=834&h=750&f=png&s=51980)
如图将 www.frontendproxy.com 代理到 127.0.0.1：8000上，配置好之后，访问域名和访问IP效果是一样的，如果上文配置好Hosts和Nginx之后的效果，同样需要配置/sockjs-node/的代理，这样更改文件后可以自动保存了，下面添加users模块的代理

![](https://user-gold-cdn.xitu.io/2018/10/2/166350a2f2cb2d4b?w=804&h=806&f=png&s=64798)
Charles代理规则的优先级是从上到下的，也就是说上面的规则会覆盖下面的规则，这点需要注意下

![](https://user-gold-cdn.xitu.io/2018/10/2/166350eec785b407?w=1036&h=126&f=png&s=32704)
OK,到这里Charles代理前端API请求的配置就已经完成了。
### Wepack + Host Switch Plus配置
现在前端开发环境中基本上都有用到webpack,其中webpack-dev-server这个包也是使用webpack搭建开发环境所必备的，我们常用的自动刷新和热更新功能就是由它提供的，今天所要讲的proxy功能也是如此。

Host Switch Plus，顾名思义就是对host的管理工具，它是一个Chrome的插件，需要你在Chrome应用商店下载。

首先我们先配置对域名的代理，支持单个添加和批量添加

![](https://user-gold-cdn.xitu.io/2018/10/2/16635302eccab4c1?w=1362&h=668&f=png&s=81685)
添加完成之后，我们点击插件的图标就能看到你刚才配置的信息，Enable为勾选，IP前面的圆点为绿色表示已生效

![](https://user-gold-cdn.xitu.io/2018/10/2/16635311e9b53f65?w=1000&h=398&f=png&s=58881)
此时，访问 http://www.frontendproxy.com已经代理到 http://localhost:8000 上，我们再打开项目中的 .webpackrc 文件，添加如下配置
```
{
  "proxy": {
  "/sockjs-node/": {
    "target": "http://127.0.0.1:8000/",
    "changeOrigin": true
  },
  "/users/": {
    "target": "http://jsonplaceholder.typicode.com/",
    "changeOrigin": true
  },
  "/todos/": {
    "target": "http://jsonplaceholder.typicode.com/",
    "changeOrigin": true
  }
}
}
```
好了，关于webpack配置代理的操作，到这里也完成了，关于[webpack-dev-server](https://webpack.docschina.org/configuration/dev-server/)中proxy配置的详细规则，请点击查看，我这里不再一一讲述。

**注意：** 如果你现在使用的是Nginx配置的代理，现在想转为webpack配置代理，假如要代理的模块为```/users/```,期望代理的接口地址为```http://localhost:8000/login```,Nginx和Webpack的配置分别如下，最终代理服务器发出的请求为```http://jsonplaceholder.typicode.com/login ```,Webpack的配置项要多加一项配置```pathRewrite```，对代理服务器的路径进行重写，将请求中的```/users/ ```替换为```/```,有时候后端会需要请求头中的某些值，这时候就需要添加自定义请求头（版本webpack-dev-server@3.1.7），如下：
```
 Nginx配置：
 location /users/ {
        proxy_pass http://jsonplaceholder.typicode.com/;
        proxy_set_header X-Real-IP $remote_addr;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
 Webpack配置：
 {
  "proxy": {
  "/users/": {
    "target": "http://jsonplaceholder.typicode.com/",
    "changeOrigin": true，
    "pathRewrite": { "^/users/": "/" }，
    "headers":{
       // 你需要定义的请求头
       Host: "localhost:8000"
    }
  },
}
}
```

### 总结
下面是划重点时间，我们从适用范围、配置难易、团队协作成本三个方面对比一下这三种配置方式

![](https://user-gold-cdn.xitu.io/2018/10/2/166355df15611831?w=1108&h=454&f=png&s=121742)
 
 国庆节的一天就这么过去了，觉得有帮助的童鞋，帮忙点个👍哟！
 
