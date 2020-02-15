# Let's Encrypt免费证书相关

## 简介

### 目的

```sh
使用 let's encrypt 免费让自己网站从http升级到https
```

### Let’s Encrypt 简介

```sh
如果要启用HTTPS,我们就需要从证书授权机构(以下简称CA)处获取一个证书,Let's Encrypt就是一个CA
Let's Encrypt是国外一个公共的免费SSL项目,由Linux基金会托管,由Mozilla、思科、Akamai、IdenTrust和EFF等组织发起,目的就是向网站自动签发和管理免费证书.以便加速互联网由HTTP过渡到HTTPS,目前Facebook等大公司开始加入赞助行列
Let's Encrypt已经得了IdenTrust的交叉签名，这意味着其证书现在已经可以被Mozilla、Google、Microsoft和Apple等主流的浏览器所信任。用户只需要在Web服务器证书链中配置交叉签名，浏览器客户端会自动处理好其它的一切
我们可以从 Let's Encrypt 获得网站域名的免费的证书
```

### Certbot 简介

```sh
Certbot是Let's Encrypt官方推荐的获取证书的客户端,可以帮我们获取免费的Let's Encrypt证书
Certbot是支持所有Unix内核的操作系统的
```

## 免费获取证书

### 安装certbot客户端

```sh
$ yum provides certbot
$ yum -y install certbot
```

### 获取证书

#### 服务有证书目录

```sh
$ certbot certonly --webroot -w /var/www/example -d example.com -d www.example.com
#--webroot模式会在/var/www/example中创建.well-know文件夹，包含一些验证文件
#certbot会通过访问example.com/.well-known/acme-challenge来验证你的域名是否绑定这个服务器
```

#### 服务无证书目录

```sh
$ certbot certonly --standalone -d example.com -d www.example.com
#--standalone这种模式不需要指定网站根目录，它会自动启动服务器的443端口，来验证域名归属
#如果与其他服务占用了443端口，就必须先停止这些服务，生成证书完毕后，再启用
```

### 获取通配符证书

#### 什么是通配符证书

```sh
域名通配符证书类似DNS解析的泛域名概念，通配符证书就是证书中可以包含一个通配符
主域名签发的通配符证书可以在所有子域名中使用
```

#### 相关协议

```sh
Let's Encrypt上的证书申请是通过ACME协议来完成的,ACME v2是ACME协议的更新版本，通配符证书只能通过ACME v2获得
要使用ACME v2协议申请通配符证书，只需一个支持该协议的客户端就可以了，官方推荐的客户端是Certbot
```

#### 获取Certbot客户端

```sh
# 下载 Certbot 客户端
$ wget https://dl.eff.org/certbot-auto

# 设为可执行权限
$ chmod a+x certbot-auto

#注：Certbot从0.22.0版本开始支持ACME v2，如果你之前已安装旧版本客户端程序需更新到新版本
#更详细安装可以参考文档： https://certbot.eff.org/
```

#### 申请通配符证书

```sh
客户在申请Let’s Encrypt证书的时候，需要校验域名的所有权，证明操作者有权利为该域名申请证书，目前支持三种验证方式
#	dns-01：给域名添加一个 DNS TXT 记录
#	http-01：在域名对应的 Web 服务器下放置一个 HTTP well-known URL 资源文件
#	tls-sni-01：在域名对应的 Web 服务器下放置一个 HTTPS well-known URL 资源文件
```

```sh
$ ./certbot-auto certonly  -d "*.xxx.com" --manual --preferred-challenges dns-01  --server https://acme-v02.api.letsencrypt.org/directory
# 	申请通配符证书，只能使用 dns-01 的方式
#	xxx.com 请根据自己的域名自行更改
#	命令在执行到最后一步时，先不要回车，申请的通配符证书是要经过DNS认证的，接下来需要在域名后台添加对应的DNS TXT记录，添加完成后，先输入一下命令确认TXT记录是否生效
$ dig  -t txt _acme-challenge.xxx.com @8.8.8.8
#	如果成功生成证书在/etc/letsencrypt/live/xxx.com/
$ openssl x509 -in  /etc/letsencrypt/live/xxx.com/cert.pem -noout -text 
#	可以看到证书包含了 SAN 扩展，该扩展的值就是 *.xxx.com
#参数说明：
	certonly 表示插件，Certbot 有很多插件。不同的插件都可以申请证书，用户可以根据需要自行选择
	-d 为哪些域名申请证书，如果是通配符，输入 *.xxx.com (根据实际情况替换为你自己的域名)
	--preferred-challenges dns-01，使用DNS方式校验域名所有权
	--server，Let's Encrypt ACME v2版本使用的服务器不同于v1版本，需要显示指定
```

### 其他

#### 证书续期

```sh
#	免费证书默认只有90天，到期后如果需要续期可以执行
$ certbot-auto renew 或者 certbot renew --dry-run
#	如果生成证书时候用的是--standalone模式，此时会报错，因为需要用443端口来验证域名，但是443已经被占用，需要先暂时关掉nginx再运行命令
```

#### 计划任务自动续期

```sh
#	利用系统的crond周期性计划任务
#	创建一个文件certbot-auto-renew-cron
$ vim certbot-auto-renew-cron
 	15 2 * * * certbot renew --pre-hook "service nginx stop" --post-hook "service nginx start"
#	--pre-hook这个参数表示执行更新操作之前要做的事情
#	--post-hook这个参数表示更新操作完后需要做的事情
$ crontab  certbot-auto-renew-cron	#启动这个定时任务
```

#### nginx相关配置

```sh
#	先检查nginx是否启用ssl模块
$ nginx -v
#	如果没有重新启用nginx的ssl模块
$ ./configure	--prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
```

```sh
server {
        server_name diamondfsd.com www.diamondfsd.com;
        listen 443;
        ssl on;
        ssl_certificate /etc/letsencrypt/live/diamondfsd.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/diamondfsd.com/privkey.pem;

        location / {
           proxy_pass http://127.0.0.1:3999;
           proxy_http_version 1.1;
           proxy_set_header X_FORWARDED_PROTO https;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header Host $host;
        }
    }
    server {
        server_name api.diamondfsd.com;
        listen 443;
        ssl on;
        ssl_certificate /etc/letsencrypt/live/api.diamondfsd.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/api.diamondfsd.com/privkey.pem;

        location / {
           proxy_pass http://127.0.0.1:4999;
           proxy_http_version 1.1;
           proxy_set_header X_FORWARDED_PROTO https;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header Host $host;

        }
    }
#	若要同时监听http和https,把ssl on；这行去掉，将ssl写在443后面，否则只需要写listen ssl;	
```









































