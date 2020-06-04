[[_TOC_]]
#Staging Orchestration Process
## Overview
The orchestration of the daily process is achieved via Talend in the form of stages:
1.	Staging Master Orchestration  
2.	Staging Control Orchestration

Each stage is explained in detail in the sections below. 

##Process Flow Diagram
The following diagram depicts the process for Staging Orchestration process. 

![image.png](/.attachments/image-3ec82f3c-4d57-42c7-9be5-56c2b72343d1.png)


## Staging Master Orchestration process

![image.png](/.attachments/image-78e9263d-8708-4679-8d1e-dd9ceddf7ae0.png)

Staging Master Orchestration is responsible for executing the Control Jobs in parallel. Control Jobs are segregated based on schema/subject area. The tParallelize component is used to achieve maximum parallelism. 

The PreJob component receives the Staging Master job name from the parent job and executes the sp_recordpackageExcution stored procedure in Snowflake. This stored procedure takes Job name, and Parent Execution ID as input arguments and returns the executionID as output. This value is then passed to the next subjob (Staging Process Orchestration) as a parameter. The parameters passed to the next sub-jobs are Job ID of Control Job, Job Name of Control Job and Execution ID of parent job (which is the Staging Master job).

Once all the Control Jobs return a successful status to this job, the PostJob component triggers the stored procedure to record Package success by stamping an end time in the PackageExecution table for the ExecutionID.



## Staging Control Orchestration

![image.png](/.attachments/image-a50fe1e7-afeb-4990-a8e2-a16da79ab1ca.png)


Staging Control Orchestration is responsible for invoking the process jobs.  The Pre-job component uses the parameters Job name and Parent Package Execution ID from the parent job and computes the Execution ID. 

Using the Job ID of Control Job which is passed as a parameter from the Parent Job, the Package table in ETLConfig database is queried to get the list of process jobs to run. Each package name is then passed to tFlowtoIterate component, which iterates through each package name in the list and executes them by passing it to tRunJob component. tFlowtoIterate component is configured to execute two process jobs in parallel. 

Once the process jobs have successfully executed, the PostJob component  executes the stored procedure to record the completion of package execution(sp_recordPackageSuccess ). It passes the ExecutionID of the current job as input parameter, which is used to find the corresponding record in the PackageExecution table in ETLConfig database, and stamp an End time.


## Individual Process Jobs

![image.png](/.attachments/image-e7f9b7a9-997a-4347-a923-4e915b55c3eb.png)


Process jobs are the most granular job executions in the daily process. These jobs are invoked by the Staging Process Orchestration job. This job performs one unit for work, for example, stages data from a SQL Server table to a Snowflake table. The PreJob Component executes the stored procedure in Snowflake to log the start of job execution by passing in the name of job, along with Parent Job Execution ID as parameters. This returns the execution ID of the current job as output.

Once the main job is complete, the successful execution is recorded by calling the Snowflake Stored procedure and passing the Execution ID of the current job as input parameter. 


## Major Components Used

- PostJob – used for closing connections and recording completion of package execution 
- PreJob- used for validating connections, recording package executions
- tParallelize – used to execute multiple control jobs in parallel threads. 
- tFlowtoIterate – iterates through list of packagenames passed as input and passes them to tRunJob
- tDBRow – used to call Snowflake Stored Procedure, mostly for package execution logging.
- tRunJob – used to specify the child jobs to execute, and also specify the parameters to be passed to child jobs.
- Initialization Joblet - reusable job  attached to PostJob component in each job that performs all the Initialization activities such as recording package execution status.
- Finalization Joblet – reusable job  attached to PostJob component in each job that performs all the finalization activities such as recording package execution status
- LogProcessingListener Joblet – reusable code that listens to the execution of each job that it is attached to, and logs Java exceptions and any other exceptions reported by the tDie and tWarn components. 
- tWarn – component used to allow LogListener joblet to catch any success messages or warnings from jobs in a controlled manner.
- tDie – component used to control the exit point in the code, by returning the error code that will be used by packageExecutionStatus.

