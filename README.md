# Azure SQL Managed Instance to Cloud SQL

Migrate from azure sql mi to cloud sql using .bacpac and convert it to .bak

In order to complete a migration from Azure SQL Managed Instance to Cloud SQL we have to perform two steps in order to obtain the backup file from Azure SQL MI databases. First of all in order to export a backup file from Azure SQL MI the only option is to get a bacpac and not a familiar .bak as needed for Cloud SQL. 
So, the first step is to obtain the bacpac file and then import it to SQL Server last edition running on GCE. 

```
As SQL bits are being upgraded regularly (every few months), Managed Instance is running on “evergreen” SQL bits. This means that current Managed Instance version is most of the time actually newer than the latest SQL Server version. As SQL Server backups are not backwards compatible, a backup from Managed Instance cannot be restored to e.g., SQL Server 2019. It's for the same reason as why backup from SQL Server 2019 cannot be restored to SQL Server 2017. 
```

In this article [Moving databases from Azure SQL Managed Instance to SQL Server](https://techcommunity.microsoft.com/t5/azure-sql-blog/moving-databases-from-azure-sql-managed-instance-to-sql-server/ba-p/2335297). Microsoft describes two main SQL technologies that can be used today for moving data from Managed Instance to SQL Server. A brief comparison of both technologies:
 
### Export/Import
 
   * Using export/import is very simple, either through SSMS or SqlPackage. It’s best to be used for offline migration of small and medium databases.
   * This technology has some limitations:
   * It does not handle databases that reference external objects (schemas with three or four-part names).
   * Its performance degrades when working with large databases or databases with large number of objects (hundreds of GBs or tens of thousands of objects in a database).
   * Produced BACPAC is not transactionally consistent.
### Transactional replication

This is a more robust way to migrate databases from Managed Instance to SQL Server. It’s best to be used for online or offline migration of large and complex databases. Some outstanding limitations that apply to transactional replication are: 

   * Tables must have primary keys.
   * Setup is very complicated. 
   * Synchronization is not trivial and can last days depending on the database size.
   * To find out more about this, see: how to migrate database from Managed Instance to SQL Server with transactional replication.
 
Since the purpose of the POC and this document is to test the migration from Azure SQL MI to Cloud SQL for non-production databases I used the first approach the Export/Import bacpac option.

## First step Import bacpac to SQL Server on GCE

And as a known issue documented by Microsoft, Exported database from Azure SQL failed to be imported to Azure SQL or to local SQL Server, it’s documented [here](https://techcommunity.microsoft.com/t5/azure-database-support-blog/exported-database-from-azure-sql-failed-to-be-imported-to-azure/ba-p/368871). Since SQL Server 2019 the “master key and encryption” is not a problem however the modification in model.xlm in the .bacpac is needed, so from option 3 from the article we should [Edit a .bacpac file](https://blogs.msdn.microsoft.com/azuresqldbsupport/2017/08/16/editing-a-bacpac-file/) 

So after editing the bacpac we should follow this [SQL Azure Tip article](https://techcommunity.microsoft.com/t5/azure-developer-community-blog/sql-azure-tip-failures-importing-bacpac/ba-p/335988) referenced by Microsoft: `The bacpac does contain a checksum that it uses to validate the package.  This needs to be updated.`  

In order to facilitate the process I’ve modified a powershell provided by the articles above and created a custom PS in order to automate the process. You can use [the RemoveKeyModelXML Powershell script to fix bacpac](https://github.com/xstabel/sqlmi-to-cloudsql/blob/main/RemoveKeyModelXML.ps1) 

So, run the PS and specigy the path to the original bacpac as a parameter


Then just upload the new patched bacpac to SQL Server:


Another approach to it, is to run the SqlPackage.exe and use the model.xml modified as an input value.


## Second step export .bak from SQL Server on GCE

Just create the .bak file from SQL Server, you can use the following command:

```
BACKUP DATABASE [YOUR-DATABASE-NAME] TO  DISK = N'PATH-TO-YOUR-FILE.bak' WITH NOFORMAT, NOINIT,  NAME = N'YOUR-DATABASENAME -Full Database Backup', SKIP, NOREWIND, NOUNLOAD,  STATS = 10
GO
```
## Third step import .bak to Cloud SQL
Upload the .bak to  Cloud Storage and then import .bak to Cloud SQL instance.

Explore your database from your SqlServer Managed Studio using the secured cloudsqlproxy:


