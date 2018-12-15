Ubuntu 18.04下 Postgresql 安装及配置



## 下载安装



1. 创建文件`/etc/apt/sources.list.d/pgdg.list`, 添加以下源：

   ```shell
   deb http://apt.postgresql.org/pub/repos/apt/ bionic-pgdg main
   ```


2. 添加仓库签名key，执行以下命令：

   ```shell
   wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
   sudo apt-get update
   ```


3. 安装`postgresql-10`：

   ```shell
   sudo apt-get install postgresql-10
   ```



   安装成功后，默认会：

   - 创建名为`postgres`的系统用户

   - 创建名为`postgres`且不带密码的默认数据库账号作为数据库管理员

   - 创建名为`postgres`的表


默认情况下，Postgres使用称为“角色”的概念来处理身份验证和授权。这些在某些方面类似于普通的Unix风格的账户，但是Postgres并没有区分用户和组，而是倾向于更灵活的术语“角色”。

安装后，Postgres被设置为使用*ident身份*验证，这意味着它将Postgres角色与匹配的Unix / Linux系统帐户相关联。如果Postgres中存在一个角色，则具有相同名称的Unix / Linux用户名可以作为该角色登录。

安装过程创建了一个名为**postgres**的用户帐户，它与默认的Postgres角色相关联。为了使用Postgres，您可以登录到该帐户。



## 配置



1. 修改`postgres`数据库用户的密码

   使用postgres用户登录：

   ```shell
   sudo -u postgres psql
   ```

   更改密码：

   ```shell
   alter user postgres with password '123456';
   ```

   使用`\q`退出。


2. 修改postgres系统用户的密码

   切换到root用户

   ```shell
   su root
   ```

   使用`passwd -d` 清空指定postgres用户的密码

   ```shell
   passwd -d postgres
   ```



3. 配置远程访问

   ```shell
   vim /etc/postgresql/10/main/postgresql.conf
   ```

   将`#listen_addresses = 'localhost'`改为`listen_addresses = '*'`



4. 重启服务

   ```shell
   /etc/init.d/postgresql restart
   ```



5. 退出root


## 登录及管理数据库



1. 登录

   如果当前系统用户不是postgres的话，使用以下命令登录：

   ```shell
   psql -U postgres -h 127.0.0.1
   ```

   如果当前登录用户是`postgres`的话，可以直接使用`psql`登录：

   ```shell
   psql
   ```

   也可以通过`psql`像**postgres**用户`sudo`一样运行单个命令来完成此操作，如下所示：

   ```shell
   sudo -u postgres psql
   ```

   这样会直接登录到Postgres中。

2. 创建角色

   如果以**postgres**账户登录，可以通过以下方式创建：

   ```shell
   createuser --interactive
   ```

   > `--interactive`标志会提示您输入新角色的名称，并询问它是否应具有超级用户权限。

   如果当前系统用户不是postgres的话，可以通过以下方式：

   ```shell
   sudo -u postgres createuser --interactive
   ```

   如果已经登录到数据库后，可以使用以下方式：

   ```shell
   create user xxx with password '123456' nocreatedb;
   ```

3. 建立数据库

   Postgres身份验证系统默认情况下的另一个假设是，对于用于登录的任何角色，该角色将具有可访问的同名数据库。

   如果您以**postgres**帐户登录，则可以键入如下所示的内容：

   ```shell
   createdb mydb
   ```

   如果不是**postgres**账户登录，则可以键入：

   ```shell
   sudo -u postgres createdb mydb
   ```

   在postgresql的命令行，可以使用以下方式：

   ```
   create database dbname with owner xxx;
   ```

4. 使用新创建的账户登录

   如果之前创建的用户名跟数据库名一样，可以使用以下方式登录：

   ```shell
   psql -U xxx -h 127.0.0.1
   ```

   如果名称不一样，需要使用`-d`指定登录的数据库名：

   ```shell
   psql -U xxx -d dbname -h 127.0.0.1
   ```


## 表操作



### 创建表

创建表的基本语法如下：

```sql
create table table_name (
	column_name1 col_type (field_length) column_constraints, 
	column_name2 col_type (field_length), 
	column_name3 col_type (field_length)
);
```

比如创建一个`tb_user`表:

```sql
create table tb_user(
	id serial primary key, 
	name varchar(50), 
	age int,
    location varchar(25) check (location in ('Beijing', 'Shanghai', 'Guangzhou')), 
    create_date date
);
```

`serial`表示该列是自动递增整数，约束是`primary key`，表示该列的值是唯一且不是空的。