#Staging Flat files form Azure Blob Storage


##Overview:

The purpose of this document is to list the various steps needed to successfully load the legacy files (IS4East and IS4West) from Azure Storage to Snowflake. The document also covers all the dependencies required to complete the task.


##Connect to Azure Storage and View files in Blob Containers

1.	To connect to Azure Storage, you need to download Microsoft Azure Storage Explorer (Free) for your version of OS (link below).
	https://azure.microsoft.com/en-us/features/storage-explorer/

2.	Open Microsoft Azure Storage Explorer, right click ‘Storage Accounts’ and select Connect to Azure Storage’.
 

![image.png](/.attachments/image-453ff228-f04c-4662-853a-a0198124b7d8.png)

3.	In the Connection setup window, select ‘Use a shared access Signature(SAS) URI’
 
![image.png](/.attachments/image-87e52928-2f5a-47ba-81ed-489063aa8d26.png)

4.	In the next section, paste the SAS URI (given below) into the ‘URI’ field. All other fields will be automatically populated. https://covuseedwdevsa.blob.core.windows.net/?sv=2018-03-28&ss=bfqt&srt=sco&sp=rwdlacup&se=2019-12-08T04:15:15Z&st=2019-08-14T19:15:15Z&spr=https&sig=V1xOn8JjkbQQA4JNXLTaaxS8aLjasbCTiQtlxrnds0k%3D

![image.png](/.attachments/image-16587663-337e-4d13-a231-8f7b32578f69.png)

5.	In the Connection Summary window, verify the details and click ‘Connect’. Note: confirm you are connecting to the dev container by confirming the keyword ‘dev’ in the URI. 

![image.png](/.attachments/image-073f73a8-3cd3-43fe-8246-fe253e24533f.png)

6.	Once connection is successful, you will be able to view the new Storage Account and the associated containers and Blobs. 

![image.png](/.attachments/image-0bc06b30-c9d6-49ae-981f-003402fcfd0a.png)

##Creating Talend Jobs to Load Flat Files 
###A.	Adding Components and Context Variables

1.	Add the required components from Palette into Designer. 

     a.	tAzureStorageGet – downloads files from specified Blob to your local directory

     b.	tFileInputDelimited – configures the flat file and prepares it for loading

     c.	tSnowflakeOutput – loads file configured in prior component

![image.png](/.attachments/image-8388942c-534a-489d-a3c9-c69891c737d8.png)

2. Add the necessary Context Variables for the job as shown below:

![image.png](/.attachments/image-77a80a14-9983-4c8a-9ff8-35e36c3eae52.png)


| **Context variable** | **Type**	 |**Value**  |
|--|--|--|
|  azurestoragecontainer	| String	 | cov-use-edw-blob-sa |
| azurestorageblobprefix	 | String	 | EDWStaging/East_Geography.Address.txt |
| azurestoragesasurl	 | String	 | https://covuseedwdevsa.blob.core.windows.net/?sv=2018-03-28&ss=bfqt&srt=sco&sp=rwdlacup&se=2019-12-08T04:15:15Z&st=2019-08-14T19:15:15Z&spr=https&sig=V1xOn8JjkbQQA4JNXLTaaxS8aLjasbCTiQtlxrnds0k%3D |
| azurestagelocaldirectory	 | String	 |  <path_to_your_local_directory>//IS4LegacyFiles|
|snowflakeoutboundlocaldir	  | String	 |  <path_to_your_local_directory>/IS4LegacyFiles/EDWStaging/East_Geography.Address.txt|
| stagingexecutionid	 | Integer	 | 8888 |
| extractionexecutionid	 |Integer	  | 8888 |
 

Note: Ensure that all strings are encapsulated in double quotes, and note the Type for ExecutionID columns. For the purpose of this tutorial, a folder named ‘IS4LegacyFiles’ was created in the local machine, and then referenced in the context variables. 

