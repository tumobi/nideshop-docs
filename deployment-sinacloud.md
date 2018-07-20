# 在新浪云上部署（CentOS）

不用买域名、不用备案、不用配置https快速部署 NideShop 微信小程序商城

### 一、购买新浪云SAE
+ 为什么选择SAE？免费二级域名和支持https访问，不用备案，可用做微信小程序服务器。
+ SAE推荐链接：[http://sae.sina.com.cn/](http://t.cn/RqUxRgi)
+ 选择对应的部署环境
  + 自定义
  + 开发言语：自定义
  + 运行环境：云容器
  + 语言版本：自定义
  + 部署方式：手工部署
  + 操作系统：centos 7.5 1804
  + 环境配置：高级 II
  + 实例个数：1（测试用选择1个即可）
  + 二级域名：填写你的域名（这里为：nideshop2.applinzi.com）
  + 应用名称：填写你的名称（nideshop2）

**文中出现 nideshop2.applinzi.com 的地方，请替换为你配置的二级域名**

![创建云服务器配置选项.png](https://upload-images.jianshu.io/upload_images/3985656-4c2bd838b8a50eb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 二、通过SSH连接云容器
windows下的配置教程：http://www.sinacloud.com/home/index/faq_detail/doc_id/173.html

### 三、安装配置nginx
```
yum update -y
yum install -y epel-release
yum install -y nginx curl vim net-tools git
nginx
curl -I localhost
```
此时发现在外网并不能访问 https://nideshop2.applinzi.com/，错误返回
502 Bad Gateway
这个错误官方文档有说明：http://www.sinacloud.com/doc/sae/docker/vm-getting-started.html

解决方法：更改nginx默认监听的端口80为5050，并重新启动nginx
```
vim /etc/nginx/nginx.conf
nginx -t
nginx -s reload
```
更改监听的端口后

![更改nginx监听的端口.png](https://upload-images.jianshu.io/upload_images/3985656-353019b86a495019.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再次访问 https://nideshop2.applinzi.com/，成功返回
`Welcome to nginx on Fedora!`

### 四、通过nvm安装node.js
+ 安装nvm
 https://github.com/creationix/nvm
```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash
```

+ nvm安装成功后，执行以下命令
```
source ~/.bashrc
```

+ 查看最新版本的Node.js并安装
```
nvm ls-remote
NVM_NODEJS_ORG_MIRROR=https://npm.taobao.org/mirrors/node nvm install v8.11.3
nvm use --delete-prefix v8.11.3
node -v
```

### 五、配置共享型MySQL并导入数据
创建MySQL成功后，选择管理操作，进入到phpmyadmin页面，选项导入
选择nideshop项目根目录下的nideshop.sql文件

### 六、本地部署NideShop
+ 下载NideShop的源码
```
cd /var/www
git clone https://github.com/tumobi/nideshop
```
+ 安装ThinkJS
```
npm install  think-cli -g --registry=https://registry.npm.taobao.org --verbose
thinkjs --version
```
+ 安装依赖
```
cd /var/www/nideshop
npm install --registry=https://registry.npm.taobao.org --verbose
```

+ 配置mysql
```
vim src/common/config/database.js
```
修改后：
```
const mysql = require('think-model-mysql');
module.exports = {
  handle: mysql,
  database:  'app_' + process.env.APPNAME,
  prefix: 'nideshop_',
  encoding: 'utf8mb4',
  host: process.env.MYSQL_HOST,
  port: process.env.MYSQL_PORT,
  user: process.env.ACCESSKEY,
  password: process.env.SECRETKEY,
  dateStrings: true
};
```
> Node.js连接MySQL参考文档：http://www.sinacloud.com/doc/sae/docker/howto-use-mysql.html#nodejs

### 七 通过nginx、pm2进行线上部署
+ 编译项目
``` 
npm run compile
```
 
+ 修改nginx配置
/etc/nginx/nginx.conf 修改后

```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       5050 default_server;
        root /var/www/nideshop/www;
        set $node_port 8360;

        index index.js index.html index.htm;
        if ( -f $request_filename/index.html ){
            rewrite (.*) $1/index.html break;
        }
        if ( !-f $request_filename ){
            rewrite (.*) /index.js;
        }

        location = /index.js {
            proxy_http_version 1.1;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-NginX-Proxy true;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_pass http://127.0.0.1:$node_port$request_uri;
            proxy_redirect off;
        }

        location ~ /static/ {
            etag         on;
            expires      max;
        }
    }
}

```
重新启动 nginx
```
nginx -s reload
```

+ 测试通过nginx访问
启动服务
```
node production.js
```
外网通过浏览器访问: https://nideshop2.applinzi.com/
***测试成功后记得： Ctrl + C 停止运行***

+ 安装配置pm2
```
npm install -g pm2
```
修改项目根目录下的pm2.json为：
```
{
  "apps": [{
    "name": "nideshop",
    "script": "production.js",
    "cwd": "/var/www/nideshop",
    "exec_mode": "fork",
    "max_memory_restart": "1G",
    "autorestart": true,
    "instances": 1,
    "node_args": [],
    "args": [],
    "env": {

    }
  }]
}
```
启动pm2
```
pm2 startOrReload pm2.json
```

+ 参考文档：ThinkJS线上部署文档：https://thinkjs.org/zh-cn/doc/3.0/deploy.html


### 八 修改NideShop微信小程序的配置
微信小程序商城GitHub: https://github.com/tumobi/nideshop-mini-program
打开文件config/api.js，修改NewApiRootUrl为自己的域名，注意 https 和后面的 api/ 不能少
```
var NewApiRootUrl = 'https://nideshop2.applinzi.com/api/';
```

### 九 微信小程序端运行效果图
![首页](http://upload-images.jianshu.io/upload_images/3985656-c543b937ac6e79bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

![专题](http://upload-images.jianshu.io/upload_images/3985656-bd606aac3b5491c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

![分类](http://upload-images.jianshu.io/upload_images/3985656-fa9565158376d439.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

![商品列表](http://upload-images.jianshu.io/upload_images/3985656-788b7fd2c4a558d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

![商品详情](http://upload-images.jianshu.io/upload_images/3985656-99a6e0a57778d85f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

![购物车](http://upload-images.jianshu.io/upload_images/3985656-60ff2307d81f6bb2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

![订单中心](http://upload-images.jianshu.io/upload_images/3985656-dff837e6b2ec87b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

> 如使用的是阿里云服务器，请参考另一篇文章：[CentOS 7.3 下部署基于 Node.js的微信小程序商城](https://www.jianshu.com/p/5d5497697b0a)