使用`\d`查看当前所有表：

```shell
testdb=> \d
                   关联列表
 架构模式 |      名称      |  类型  | 拥有者  
----------+----------------+--------+---------
 public   | tb_user        | 数据表 | wangzhf
 public   | tb_user_id_seq | 序列数 | wangzhf
(2 行记录)

testdb=> 
```



### 添加

```sql
insert into tb_user (name, age, location, create_date) values ('zhangsan', 20, 'Beijing', '2018-12-15');
insert into tb_user (name, age, location, create_date) values ('lisi', 25, 'Shanghai', '2018-12-15');
```

- 对于id字段，因为它是`serial`类型，所以不需要插入；
- 对于`location`字段，由于它被限定为'Beijing', 'Shanghai', 'Guangzhou'中的一个，如果不是这三项将会出错：

```sql
testdb=> insert into tb_user (name, age, location, create_date) values ('lisi', 25, 'Shenzhen', '2018-12-15');
错误:  关系 "tb_user" 的新列违反了检查约束 "tb_user_location_check"
描述:  失败, 行包含(3, lisi, 25, Shenzhen, 2018-12-15).
testdb=> 
```



### 查询

```sql
testdb=> select * from tb_user;
 id |   name   | age | location | create_date 
----+----------+-----+----------+-------------
  1 | zhangsan |  20 | Beijing  | 2018-12-15
  2 | lisi     |  25 | Shanghai | 2018-12-15
(2 行记录)

testdb=>
```



### 删除

```sql
testdb=> delete from tb_user where name = 'lisi';
DELETE 1
testdb=> select * from tb_user;
 id |   name   | age | location | create_date 
----+----------+-----+----------+-------------
  1 | zhangsan |  20 | Beijing  | 2018-12-15
(1 行记录)

testdb=>
```



### 修改表结构

1. 新增列

   ```sql
   testdb=> alter table tb_user add column update_date date;
   ALTER TABLE
   testdb=> select * from tb_user;
    id |   name   | age | location | create_date | update_date 
   ----+----------+-----+----------+-------------+-------------
     1 | zhangsan |  20 | Beijing  | 2018-12-15  | 
   (1 行记录)
   
   testdb=>
   ```

   我们新增一个类型为`date`的`update_date`字段，并重新查询，可以看到字段已经显示。