###B.	Configure each Talend component
**•	tAzureStorageGet**

Select the ‘Use Azure Shared Access Signature’ checkbox. 

Specify the context variables for Azure Shared Access Signature, Container name, Local folder and Blobs prefix. 

Select the ‘Create Parent directories’ checkbox. 	

![image.png](/.attachments/image-d68c9043-bb25-428b-9fa4-f6c9a0a334ea.png)


**Additional information on Parameters:**

**Prefix**: this allows you to filter the blobs which have the specified prefix in their names in the given container. For the flat files, you need to specify the file name, including the full path. You can get this information from Azure Storage Explorer by right clicking a file and selecting properties, and then copying the ‘Name’ field. A blob name contains the virtual hierarchy of the blob itself. This hierarchy is a virtual path to that blob and is relative to the container where that blob is stored. 

For example, in a container named cov-use-edw-blob-sa, the name of a flat file  blob might be EDW_Staging/East_Geography.Address.txt.

![image.png](/.attachments/image-71ea390c-0b22-4bb0-9824-8f0bde56784f.png)
![image.png](/.attachments/image-dd7ed055-c2f6-4f24-b9c4-7f423e4e9321.png)

**Include sub-directories**: select this check box to retrieve all of the sub-folders and the blobs in those folders beneath the designated directory level in the Blob prefix column. If you leave this check box clear, tAzureStorageGet returns only the blobs directly beneath that directory level. We do not need to check this since we’re fetching one specific file at a time. 

**Create parent directories**: select this check box to replicate the virtual directory of the retrieved blobs in the local folder. Note that if you leave this check box clear, there must be the same directory in the local folder as the retrieved blobs have in the container; otherwise, those blobs cannot be retrieved. Refer FAQ section below for associated errors. 

•	**tFileInputDelimited**

Specify the context variable for file name in the ‘File name/Stream’ field. 
If your data contains date columns, ensure the format in ‘Edit schema’ window matches the actual date formats in your data. 

Specify the Field separator and Header values. 

Open Edit schema, and include StagingExecutionID and ExtractionExecutionID, and specify the corresponding type and context variables. 


![image.png](/.attachments/image-960f80af-b05e-4556-b905-61def940ae55.png)

![image.png](/.attachments/image-be086bf8-dc20-46de-a126-f3295d9a0ef3.png)

•	**tSnowflakeOutput**

Specify the Connection details, and select the correct table name. 

Select ‘TRUNCATE’ under Table Action. 

![image.png](/.attachments/image-bdd166f0-c2dd-420e-8ac8-6d325ae1cbfb.png)

###C.	**Complete your Talend Job**

Make necessary connections as shown below to complete the configuration of your Talend job. 

![image.png](/.attachments/image-8d9b16a3-fa12-4e16-8709-ef764ff06085.png)


# SSIS Package to SnowSQL Conversion

The objective of this task is to convert the latest SSIS package for the package assigned, and convert them into a series of SnowSQL scripts. For each step within the SSIS Package, it is necessary to convert the SQL from TransactSQL to SnowSQL.
We are selecting the package for Party Helper ASW as an example in this document. Package name = CoverysEDW_Staging_Process_Aux_Party_Helper_ASW.dtsx

## 1.	Download and View Latest SSIS Package in Visual Studio

Download the latest SSIS package from SSIS_Staging folder from Visual Studio Team Web Access site.

![image.png](/.attachments/image-2ecd584b-424c-409a-8757-3296a09a7f5c.png)


Open the file in Visual Studio, and view all the tasks under ‘Work’. For each task, note the stored procedure being executed. 

![image.png](/.attachments/image-cc659490-468e-4fd4-af76-18b33171ebbd.png)

For example, the first task ‘SQL Task Staging usp_Populate_Bridge_PartyRole_Source’ connects to ‘Staging’ database and executes stored procedure [PPIC_Party_Helper].[usp_Populate_Bridge_PartyRole_Source];. 

