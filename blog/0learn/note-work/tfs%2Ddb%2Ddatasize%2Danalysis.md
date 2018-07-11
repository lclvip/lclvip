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
##  find out which tables are large

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