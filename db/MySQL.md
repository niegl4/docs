[TOC]

# 基本SQL

## 服务

配置文件路径

/etc/mysql/mysql.conf.d/mysqld.cnf

登录

mysql -uroot -p

mysql -hlocalhost -P3306 -uroot -p



## 库操作

展示所有数据库:

**show databases;**

 

创建数据库:

create database 数据库名 charset=utf8;

展示建库语句:

show create database 数据库名;

设置数据库字符集:

alter database 数据库名 charset=utf8;

 

选择使用的数据库:

**use** **数据库名**;

查看当前使用的数据库:

**select database();**

 

删除数据库:

drop database 数据库名;



## 表操作

展示所有数据表:

**show tables;**

 

创建数据表:

create table 表名(

id int(6) unsigned zerofill primary key not null auto_increment,

name varchar(20) not null,

gender enum("man","woman") default "woman",

age int(3) unsigned default 0

);

展示创表语句:

show create table 表名;

show create table 表名 \G --以键值对的形式展示,不带分号结尾

查看表结构:

desc 表名;

 

对表的字段增加,小改,大改,删除

alter table 表名 add 字段名 类型 约束;

alter table 表名 modify 字段名 类型 约束;

alter table 表名 change 原名 新名 类型 约束;

alter table 表名 drop 字段名;

 

删除数据表:

drop table 表名;



## 数据操作

### **增加数据**

1.全列插入

insert into 表名 values(0,xx,xx,xx,default,...); --全列插入时,顺序与字段顺序一致,即使有默认值,也要用default占位.

2.部分列插入

insert into 表名(字段1,字段2...) values(值1,值2..); --部分列插入时,也是对数据行的完整插入,所以只有设置了默认值或允许为null的字段,才允许省略不写.

3.增加多行时,

无论哪种方式,values后用多个括号即可.



### **删除数据**

1.只是删除数据,或者指定的数据.但是主键值会在原基础上变化.

delete from 表名 where 条件;

2.删除全部数据,同时主键值会清空.相当于初始化表.

truncate 表名;

3.推荐使用逻辑删除,而不用delete.

增加 isdelete 字段,表示是否被"删除".



### **修改数据**

update 表名 set 字段1=值1,字段2=值2... where 条件;



### **查找数据**

**select** <u>显示内容</u> **from** <u>数据源</u> **where** <u>条件</u> <u>附加(group by/order by/limit)</u>

#### 顺序

->from 确定数据源

->where 条件筛选

->group by 分组

->select 显示内容

->order by 排序

->limit 分页



1. 查找所有字段

   select * from 表名;

2. 查找指定字段

   select 字段1,字段2 from 表名;

3. select 的附加之 as起别名 (as可省略)

   给字段起别名

   select 字段1 as 字段1新名,字段2 as 字段2新名 from 表名;

   给表起别名

   select 表别名.字段1,表别名.字段2 from 表名 as 表别名;

4. select 的附加之 distinct去重查找

   <u>显示内容</u>

   select **distinct** 字段1 from 表名;

   select distinct 字段1,字段2 from 表名; --把两个字段作为组合条件,去重. 类似于两个字段作为组合条件分组.

5. select的附加之 聚合函数

   <u>显示内容</u>

   **聚合函数**要么在"显示内容"中使用,要么在分组后的having中充当条件(与group by配合).

   - count

     统计行数

     select count(字段名) from 表名; 

     统计指定字段的行数,默认不统计null.

     使用count(ifnull(字段名,设定值)),设定值为null,不统计null.设定值为其他,按1统计

     =>也就是说count统计行数时,始终忽略null行.

   - max(字段名),min(字段名),sum(字段名),avg(字段名) --默认排除null.

      select max(字段名),min(字段名),sum(字段名),avg(字段名) from 表名;

      select sum(ifnull(字段名,指定值)) from 表名; --如果为null,按指定值代替,再计算.

     自定义小数显示的位

      round(avg(字段名),保留的小数位)

      select round(avg(c_age),2) from t_student;

   - 聚合函数可以出现在分组后的having里.

     select gender,avg(age),group_concat(name) from students group by gender having avg(age) > 30;

     --统计每个分组指定字段的信息集合,每个信息之间用逗号分隔

