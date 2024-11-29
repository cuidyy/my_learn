# SQL基础知识总结笔记

## 一、SQL分类
1. **DDL（Data Definition Language）数据定义语言**
   - 用于定义数据库对象，如库、表、列等。
2. **DML（Data Manipulation Language）数据操作语言**
   - 用于操作数据库记录（数据）。
3. **DCL（Data Control Language）数据控制语言**
   - 用于定义访问权限和安全级别。
4. **DQL（Data Query Language）数据查询语言**
   - 用于查询记录（数据）。

## 二、DDL操作
1. **数据库操作**
   - **查看所有数据库**：`show databases;`
   - **切换数据库**：`use [数据库名]`，如`use mydb1`切换到mydb1数据库。
   - **创建数据库**：
     - `CREATE DATABASE [IF NOT EXISTS] [数据库名]`，如`CREATE DATABASE mydb1`创建名为mydb1的数据库，`CREATE DATABASE IF NOT EXISTS mydb1`避免创建已存在数据库时报错。
   - **删除数据库**：
     - `DROP DATABASE [IF EXISTS] [数据库名]`，如`DROP DATABASE mydb1`删除mydb1数据库，`DROP DATABASE IF EXISTS mydb1`避免删除不存在数据库时报错。
   - **修改数据库编码**：`ALTER DATABASE [数据库名] CHARACTER SET [编码名]`，如`ALTER DATABASE mydb1 CHARACTER SET utf8`（MySQL中UTF - 8需写为UTF8）。
2. **数据类型（MySQL中主要应用在列上）**
   - **常用类型**
     - `int`：整型。
     - `double`：浮点型，如`double(5,2)`表示最多5位，其中必须有2位小数，最大值为999.99。
     - `decimal`：用于表单线方面，不会出现精度缺失问题。
     - `char`：固定长度字符串类型（输入字符不够长度时补空格）。
     - `varchar`：可变长度字符串类型。
     - `text`：字符串类型。
     - `blob`：字节类型。
     - `date`：日期类型，格式为`yyyy-MM-dd`。
     - `time`：时间类型，格式为`hh:mm:ss`。
     - `timestamp`：时间戳类型。
3. **表操作**
   - **创建表**
     - 语法：`CREATE TABLE [表名]([列名] [列类型], [列名] [列类型],...... );`
     - 例如创建stu表：
```sql
CREATE TABLE stu(
  sid CHAR(6), 
  sname VARCHAR(20), 
  age INT, 
  gender VARCHAR(10)
);
```
   - **查看表结构**：`DESC [表名]`。
   - **删除表**：`DROP TABLE [表名]`。
   - **修改表**
     - **添加列**：`ALTER TABLE [表名] ADD ([列名] [列类型]);`，如给stu表添加classname列：`ALTER TABLE stu ADD (classname varchar(100));`
     - **修改列数据类型**：`ALTER TABLE [表名] MODIFY [列名] [新列类型];`，如修改stu表gender列类型为CHAR(2)：`ALTER TABLE stu MODIFY gender CHAR(2);`
     - **修改列名**：`ALTER TABLE [表名] CHANGE [旧列名] [新列名] [新列类型];`，如修改stu表gender列名为sex：`ALTER TABLE stu change gender sex CHAR(2);`
     - **删除列**：`ALTER TABLE [表名] DROP [列名];`，如删除stu表classname列：`ALTER TABLE stu DROP classname;`
     - **修改表名称**：`ALTER TABLE [表名] RENAME TO [新表名];`，如修改stu表名称为student：`ALTER TABLE stu Rename TO student;`

## 三、DML操作
1. **插入数据**
   - **语法1**：`INSERT INTO [表名]([列名1],[列名2],…) VALUES([值1],[值2],…);`
```sql
INSERT INTO stu(sid, sname,age,gender) VALUES('s_1001', 'zhangSan', 23, 'male');
INSERT INTO stu(sid, sname) VALUES('s_1001', 'zhangSan');
```
   - **语法2**：`INSERT INTO [表名] VALUES([值1],[值2],…);`（按创建表时列顺序插入所有列的值）
```sql
INSERT INTO stu VALUES('s_1002', 'liSi', 32, 'female');
```
   - 注意：所有字符串数据必须使用单引号。
2. **修改数据**
   - 语法：`UPDATE [表名] SET [列名1]=[值1], … [列名n]=[值n] [WHERE条件];`
```sql
UPDATE stu SET sname=’zhangSanSan’, age=’32’, gender=’female’ WHERE sid=’s_1001’;
UPDATE stu SET sname=’liSi’, age=’20’WHERE age>50 AND gender=’male’;
UPDATE stu SET sname=’wangWu’, age=’30’WHERE age>60 OR gender=’female’;
UPDATE stu SET gender=’female’WHERE gender IS NULL
UPDATE stu SET age=age+1 WHERE sname=’zhaoLiu’;
```
3. **删除数据**
   - **语法1**：`DELETE FROM [表名] [WHERE条件];`