2. 修改列

   修改上述列`update_date`为`update_time`：

   ```sql
   testdb=> alter table tb_user rename update_date to update_time;
   ALTER TABLE
   testdb=> select * from tb_user;
    id |   name   | age | location | create_date | update_time 
   ----+----------+-----+----------+-------------+-------------
     1 | zhangsan |  20 | Beijing  | 2018-12-15  | 
   (1 行记录)
   
   testdb=>
   ```

   改变`update_time`类型为`varchar`：

   ```sql
   testdb=> alter table tb_user alter update_time type varchar(20);
   ALTER TABLE
   testdb=> select * from tb_user;
    id |   name   | age | location | create_date | update_time 
   ----+----------+-----+----------+-------------+-------------
     1 | zhangsan |  20 | Beijing  | 2018-12-15  | 
   (1 行记录)
   
   testdb=>
   ```

   给`update_time`列设置默认值：

   ```sql
   testdb=> alter table tb_user alter update_time set default '2018-12-15';
   ALTER TABLE
   testdb=> select * from tb_user;
    id |   name   | age | location | create_date | update_time 
   ----+----------+-----+----------+-------------+-------------
     1 | zhangsan |  20 | Beijing  | 2018-12-15  | 
   (1 行记录)
   
   testdb=> insert into tb_user (name, age, location, create_date) values ('wangwu', 30, 'Guangzhou', '2018-12-15');
   INSERT 0 1
   testdb=> select * from tb_user;
    id |   name   | age | location  | create_date | update_time 
   ----+----------+-----+-----------+-------------+-------------
     1 | zhangsan |  20 | Beijing   | 2018-12-15  | 
     4 | wangwu   |  30 | Guangzhou | 2018-12-15  | 2018-12-15
   (2 行记录)
   
   testdb=>
   ```

   可以看到新插入的记录的`update_time`字段默认值已经设置。

   要设置`update_time`的**NOT NULL**约束，可以使用以下语法：

   ```sql
   alter table table_name alter column_name {set|drop} not null;
   ```

   > set表示设置，drop表示删除

   由于表中`update_time`列有空值，如果直接设置的话，将会报错：

   ```sql
   testdb=> alter table tb_user alter update_time set not null;
   错误:  字段 "update_time" 包含空值
   testdb=>
   ```

   所以我们需要先设置一个值然后再设置：

   ```sql
   testdb=> update tb_user set update_time = '2018-12-10' where id = 1;
   UPDATE 1
   testdb=> select * from tb_user;
    id |   name   | age | location  | create_date | update_time 
   ----+----------+-----+-----------+-------------+-------------
     4 | wangwu   |  30 | Guangzhou | 2018-12-15  | 2018-12-15
     1 | zhangsan |  20 | Beijing   | 2018-12-15  | 2018-12-10
   (2 行记录)
   
   testdb=> alter table tb_user alter update_time set not null;
   ALTER TABLE
   testdb=> insert into tb_user (name, age, location, create_date, update_time) values ('zhaoliu', 22, 'Guangzhou', '2018-12-15', null);
   错误:  在字段 "update_time" 中空值违反了非空约束
   描述:  失败, 行包含(5, zhaoliu, 22, Guangzhou, 2018-12-15, null).
   testdb=>
   ```

   可以看到设置`update_time`为非空后，如果再插入空值，将会报错。

   然后将非空约束删除：

   ```sql
   testdb=> alter table tb_user alter update_time drop not null;
   ALTER TABLE
   testdb=> insert into tb_user (name, age, location, create_date, update_time) values ('zhaoliu', 22, 'Guangzhou', '2018-12-15', null);
   INSERT 0 1
   testdb=> select * from tb_user;
    id |   name   | age | location  | create_date | update_time 
   ----+----------+-----+-----------+-------------+-------------
     4 | wangwu   |  30 | Guangzhou | 2018-12-15  | 2018-12-15
     1 | zhangsan |  20 | Beijing   | 2018-12-15  | 2018-12-10
     6 | zhaoliu  |  22 | Guangzhou | 2018-12-15  | 
   (3 行记录)
   
   testdb=>
   ```

   删除列：

   ```sql
   testdb=> alter table tb_user drop update_time;
   ALTER TABLE
   testdb=> select * from tb_user;
    id |   name   | age | location  | create_date 
   ----+----------+-----+-----------+-------------
     4 | wangwu   |  30 | Guangzhou | 2018-12-15
     1 | zhangsan |  20 | Beijing   | 2018-12-15
     6 | zhaoliu  |  22 | Guangzhou | 2018-12-15
   (3 行记录)
   
   testdb=>
   ```



## pgAdmin

