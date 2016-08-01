# X-NUCA writeup #
## 0x01 sign ##
题目：Good Luck！flag{X-nuca@GoodLuck!}
## 0x02 BaseCoding ##
提示：这是编码不是加密哦!一般什么编码里常见等号？

题目：这一串字符好奇怪的样子，里面会不会隐藏什么信息？http://question1.erangelab.com/
Base64……
## 0x03 BaseInjection ##
提示：试试万能密码
题目：不知道密码也能登录。

http://question2.erangelab.com/

1'or'1'='1直接绕过（30，50的payload一样）
## 0x04 BaseReconstruction ##
提示：对数据包进行重构是基本技能

题目：此题看似和上题一样，其实不然。http://question3.erangelab.com/

只不过是本地验证，抓包绕过，payload同上 
## 0x05 CountingStars ##
提示：一不小心Mac也侧漏

题目：No more $s counting stars. http://question4.erangelab.com/

查看源码可以看到有个提示是说mac系统的，所以直接下载DS_Store，里面可以看到有一个zip
![1.png](1.png)
下载后拿到index源码
![2.png](2.png)
直接echo $$$……S 就出来了
![3.png](3.png)
 
## 0x06 Invisible ##
题目：隐藏IP来保护自己。http://121.195.186.234

这个没有印象了，应该是改xff 127.0.0.1
## 0x07 Normal_normal ##
提示：phpwind 后台getxxxxx

题目：又是一个bbs。http://question6.erangelab.com/

一开始写的是phpbb，搜了半天发现那些12年的漏洞没什么卵用，论坛里面有个帖子，留了一个邮箱zhangrendao2008#126.com，然后无聊的我跟邮箱发了个邮件，然后不知道哪个师傅就开始跟我聊天了

![4.png](4.png)

然后就懂了啊，立马去126的裤子里面找到了密码

![5.png](5.png)

登了邮箱发现没什么用，直接拿密码去登了后台，后台的功能都被删了，只有个模板管理的，进去后发现，，，全是这个！！！我服……谁写的，出来保证不打你！

![6.png](6.png)

最后在调用代码里面找到了几个链接，挨个试了试，把xml改成txt就出来了
http://question6.erangelab.com/index.php?m=design&c=api&token=BvNWdJUDHV&id=11&format=txt
![7.png](7.png)
## 0x08 DBexplorer ##
提示：a.SELECT @@datadir 。。。mysql/user.MYD b.user.MYD

题目：Where is my data。http://question7.erangelab.com/（请不要修改密码！）

这个题也是奇葩，其实不是难。最开始源码理由提示，vim老套路，.db.php.swp，但是没有index的源码，从db中找到了数据库账号和密码，还有phpmyadmin路径
 ![8.png](8.png)
登上去还没玩一会密码就被改了，然后整个目录被秒删。也是醉了，后来就没什么了，最开始早就找到了路径，但是一直没想到去导出user.MYD，一直在想办法怎么把外面的文件导进去，tmp目录是可以写的，从swp里面找到了绝对路径，但是/var/www/html目录下的文件一直都是NULL，后来有了提示，最开始想到的是自己本机的传上去然后覆盖user.MYD，但是路径怎么也不对，/var/lib/mysql这个路径早就找到了，但是智障的我，以为win跟linux是一样的，总以为数据库应该在data目录下面，mdzz，所以就成了
/var/lib/mysql/data/mysql/user.MYD，然后就一直智障了下去……
不过在执行语句的时候，看到了各种大牛在导出，源码，passwd，各种都有
 ![9.png](9.png)
所以就留意了一下，如果有大牛成功导出的话，应该所有人都可以看到的，所以就一直在刷新，，，终于大牛出现了，然后直接就复制了下来……Orz
 ![10.png](10.png)
反正最后我也没有导出成功，之所以拿下是在执行命令的时候，突然看到了一个大牛导出的user.MYD,然后瞬间保存了下来，最后topsec  topsec123456登上去，拿下了flag（手速慢了，一血也是这么被拿下的，给@ming师傅手速点赞 ）
 ![11.png](11.png)
最后问了下师傅们，师傅们给的命令，没法复现了，自己本地搭环境试试吧
LOAD DATA INFILE '/var/lib/mysql/mysql/user.MYD' INTO TABLE q fields terminated by 'LINES' TERMINATED BY '\0'  
## 0x09 RotatePicture ##
提示：urlopen file schema

题目：转转转。http://question8.erangelab.com/picrotate