```sql
DELETE FROM stu WHERE sid=’s_1001’;
DELETE FROM stu WHERE sname=’chenQi’ OR age > 30;
DELETE FROM stu;
```
   - **语法2**：`TRUNCATE TABLE [表名];`
   - 两者区别：
     - DELETE效率低于TRUNCATE。
     - TRUNCATE属于DDL语句，先DROP TABLE再CREATE TABLE，删除记录无法回滚；DELETE删除记录可回滚（涉及事务知识）。

## 四、DCL操作
1. **创建用户**
   - 语法：`CREATE USER ‘[用户名]’@[地址] IDENTIFIED BY '[密码]';`
```sql
CREATE USER ‘user1’@localhost IDENTIFIED BY ‘123’;
CREATE USER ‘user2’@’%’ IDENTIFIED BY ‘123’;
```
2. **给用户授权**
   - 语法：`GRANT [权限1], …, [权限n] ON [数据库名].* TO ‘[用户名]’@[地址];`
```sql
GRANT CREATE,ALTER,DROP,INSERT,UPDATE,DELETE,SELECT ON mydb1.* TO 'user1'@localhost;
GRANT ALL ON mydb1.* TO user2@localhost;
```
3. **撤销授权**
   - 语法：`REVOKE [权限1], …, [权限n] ON [数据库名].* FROM ‘[用户名]’@[地址];`
```sql
REVOKE CREATE,ALTER,DROP ON mydb1.* FROM 'user1'@localhost;
```
4. **查看用户权限**
   - 语法：`SHOW GRANTS FOR ‘[用户名]’@[地址];`
```sql
SHOW GRANTS FOR 'user1'@localhost;
```
5. **删除用户**
   - 语法：`DROP USER ‘[用户名]’@[地址];`
```sql
DROP USER ‘user1’@localhost;
```
6. **修改用户密码（以root身份）**
   - 语法：`use mysql; alter user '[用户名]'@localhost identified by '[新密码]';`

## 五、DQL操作
**语法结构**
   - `select [列名] ----> 要查询的列名称`
   - `from [表名] ----> 要查询的表名称`
   - `where [条件] ----> 行条件`
   - `group by [分组列] ----> 对结果分组`
   - `having [分组条件] ----> 分组后的行条件`
   - `order by [排序列] ----> 对结果分组`
   - `limit [起始行], [行数] ----> 结果限定`

1. **基础查询**
   - **查询所有列**：`SELECT * FROM [表名];`（`*`为通配符，表示所有列），如`SELECT * FROM stu;`
   - **查询指定列**：`SELECT [列名1], [列名2], …[列名n] FROM [表名];`，如`SELECT sid, sname, age FROM stu;`
2. **条件查询**
   - 条件查询在`WHERE`子句中使用运算符及关键字，如`=、!=、<>、<、<=、>、>=、BETWEEN…AND、IN(set)、IS NULL、AND、OR、NOT`。
   - 示例：
     - 查询性别为女且年龄小于50的记录：`SELECT * FROM stu WHERE gender='female' AND age<50;`
     - 查询学号为S_1001或姓名为liSi的记录：`SELECT * FROM stu WHERE sid ='S_1001' OR sname='liSi';`
     - 查询学号为S_1001、S_1002、S_1003的记录：`SELECT * FROM stu WHERE sid IN ('S_1001','S_1002','S_1003')`
     - 查询学号不是S_1001、S_1002、S_1003的记录：`SELECT * FROM stu WHERE sid NOT IN ('S_1001','S_1002','S_1003');`
     - 查询年龄为null的记录：`SELECT * FROM stu WHERE age IS NULL;`
     - 查询年龄在20到40之间的学生记录：`SELECT * FROM stu WHERE age>=20 AND age<=40;`或`SELECT * FROM stu WHERE age BETWEEN 20 AND 40;`
     - 查询性别非男的学生记录：`SELECT * FROM stu WHERE gender!='male';`或`SELECT * FROM stu WHERE gender<>'male';`或`SELECT * FROM stu WHERE NOT gender='male';`
     - 查询姓名不为null的学生记录：`SELECT * FROM stu WHERE NOT sname IS NULL;`或`SELECT * FROM stu WHERE sname IS NOT NULL;`
3. **模糊查询**
   - 语法：`SELECT [字段] FROM [表] WHERE [某字段] Like [条件]`，条件有两种匹配模式：
     - `%`：表示任意0个或多个字符（中文情况可能需用`%%`）。
     - `_`：表示任意单个字符。
   - 示例：
     - 查询姓名由5个字母构成的学生记录：`SELECT * FROM stu WHERE sname LIKE '_ _ _ _ _';`
     - 查询姓名由5个字母构成且第5个字母为“i”的学生记录：`SELECT * FROM stu WHERE sname LIKE '_ _ _ _i';`
     - 查询姓名以“z”开头的学生记录：`SELECT * FROM stu WHERE sname LIKE 'z%';`
     - 查询姓名中第2个字母为“i”的学生记录：`SELECT * FROM stu WHERE sname LIKE '_i%';`
     - 查询姓名中包含“a”字母的学生记录：`SELECT * FROM stu WHERE sname LIKE '%a%';`
