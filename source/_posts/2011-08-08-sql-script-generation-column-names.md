title: "SQL Script Generation Column Names"
date: 2011-08-8 12:00:00
categories: 
    - Development
tags:
    - SQL
---

When I’m doing heavy database development, one of the biggest annoyances I run into is having to hand key (or clumsily generate) the column names for stored procedures (especially those containing MERGE statements). I really wished I could use something to quickly output the columns to text so I could copy and paste the formatted column names as I needed them… well voila.

Our head DBA always has a few tricks up his sleeve, so we setup a system stored procedure to print out the column names (comma separated of course) for what ever table we want.

The PROC allows you to specify:
- Table Name (as it appears in the SYS.tables table)
- ‘L’ or ‘W’ depending if you want the columns listed with a {CRLF} after each column (‘L’) or all on one line (‘W)
- A ColumnName prefix (for doing table aliases)
- A table schema (if you have other table schema’s besides ‘dbo’)

Here’s the code (make sure to look at the bottom):
```sql
USE [master]
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[sp_listcolumns]
    @table  SYSNAME,
    @list   CHAR(1) = 'L',
    @prefix SYSNAME = '',
    @schema SYSNAME = 'dbo'
AS
BEGIN
SET NOCOUNT ON;
SET QUOTED_IDENTIFIER OFF

DECLARE @columnlist NVARCHAR(4000)
       ,@colctr INT = 1
       ,@sqlcmd NVARCHAR(500)

CREATE TABLE #column
(
    ColumnName NVARCHAR(256),
    Ordinal INT
)

CREATE UNIQUE CLUSTERED INDEX IC_ColumnList99 ON #column (Ordinal)

SET @columnlist =    ''
SET @table =        LTRIM(RTRIM(@table))
SET @list =         LTRIM(RTRIM(@list))
SET @prefix =       LTRIM(RTRIM(@prefix))
SET @schema =       LTRIM(RTRIM(@schema))

INSERT INTO #column (ColumnName, Ordinal)
SELECT  '['+LTRIM(RTRIM(COLUMN_NAME))+']', ORDINAL_POSITION
FROM    INFORMATION_SCHEMA.COLUMNS
WHERE   TABLE_NAME = @table AND
        TABLE_SCHEMA = @schema
ORDER BY ORDINAL_POSITION

-- Check to make sure we actually got usuable input from the User
IF ((SELECT COUNT(*) FROM #column) = 0)
RAISERROR('Bad Table Information - Please Try Again', 16, 1, 1)

IF(@prefix <> '')
BEGIN
    UPDATE #column
    SET    ColumnName = @prefix + ColumnName
END

IF(@list = 'L')
    BEGIN
        -- If it is 'L' just append a ',' to each line, output would be:
        -- TestCol1,
        -- TestCol2

        UPDATE  C
        SET     @columnlist = @columnlist + C.ColumnName + ',' + CHAR(10)
        FROM    #column C
    END
ELSE
    BEGIN
        -- else we output a 'w' where it just a ',' separated list of Column names
        UPDATE  C
        SET     @columnlist = @columnlist + C.ColumnName + ','
        FROM    #column C

        -- Remove the last comma
        SET @columnlist = LEFT(@columnlist,LEN(@columnlist) - 1)
    END

-- Output the results so that they can be nicely copy
-- and pasted from the comment window
PRINT(@columnlist)

SET QUOTED_IDENTIFIER ON
SET NOCOUNT OFF

END
```

Lastly we need to add this as a system proc, or else it never changes its scope to the DB you are running it in

```sql
EXECUTE sp_MS_marksystemobject 'sp_listcolumns'
```

After we have this proc build and added, we can get output like this:
```sql
CREATE TABLE dbo.Test
(
    Id UNIQUEIDENTIFIER
    ,TestCol1 NVARCHAR(64)
    ,TestCol2 INT
    ,TestCol3 BIT
)
GO

sp_listcolumns 'test'

--Outputs:
[Id],
[TestCol1],
[TestCol2],
[TestCol3],
--

sp_listcolumns 'test','w'

--Outputs:
[Id],[TestCol1],[TestCol2],[TestCol3]
--

sp_listcolumns 'test','l','testTable.'

--Outputs:
testTable.[Id],
testTable.[TestCol1],
testTable.[TestCol2],
testTable.[TestCol3],
--

sp_listcolumns 'test','w','testTable.'

--Outputs:
testTable.[Id],testTable.[TestCol1],testTable.[TestCol2],testTable.[TestCol3]
```