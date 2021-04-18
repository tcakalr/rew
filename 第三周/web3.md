1.本视频一开始介绍了哪两个工具，他们的作用分别是什么？为什么作者会推荐firefox，它的优点是什么？

```
burp代理和firefox burp代理

firefox的代理可以在游览器设置，而不用对系统做设置。捕抓到的流量也是我们需要的web流量而不是一些与系统涉及的流量

还有相关的工具用于查看cookie和修改cookie等。
```

 2.本视频中体现了哪些攻防上的哲学观点？作者希望你养成什么样的思维？这些思维在帮助你挖掘漏洞的时候有什么帮助？结合你的经历与视频内容谈谈你的看法。（10分） 

```
不要像防守者一样思考，而是应该攻击者一样思考
 想要去了解一个应用最好的方式就是点击每一个功能，然后使用，不停地尝试这个是干什么的，这样会有助于我们去发现这个功能的问题。
 通常来说防守者需要找到每个bug，兼顾每个地方 ，而攻击者只需要找少量甚至一个漏洞点即可。
 思考攻击者的目的与网站功能相关起来，才能更好的了解攻击者想要什么。
```

3.审计以下代码：

```
<?php 
if(isset($_GET[ ' name ' ]))
{ echo "<h1>Hello {$_GET['name']} !</h1>"; } 
?> 
<form method="GET">
Enter your name: <input type="input" name="name"><br> 
<input type=" submit"> 
```

本段代码涉及到客户端，服务端以及通信协议。运行在客户端的代码主要有HTML以及javascript，由浏览器核心负责解释 通信协议为HTTP协议，有多种格式的请求包，常见的为POST与GET 运行在服务端的代码为php，由php核心负责解释。 用户端与服务端通过HTTP通信协议进行交互。 

那么，以上代码中，哪些部分属于客户端的内容，哪些属于服务端的内容？（1分） 

```
服务端：`<?php if(isset($_GET[ ' name ' ])){ echo "<h1>Hello {$_GET['name']} !</h1>"; } ?>`

客户端：`<form method="GET"> Enter your name: <input type="input" name="name"><br> <input type=" submit">` 
```

客户端是通过传递什么参数来控制服务端代码的？（1分） 

```
通过GET传参为name传入参数。
```

客户端通过控制该参数会对服务端造成什么影响，继而使得客户端本身收到影响，从而造成了什么漏洞？如果是xss漏洞，具体又是什么类型的xss漏洞，为什么？（3分） 

```
通过执行恶意代码，在游览器端账户环境下，执行任意javascript代码，造成xss漏洞，反射型的xss漏洞，我们输入的代码并未存储在服务端的数据库内，只有当前用户自己可见。
```

4.思考：现实中如何利用xss漏洞实施攻击，我们应该如何预防？（1分）

盗取cookie，xss蠕虫，xss模拟点击，xss远控都是可以利用的，多见用于盗取cookie（如果存在xss漏洞，那么通过xss平台可以获取它的cookie），用得到的cookie可以通过一些工具来修改cookie。不用登陆就能直接进入后台。当然有些网站修改后也是没有用的

防范：不信任任何用户的输入，对每个用户的输入都做严格检查，过滤，在输出的时候，对某些特殊字符进行转义，替换等。



# [WesternCTF2018]shrine    

```
import flask
import os
 
app = flask.Flask(__name__)
 
app.config['FLAG'] = os.environ.pop('FLAG')
 
@app.route('/')
def index():
    return open(__file__).read()
 
@app.route('/shrine/<path:shrine>')
def shrine(shrine):
 
    def safe_jinja(s):
        s = s.replace('(', '').replace(')', '')
        blacklist = ['config', 'self']
        return ''.join(['{{% set {}=None%}}'.format(c) for c in blacklist]) + s
 
    return flask.render_template_string(safe_jinja(shrine))
 
if __name__ == '__main__':
    app.run(debug=True)
```

看代码应该要在shrine路径下执行。

写入/shrine/{{5*5}}

执行成功

