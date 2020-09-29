# mysql笔记--基础知识






## SQL基础



SQL语句的分类：

- DQL: 数据库查询语句，基本的就是select查询命令，用于查询数据
- DML: 数据操纵语句，用于插入，更新，删除数据，即INSERT, UPDATE,DELETE
- DDL: 数据定义语句，用于创建，删除，以及修改表，索引等数据库对象,CREATE,DRIO,ALTER



创建表：

```sql
CREATE TABLE table_name (
	col01_name data_type,
	col02_name data_type,
	col03_name data_type,
	....
)
```

关于data_type, 常用的有以下数据类型：

整数：

   1. tinyint: 微整数：很小的整数占8位二进制

   2. smallint: 小整数： 小的整数 占16位二进制

   3. mediumint: 中整数：中等长度的整数，占24位二进制

   4. int: 整型，整数类型，占32位二进制

      

小数

1. float:单精度浮点数，占4个字节
2. double:双精度，占8个字节



日期

1. time : 表示时间类型
2. date: 表示日期类型 (只包含年月日 yyyy-MM-dd)
3. datetime: 同时可以表示日期和时间类型(包含年月日十分秒，yyyy-MM-dd HH:mm:ss)
4. timestamp: 时间戳类型，(包含年月日十分秒，yyyy-MM-dd HH:mm:ss)，如果添加数据时没有赋值，则自动插入当前的系统时间



字符串

1. char(m): 固定长度的字符串，使用几个字符就占用几个，M 为 0~65535 之间的整数

2. varchar(m):可变长度的字符串，使用几个字符就占用几个，M 为 0~65535 之间的整数

   

大二进制

1. tinyblob: 允许长度0-255字节
2. blog: 允许长度为0-65535字节
3. longblog: 允许长度为0~4294967295 字节



大文本

      1. tinytext: 允许长度 0~255 字节
      2. text: 允许长度 0~65535 字节
      3. mediumtext: 允许长度 0~167772150 字节
      4. longtext: 允许长度 0~4294967295 字节



一个简单的例子：

```sql
mysql> create table student (
    -> id int,
    -> name varchar(32),
    -> age int,
    -> score double(4,1),
    -> birthday date,
    -> insert_time timestamp
    -> );
Query OK, 0 rows affected (0.01 sec)

mysql> desc student;
+-------------+-------------+------+-----+-------------------+-----------------------------+
| Field       | Type        | Null | Key | Default           | Extra                       |
+-------------+-------------+------+-----+-------------------+-----------------------------+
| id          | int(11)     | YES  |     | NULL              |                             |
| name        | varchar(32) | YES  |     | NULL              |                             |
| age         | int(11)     | YES  |     | NULL              |                             |
| score       | double(4,1) | YES  |     | NULL              |                             |
| birthday    | date        | YES  |     | NULL              |                             |
| insert_time | timestamp   | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+-------------+-------------+------+-----+-------------------+-----------------------------+
6 rows in set (0.00 sec)

mysql> 
```



删除表

```sql
DROP TABLE table_name;
```



修改表

```sql
ALTER TABLE student RENAME TO new_student; //修改表名
ALTER TABLE new_student character SET utf8; // 修改表的字符集
ALTER TABLE new_student ADD 列名 数据类型;  // 给表添加列
ALTER TABLE new_student CHANGE 要更改列名称 要更改为的列名称;  // 更改列名称
ALTER TABLE new_student MODIFY 列名 类型;  // 更改列的类型
ALTER TABLE new_student DROP 列名; // 删除表中一个列
```



插入语句

```sql
INSERT INTO student VALUES (1, '张三', 18);
INSERT INTO student (no,age,name) VALUES (1,'张三'，18)；
```



更新语句

```sql
UPDATE student SET age=16;
UPDATE student SET age=16, name="李四" WHERE no=2;
```



删除语句

```sql
DELETE FROM student WHERE no=2;
```

注意：如果没有WHERE 条件，将删除student表中的所有数据,

不推荐使用，因为这里是有多少数据就执行多少次删除数据操作



查询语句

```sql
select 
	字段列表
from
	表名列表
where
	条件列表
group
	分组字段
having
	分组之后的条件
order by
	排序
limit
	分页限定
```



**基础查询**

- select * from student；查询数据
- select name,age from student; 查询某些列的数据
- select distinct address from student; 去除重复的结果集
- select distinct name,address from student; 这里值name 和 address都相同的进行去重
- select name,math,english , math+english from student; 计算math和english成绩之和
- select name,math,english,math+IFNULL(english,0) from student; 这里的查询表示如果english是nulll的时候用0进行替换
- select name 名字,math 数学,english 英语,math+IFNULL(english,0) as 总分 from student;  这里是给查询的结果起别名这里是通过as实现，也可以直接空格之后加别名

**条件查询：**

where子句后跟条件

运算符：

