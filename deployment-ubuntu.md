# 在阿里云上部署（Ubuntu）

### 环境介绍
阿里云 ECS Ubuntu 16.04 64 [直达链接](https://promotion.aliyun.com/ntms/act/ambassador/sharetouser.html?userCode=3grpysgf&productCode=vm&utm_source=3grpysgf)

### 更新系统和安装 git、vim、curl
```
apt update -y
apt upgrade -y
apt install curl git -y
```

### 通过 nvm 安装 Node.js
+ 安装 nvm
```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.5/install.sh | bash
```
验证安装是否成功
```
source ~/.bashrc
nvm --version
```
看到输出版本信息 0.33.5 表示安装成功

+ 查看最新 8.x 版本 Node.js 版本并安装
```
nvm ls-remote
nvm install v8.2.1
node -v
```
看到输出版本信息 v8.2.1 表示安装成功
>**必须安装 Node.js 8.x 以上版本**

### 安装 MySQL 5.7
```
apt  install mysql-server -y
```
安装过程会要求设置 root 用户的密码，并记住密码

验证 mysql 是否安装成功
```
mysql -uroot -p 
```
回车后输入安装时输入的密码，登录成功后的样子
![登录成功后](http://upload-images.jianshu.io/upload_images/3985656-32da7cea7c096022.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 开始运行 NideShop
+ 下载 NideShop 的源码
```
mkdir /var/www
cd /var/www
git clone https://github.com/tumobi/nideshop
```
+ 全局安装 ThinkJS 命令
```
npm install -g think-cli
thinkjs -v
```

+ 安装依赖
```
cd /var/www/nideshop
npm install 
```

+ 创建数据库并导入数据
```
 mysql -uroot -p -e "create database nideshop character set utf8mb4"
 mysql -uroot -p nideshop < /var/www/nideshop/nideshop.sql
```

+ 修改 Nideshop 的数据库配置
```
vim src/common/config/adapter.js
```
修改后

```
 24 /**
 25  * model adapter config
 26  * @type {Object}
 27  */
 28 exports.model = {
 29   type: 'mysql',
 30   common: {
 31     logConnect: isDev,
 32     logSql: isDev,
 33     logger: msg => think.logger.info(msg)
 34   },
 35   mysql: {
 36     handle: mysql,
 37     database: 'nideshop',
 38     prefix: 'nideshop_',
 39     encoding: 'utf8mb4',
 40     host: '127.0.0.1',
 41     port: '3306',
 42     user: 'root',
 43     password: '你的密码',
 44     dateStrings: true
 45   }
 46 };
```

>注意 encoding，prefix 的值

编译项目
```
npm run compile
```

以生产模式启动
```
node production.js
```

打开另一个终端验证是否启动成功
```
curl -I http://127.0.0.1:8360/
```

输出 HTTP/1.1 200 OK，则表示成功
** Ctrl + C 停止运行**
> 为防止后面操作出现[Error] Error: Address already in use, port:8360. 的错误，一定要记得Ctrl + C停止运行，并确保curl -I http://127.0.0.1:8360/ 不能访问

### 使用 PM2 管理服务

+ 安装配置 pm2
```
npm install -g pm2
```

修改项目根目录下的 pm2.json 为：
```
vim pm2.json
```

修改后的内容如下 ：
```
{
  "apps": [{
    "name": "nideshop",
    "script": "production.js",
    "cwd": "/var/www/nideshop",
    "exec_mode": "fork",
    "max_memory_restart": "256M",
    "autorestart": true,
    "node_args": [],
    "args": [],
    "env": {

    }
  }]
}
```

如果服务器配置较高，可适当调整 max_memory_restart 和instances的值
+ 启动pm2
```
pm2 start pm2.json
```

成功启动

![image.png](http://upload-images.jianshu.io/upload_images/3985656-21a6aa802f7bb1ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再次验证是否可以访问
```
curl -I http://127.0.0.1:8360/
```

### 使用 nginx 做反向代理
```
apt install nginx -y
```

测试本地是否可以正常访问
```
curl -I localhost 
```

修改nginx配置
```
cp  /etc/nginx/sites-available/default  /etc/nginx/sites-available/default.bak
vim /etc/nginx/sites-available/default
```

修改后的内容

```
server {
    listen 80;
    server_name nideshop.com www.nideshop.com; # 改成你自己的域名
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
```

+ 重新启动nginx并验证nginx是否还可以正常访问
```
nginx -t 
service nginx restart
curl  http://127.0.0.1/
```

如果返回的是下图的json数据则表示nginx反向代理配置成功

![nginx转发成功](http://upload-images.jianshu.io/upload_images/3985656-ff191d58e075c41c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 注：阿里云默认外网不能访问80/443端口，请更改实例的安全组配置，配置教程：https://help.aliyun.com/document_detail/25475.html?spm=5176.doc25475.3.3.ZAx4Uo

### 配置https访问
+ 安装certbot
```
apt install software-properties-common
add-apt-repository ppa:certbot/certbot
apt update -y
apt install python-certbot-nginx  -y
certbot --nginx
```

+ 配置自动更新证书
```
certbot renew --dry-run
```

> 详细文档请查看：https://certbot.eff.org/#ubuntuxenial-nginx

+ 测试浏览器使用https形式访问是否成功

![配置https访问成功](http://upload-images.jianshu.io/upload_images/3985656-9b0cfb1db7c99c3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 修改NideShop微信小程序客户端的配置
微信小程序商城客户端GitHub:  https://github.com/tumobi/nideshop-mini-program 

打开文件 config/api.js，修改 NewApiRootUrl 为自己的域名
```
var NewApiRootUrl = 'https://www.nideshop.com/api/';
```
> 注意 https 和后面的 api/ 不能少

到此部署成功。如有问题请加QQ群：497145766