前面我们都是采用命令行的形式来操作表，我们还是需要一个图形化的界面来方便操作。`pgAdmin`是Postgresql官方出品的一个图形化管理工具（基于浏览器的web页面），我们可以从这里下载：[pgAdmin](https://www.pgadmin.org/download/) 



### 安装

参考：https://wiki.postgresql.org/wiki/Apt



## 控制台命令

```shell
一般性
  \copyright            显示PostgreSQL的使用和发行许可条款
  \crosstabview [COLUMNS] 执行查询并且以交叉表显示结果
  \errverbose            以最冗长的形式显示最近的错误消息
  \g [文件] or;     执行查询 (并把结果写入文件或 |管道)
  \gexec                 执行策略，然后执行其结果中的每个值
  \gset [PREFIX]     执行查询并把结果存到psql变量中
  \gx [FILE]             as \g, but forces expanded output mode
  \q             退出 psql
  \watch [SEC]          每隔SEC秒执行一次查询

帮助
  \? [commands]          显示反斜线命令的帮助
  \? options             显示 psql 命令行选项的帮助
  \? variables           显示特殊变量的帮助
  \h [名称]          SQL命令语法上的说明，用*显示全部命令的语法说明

查询缓存区
  \e [FILE] [LINE]        使用外部编辑器编辑查询缓存区(或文件)
  \ef [FUNCNAME [LINE]]   使用外部编辑器编辑函数定义
  \ev [VIEWNAME [LINE]]  用外部编辑器编辑视图定义
  \p                    显示查询缓存区的内容
  \r                    重置(清除)查询缓存区
  \s [文件]        显示历史记录或将历史记录保存在文件中
  \w 文件          将查询缓存区的内容写入文件

输入/输出
  \copy ...             执行 SQL COPY，将数据流发送到客户端主机
  \echo [字符串]       将字符串写到标准输出
  \i 文件          从文件中执行命令
  \ir FILE               与 \i类似, 但是相对于当前脚本的位置
  \o [文件]        将全部查询结果写入文件或 |管道
  \qecho [字符串]      将字符串写到查询输出串流(参考 \o)

Conditional
  \if EXPR               begin conditional block
  \elif EXPR             alternative within current conditional block
  \else                  final alternative within current conditional block
  \endif                 end conditional block

资讯性
  (选项: S = 显示系统对象, + = 其余的详细信息)
  \d[S+]          列出表,视图和序列
  \d[S+]  名称      描述表，视图，序列，或索引
  \da[S]  [模式]    列出聚合函数
  \dA[+]  [PATTERN]      list access methods
  \db[+]  [模式]     列出表空间
  \dc[S+] [PATTERN]      列表转换
  \dC[+]  [PATTERN]      列出类型强制转换
  \dd[S]  [PATTERN]      显示没有在别处显示的对象描述
  \dD[S+] [PATTERN]      列出共同值域
  \ddp     [模式]    列出默认权限
  \dE[S+] [PATTERN]      列出引用表
  \det[+] [PATTERN]      列出引用表
  \des[+] [模式]    列出外部服务器
  \deu[+] [模式]     列出用户映射
  \dew[+] [模式]       列出外部数据封装器
  \df[antw][S+] [模式]    列出[只包括 聚合/常规/触发器/窗口]函数 
  \dF[+]  [模式]   列出文本搜索配置
  \dFd[+] [模式]     列出文本搜索字典
  \dFp[+] [模式]     列出文本搜索解析器
  \dFt[+] [模式]   列出文本搜索模版
  \dg[S+] [PATTERN]      列出角色
  \di[S+] [模式]  列出索引
  \dl                   列出大对象， 功能与\lo_list相同
  \dL[S+] [PATTERN]      列出所有过程语言
  \dm[S+] [PATTERN]      列出所有物化视图
  \dn[S+] [PATTERN]     列出所有模式
  \do[S]  [模式]   列出运算符
  \dO[S+] [PATTERN]      列出所有校对规则
  \dp     [模式]     列出表，视图和序列的访问权限
  \drds [模式1 [模式2]] 列出每个数据库的角色设置
  \dRp[+] [PATTERN]      list replication publications
  \dRs[+] [PATTERN]      list replication subscriptions
  \ds[S+] [模式]    列出序列
  \dt[S+] [模式]     列出表
  \dT[S+] [模式]  列出数据类型
  \du[S+] [PATTERN]      列出角色
  \dv[S+] [模式]   列出视图
  \dx[+]  [PATTERN]      列出扩展
  \dy     [PATTERN]      列出所有事件触发器
  \l[+]   [PATTERN]      列出所有数据库
  \sf[+]  FUNCNAME       显示一个函数的定义
  \sv[+]  VIEWNAME       显示一个视图的定义

格式化
  \a                  在非对齐模式和对齐模式之间切换
  \C [字符串]        设置表的标题，或如果没有的标题就取消
  \f [字符串]         显示或设定非对齐模式查询输出的字段分隔符
  \H                    切换HTML输出模式 (目前是 关闭)
  \pset [NAME [VALUE]]   set table output option
                         (NAME := {border|columns|expanded|fieldsep|fieldsep_zero|
                         footer|format|linestyle|null|numericlocale|pager|
                         pager_min_lines|recordsep|recordsep_zero|tableattr|title|
                         tuples_only|unicode_border_linestyle|
                         unicode_column_linestyle|unicode_header_linestyle})
  \t [开|关]       只显示记录 (目前是 关闭)
  \T [字符串]         设置HTML <表格>标签属性, 或者如果没有的话取消设置
  \x [on|off|auto]       切换扩展输出模式(目前是 关闭)

连接
  \c[onnect] {[DBNAME|- USER|- HOST|- PORT|-] | conninfo}
                         连接到新数据库（当前是"testdb"）
  \conninfo              显示当前连接的相关信息
  \encoding [编码名称] 显示或设定客户端编码
  \password [USERNAME]  安全地为用户更改口令

操作系统
  \cd [目录]     更改目前的工作目录
  \setenv NAME [VALUE]   设置或清空环境变量
  \timing [开|关]       切换命令计时开关 (目前是 关闭)
  \! [命令]      在 shell中执行命令或启动一个交互式shell

变量
  \prompt [文本] 名称 提示用户设定内部变量
  \set [名称 [值数]] 设定内部变量，若无参数则列出全部变量
  \unset 名称    清空(删除)内部变量

大对象
  \lo_export LOBOID 文件
  \lo_import 文件 [注释]
  \lo_list
  \lo_unlink LOBOID   大对象运算

```