![image.png](/.attachments/image-138b4021-a727-4294-9566-c8fc691e88da.png)

##2.	View SQL Code for Stored Procedure in SQL Server Management Studio

Open SQL Server Management Studio and navigate to the required Stored Procedure and view the code. 

![image.png](/.attachments/image-af2c3984-8513-4bdb-b410-0917b7af91dd.png)

##3.	Convert the Transact SQL Code to SnowSQL in Snowflake GUI 

Development/Design can take place in the Snowflake GUI providing the required source staging tables have been populated. Ensure that the code is executed in Snowflake console before saving into a .sql file. 

![image.png](/.attachments/image-dc317cc0-7601-4fab-9349-cc6d65c4cc0e.png)

All tasks within ‘Work’ can be combined and saved into one .sql file, if there are no other dependencies for their execution. 

##4.	Update SnowSQL scripts in Git Repository

Please make sure your scripts are periodically checked into Git, in the location specified in the screenshot below. The naming convention for the file is the same as SSIS package naming convention, followed by a .Sql extension. For example, ‘CoverysEDW_Staging_Process_Aux_Party_Helper_ASW.sql’ will be the name of the script we walked through in this document.

![image.png](/.attachments/image-a2139ec2-e0d2-45b3-84b0-6a2ffb7e27f5.png)

For information on GitHub Desktop setup, please refer the following link:

https://ecoverys.sharepoint.com/:w:/r/sites/extEDWCloudMigration/_layouts/15/Doc.aspx?sourcedoc=%7B02160515-27E9-4B61-BE71-6D897005A4A8%7D&file=Talend_Install_and_Configuration_Git_Hub_Overview.docx&action=default&mobileredirect=true

###NOTES AND HELPFUL RESOURCES:

**Notes:** 

- Write all the SnowSQL scripts for tasks specified in package in one file.
- Encapsulating table names or column names in quotes make them case sensitive. Avoid typing objects in quotes unless it is required. 
- It is best practice to specify the columns you are inserting into in your ‘INSERT INTO table_name’ statement. 
- You can alias a column using the ‘AS’ keyword. The ‘AS’ keyword is also used to assign values to columns. 

**Snowflake Community Documentation:**

|**Topic**  | **Link** |
|--|--|
|  SnowSQL Commands Glossary| https://docs.snowflake.net/manuals/sql-reference/sql-all.html |
| Pivot Function in SnowSQL | https://docs.snowflake.net/manuals/sql-reference/constructs/pivot.html |
| Unpivot	 | https://docs.snowflake.net/manuals/sql-reference/constructs/unpivot.html |
| SnowSQL Query Syntax | https://docs.snowflake.net/manuals/sql-reference/constructs.html |

**Common SQL to SnowSQL Conversions**

|**SQL**| **SnowSQL** |
|--|--|
| [RowIsCurrent] = CAST('Y' AS [nchar] (1)) | CAST('Y' AS VARCHAR) AS ROWISCURRENT |
|  [RowEndDate] = CAST('12/31/9999' AS [datetime])| CAST('12/31/9999' AS datetime) AS ROWENDDATE |
| ColumnName='Value' | ‘Value' AS ColumnName |
| ISNULL(expr1,expr2) | IFNULL(expr1,expr2) |
 
For a detailed example on conversion of a script, refer: 

- [https://ecoverys.sharepoint.com/sites/extEDWCloudMigration/Documents/Sprint%203_%20SSIS%20Package%20to%20SnowSQL%20Conversion%20(Helper%20Tables).docx?d=w12c2299dfcb64864b9b623feb1a1f0af]()
	
- https://ecoverys.sharepoint.com/sites/extEDWCloudMigration/Documents/Sprint3_SnowSQLScript_PPIC_Party_Helper_usp_Bridge_PartyRole_Source.sql
	
	
	