![微信图片_20210417195437](https://tcakalr.gitee.io/tc/微信图片_20210417195437.png)

代码中设置了一个blacklist，过滤了config和self。

```
app.config['FLAG'] = os.environ.pop('FLAG')
```

注册了一个名为FLAG的config

flag应该就在config里面了。

但是没法直接读取这个config

读取就返回了NONE

![微信图片_202104171954371](https://tcakalr.gitee.io/tc/微信图片_202104171954371.png)

然后就不会了，

又到了看wp的时候了

wp上给出了两个python内置函数url_for和get_flashed_messages

**当config,self,()都被过滤的时候，为了获取讯息，我们需要读取一些例如current_app这样的全局变量**

上述两个函数是包含current_app的

注入一下

![微信图片_202104171954372](https://tcakalr.gitee.io/tc/微信图片_202104171954372.png)

再读取

![微信图片_202104171954373](https://tcakalr.gitee.io/tc/微信图片_202104171954373.png)

发现flag在里面

# [护网杯 2018]easy_tornado

打开有三个文件可查看

![微信图片_20210417195726](https://tcakalr.gitee.io/tc/微信图片_202104171954371.png)

![微信图片_202104171957261](https://tcakalr.gitee.io/tc/微信图片_202104171957261.png)

![微信图片_202104171957262](https://tcakalr.gitee.io/tc/微信图片_202104171957262.png)

看到render，应该是考python模板注入

查看/fllllllllllllag时报错了

![微信图片_202104171954374](https://tcakalr.gitee.io/tc/微信图片_202104171954374.png)

发现msg传入参数可控

![微信图片_20210417200154](https://tcakalr.gitee.io/tc/微信图片_20210417200154.png)

但是运算被ban了

再根据查看文件时文件的字符串可以确定在读取文件时需要文件哈希

计算方式在hint.txt里给了

现在就要想办法得到cookie_secret

但是不知道怎么得到

查wp：在tornado模板中，存在一些可以访问的快速对象,这里用到的是handler.settings，handler  指向RequestHandler，而RequestHandler.settings又指向self.application.settings，所以handler.settings就指向RequestHandler.application.settings了，这里面就是我们的一些环境变量

直接查handler.settings

![微信图片_202104171954375](https://tcakalr.gitee.io/tc/微信图片_202104171954375.png)

得到了cookie_secret，写个php脚本计算一下

```
<?php
$cookie_secret='cd3d520f-51a9-4ad4-8fa3-066e80323ae8';
echo md5($cookie_secret.md5('/fllllllllllllag'));
?>#93c5b70dfec0fd6bfacf13e0973415ca
```

写入读取

![微信图片_202104171954376](https://tcakalr.gitee.io/tc/微信图片_202104171954376.png)

# ssti的一些绕过姿势

除了做题外还学习了ssti的一些绕过姿势

###### 过滤了**.**

可以使用[]，attr()，getattr()来绕

- []

```
例如可以用{{()[__class__]}}来代替{{().__class__}}
```

- attr()

```
用{{()|attr('__class__')}}代替{{().__class__}}
```

- getattr()

  ```
  {{getattr((),"__class__")}}代替{{().__class__}}
  ```

  

###### chr绕过

可以利用chr函数构造想要的ascii字符

网上有相应脚本可以利用

```
<?php
$a = 'whoami';
$result = '';
for($i=0;$i<strlen($a);$i++)
{
 $result .= 'chr('.ord($a[$i]).')%2b';
}
echo substr($result,0,-3);
?>
```

###### 过滤了_

下划线是关键字里要用到的

但是被过滤后可以用编码的方式绕过

1.使用十六进制编码绕过

使用十六进制绕过的话其实也可以直接把过滤的关键字也一同绕过

###### 过滤关键字

过滤关键字的方式有很多，包括但不局限于黑名单，替换为空。

1.用+来拼接

在[]中使用+来拼接字符，如：`['__cla'+'ss__']`或者直接不加+也行就只留下' '

2.join

```bash
["a","b","c"]|join
```

上述例子等价于abc。

3.decode绕过

只能用于python2

4.替代

查找可以替代相关的模块

如init可以用`__enter__`或`__exit__`替代

config，前面的一道题就是用到这个，但是被过滤了

查找文章可以使用

```
{{self}} ⇒ <TemplateReference None>
{{self.__dict__._TemplateReference__context}}
```

但是那道题连self都过滤了

所以才采用url_for或者get_flashed_messages查看相关配置。

###### 绕{{

使用{% if ... %}1{% endif %}

###### 数字过滤

用特殊数字绕过（参照明御攻防那题）

# 总结：

看了视频回答了相关问题，还算是学过一点点xss，后面的ssti是最近在学的一些东西，绕过姿势也只是最基础的，还有很多骚操作还没搞懂，最近在准备搭个靶场，github上有个ssti的靶场，准备搭起来用来更好的学习ssti。