```
>、 <、<=、>=、=、<> 、!=
between....and: 如between 100 and 200
in(集合):集合表示多个值，使用都好分隔
like:模糊查询， 占位符：_ 表示单个任意字符 ;  % 表示多个任意字符
is null : 查询某一列为null的值，注：不能写=null
and 或 &&
or 或 ||
not 或 ！
```

- 查询年龄大于20的数据：select * from student where age>20;
- 查询年龄大于等于20,小于等于30：select * from student where age>=20 and age<=30; 
- 查询年龄大于等于20,小于等于30：select * from student where age between 20 and 30; 
- 查询22,18,24岁的数据：select * from student where age=22 or age=18 or age=24;
- 查询22,18,24岁的数据：select * from student where age in (22,18,24);
- select * from student where english is null; 这里切忌不能使用= 或者 !=
- select * from student where english is not null;
- select * from student where name like '赵%'; 查询姓赵的人
- select * from student where name like '_华%'; 查询名字第二个是华的数据
- select * from student where name like '___'; 查询名字为3个字的数据
- select * from student where name like '%马%'; 查询名字包含马的数据



**排序查询**

oder by 排序字段 排序方式（asc,desc）

- select * from student order by math; 按照数学成绩排序，默认位升序(asc)
- select * from student order by math desc, english desc; 按数学成绩进行排序，如果math成绩一样，按照english成绩进行排序。 第二排序条件只有当第一排序条件无法排序时才会执行



**聚合函数**

将一列数据作为一个整体，进行纵向的计算。

count: 计算个数

max: 计算最大值

min: 计算最小值

sum: 求和

注意：聚合函数的计算都是排除了null值

avg: 计算平均值

- select count(name) from student;  查询student表一共有多少学生
- select count(ifnull(english,0)) from student; 防止为null计算数目不对的问题
- select count(*) from sutdent;
- select count(id) from student;
- select max(english) from student; 查询英语成绩最高的数据
- select min(english) from student; 查询英语成绩最低的数据
- select sum(english) from student; 查询英语成绩的总分
- select avg(english) from student; 查询英语成绩的平均分



**分组查询**

语法： group by 分组字段

注意：

分组之后查询的字段：要么是分组字段，要么是聚合函数

where 和 having 的区别：

where 在分组之前进行限定，如果不满足条件，则不参与分组。having 在分组之后进行限定，如果不满足结果，则不会被查询出来。

where 后不可以跟聚合函数，having可以进行聚合函数的判断



- select sex, avg(math) from student group by sex; 计算男女同学的数据平均成绩
- select sex, count(id) from student group by sex; 计算男女的人数
- select sex, avg(math) from student where math>70 group by sex;   计算数学成绩大于70分的男女同学的数据平均成绩
- select sex, avg(math), count(id) from student where math>70 gorup by sex having count(id) >2;  having条件一定是在group by 分组之后



**分页查询**

语法： limit 开始索引，每页查询的条数

- select * from student limit 0,3; 查询第一页
- select * from student limit 3,3;查询第二页
- select * from student limit 6,3;查询第三页



## 约束



对表中的数据进行限定，保证数据的正确性，有效性和完整性

分类：

- 主键约束：primary key
- 非空约束:  not null
- 唯一约束：unique
- 外键约束: foreign key



### 非空约束 not null

添加非空约束的方式：

- 创建表的时候添加约束

  例子如下：

```mysql
CREATE TABLE  stu (
  id INT,
  name varchar(20) NOT NULL -- name 为非空
);
```

还有的情况是我们创建表的时候没有添加非空的约束

```mysql
alter table stu modify name varchar(20);
```

创建表之后,手动更改表字段添加非空限制

```mysql
alter table stu modify name varchar(20) NOT NULL ;
```



### 唯一约束 unique

其实就是某一列的值不能重复

- 创建唯一约束

```mysql
-- 创建表时添加唯一约束
CREATE TABLE  stu (
  id INT,
  phone_number varchar(20) UNIQUE -- 手机号
);
```

**注意：唯一约束可以有null值，但是只能有一条记录为null**

- 删除唯一约束

```mysql
alter table stu drop index phone_number;
```

- 在表创建完之后，添加唯一约束

```mysql
-- 创建表
CREATE TABLE  stu (
  id INT,
  phone_number varchar(20) -- 手机号
);
-- 添加唯一约束
alter table stu modify phone_number varchar(20) UNIQUE ;
```

### 主键约束 primary key 

概念： 非空且唯一， 且一张表只能有一个字段位主键，通俗说主键就是表中记录的唯一标识

- 创建表时候添加主键约束

```mysql
-- 创建表时添加主键
CREATE TABLE  stu (
  id INT primary key , -- 给id添加主键约束
  name varchar(20)
);
```

- 删除主键

```mysql
-- 删除主键
alter table stu drop primary key ;
```

- 创建表之后添加主键

```mysql
-- 创建表
CREATE TABLE  stu (
  id INT , -- 给id添加主键约束
  name varchar(20)
);

-- 添加主键
alter table stu modify id int primary key ;
```

