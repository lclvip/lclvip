## shows an increase of the tbl_Content over the last two months:

https://stackoverflow.com/questions/44158019/tfs2015-tbl-content-increase

```
select DATEPART(yyyy, CreationDate) as [year],
  DATEPART(mm, CreationDate) as [month],
  count(*) as [count],
  SUM(DATALENGTH(Content)) / 1048576.0 as [Size in Mb],
  (SUM(DATALENGTH(Content)) / 1048576.0) / count(*) as [Average Size]
from tbl_Content
group by DATEPART(yyyy, CreationDate),
    DATEPART(mm, CreationDate)
order by DATEPART(yyyy, CreationDate),
    DATEPART(mm, CreationDate)
```


## get all the tbl_content rows which have no matching in the tbl_FileMetadata

https://social.msdn.microsoft.com/Forums/en-US/fe992926-dd58-488e-9fd1-9862a5f6ebe8/tfs-2017-u1-dbotblcontent-growing-dramatically-rowssize?forum=tfsversioncontrol

```
SELECT A.[ResourceId]  
  FROM [Tfs_DefaultCollection].[dbo].[tbl_Content] As A
  left join [Tfs_DefaultCollection].[dbo].[tbl_FileMetadata] As B on A.ResourceId=B.ResourceId
  where B.[ResourceId] IS Null
```

## check which collection database has big size

```
use master
select DB_NAME(database_id) AS DBName, (size/128) SizeInMB
 FROM sys.master_files 
 where type=0  and substring(db_name(database_id),1,4)='Tfs_' and DB_NAME(database_id)<>'Tfs_Configuration' order by size desc
```

![image.png](/.attachments/image-bb562b5c-e1d1-42c6-add7-b450f1b1708e.png)

##  find out which tables are large

https://developercommunity.visualstudio.com/content/problem/63712/tfs-database-size.html

```
USE Tfs_CollectionName
CREATE TABLE #t 
( [name] NVARCHAR(128), [rows] CHAR(11), reserved VARCHAR(18), 
data VARCHAR(18), index_size VARCHAR(18), unused VARCHAR(18))
GO
INSERT #t
EXEC [sys].[sp_MSforeachtable] 'EXEC sp_spaceused "?"'
GO
SELECT
name as TableName,
Rows,
ROUND(CAST(REPLACE(reserved, ' KB', '') as float) / 1024,2) as ReservedMB,
ROUND(CAST(REPLACE(data, ' KB', '') as float) / 1024,2) as DataMB,
ROUND(CAST(REPLACE(index_size, ' KB', '') as float) / 1024,2) as IndexMB,
ROUND(CAST(REPLACE(unused, ' KB', '') as float) / 1024,2) as UnusedMB
FROM #t
ORDER BY CAST(REPLACE(reserved, ' KB','') as float) DESC
GO
Drop table #t
```

![image.png](/.attachments/image-bf72d987-42b5-47a7-a64b-05212fa58be2.png)


```
SELECT TOP 10 o.name, 
SUM(reserved_page_count) * 8.0 / 1024 SizeInMB,
SUM(CASE 
WHEN p.index_id <= 1 THEN p.row_count
ELSE 0
END) Row_Count
FROM sys.dm_db_partition_stats p
JOIN sys.objects o
ON p.object_id = o.object_id
GROUP BY o.name
ORDER BY SUM(reserved_page_count) DESC
```

## look at the distribution of "owners" for the data in tbl_Content.
```
SELECT Owner = 
CASE
WHEN OwnerId = 0 THEN 'Generic' 
WHEN OwnerId = 1 THEN 'VersionControl'
WHEN OwnerId = 2 THEN 'WorkItemTracking'
WHEN OwnerId = 3 THEN 'TeamBuild'
WHEN OwnerId = 4 THEN 'TeamTest'
WHEN OwnerId = 5 THEN 'Servicing'
WHEN OwnerId = 6 THEN 'UnitTest'
WHEN OwnerId = 7 THEN 'WebAccess'
WHEN OwnerId = 8 THEN 'ProcessTemplate'
WHEN OwnerId = 9 THEN 'StrongBox'
WHEN OwnerId = 10 THEN 'FileContainer'
WHEN OwnerId = 11 THEN 'CodeSense'
WHEN OwnerId = 12 THEN 'Profile'
WHEN OwnerId = 13 THEN 'Aad'
WHEN OwnerId = 14 THEN 'Gallery'
WHEN OwnerId = 15 THEN 'BlobStore'
WHEN OwnerId = 255 THEN 'PendingDeletion'
END,
SUM(CompressedLength) / 1024.0 / 1024.0 AS BlobSizeInMB
FROM tbl_FileReference AS r
JOIN tbl_FileMetadata AS m
ON r.ResourceId = m.ResourceId
AND r.PartitionId = m.PartitionId
WHERE r.PartitionId = 1
GROUP BY OwnerId
ORDER BY 2 DESC
```

## the FileContainer space
```
SELECT CASE WHEN Container = 'vstfs:///Buil' THEN 'Build'
WHEN Container = 'vstfs:///Git/' THEN 'Git'
WHEN Container = 'vstfs:///Dist' THEN 'DistributedTask'
ELSE Container 
END AS FileContainerOwner,
SUM(fm.CompressedLength) / 1024.0 / 1024.0 AS TotalSizeInMB
FROM (SELECT DISTINCT LEFT(c.ArtifactUri, 13) AS Container,
fr.ResourceId,
ci.PartitionId
FROM tbl_Container c
INNER JOIN tbl_ContainerItem ci
ON c.ContainerId = ci.ContainerId
AND c.PartitionId = ci.PartitionId
INNER JOIN tbl_FileReference fr
ON ci.fileId = fr.fileId
AND ci.DataspaceId = fr.DataspaceId
AND ci.PartitionId = fr.PartitionId) c
INNER JOIN tbl_FileMetadata fm
ON fm.ResourceId = c.ResourceId
AND fm.PartitionId = c.PartitionId
GROUP BY c.Container
ORDER BY TotalSizeInMB DESC
```

## clear at once 

`exec prc_DeleteUnusedContent`

```
use master 
go 
dbcc shrinkdatabase (database_name,10) 
go
```

-- 查询各个磁盘分区的剩余空间：
`Exec master.dbo.xp_fixeddrives`

-- 查询数据库的数据文件及日志文件的相关信息（包括文件组、当前文件大小、文件最大值、文件增长设置、文件逻辑名、文件路径等）
`select * from [数据库名].[dbo].[sysfiles]`


-- 转换文件大小单位为MB：
`select name, convert(float,size) * (8192.0/1024.0)/1024. from [数据库名].dbo.sysfiles `


-- 查询当前数据库的磁盘使用情况：
`Exec sp_spaceused`

-- 查询数据库服务器各数据库日志文件的大小及利用率
`DBCC SQLPERF(LOGSPACE)`
