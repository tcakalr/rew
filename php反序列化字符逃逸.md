---
title: PHP反序列化字符逃逸
---

最近遇到相关的题目，就去学习一下相关的知识。

php在反序列化时会以{开始，在遇到;}时结束，接下来做个小测试

```php
<?php
class xinxi
{
   public $name = "xiaoming";
   public $sex = "boy";
   public $age = "15";  
}

$a = new xinxi();
$b = serialize($a);
print_r($b)."\n";
$c = unserialize($b);
print_r($c)."\n";
print_r(unserialize('O:5:"xinxi":3{s:4:"name";s:8:"xiaoming";s:3:"sex";s:3:"boy";s:3:"age";s:2:"15";}123456'));
?>
/*O:5:"xinxi":3:{s:4:"name";s:8:"xiaoming";s:3:"sex";s:3:"boy";s:3:"age";s:2:"15";}xinxi Object
(
    [name] => xiaoming
    [sex] => boy
    [age] => 15
)
xinxi Object
(
    [name] => xiaoming
    [sex] => boy
    [age] => 15
)*/
```

可以发现在;}后的东西并不会被当做反序列化需要执行的字符串。

那我们是不是可以构造我们要的字符串让反序列化提前结束，同时修改掉相关的数据。

继续做测试：

这里逃逸的方法主要是两种：

1.字符串的增加

2.字符串的减少

先说说字符串的增加，举个例子

```php
<?php
$a = array("name" => "xiaoming","age" => "15");
print_r(serialize($a))."\n";
$k = serialize($a);
$b = preg_replace('/o/', 'oo', $k);
echo $b;
print_r(unserialize($b));
?>
/*a:2:{s:4:"name";s:8:"xiaoming";s:3:"age";s:2:"15";}a:2:{s:4:"name";s:8:"xiaooming";s:3:"age";s:2:"15";}
Notice: unserialize(): Error at offset 29 of 52 bytes in C:\Users\luorui\Desktop\1.php on line 7
PHP Notice:  unserialize(): Error at offset 29 of 52 bytes in C:\Users\luorui\Desktop\1.php on line 7
```

可以发现在字符数与s:后的数字对应不上时是会报错的，但是字符数对应上就不会报错了。

那我们可以试试将age修改利用字符的增多。

```php
<?php
$a = array('name' => 'xiaoming;s:3:"age";s:2:"20";}','age' => '15');
print_r(serialize($a))."\n";
?>
//a:2:{s:4:"name";s:29:"xiaoming;s:3:"age";s:2:"20";}";s:3:"age";s:2:"15";}
```

那接下来我们就将xiaoming增长使得s:29在识别后面字符串时不把;s:3:"age";s:2:"20";}包含进去，就能成功将后面一串失效从而使我们构造的age为20包含进去，

```php
<?php
$a = array('name' => 'xiaoming;s:3:"age";s:2:"20";}','age' => '15');
print_r(serialize($a))."\n";

$b='a:2:{s:4:"name";s:29:"xiaooooooooooooooooooooooming";s:3:"age";s:2:"20";}";s:3:"age";s:2:"15";}';
print_r(unserialize($b));
?>
/*a:2:{s:4:"name";s:29:"xiaoming;s:3:"age";s:2:"20";}";s:3:"age";s:2:"15";}Array
(
    [name] => xiaooooooooooooooooooooooming
    [age] => 20
)
```

在xiaooooooooooooooooooooooming后加上"使其与前面的"闭合，且xiaooooooooooooooooooooooming符合前面字数，所以并不会报错，使得我们后面构造的字符串成功逃逸，修改掉age的数据。





接下来说一下字符减少:

还是一个思路，前面利用增加让它不包含我们的字符串，现在用减少让它吞掉一些我们不要的字符

还是举个例子：

```php
<?php
$a = array('name' => 'xiaoooooooooooooming', 'age' => '15');
print_r(serialize($a));
?>
/*a:2:{s:4:"name";s:20:"xiaoooooooooooooming";s:3:"age";s:2:"15";}
```

我们构造字符串后要让s:12往后吞，那我们就要减短xiaoooooming这个字符串的长度，

先构造我们要的字符

讲年龄修改为20

```php
<?php
$a = array('name' => 'xiaoooooooooooooming', 'age' => '20');
print_r(serialize($a));
?>
```

输出的东西里面";s:3:"age";s:2:"20";}这一串是我们要的。

我们把xiaooooooooooooooming删减掉，删减成2位，因为我们构造的字符串为18位要符合就要使原本的字符串减少为2位。

```php
<?php
$a = array('name' => 'xiaoooooooooooooming', 'age' => '15');
print_r(serialize($a))."\n";

$b = array('name' => 'xiaoooooooooooooming', 'age' => '";s:3:"age";s:2:"20";}');
print_r(serialize($b))."\n";

$c = 'a:2:{s:4:"name";s:20:"xi";s:3:"age";s:22:"";s:3:"age";s:2:"20";}";}';
print_r(unserialize($c));
?>
/*a:2:{s:4:"name";s:20:"xiaoooooooooooooming";s:3:"age";s:2:"15";}a:2:{s:4:"name";s:20:"xiaoooooooooooooming";s:3:"age";s:22:"";s:3:"age";s:2:"20";}";}Array
(
    [name] => xi";s:3:"age";s:22:"
    [age] => 20
)
```

可以看到age成功被修改为20。



结语：只是学习了相关基础知识，但是并未做题实战，到时候再补回相关题目。