#### 自动增长 auto_increment

概念：如果某一列是数值类型的，使用auto_increment可以来完成值的自动增长,通常和主键一起使用，很少单独使用

- 在创建表时添加主键约束，并完整主键自增长

```mysql
-- 创建表，添加主键约束，并设置主键自增长
CREATE TABLE  stu (
  id INT primary key auto_increment, -- 给id添加主键约束,并设置自增长
  name varchar(20)
);
```

- 删除自动增长

```mysql
alter table stu modify id int;
```

- 创建表之后手动添加自增长

```mysql
-- 创建表，添加主键约束
CREATE TABLE  stu (
  id INT primary key, -- 给id添加主键约束
  name varchar(20)
);

alter table stu modify id int auto_increment ;
```



### 外键约束 foreign key

先通过一个例子来看为啥会有外键约束，通过下列sql语句创建表并插入数据：

```mysql
CREATE TABLE emp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    NAME VARCHAR(30),
    age INT,
    dep_name VARCHAR(30),
    dep_location VARCHAR(30)
);
-- 添加数据
INSERT INTO emp (name, age, dep_name, dep_location) VALUES ('张三', 20, '研发部', '广州');
INSERT INTO emp (name, age, dep_name, dep_location) VALUES ('李四', 21, '研发部', '广州');
INSERT INTO emp (name, age, dep_name, dep_location) VALUES ('王五', 20, '研发部', '广州');
INSERT INTO emp (name, age, dep_name, dep_location) VALUES ('老王', 20, '销售部', '深圳');
INSERT INTO emp (name, age, dep_name, dep_location) VALUES ('大王', 22, '销售部', '深圳');
INSERT INTO emp (name, age, dep_name, dep_location) VALUES ('小王', 18, '销售部', '深圳');
```



生成如下数据：

```
1	张三	20	研发部	广州
2	李四	21	研发部	广州
3	王五	20	研发部	广州
4	老王	20	销售部	深圳
5	大王	22	销售部	深圳
6	小王	18	销售部	深圳
```

从上面的数据我们可以看到前三个人的数据和后三个人的数据显得非常冗余

这里有一个方案就是把这张表拆程两张表

```mysql
-- 创建部门表(id,dep_name,dep_location)
-- 一方，主表
create table department(
    id int primary key auto_increment,
    dep_name varchar(20),
    dep_location varchar(20)
);
-- 创建员工表(id,name,age,dep_id)
-- 多方，从表
create table employee(
    id int primary key auto_increment,
    name varchar(20),
    age int,
    dep_id int -- 外键对应主表的主键
);
-- 添加 2 个部门
insert into department values('研发部','广州'),('销售部', '深圳');
select * from department;
-- 添加员工,dep_id 表示员工所在的部门
INSERT INTO employee (name, age, dep_id) VALUES ('张三', 20, 1);
INSERT INTO employee (name, age, dep_id) VALUES ('李四', 21, 1);
INSERT INTO employee (name, age, dep_id) VALUES ('王五', 20, 1);
INSERT INTO employee (name, age, dep_id) VALUES ('老王', 20, 2);
INSERT INTO employee (name, age, dep_id) VALUES ('大王', 22, 2);
INSERT INTO employee (name, age, dep_id) VALUES ('小王', 18, 2);
```

生成如下数据：

```
1	研发部	广州
2	销售部	深圳
```

```
1	张三	20	1
2	李四	21	1
3	王五	20	1
4	老王	20	2
5	大王	22	2
6	小王	18	2

```

这样我们虽然解决了数据冗余的问题，但是还有一个问题，假如有一天我们们不小心把部门数据删除了，这个时候员工表中的字段依然还存在dep_id，以及当我们添加了一个员工，但是我们填写了一个不存在dep_id

这里我们就需要解决员工表中的dep_id 只能是department表中存在的id, 这个时候就需要用到外键



#### 什么是外键？

在从表中与主表主键对应的那一列，就像上面员工表中的dep_id

主表： 一方，用来约束别人的表

从表： 多方， 被别人约束的表



- 创建表的时候添加外键

```mysql
-- 创建部门表(id,dep_name,dep_location)
-- 一方，主表
create table department(
    id int primary key auto_increment,
    dep_name varchar(20),
    dep_location varchar(20)
);
-- 创建员工表(id,name,age,dep_id)
-- 多方，从表
create table employee(
    id int primary key auto_increment,
    name varchar(20),
    age int,
    dep_id int,-- 外键对应主表的主键
    -- 创建外键约束
    constraint emp_depid_fk foreign key  (dep_id) references department(id)

);
-- 添加 2 个部门
insert into department values(null, '研发部','广州'),(null, '销售部', '深圳');
select * from department;
-- 添加员工,dep_id 表示员工所在的部门
INSERT INTO employee (NAME, age, dep_id) VALUES ('张三', 20, 1);
INSERT INTO employee (NAME, age, dep_id) VALUES ('李四', 21, 1);
INSERT INTO employee (NAME, age, dep_id) VALUES ('王五', 20, 1);
INSERT INTO employee (NAME, age, dep_id) VALUES ('老王', 20, 2);
INSERT INTO employee (NAME, age, dep_id) VALUES ('大王', 22, 2);
INSERT INTO employee (NAME, age, dep_id) VALUES ('小王', 18, 2);
```