6. select的数据源之 连接

   <u>数据源</u>

   - 内连接

     from 表1 as 表1别名 inner join 表2 as 表2别名 on 表1别名.字段名=表2别名.字段名;

   - 左连接/右连接

     左连接/右连接有一个"强势地位问题",左连接时,左表的字段名强势,左表存在的数据而右表不存在,则右表补充一个null.

     from 表1 as 表1别名 left/right join 表2 as 表2别名 on 表1别名.字段名=表2别名.字段名;

   - 自连接

     表自身与自身连接,表的结构比较特殊,比如:一些数据的字段值,是另一些数据的主键值.他们是有关联的==>自关联.

     from 表 as 表别名1 inner join 表 as 表别名2 on 表别名1.字段名=表别名2.字段名;

     举个有趣的例子:

     需求:找出河南省辖区内的所有市信息.

     select * from 地域表 as a inner join 地域表 as b on a.pid=b.id where b.name="河南省";

     语句中,一定是用id的那个表的名字叫"河南省",才能查出城市.与on后等号前后的顺序,inner join前后的顺序无关.

     其实,虽然是一个表起了两个名字,但是在自连接中,它们就是有区别的,有标识的.

     ====>**where**, **having**, **on**

7. select 的附加之 where条件查询

   <u>条件</u>

   where 比较(=, !=),逻辑(and, or, not),模糊(like,%,_),范围( between...and...,in (...,...,) ),空判断(is null,is not null)

    %任意多个任意字符

   _一个任意字符

   select * from students where name like "黄%";

   select * from students where name like "黄_";

8. select的子查询

   <u>条件/数据源</u>

   子查询要么充当条件,要么充当数据源.子查询充当数据源时,需要给数据源起一个别名.

   子查询可以独立存在,是一条完整的select语句.它的语法必须完整,可以用来初步检查笔误.

   需求:挑选出年龄最大的女性,在她们中再挑选班级ID最小的人员信息.

   select * from (select * from t_student where c_age=(select max(c_age) from t_student where c_gender="女")) as a 

   where a.c_class_id=

   (select min(b.c_class_id) from (select * from t_student where c_age=(select max(c_age) from t_student where c_gender="女")) as b);

9. select 的附加之 where后面加order by 字段 asc/desc

   <u>附加</u>

   select * from 表名 where 条件 order by 字段1 asc/desc [字段2 asc/desc ...];

   其中asc升序为默认,可以省略.

   asc:ascend

   desc:descend

10. select的附加之 where后面加limit 分页查询

    <u>附件</u>

    select * from 表名 where 条件 limit 起始行索引(不等于id),显示行数;

11. select的附加之 分组

    <u>附加</u>

    group by 字段名[,字段名2...] [having 条件表达式] [with rollup]

    字段名: 是指按照指定字段的值进行分组。

    having 条件表达式: 用来过滤分组后的数据。

    with rollup：在所有记录的最后加上一条记录，显示select查询时聚合函数的统计和计算结果.

    having 与 with rollup 好像只能二选一.???

    - select后字段名与by后字段名要完全一致! 或 使用group_concat(字段名2)

      select 字段名 from 表名 group by 字段名;

      select group_concat(字段名2) from 表名 group by 字段名1;

    - 如需增加显示的信息,需要使用group_concat(字段名).显示的是,在分组内,指定字段的信息集.

      select 字段名,group_concat(字段名2) from 表名 group by 字段名;

    - having只能在group by后用.

      select 字段名,group_concat(字段名2) from 表名 group by 字段名 having 条件;



### 外键约束

#### 新增

它强调的是一种约束关系,所以在新增外键约束前,首先要有该键,要有约束表.

- 已经有表.(先确认已有外键字段)

  alter table *被约束表* add foreign key(*外键字段*) references *约束表*(*主键*);

  alter table *students* add foreign key(*cls_id*) references *classes*(*id*);

-  新建表时即添加外键约束.建议写在所有字段后面.

  foreign key(*外键字段*) references *约束表*(*主键*);

- \*自定义外键约束名,不自定义外键约束名,系统默认 "表名_ibfk_1"

  Alter table 表名 add [constraint 外键约束名] foreign key (外键字段) references 父表(主键);

#### 删除

外键约束名不是外键字段名.外键约束名可以通过 show create table 表名; 查看.

只有取消外键约束,约束表才能被删除.

alter table 表名 drop foreign key 外键约束名;



### 索引

显示已有索引

show index from 表名;

#### 新增

alter table 表名 add index \[idx_字段名\](字段1,字段2...);

[]自定义字段名为可选项,如果不写,默认为字段名.

(多个字段)即为联合索引,遵循"最左原则"

#### 删除

alter table 表名 drop index 索引名;

# 高阶原理























