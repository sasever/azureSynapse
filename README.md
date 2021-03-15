# Azure Synapse SQL Pools

In this example we will be using New York Taxi dataset.
Before creating the Azure Synapse Workspace we will prepare our data.

Please follow below steps:

1 . Create a Resource Group to host the Azure Synapse Workspace. You may chose to deploy another one for Azure Data Lake.

1. We need to create an [Azure Data Lake Gen2 storage](https://docs.microsoft.com/en-us/azure/storage/blobs/create-data-lake-storage-account). We will create the storage first as publicly accessible. After copying required data, we can close the public access.

1. We will copy the data to the new storage. You can use storage explorer or azcopy utility from cloud shell for this purpose.

    * We will use below storage accounts to copy the data:
        * CSV example data: https://nytaxiblob.blob.core.windows.net/2013/
        * Parquet example data:https://azureopendatastorage.blob.core.windows.net/nyctlc/

    * [How to install Azure Storage Explorer](https://docs.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer?tabs=windows#connect-to-a-storage-account-or-service) After installing storage explorer you can login to your azure account and also define public storage accounts as well.

    * To use azcopy via cloudshell, O
        * Open a cloud shell from azure portal.
        * Then authenticate with azcopy:

            ```command
            azcopy login
            ```

            This will respond with a device login URL and code. Click on URL and use the code to authenticate further. If this does not work you can try with  your AD tenant id:

            ```command
            azcopy login --tenant-id=<tenant-id>
            ```

        * Use below commands by changing the target storage URL and container information, to copy NY Taxi data sample parquet files and sample csv files respectively. If you receive error in any below, please use STORAGE EXPLORER to copy those items.

        ```command
        azcopy copy "https://azureopendatastorage.blob.core.windows.net/nyctlc/yellow/*" "https://<your-adls>.blob.core.windows.net/<your-dfs-name>/nyttripdataparquet/yellow/*" --overwrite=prompt --s2s-preserve-access-tier=false --recursive;
        ```

        ```command
        azcopy copy "https://nytaxiblob.blob.core.windows.net/2013/*" "https://<your-adls>.dfs.core.windows.net/<your-dfs-name>/nytcsvdata/" --overwrite=true --s2s-preserve-access-tier=false --recursive;
        ```

Once the data is copied we can be ready for creating the workspace. To be able to use private link to access storages in Azure via Azure Synapse, We need to provision a Managed V-NET using Azure Synapse Workspace.

## Azure Synapse Workspace Managed Virtual Network and Private Endpoints

**Please follow below steps:**

1. Azure Synapse requires  Managed Virtual Network configuration to be able to use Private Endpoints to access sources.
Use [this guidance](https://docs.microsoft.com/en-us/azure/synapse-analytics/security/synapse-workspace-managed-vnet#create-an-azure-synapse-workspace-with-a-managed-workspace-virtual-network) to deploy a synapse workspace from the portal which resides in a managed vnet ito be able to use private link.


1. The user who created the Workspace will require to grant contributor role to their colleagues that they are collaborating with, to let them also access the repository.  to do that please follow [this guidance](https://docs.microsoft.com/en-us/azure/synapse-analytics/security/how-to-manage-synapse-rbac-role-assignments)

1. Once everyone is able to log in and see the synapse environment, we can start defining private link configurations for our sources with using [this guidance](https://docs.microsoft.com/en-us/azure/synapse-analytics/security/how-to-create-managed-private-endpoints). When you create the private link it will wait in pending state requiring approval.

1. To approve the managed Private Endpoint request:    
    * Go directly to the Azure portal Storage Account and go into the Private endpoint connections blade.
    * Tick the Private endpoint you created in the Studio and select Approve.
    * Add a description and select yes
    * Go back to Synapse Studio in under the Managed Virtual Networks section of the Manage tab.

    It should take around 1 minute to get the approval reflected for your private endpoint.

1.To help in further process, we will also enable "interactive authoring" capability. To do that on the most left blade we need to go to MANAGE>Integration runtimes and click on AutoResolveIntegrationRuntime. From the menu that opens, Select **Enable** for **Interactive authoring**.

1. For the Storage that we  created a managed private link, We will create a linked server. To do this,
    * On the Storage Account in access control we need to grant **Storage Blob Contributor** role to the Synapse Workspace's Managed Identity. Go to Grant access to this resource>Add role assignment, Select Role: Storage Blob Contributor, Assign Access to: User,group, or service principal. Write the  Managed Identity Name of the Workspace(name of the workspace) to select box. The identity will show itself below. select that, and click save, which should have been activated after selecting. As an alternative you can grant the same rights from storage explorer.

    * On the Data Section, we will click on **+** from the opening menu under **Linked** Click on **Connect to external data**.
    * Select the storage technology, in our case it should be **Azure Data Lake Storage Gen2**, and click continue.
    * Provide a name for the new linked Service. *here you should see **Interactive authoring enabled** with a green tick*

    * From **Authentication method** select Managed Identity.

    * Select the subscription where the storage account is, and the name of the storage account *The managed private endpoint should be found automatically.*

1. Now we  create a dedicated SQL pool. You can do that by following [this guidance](https://docs.microsoft.com/en-us/azure/synapse-analytics/quickstart-create-sql-pool-portal). We suggest to have a minimum DW300c to max DW1000c Dedicated SQL Pool.

1. Now we  create a new database other than MASTER in Serverless SQL pool. Create an Empty SQL script and Select Built in from the drop down list. Execute below script by giving database a name: 
```sql
CREATE DATABASE  <DATABASE_NAME>
```

## How to access Azure Synapse SQL environments
You can use various clients to work with data through Azure Synapse.

1. Azure Synapse Workspace
1. Azure Data Studio
1. SQL Sever Management Services (SSMS)

You can use all at the same time or prefer one. To discover more on the Workspace Capabilities, We suggest moving on with the **Azure Synapse Workspace**. To use the other two, you can get the access URLs from Azure Portal>Your Synapse Resource> Overview section respectively..

## How to  explore data in-place with Azure Synapse 

You can use the default Serverless SQL Pools for ad hoc data exploration purposes on Azure Storage. You can profile data before importing & selecting. 

To be able to use Serverless SQL Pools, you should go to the most left blade, click on **Data>Workspace>Databases>Your Dedicated Pool>...>New SQL Script>Empty Script** in given order. Now from the top menu you can change the database to the serverless by selecting **Built-in**.

Below query enables you to profile a data on a storage environment. Given Storage is a public storage which can be accessed also from a Managed Azure Synapse Vnet.


```sql
EXEC sp_describe_first_result_set N'
	SELECT
        *
	FROM 
		OPENROWSET(
        	BULK ''https://azureopendatastorage.blob.core.windows.net/nyctlc/yellow/puYear=*/puMonth=*/*.parquet'',
	        FORMAT=''PARQUET''
    	) AS nyc';
```

You can also select the parquet data directly as below:

```sql
	SELECT
        *
	FROM 
		OPENROWSET(
        	BULK 'https://azureopendatastorage.blob.core.windows.net/nyctlc/yellow/puYear=*/puMonth=*/*.parquet',
	        FORMAT='PARQUET'
    	) AS nyc;
```


You can use the same method to select csv as well:
```sql
SELECT
    TOP 100 *
FROM
    OPENROWSET(
        BULK     'https://nytaxiblob.blob.core.windows.net/2013/Weather/*.txt',
        FORMAT = 'csv'
    ) WITH (    [DateID] int,
    [GeographyID] int ,
    [PrecipitationInches] float,
    [AvgTemperatureFahrenheit] float) 
    AS [result];
```

### Loading CSV data from blob to SQL Pools with COPY TO command

Create tables for loading:


*The original tutorial used as reference can be found [here](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/load-data-from-azure-blob-storage-using-copy#create-tables-for-the-sample-data)*


```sql
IF OBJECT_ID('[dbo].[Date]') IS NOT NULL
	DROP TABLE [dbo].[Date];
CREATE TABLE [dbo].[Date]
(
    [DateID] int NOT NULL,
    [Date] datetime NULL,
    [DateBKey] char(10) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [DayOfMonth] varchar(2) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [DaySuffix] varchar(4) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [DayName] varchar(9) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [DayOfWeek] char(1) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [DayOfWeekInMonth] varchar(2) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [DayOfWeekInYear] varchar(2) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [DayOfQuarter] varchar(3) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [DayOfYear] varchar(3) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [WeekOfMonth] varchar(1) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [WeekOfQuarter] varchar(2) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [WeekOfYear] varchar(2) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [Month] varchar(2) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [MonthName] varchar(9) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [MonthOfQuarter] varchar(2) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [Quarter] char(1) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [QuarterName] varchar(9) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [Year] char(4) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [YearName] char(7) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [MonthYear] char(10) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [MMYYYY] char(6) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [FirstDayOfMonth] date NULL,
    [LastDayOfMonth] date NULL,
    [FirstDayOfQuarter] date NULL,
    [LastDayOfQuarter] date NULL,
    [FirstDayOfYear] date NULL,
    [LastDayOfYear] date NULL,
    [IsHolidayUSA] bit NULL,
    [IsWeekday] bit NULL,
    [HolidayUSA] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL
)
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED COLUMNSTORE INDEX
);

IF OBJECT_ID('[dbo].[Geography]') IS NOT NULL
	DROP TABLE [dbo].[Geography];
CREATE TABLE [dbo].[Geography]
(
    [GeographyID] int NOT NULL,
    [ZipCodeBKey] varchar(10) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [County] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [City] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [State] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [Country] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL,
    [ZipCode] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL
)
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED COLUMNSTORE INDEX
);

IF OBJECT_ID('[dbo].[HackneyLicense]') IS NOT NULL
	DROP TABLE [dbo].[HackneyLicense];
CREATE TABLE [dbo].[HackneyLicense]
(
    [HackneyLicenseID] int NOT NULL,
    [HackneyLicenseBKey] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [HackneyLicenseCode] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL
)
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED COLUMNSTORE INDEX
);


IF OBJECT_ID('[dbo].[Medallion]') IS NOT NULL
	DROP TABLE [dbo].[Medallion];
CREATE TABLE [dbo].[Medallion]
(
    [MedallionID] int NOT NULL,
    [MedallionBKey] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [MedallionCode] varchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL
)
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED COLUMNSTORE INDEX
);

IF OBJECT_ID('[dbo].[Time]') IS NOT NULL
	DROP TABLE [dbo].[Time];
CREATE TABLE [dbo].[Time]
(
    [TimeID] int NOT NULL,
    [TimeBKey] varchar(8) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [HourNumber] tinyint NOT NULL,
    [MinuteNumber] tinyint NOT NULL,
    [SecondNumber] tinyint NOT NULL,
    [TimeInSecond] int NOT NULL,
    [HourlyBucket] varchar(15) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [DayTimeBucketGroupKey] int NOT NULL,
    [DayTimeBucket] varchar(100) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL
)
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED COLUMNSTORE INDEX
);

IF OBJECT_ID('[dbo].[Weather]') IS NOT NULL
	DROP TABLE [dbo].[Weather];
CREATE TABLE [dbo].[Weather]
(
    [DateID] int NOT NULL,
    [GeographyID] int NOT NULL,
    [PrecipitationInches] float NOT NULL,
    [AvgTemperatureFahrenheit] float NOT NULL
)
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED COLUMNSTORE INDEX
);

```

Load data with [COPY command](https://docs.microsoft.com/en-us/azure/data-factory/connector-azure-sql-data-warehouse#use-copy-statement) While you can manually load data with below given scripts, you can also browse the storage account from linked services menu and right click to ... and click open. From the listed directories you can right click which one you want to populate  to the database, and from the menu click on new SQL Script, Bulk load, pick a table name and generate load script.

*The original tutorial used as reference can be found [here](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/load-data-from-azure-blob-storage-using-copy?bc=/azure/synapse-analytics/breadcrumb/toc.json&toc=/azure/synapse-analytics/toc.json#load-the-data-into-your-data-warehouse)*


*Copy Command Spec:https://docs.microsoft.com/en-us/sql/t-sql/statements/copy-into-transact-sql?view=azure-sqldw-latest*

```sql
COPY INTO [dbo].[Date]
FROM 'https://yourblob.blob.core.windows.net/yourcontainer/Date'
WITH
(
    FILE_TYPE = 'CSV',
	FIELDTERMINATOR = ',',
	FIELDQUOTE = '',
    CREDENTIAL = ( IDENTITY = 'Managed Identity' )
)
OPTION (LABEL = 'COPY : Load [dbo].[Date] - Taxi dataset');


COPY INTO [dbo].[Geography]
FROM 'https://yourblob.blob.core.windows.net/yourcontainer/Geography'
WITH
(
    FILE_TYPE = 'CSV',
	FIELDTERMINATOR = ',',
	FIELDQUOTE = '',
    CREDENTIAL = ( IDENTITY = 'Managed Identity' )
)
OPTION (LABEL = 'COPY : Load [dbo].[Geography] - Taxi dataset');

COPY INTO [dbo].[HackneyLicense]
FROM 'https://yourblob.blob.core.windows.net/yourcontainer/HackneyLicense'
WITH
(
    FILE_TYPE = 'CSV',
	FIELDTERMINATOR = ',',
	FIELDQUOTE = '',
    CREDENTIAL = ( IDENTITY = 'Managed Identity' )
)
OPTION (LABEL = 'COPY : Load [dbo].[HackneyLicense] - Taxi dataset');

COPY INTO [dbo].[Medallion]
FROM 'https://yourblob.blob.core.windows.net/yourcontainer/Medallion'
WITH
(
    FILE_TYPE = 'CSV',
	FIELDTERMINATOR = ',',
	FIELDQUOTE = '',
    CREDENTIAL = ( IDENTITY = 'Managed Identity' )
)
OPTION (LABEL = 'COPY : Load [dbo].[Medallion] - Taxi dataset');

COPY INTO [dbo].[Time]
FROM 'https://yourblob.blob.core.windows.net/yourcontainer/Time'
WITH
(
    FILE_TYPE = 'CSV',
	FIELDTERMINATOR = ',',
	FIELDQUOTE = '',
    CREDENTIAL = ( IDENTITY = 'Managed Identity' )
)
OPTION (LABEL = 'COPY : Load [dbo].[Time] - Taxi dataset');

COPY INTO [dbo].[Weather]
FROM 'https://yourblob.blob.core.windows.net/yourcontainer/Weather'
WITH
(
    FILE_TYPE = 'CSV',
	FIELDTERMINATOR = ',',
	FIELDQUOTE = '',
	ROWTERMINATOR='0X0A',
    CREDENTIAL = ( IDENTITY = 'Managed Identity' )
)
OPTION (LABEL = 'COPY : Load [dbo].[Weather] - Taxi dataset');
```

Lets load the Trips table now,, which is stored in parquet format.
to test another way of loading ,using [POLYBASE](https://docs.microsoft.com/en-us/azure/data-factory/connector-azure-sql-data-warehouse#use-polybase-to-load-data-into-azure-synapse-analytics)
we will first create an external table for this table and then load the actual table with a CTAS (CREATE AS SELECT) statement.

To create an external table we need to first define the data format, data source and  credential:

```sql
IF NOT EXISTS (SELECT * FROM sys.external_file_formats WHERE name = 'SynapseParquetFormat') 
	CREATE EXTERNAL FILE FORMAT [SynapseParquetFormat] 
	WITH ( FORMAT_TYPE = PARQUET)
GO

IF NOT EXISTS (SELECT * FROM sys.external_data_sources WHERE name = 'nyctlc_azureopendatastorage_blob_core_windows_net') 
	CREATE EXTERNAL DATA SOURCE [nyctlc_azureopendatastorage_blob_core_windows_net] 
	WITH (
		LOCATION = 'wasbs://nyctlc@azureopendatastorage.blob.core.windows.net', 
		TYPE     = HADOOP 
	)
GO
```

Then we will  create an external table to access the data via above definitions.The CREATE EXTERNAL TABLE command on Azure Synapse SQL is called [CETAS](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql/develop-tables-cetas) and is a very powerful method utilizing polybase technology underneath.

```sql
IF OBJECT_ID('[dbo].[Ext_Yellow_Trips]') IS NOT NULL
	DROP TABLE [dbo].[Ext_Yellow_Trips];

CREATE EXTERNAL TABLE [dbo].[Ext_Yellow_Trips] (
	[vendorID] varchar(8000),
	[tpepPickupDateTime] datetime2(7),
	[tpepDropoffDateTime] datetime2(7),
	[passengerCount] int,
	[tripDistance] float,
	[puLocationId] varchar(8000),
	[doLocationId] varchar(8000),
	[startLon] float,
	[startLat] float,
	[endLon] float,
	[endLat] float,
	[rateCodeId] int,
	[storeAndFwdFlag] varchar(2)),
	[paymentType] varchar(8000),
	[fareAmount] float,
	[extra] float,
	[mtaTax] float,
	[improvementSurcharge] varchar(8000),
	[tipAmount] float,
	[tollsAmount] float,
	[totalAmount] float
	)
WITH (
	LOCATION = 'yellow',
	DATA_SOURCE = [nyctlc_azureopendatastorage_blob_core_windows_net],
	FILE_FORMAT = [SynapseParquetFormat],
	REJECT_TYPE = VALUE,
	REJECT_VALUE = 0
	)
GO
```

Below you can find how Synapse auto creates the external table script for this dataset.
*This data exists  also on the [Knowledge Center Gallery Samples](https://docs.microsoft.com/en-us/azure/synapse-analytics/get-started-knowledge-center), from there you can create the create script like you can do for any existing table.*
Having very big text data types can effect your CCI Quality and hence query performance, hence it is suggested to  analyse the data and decide the permanent internal table data types accordingly, to prevent possible performance issues.

```sql
CREATE EXTERNAL TABLE Ext_Yellow_Trips (
	[vendorID] varchar(8000),
	[tpepPickupDateTime] datetime2(7),
	[tpepDropoffDateTime] datetime2(7),
	[passengerCount] int,
	[tripDistance] float,
	[puLocationId] varchar(8000),
	[doLocationId] varchar(8000),
	[startLon] float,
	[startLat] float,
	[endLon] float,
	[endLat] float,
	[rateCodeId] int,
	[storeAndFwdFlag] varchar(8000),
	[paymentType] varchar(8000),
	[fareAmount] float,
	[extra] float,
	[mtaTax] float,
	[improvementSurcharge] varchar(8000),
	[tipAmount] float,
	[tollsAmount] float,
	[totalAmount] float
	)
WITH (
	LOCATION = 'yellow',
	DATA_SOURCE = [nyctlc_azureopendatastorage_blob_core_windows_net],
	FILE_FORMAT = [SynapseParquetFormat],
	REJECT_TYPE = VALUE,
	REJECT_VALUE = 0
	)
```

You can see how to query as well:
```sql
SELECT TOP 100 * FROM Yellow_Trips
GO
```

Then we will use a CTAS (CREATE AS SELECT) statement to create the actual table *(Here we could also have been creating a blank table and using insert. But DROP&CTAS combination has generally a much better performance comparing to TRUNCATE&INSERT, due to recreation of  [Compressed ColumnStore Index (CCI)](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview?toc=/azure/synapse-analytics/toc.json&bc=/azure/synapse-analytics/breadcrumb/toc.json&view=azure-sqldw-latest&preserve-view=true) rowgroups.)*:

```sql
IF OBJECT_ID('[dbo].[Yellow_Trips]') IS NOT NULL
	DROP TABLE [dbo].[Yellow_Trips];
CREATE TABLE [dbo].[Yellow_Trips]
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED COLUMNSTORE INDEX
)
AS
SELECT  ISNULL(CAST([vendorID]  AS INTEGER),0) as [vendorID]
,[tpepPickupDateTime]
,[tpepDropoffDateTime]
,[passengerCount]
,[tripDistance]
,CAST([puLocationId] AS INTEGER ) as [puLocationId]
,CAST([doLocationId] AS INTEGER) as [doLocationId]
,[startLon]
,[startLat]
,[endLon]
,[endLat]
,[rateCodeId]
,[storeAndFwdFlag]
,CAST([paymentType] AS INTEGER) as [paymentType]
,[fareAmount]
,[extra]
,[mtaTax]
,CAST([improvementSurcharge] AS DECIMAL(7,4)) as [improvementSurcharge]
,[tipAmount]
,[tollsAmount]
,[totalAmount]
 FROM [dbo].[Ext_Yellow_Trips];
```

If you want to test TRUNCATE & INSERT or COPY TO  you can use below script to create the table initially, but you should still do the type casting as used i n above CTAS statements SELECT part in the INSERT statement's SELECT part that you will use to populate the date to the target table.

```sql
IF OBJECT_ID('[dbo].[Yellow_Trips]') IS NOT NULL
	DROP TABLE [dbo].[Yellow_Trips];
CREATE TABLE [dbo].[Yellow_Trips]
	[vendorID] int,
	[tpepPickupDateTime] datetime2(7),
	[tpepDropoffDateTime] datetime2(7),
	[passengerCount] int,
	[tripDistance] float,
	[puLocationId] int,
	[doLocationId] int,
	[startLon] float,
	[startLat] float,
	[endLon] float,
	[endLat] float,
	[rateCodeId] int,
	[storeAndFwdFlag] varchar(2),
	[paymentType] int,
	[fareAmount] float,
	[extra] float,
	[mtaTax] float,
	[improvementSurcharge] float,
	[tipAmount] float,
	[tollsAmount] float,
	[totalAmount] float
	)
WITH
(
    DISTRIBUTION = ROUND_ROBIN,
    CLUSTERED COLUMNSTORE INDEX
);
```

**Note:** Round robin is not a very good choice to distribute the tables, but that is not our subject for this exercise. You can get more information about how to efficiently design the table layout for good query performance from the [Guidance for designing distributed tables](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-tables-distribute?toc=/azure/synapse-analytics/toc.json&bc=/azure/synapse-analytics/breadcrumb/toc.json) and [CCI best practices](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql/data-load-columnstore-compression) on microsoft docs.

Once you have your data loaded you can query your data as you want.

More detail about the MPP architecture which the Azure SQL Pools base on can be found [here](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql/overview-architecture) 

## High Availability for Synapse

Azure Synapse workspaces are backed up daily basis.
You can find detailed information on HA DR [here](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/migrate/azure-best-practices/analytics/azure-synapse)
