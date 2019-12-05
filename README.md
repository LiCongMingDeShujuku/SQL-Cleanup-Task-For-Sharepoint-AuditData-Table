![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# Sharepoint AuditData表的SQL清理任务
#### SQL Cleanup Task For Sharepoint AuditData Table

![#](images/##############?raw=true "#")

## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
某些Sharepoint WSS_Content数据库会有一个名为[AuditData]的大型表。这是Sharepoint环境的访问时间的基本记录。大多数这些数据都是良性的，很少被人看到，并且只需要一点点的Sharepoint managmenet，这些记录可以定期维护，但通常在大多数环境中，我们都忽视这些小细节。只有在很长一段时间后，这些日志才会错乱。在这种情况下，会出现数百万行的错乱。幸运的是桌面上不会有任何外键（截至本文的写作），因此维护是非常必须和基础的工作。

在这种情况下，我创建了一个SQL清理任务来删除一些旧的[AuditData]条目。它可以最长保持91天。为什么是91天？因为它将保留所有前90天（加上明天）的信息，这将保证在清理任务运行时添加的任何新行都将被保存。

注意：这是一个批量处理过程。目前它设置为一次只删除10行。这当然是可以调整，但按照我的方式做就会是一个非常简单的操作。


## English
Some Sharepoint WSS_Content databases will have a substantially large table called [AuditData]. This is a basic record of access times for the Sharepoint environment. Most of this data is benign and rarely looked at, and with alittle bit of Sharepoint managmenet these records can be regularly maintained, but often with most environments; the little things go unseen. It’s only after a long period of time that these logs can get out of hand. In this case millions of rows. Fortunately there aren’t any foreign keys on the table (as of the writing on this post) so maintenance can be pretty basic.

In this case I created an SQL Cleanup Task to remove some of the old [AuditData] entries. This keeps the most recent 91 days. Why 91 days? Cause it will keep all previous 90 days (plus tomorrow) this will guarantee any new rows added while the cleanup task is running will be saved.

Note: This is a batch process. Presently it’s set to only delete 10 rows at a time. This of course can be adjusted, but I have it this way so the task it’s self is an extremely minimal operation.

---
## Logic
```SQL
use My_WSS_CONTENT_DATABASE;
set nocount on
 
declare @start_date datetime
declare @end_date   datetime
set @start_date = (select dateadd(d, +1, getdate()))
set @end_date   = (select dateadd(d, -90, getdate()))
 
declare @cleanup_task   varchar(max)
set @cleanup_task   =
'
set nocount on
while exists ( select * from [AuditData] where [occurred] not between ''' + CONVERT(nvarchar(24), @start_date, 121) + ''' and ''' + CONVERT(nvarchar(24), @end_date, 121) + ''')
    begin
        begin tran spauditmaint
        delete top (10) from [AuditData] where [occurred] not between ''' + CONVERT(nvarchar(24), @start_date, 121) + ''' and ''' + CONVERT(nvarchar(24), @end_date, 121) + '''
        commit tran spauditmaint
        checkpoint;
        print ''10 rows deleted''
    end
'
exec (@cleanup_task)
```
如果你想要覆盖所有内容数据库，你可以尝试这个。
每个内容数据库在其AuditData表下很容易有数百万行，因此一次批量删除500行是最好的，因为系统可以更好地消化删除。每个删除都有一个提交和检查点，因此更容易管理，但这个过程确实需要一段时间，也许是几天。因此，我将“之间”设置为在过去90天和未来7天之间。这将使你有时间在各种数据库中完成。另外，确保定期进行缩减操作，在事务日志备份中进行上升会更有帮助，或者在执行此过程时将数据库设置为SIMPLE恢复。

If you’re looking to hit ALL content databases you can try this one.
Each content database could easily have millions of rows under it’s AuditData tables so doing batch deletes of 500 rows at a time is good as the system can better digest the deletes. Each delete has a commit, and checkpoint so it’s easier to manage, but the process does take a while. Days perhaps. Because of this I set the ‘between’ days to look between 90 days past, and 7 days into the future. This will give your process time to complete among a variety of databases. Again; be sure to run periodic shrink operations, and it would be helpful to have an uptick in your transaction log backups, or better yet; setting your database to SIMPLE recovery while this process is carried out.

```SQL
use master;
set nocount on
 
declare @clean_process  varchar(max)
set @clean_process  = ''
select  @clean_process  = @clean_process +
'use [' + [name] + '];
set nocount on
 
declare @start_date_' + cast([database_id] as varchar) + '  datetime
declare @end_date_' + cast([database_id] as varchar) + '    datetime
set @start_date_' + cast([database_id] as varchar) + '  = (select dateadd(d, +7,    getdate()))
set @end_date_' + cast([database_id] as varchar) + '    = (select dateadd(d, -90,   getdate()))
 
declare @cleanup_task_' + cast([database_id] as varchar) + '    varchar(max)
set @cleanup_task_' + cast([database_id] as varchar) + '    =
''
set nocount on
while exists ( select * from [AuditData] where [occurred] not between '''''' + CONVERT(nvarchar(24), @start_date_' + cast([database_id] as varchar) + ', 121) + '''''' and '''''' + CONVERT(nvarchar(24), @end_date_' + cast([database_id] as varchar) + ', 121) + '''''')
    begin
        begin tran spauditmaint
        delete top (500) from [AuditData] where [occurred] not between '''''' + CONVERT(nvarchar(24), @start_date_' + cast([database_id] as varchar) + ', 121) + '''''' and '''''' + CONVERT(nvarchar(24), @end_date_' + cast([database_id] as varchar) + ', 121) + ''''''
        commit tran spauditmaint
        checkpoint;
        print ''''500 rows deleted under log table for database [' + upper([name]) + ']''''
    end
''
exec (@cleanup_task_' + cast([database_id] as varchar) + ')
' + char(10) + char(10)
from    sys.databases where [name] like '%content%'
order by [name] desc
 
exec (@clean_process) --for xml path (''), type

```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