4. **字段控制查询**
   - **去掉重复记录**：使用`DISTINCT`关键字，如`SELECT DISTINCT sal FROM emp;`
   - **列运算与别名**：
     - 若列类型为数值类型可进行运算，如`SELECT *, sal+comm FROM emp;`，若列中有`NULL`值相加结果可能为`NULL`，可使用`IFNULL`函数转换，如`SELECT *, sal+IFNULL(comm,0) FROM emp;`
     - 给列起别名，可使用`AS`关键字或省略，如`SELECT *, sal+IFNULL(comm,0) AS total FROM emp;`或`SELECT *, sal+IFNULL(comm,0) total FROM emp;`
5. **排序**
   - 语法：`SELECT * FROM [表名] ORDER BY [列名] [ASC/DESC];`（`ASC`升序，`DESC`降序，`ASC`可省略）。
   - 示例：
     - 查询所有学生记录按年龄升序排序：`SELECT * FROM stu ORDER BY age;`或`SELECT * FROM stu ORDER BY age ASC;`
     - 查询所有学生记录按年龄降序排序：`SELECT * FROM stu ORDER BY age DESC;`
     - 查询所有雇员按月薪降序排序，若月薪相同按编号升序排序：`SELECT * FROM emp ORDER BY sal DESC,empno ASC;`
6. **聚合函数**
   - `COUNT()`：统计指定列不为`NULL`的记录行数。
   - `MAX()`：计算指定列最大值（字符串类型按字符串排序运算）。
   - `MIN()`：计算指定列最小值（字符串类型按字符串排序运算）。
   - `SUM()`：计算指定列数值和（非数值类型计算结果为0）。
   - `AVG()`：计算指定列平均值（非数值类型计算结果为0）。
   - 示例：
     - 查询emp表中记录数：`SELECT COUNT(*) AS cnt FROM emp;`
     - 查询emp表中有佣金的人数：`SELECT COUNT(comm) cnt FROM emp;`
     - 查询emp表中月薪大于2500的人数：`SELECT COUNT(*) FROM emp WHERE sal > 2500;`
     - 统计月薪与佣金之和大于2500元的人数：`SELECT COUNT(*) AS cnt FROM emp WHERE sal+IFNULL(comm,0) > 2500;`
     - 查询有佣金的人数以及有领导的人数：`SELECT COUNT(comm), COUNT(mgr) FROM emp;`
     - 查询所有雇员月薪和：`SELECT SUM(sal) FROM emp;`
     - 查询所有雇员月薪和以及所有雇员佣金和：`SELECT SUM(sal), SUM(comm) FROM emp;`
     - 查询所有雇员月薪+佣金和：`SELECT SUM(sal+IFNULL(comm,0)) FROM emp;`
     - 统计所有员工平均工资：`SELECT SUM(sal), COUNT(sal) FROM emp;`或`SELECT AVG(sal) FROM emp;`
     - 查询最高工资和最低工资：`SELECT MAX(sal), MIN(sal) FROM emp;`
7. **分组查询**
   - 使用`GROUP BY`子句，如查询每个部门的工资和：`SELECT deptno, SUM(sal) FROM emp GROUP BY deptno;`
   - 可结合`WHERE`子句对分组前记录筛选，如查询每个部门工资大于1500的人数：`SELECT deptno, COUNT(*) FROM emp WHERE sal>1500 GROUP BY deptno;`
   - `HAVING`子句用于对分组后数据约束，如查询工资总和大于9000的部门编号以及工资和：`SELECT deptno, SUM(sal) FROM emp GROUP BY deptno HAVING SUM(sal) > 9000;`
8. **LIMIT**
   - 用于限定查询结果的起始行和总行数，语法：`SELECT * FROM [表名] LIMIT [起始行], [行数];`（起始行从0开始）。
   - 示例：
     - 查询5行记录，起始行从0开始：`SELECT * FROM emp LIMIT 0, 5;`
     - 查询10行记录，起始行从3开始：`SELECT * FROM emp LIMIT 3, 10;`
   - 分页查询：若一页记录为10条，查看第3页记录，起始行为20，查询10行，即`SELECT * FROM emp LIMIT 20, 10;`
9. **多表连接查询**
   - 表连接分为内连接和外连接，内连接仅选两张表中互相匹配记录，外连接会选其他不匹配记录。
   - **内连接**：语法如`select [表1.列名], [表2.列名] from [表1], [表2] where [表1.连接列]=[表2.连接列];`，例如查询员工表和职位表中匹配的记录：`select staff.name, deptname from staff, deptno where staff.name = deptno.name;`
   - **外连接**
     - **左连接**：包含左边表中所有记录，右边表中无匹配记录显示为`NULL`。语法如`select [表1.列名], [表2.列名] from [表1] left join [表2] on [表1.连接列]=[表2.连接列];`，例如`select staff.name, deptname from staff left join deptno on staff.name = deptno.name;`
     - **右连接**：包含右边表中所有记录，左边表中无匹配记录显示为`NULL`。语法如`select [表1.列名], [表2.列名] from [表1] right join [表2] on [表1.连接列]=[表2.连接列];`，例如`select deptname, deptno.name from staff right join deptno on deptno.name = staff.name;`