上面的表和我们之前拆分之后的表表面上看着没啥区别，但是其实内在它们已经建立了联系

这个时候我们可以尝试删除部门表中的一个数据

```mysql
db1> DELETE FROM db1.department WHERE id = 1
[2020-09-29 16:27:55] [23000][1451] Cannot delete or update a parent row: a foreign key constraint fails (`db1`.`employee`, CONSTRAINT `emp_depid_fk` FOREIGN KEY (`dep_id`) REFERENCES `department` (`id`))
[2020-09-29 16:27:55] [23000][1451] Cannot delete or update a parent row: a foreign key constraint fails (`db1`.`employee`, CONSTRAINT `emp_depid_fk` FOREIGN KEY (`dep_id`) REFERENCES `department` (`id`))

```

这个时候我们就可以看到因为员工表与部门表有外键关联关系，所以会提示我们无法`Cannot delete or update a parent row: a foreign key constraint fails...`

同时这个时候我们给员工表添加一个表，同时dep_id 我们随便写一个

```mysql
db1> INSERT INTO db1.employee (id, name, age, dep_id) VALUES (7, '哈哈哈', 222, 2323)
[2020-09-29 16:30:22] [23000][1452] Cannot add or update a child row: a foreign key constraint fails (`db1`.`employee`, CONSTRAINT `emp_depid_fk` FOREIGN KEY (`dep_id`) REFERENCES `department` (`id`))
[2020-09-29 16:30:22] [23000][1452] Cannot add or update a child row: a foreign key constraint fails (`db1`.`employee`, CONSTRAINT `emp_depid_fk` FOREIGN KEY (`dep_id`) REFERENCES `department` (`id`))

```

这个时候同样因为外键的关系，我们因为上面指定的外键dep_id是个不存在的值，所以提示了：`Cannot add or update a child row: a foreign key constraint fails...`

**注意： 外键值可以是null**

- 删除外键

```mysql
--  删除 employee 表的 emp_depid_fk 外键
alter table employee drop foreign key emp_depid_fk;
```

- 在employee 表已经创建的情况下，添加外键的方法

```mysql
-- 在 employee 表情存在的情况下添加外键
alter table employee add constraint emp_depid_fk foreign key  (dep_id) references department(id);
```

#### 外键的级联操作

还是我们的员工表和部门表，在我们已经建立了外键关系的情况下，加入我们更新了部门表中的id字段，这个时候需要怎么做？

想到的笨办法就是把员工表中和这个id相关的数据的dep_id 设置为null, 然后update 部门表中的id, 最后更新员工表中的数据的dep_id 位更新后的id.但这显然不是好办法，太麻烦了，其实外键提供了更方便的设置，**就是外键的级联**

| 级联操作语法      | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| ON UPDATE CASCADE | 级联更新，只能是创建表的时候创建级联关系。更新主表中的主键，从表中的外键 列也自动同步更新 |
| ON DELETE CASCADE | 级联删除                                                     |



我们重新创建表，并添加级联：

```mysql
-- 创建部门表(id,dep_name,dep_location)
-- 一方，主表
create table department(
    id int primary key auto_increment,
    dep_name varchar(20),
    dep_location varchar(20)
);
-- 创建员工表(id,name,age,dep_id)
-- 多方，从表
create table employee(
    id int primary key auto_increment,
    name varchar(20),
    age int,
    dep_id int,-- 外键对应主表的主键
    -- 创建外键约束, 并设置级联删除和级联更新
    constraint emp_depid_fk foreign key  (dep_id) references department(id) on update cascade on delete cascade 

);
-- 添加 2 个部门
insert into department values(null, '研发部','广州'),(null, '销售部', '深圳');
select * from department;
-- 添加员工,dep_id 表示员工所在的部门
INSERT INTO employee (name, age, dep_id) VALUES ('张三', 20, 1);
INSERT INTO employee (name, age, dep_id) VALUES ('李四', 21, 1);
INSERT INTO employee (name, age, dep_id) VALUES ('王五', 20, 1);
INSERT INTO employee (name, age, dep_id) VALUES ('老王', 20, 2);
INSERT INTO employee (name, age, dep_id) VALUES ('大王', 22, 2);
INSERT INTO employee (name, age, dep_id) VALUES ('小王', 18, 2);
```

这样当我们更新部门表的id的时候都同时更新员工表中的dep_id

同时删除了部门的数据也会把员工表dep_id 等于 该部门iid的数据进行删除



### 约束小结

