# 在新浪云上部署

### 一、购买新浪云SAE
+ 为什么选择SAE？免费二级域名和支持https访问，不用备案，可用做微信小程序服务器。
+ SAE推荐链接：[http://sae.sina.com.cn/](http://t.cn/RqUxRgi)
+ 选择对应的部署环境
  + 自定义
  + 开发言语：自定义
  + 运行环境：云容器
  + 语言版本：自定义
  + 部署方式：手工部署
  + 环境配置：选择第一项
  + 实例个数：1（测试用选择1个即可）
  + 二级域名：填写你的域名（这里为：nideshop.applinzi.com）
  + 应用名称：填写你的名称（nideshop）

>文中出现nideshop.applinzi.com的地方，请替换为你配置的二级域名

![选择部署环境](http://upload-images.jianshu.io/upload_images/3985656-634a56e06b3b77f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 二、通过SSH连接云容器
windows下的配置教程：http://www.sinacloud.com/home/index/faq_detail/doc_id/173.html

### 三、安装配置nginx
```
apt update -y
apt upgrade -y
apt install nginx curl vim -y
service nginx start 
curl localhost
```
此时发现在外网并不能访问 https://nideshop.applinzi.com/ ，错误返回
502 Bad Gateway
这个错误官方文档有说明： http://www.sinacloud.com/doc/sae/docker/vm-getting-started.html

解决方法：更改nginx默认监听的端口80为5050，并重新启动nginx
```
vim /etc/nginx/sites-available/default
nginx -t
service nginx restart
```
![此处输入图片的描述](http://upload-images.jianshu.io/upload_images/3985656-fde98309d0b01249.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再次访问 https://nideshop.applinzi.com/ ，成功返回
Welcome to nginx!


### 四、通过nvm安装node.js
+ 安装 nvm
 https://github.com/creationix/nvm 
```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.4/install.sh | bash
```
+ nvm安装成功后，执行以下命令
```
source ~/.bashrc  
```

+ 查看最新版本的Node.js并安装
```
nvm ls-remote
NVM_NODEJS_ORG_MIRROR=https://npm.taobao.org/mirrors/node nvm install v8.1.4
node -v
```

### 五、配置共享型MySQL并导入数据
创建MySQL成功后，选择管理操作，进入到 phpmyadmin 页面，选项导入
选择nideshop项目根目录下的 nideshop.sql 文件

### 六、本地部署NideShop
+ 下载NideShop的源码
```
apt install git -y
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
> Node.js连接MySQL参考文档： http://www.sinacloud.com/doc/sae/docker/howto-use-mysql.html#nodejs

+ 启动：
```
npm start
curl localhost:8360
```
***测试成功后记得： Ctrl + C 停止运行***

+ 如果你遇到npm start 失败，错误信息如下：
![image.png](http://upload-images.jianshu.io/upload_images/3985656-d47d276f61dcda10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可参考如下解决方法：
thinkjs 3默认使用当前 cpu 的个数来启用子进程的数量，所以修改workers数量为1
  ```
    vim vim src/common/config/config.js
  ```
修改后：

![image.png](http://upload-images.jianshu.io/upload_images/3985656-093c18878ebc7306.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**如果还是未解决，可以尝试选择环境配置最高的**

![image.png](http://upload-images.jianshu.io/upload_images/3985656-6b476ea89957feba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 七 通过nginx、pm2进行线上部署
+ 编译项目
``` 
npm run compile
```
 
+ 修改nginx配置
 /etc/nginx/sites-available/default 修改后

```
server {
    listen 5050;
    server_name nideshop.applinzi.com;
    root /var/www/nideship/www;
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

```

+ 测试通过nginx访问
启动服务

```
node www/production.js
```

外网通过浏览器访问: https://nideshop.applinzi.com/
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
    "max_memory_restart": "256M",
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

+ 参考文档：ThinkJS线上部署文档： https://thinkjs.org/zh-cn/doc/3.0/deploy.html


### 八 修改NideShop微信小程序的配置
微信小程序商城GitHub: https://github.com/tumobi/nideshop-mini-program 

打开文件config/api.js，修改 NewApiRootUrl 为自己的域名，注意是 https 和后面的 api/ 不能少

```
var NewApiRootUrl = 'https://nideshop.applinzi.com/api/';
```

到此部署成功。如有问题请加QQ群：497145766