看到题目直接试了下file:///etc.passwd 可以读到一行，然后就不会玩了，后来看了下源码有个view.py。直接进进去试试，得到了：we hava a view named "getredisvalue"
直接访问：http://question8.erangelab.com/getredisvalue
![12.png](12.png)
 
搜了下前短时间出来的那个Python urllib HTTP头注入
http://www.tuicool.com/articles/2iIj2eR
直接上payload试了下，随便设置个uuid
 ![13.png](13.png)
提交uuid得到flag
 ![14.png](14.png)
## 0x10 AdminLogin ##
题目：On the way in。http://121.195.186.238/index.php
这个题一开始是q9，后来改成了ip，所以最后一步xff一直过不去，不过最后还是被机智的我发现了……
题目可以直接加referer用sqlmap跑，不过我是直接手工注的，解密是administrat0r
 ![15.png](15.png)
robots.txt里面有后台路径
 ![16.png](16.png)
然后估计大家都被坑在这里了，进去之后一直都是where……
 ![17.png](17.png)
也是醉了，最后脑洞打开，bugsacn扫了下，扫到了svn，直接脚本跑了下找到了路径http://question9.erangelab.com/xnucactfwebadmin/welcometoctfloginxxx.php
直接登录提示：Illegal IP ... expect 8.8.8.8
这很简单啊，xff就过了，然并卵
 ![18.png](18.png)
怎么改都不过，辣鸡，后来我都试了最新的xff格式，还是不行
Forwarded: for=192.0.2.60; proto=http; by=203.0.113.43
只能又一个解释啊，题目有坑，回去重新看了下题目，链接竟然改了，改成了
http://121.195.186.238/ xff秒过，厉害……
 ![19.png](19.png)
得到一张图片，直接扔火狐就好了
 ![20.png](20.png)
## 0x11 WeirdCamel ##
提示：a.小骆驼的%和@真是蛋疼 b.嗯……URL转义有时候会失效 c.也许变量能够覆盖哦
题目：欢迎报名夏令营，请您仔细阅读公告，之后我们将会审核您的报名信息。http://question10.erangelab.com/
这个一开始完全没思路，500太坑，最后又来了个提示，变量覆盖（post:name=a&name=STATEMENT&name=register.pl），直接拿到了源码
 ![21.png](21.png)
 ![22.png](22.png)
可以直接命令执行（post: name=1&name=STATEMENT&name=|ls|）
然后翻了下目录，没找到flag，不过有个xnuca_looktheregisternews.pl
 ![23.png](23.png)
源码：
 ![24.png](24.png)
然后就是弹shell了，上py脚本，直接反弹shell出来
登mysql提示：
Access denied for user 'xnuca_user'@'localhost' to database 'xnuca_news_db' when using LOCK TABLES
我服，写了个py脚本去读所有字段，提示：
ImportError: No module named MySQLdb
我服，还打算写个php的，后来发现内核版本有点老啊，ubuntu的，上exp
https://www.exploit-db.com/exploits/37292/
 ![25.png](25.png)
我服，又穿了，中午穿过一次了……
最后看师傅们都在使劲传脚本，弹shell，无奈了，太菜害怕被超，最后十几分钟干了点缺德事，抱歉抱歉……
 ![26.png](26.png)
不过最后发现，删了以后有点亏，因为拿到了root密码xgsqggxwalspassw0rd，不知道是否通用啊，如果通用的话，那500也就可以拿下了……Orz
## 0x12 OneWayIn ##
题目：How can I get in。http://question11.erangelab.com/
（这题全靠记忆，流程一点都记不得了，凑合看吧……）
首先查看源码，弱类型登录
 ![27.png](27.png)
跳转http://question11.erangelab.com/flag_manager/index.php?file=dGVzdC50eHQ=
dGVzdC50eHQ=解密后是test.txt，
去掉参数后会直接跳转到index.php?file=dGVzdC50eHQ=&num=
将file换成index.php读源码，num是行数
http://question11.erangelab.com/flag_manager/index.php?file=aW5kZXgucGhw&num=2
 ![28.png](28.png)
把cookie改成flagadmin去读取flag.php源码，是混淆加密过的
 ![29.png](29.png)
这样直接copy去解密肯定是不行的，我用python request取解密也没成功，最后curl取出来，在线解密三块钱搞定了，然后base64。
 ![30.png](30.png)
 ![31.png](31.png)
再
往
下
看
有
惊
喜
2
3
3
3
还有一种不用解密的方法，直接在本地运行php，localhost会提示

 ![32.png](32.png)

改成127.0.0.1……钱都花了，才想起来，hhh
 ![33.png](33.png)