| 约束名 | 关键字      | 说明                         |
| ------ | ----------- | ---------------------------- |
| 主键   | primary key | 1) 唯一    2) 非空           |
| 默认   | default     | 如果一列没有值，使用默认值   |
| 非空   | not null    | 这一列必须有值               |
| 唯一   | unique      | 这一列不能有重复值           |
| 外键   | foreign key | 主表中主键列，在从表中外键列 |



**注意：外键的级联慎用**



## 数据库的设计



### 多表之间的关系

#### 分类

- 一对一

  相对了来说用的场景比较少，

  例如： 一个人和身份证号码的关系，一个人只能有一个身份证号码

- 一对多（多对一）

  例如之前笔记中整理的：部门和员工的关系

  一个部门可以有多个员工，一个员工只能属于一个部门

- 多对多

  例如学生和课程之间的关系

  一个学院可以选修多门课程，同样一门课程也可以被多个学生选修

#### 实现关系

- 一对多（多对一）

  如： 部门和员工的关系

  实现方式：在**多的一方**(员工表)去建立外键，来指向**一的一方**（部门表）的主键
  例子:

  ```mysql
  -- 创建部门表(id,dep_name,dep_location)
  -- 一方，主表
  create table department(
      id int primary key auto_increment,
      dep_name varchar(20),
      dep_location varchar(20)
  );
  -- 创建员工表(id,name,age,dep_id)
  -- 多方，从表
  create table employee(
      id int primary key auto_increment,
      name varchar(20),
      age int,
      dep_id int,-- 外键对应主表的主键
      -- 创建外键约束, 并设置级联删除和级联更新
      constraint emp_depid_fk foreign key  (dep_id) references department(id) on update cascade on delete cascade
  
  );
  ```

- 多对多

  如：学生和课程的关系

  实现方式：多对多关系实现需要借助第三张中间表，中间表至少包含两个字段，这两个字段作为第三张表的外键，分别指向两张表（课程表和学生表）的主键

- 一对一

  如：人和身份证

  实现方式： 一对一关系实现，可以在任意一方添加唯一外键指向另一方的主键

  备注： 这种用法较少，因为这个时候通常会合成一张表

  

#### 实际案例

- 一对多案例

  京东和淘宝的手机分类中会有各个手机品牌，而每个手机品牌下会有很多手机型号，这就是非常典型的一对多关系

  ```mysql
  -- 创建手机品牌类别表phone_category
  -- pid 手机分类主键自动增长
  -- pname 手机类别的名字，如苹果, 小米
  create table phone_category (
      cid int primary key auto_increment,
      cname varchar(20) not null unique
  );
  
  -- 添加手机牌子
  insert into phone_category (cname) values ("苹果"), ("小米"),("三星");
  
  
  -- 创建具体手机表
  /*
  pid 手机表主键，自动增长
  pname 手机名字
  price 价格
  cid 外键
  */
  create table phone_name (
      pid int primary key auto_increment,
      pname varchar(20) not null unique ,
      price double,
      cid int,
      constraint foreign key (cid) references phone_category(cid)
  );
  
  -- 添加手机数据
  insert into phone_name values
                                (NULL, "IPhonex",4999.9,1),
                                (NULL, "IPhone11",5999.9,1),
                                (NUll, "mi6",1999.9,2),
                                (NUll, "mi10",2999.9,2),
                                (NUll, "s8",6999.9,3),
                                (NUll, "s10",7999.9,3);
  ```

  

- 多对多案例

  还是关于手机的案例，接着上面一对多的案例，其实我们再添加一个关系手机存储，就像手机有64G，128G, 256G, 不管那个手机都会有多个存储的，所以这里的手机和存储之间的关系就是多对多的关系

  ```mysql
  /*
  创建手机存储表
  mid 存储表的id,自增长
  memory 内存大小，唯一，非空
  */
  create table phone_memory (
      mid int primary key auto_increment,
      memory int not null unique
  );
  
  
  -- 添加内存数据
  insert into phone_memory values
                                  (NULL, 64),
                                  (NULL, 128),
                                  (NULL, 256);
  
  
  /*
   创建第三张表存手机和内存之间的关系
   mid 内存的id 外键
   pid 手机的id 外键
   */
  
   create table phone_m2m_memory(
       mid int,
       pid int,
       primary key (mid,pid),
       foreign key (mid) references phone_memory(mid),
       foreign key (pid) references phone_name(pid)
   );
  
  -- 增加手机内存数据
  
  insert into phone_m2m_memory values
                                      (1, 1),
                                      (1,2),
                                      (1,3),
                                      (2,1),
                                      (2,2),
                                      (2,3),
                                      (3,1),
                                      (3,2),
                                      (3,3);
  ```



#### 表与表关系小结

| 表与表关系 | 关系的维护                                        |
| ---------- | ------------------------------------------------- |
| 一对多     | 主外键的关系                                      |
| 多对多     | 中间表，两个一对多                                |
| 一对一     | 1）特殊一对多，从表中的外键设置为唯一 2）从表中的 |

### 数据库设计的范式

概念： 在设计数据库的时候需要遵循的规范。 要遵循后面的范式，必须先遵循前面的所有范式。

目前关系数据库有六种范式：第一范式（1NF）、第二范式（2NF）、第三范式（3NF）、巴斯-科德范式（BCNF）、第四范式(4NF）和第五范式（5NF，又称完美范式）。

掌握前三个设计出来的数据库基本不会有啥大问题

第一范式（1NF）：用一句话总结就是每一列是不可分割的原子数据项

第二范式（2NF）:   在1NF的基础上，非码属性必须完全依赖于候选码（在1NF基础上消除非主属性对主码的部分函数依赖）

概念补充：

- 函数依赖：

  A-->B,如果通过A属性(属性组)的值，可以确定唯一B属性的值。则称B依    赖于A

  例如：学号-->姓名。  （学号，课程名称） --> 分数               

- 完全函数依赖：

  A-->B， 如果A是一个属性组，则B属性值得确定需要依赖于A属性组中所有的属性值。

  例如：（学号，课程名称） --> 分数

- 部分函数依赖：

  A-->B， 如果A是一个属性组，则B属性值得确定只需要依赖于A属性组中某一些值即可。

  例如：（学号，课程名称） -- > 姓名

- 传递函数依赖：

  A-->B, B -- >C . 如果通过A属性(属性组)的值，可以确定唯一B属性的值，在通过B属性（属性组）的值可以确定唯一C属性的值，则称 C 传递函数依赖于A

  例如：学号-->系名，系名-->系主任

- 码：

  如果在一张表中，一个属性或属性组，被其他所有属性所完全依赖，则称这个属性(属性组)为该表的码

  例如：该表中码为：（学号，课程名称）

  主属性：码属性组中的所有属性

  非主属性：除了码属性组的属性

传递函数依赖：

A-->B, B -- >C . 如果通过A属性(属性组)的值，可以确定唯一B属性的值，在通过B属性（属性组）的值可以确定唯一C属性的值，则称 C 传递函数依赖于A

第三范式 (3NF):    在2NF基础上，任何非主属性不依赖于其它非主属性（在2NF基础上消除传递依赖）

#### 案例

[![3j27AH.png](https://s2.ax1x.com/2020/03/07/3j27AH.png)](https://imgchr.com/i/3j27AH)



![3jRM59.png](https://s2.ax1x.com/2020/03/07/3jRM59.png)



![3jRtbD.png](https://s2.ax1x.com/2020/03/07/3jRtbD.png)



[![3jR52q.md.png](https://s2.ax1x.com/2020/03/07/3jR52q.md.png)](https://imgchr.com/i/3jR52q)



## 多表查询



### 基础数据

```mysql
create table dept(
	id int primary key auto_increment,
	name varchar(20)
);
insert into dept(name) values('研发部'),('市场部'),('财务部');

create table emp(
	id int primary key auto_increment,
	name varchar(10),
	gender char(1), -- 性别
	salary double, -- 工资
	join_date date, -- 入职日期
	dept_id INT,
	foreign key (dept_id) references dept(id) -- 外键，关联部门表的主键
);

insert into emp(name, gender,salary, join_date,dept_id) values('张三','男', 12000, '2020-01-01',1);
insert into emp(name, gender,salary, join_date,dept_id) values('李四','男', 22000, '2019-03-01',2);
insert into emp(name, gender,salary, join_date,dept_id) values('王五','女', 3200, '2025-02-01',3);
insert into emp(name, gender,salary, join_date,dept_id) values('哈哈','女', 1000, '2000-12-01',1);
```



## 多表查询



### 内连接查询

#### 隐式内连接

使用where条件消除无用信息

例如：查询所有员工信息和对应的部门信息

```mysql
select * from emp,dept where emp.dept_id=dept.id
```

例如：查询员工表的名称，性别，部门表的名称

```mysql
select emp.name, gender,dept.name from emp,dept where emp.dept_id= dept.id;
```

```mysql
select
	t1.name,t1.gender,t2.name
from 
	emp t1, dept t2
where
	t1.dept_id=t2.id;
```



#### 显式内连接

语法： select 字段列表 from 表名 inner join 表名2 on 条件

inner 关键字可以省略

```mysql
select * from emp inner join dept on emp.dept_id = dept.id;
```



### 外连接查询

#### 左外连接

语法：select 字段列表 from 表1 left outer join 表2 on 条件

outer 关键字可以省略

```mysql
select
	t1.*,t2.name
from
	emp t1
left join
	dept t2
on
	t1.dept_id=t2.id;
```



总结过可以看出其实左外连接查询的是左表所有数据以及其他交集部分



#### 右外连接

语法：select 字段列表 from 表1 right outer join 表2 on 条件

outer 关键字可以省略

其实右外连接查询的是右表所有数据以及其他交集部分

```mysql
select
	t1.*,t2.name
from
	emp t1
right join
	dept t2
on
	t1.dept_id=t2.id;
```



### 子查询

概念： 查询中嵌套查询，称为子查询. 

```mysql
select * from emp where emp.salary=(select max(salary) from emp);
```



子查询包含几种不同的情况：

- 子查询的结果是单行单列
- 子查询的结果是多行单列的
- 子查询的结果是多行多列的



#### 子查询的结果是单行单列

子查询可以作为条件，使用运算符去判断。  运算符： **> , >= , <=, =**

```mysql
--- 查询员工工资小于平均工资的人
select * from emp where emp.salary < (select avg(salary) from emp);
```



####   子查寻的结果是多行单列

子查询可以作为条件，使用运算符**in**来判断

```mysql
-- 查询财务部和市场部所有员工的信息
select * from emp where dept_id in (select id from dept where name='财务部' or name='市场部');
```



#### 子查询的结果是多行多列的

```mysql
-- 查询入职日期在2019-01-01 之后的员工信息和部门信息
select * from dept t1, (select * from emp where emp.join_date>'2019-01-01') t2 where t1.id = t2.id;
```

```mysql
--  通过内连接也可以;
select * from dept t1, emp t2 where t2.join_date>'2019-01-01' and t1.id=t2.id;
```



### 多表查询练习



基础数据

```mysql
-- 部门表
CREATE TABLE dept (
  id INT PRIMARY KEY PRIMARY KEY, -- 部门id
  dname VARCHAR(50), -- 部门名称
  loc VARCHAR(50) -- 部门所在地
);

-- 添加4个部门
INSERT INTO dept(id,dname,loc) VALUES 
(10,'教研部','北京'),
(20,'学工部','上海'),
(30,'销售部','广州'),
(40,'财务部','深圳');



-- 职务表，职务名称，职务描述
CREATE TABLE job (
  id INT PRIMARY KEY,
  jname VARCHAR(20),
  description VARCHAR(50)
);

-- 添加4个职务
INSERT INTO job (id, jname, description) VALUES
(1, '董事长', '管理整个公司，接单'),
(2, '经理', '管理部门员工'),
(3, '销售员', '向客人推销产品'),
(4, '文员', '使用办公软件');



-- 员工表
CREATE TABLE emp (
  id INT PRIMARY KEY, -- 员工id
  ename VARCHAR(50), -- 员工姓名
  job_id INT, -- 职务id
  mgr INT , -- 上级领导
  joindate DATE, -- 入职日期
  salary DECIMAL(7,2), -- 工资
  bonus DECIMAL(7,2), -- 奖金
  dept_id INT, -- 所在部门编号
  CONSTRAINT emp_jobid_ref_job_id_fk FOREIGN KEY (job_id) REFERENCES job (id),
  CONSTRAINT emp_deptid_ref_dept_id_fk FOREIGN KEY (dept_id) REFERENCES dept (id)
);

-- 添加员工
INSERT INTO emp(id,ename,job_id,mgr,joindate,salary,bonus,dept_id) VALUES 
(1001,'孙悟空',4,1004,'2000-12-17','8000.00',NULL,20),
(1002,'卢俊义',3,1006,'2001-02-20','16000.00','3000.00',30),
(1003,'林冲',3,1006,'2001-02-22','12500.00','5000.00',30),
(1004,'唐僧',2,1009,'2001-04-02','29750.00',NULL,20),
(1005,'李逵',4,1006,'2001-09-28','12500.00','14000.00',30),
(1006,'宋江',2,1009,'2001-05-01','28500.00',NULL,30),
(1007,'刘备',2,1009,'2001-09-01','24500.00',NULL,10),
(1008,'猪八戒',4,1004,'2007-04-19','30000.00',NULL,20),
(1009,'罗贯中',1,NULL,'2001-11-17','50000.00',NULL,10),
(1010,'吴用',3,1006,'2001-09-08','15000.00','0.00',30),
(1011,'沙僧',4,1004,'2007-05-23','11000.00',NULL,20),
(1012,'李逵',4,1006,'2001-12-03','9500.00',NULL,30),
(1013,'小白龙',4,1004,'2001-12-03','30000.00',NULL,20),
(1014,'关羽',4,1007,'2002-01-23','13000.00',NULL,10);



-- 工资等级表
CREATE TABLE salarygrade (
  grade INT PRIMARY KEY,   -- 级别
  losalary INT,  -- 最低工资
  hisalary INT -- 最高工资
);

-- 添加5个工资等级
INSERT INTO salarygrade(grade,losalary,hisalary) VALUES 
(1,7000,12000),
(2,12010,14000),
(3,14010,20000),
(4,20010,30000),
(5,30010,99990);

-- 需求：

-- 1.查询所有员工信息。查询员工编号，员工姓名，工资，职务名称，职务描述

-- 2.查询员工编号，员工姓名，工资，职务名称，职务描述，部门名称，部门位置
   
-- 3.查询员工姓名，工资，工资等级

-- 4.查询员工姓名，工资，职务名称，职务描述，部门名称，部门位置，工资等级

-- 5.查询出部门编号、部门名称、部门位置、部门人数
 
-- 6.查询所有员工的姓名及其直接上级的姓名,没有领导的员工也需要查询

```



- 查询所有员工信息。查询员工编号，员工姓名，工资，职务名称，职务描述

  ```mysql
  -- 查询所有员工信息。查询员工编号，员工姓名，工资，职务名称，职务描述
  select t1.id,t1.ename,t1.salary,t2.jname,t2.description from emp t1, job t2 where t1.job_id = t2.id;
  ```



- 查询员工编号，员工姓名，工资，职务名称，职务描述，部门名称，部门位置

  ```mysql
  -- 查询员工编号，员工姓名，工资，职务名称，职务描述，部门名称，部门位置
  select t1.id,t1.ename,t1.salary,t2.jname,t2.description,t3.dname,t3.loc from emp t1, job t2, dept t3  where t1.job_id = t2.id and t1.dept_id=t3.id;
  ```

- 查询员工姓名，工资，工资等级

  ```mysql
  -- 查询员工姓名，工资，工资等级
  select t1.ename, t1.salary, (select grade from salarygrade where  t1.salary> losalary and salary<hisalary)  grade from emp t1
  ```

  或者

  ```mysql
  select t1.ename,t1.salary,t2.grade from emp t1, salarygrade t2 where t1.salary > t2.losalary and t1.salary < t2.hisalary;
  ```

- 查询员工姓名，工资，职务名称，职务描述，部门名称，部门位置，工资等级

  ```mysql
  -- 查询员工姓名，工资，职务名称，职务描述，部门名称，部门位置，工资等级
  select
         t1.id,t1.ename,t1.salary,t2.jname,t2.description,t3.dname,t3.loc ,t4.grade
  from emp t1, job t2, dept t3, salarygrade t4
  where
        t1.job_id = t2.id
    and
        t1.dept_id=t3.id
  and (t1.salary > t4.losalary and t1.salary < t4.hisalary);
  ```

- 查询出部门编号、部门名称、部门位置、部门人数

  ```mysql
  -- 查询出部门编号、部门名称、部门位置、部门人数
  select t1.dname, t1.loc, (select count(id)  from emp where emp.dept_id=t1.id ) count from dept t1;
  ```

  ```mysql
  select t1.id, t1.dname,t1.loc, t2.total from dept t1, (select dept_id,count(id) total from emp group by dept_id) t2 where t1.id = t2.dept_id
  ```

- 查询所有员工的姓名及其直接上级的姓名,没有领导的员工也需要查询

  ```mysql
  -- 查询所有员工的姓名及其直接上级的姓名,没有领导的员工也需要查询
  select t1.ename, (select ename from emp where t1.mgr=id)from emp t1;
  ```

  ```mysql
  select
         t1.ename,
         t1.mgr,
         t2.id,
         t2.ename
  from emp t1
  left join  emp t2
  on t1.mgr = t2.id
  ```



## 事务



### 事务的基本介绍

概念：如果一个包含多个步骤的业务操作，被事务管理，那么这些操作要么同时成功，要么同时失败

操作：

1. 开启事务：start transaction
2. 回滚：rollback
3. 提交：commit



注意： 

Mysql数据库中事务默认自动提交

一条DML（增删改）语句会自动提交奥一次事务

事务提交的两种方式：

自动提交：mysql的默认方式

手动提交：需要先开启事务，再提交



查看事务的默认提交方式：

```mysql
select @@autocommit; -- 1 表示自动提交，0表示手动提交
```

如果想要更改：

```mysql
set @@autocommit = 0;
```



### 事务的四大特征



- 原子性：是不可分割的最小单位，要么同时成功，要么同时失败
- 持久性：当事务提交或回滚后，数据会持久化的保存数据
- 隔离性： 多个事务之间。相互独立
- 一致性： 事务操作前后，数据总量不变



### 事务的隔离级别

概念：多个事务之间是相互隔离的，相互独立。但是多个事务操作同一批数据，会引发一些问题。设置不同的隔离级别就卡一解决这些问题

存在的问题：

- 脏读：一个事务，读取到另外一个事务中没有提交的数据
- 不可重复读（虚读）：在同一个事务中，两次读取到的数据不一样
- 幻读：一个事务操作数据表中所有记录，另外一个事务添加了一条数据，则第一个事务查询不到自己的修改

隔离级别：

- read umcommitted:读未提交。 会产生的问题：脏读，不可重复读，幻读
- read commit:读已提交。会产生的问题：不可重复读，幻读
- repeatable read: 可重复读。 会产生幻读 （Mysql默认级别）
- serializable: 串行化。 可以解决所有的问题

注意：隔离级别从小到大安全性越来越高，但是效率越来越低

数据库查询隔离级别： 

```mysql
select @@tx_isolation;
```



数据设置隔离级别：

```mysql
set global transaction isolation level 级别字符串 
```
