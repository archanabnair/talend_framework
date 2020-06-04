
**QA Checklist**


**1. Packageexecution table in Snowflake (QA_ETLConfig):**

- Check the Master staging package id (null) , control package id and all process package IDs.
- Ensure process package IDs are executed in the correct execution order as indicated in QA_ETLConfig.Control.Package table.
- Ensure all process packages with Active=1 , for the given Control Package ID are executed.
- Ensure all execution status=2 (success). If status = 1 or 3, flag test failed.

**2. Package Execution Order:**
- Verify the packages are being Executed in Correct order as specified in QA_ETLConfig.Control.Package table Archana: Need to update the package names in QA
- Query to get list of packages for a control package, sorted by execution order:

  SELECT *
FROM control.package
where PARENTPACKAGEID =16
and Active=1
ORDER BY ExecutionOrder ASC;

**3. Staging tables in Snowflake (QA_Staging):** 

- select * from staging table - to confirm data was loaded
- check record counts in Snowflake staging table matches original SQL source server (and not uatfix server)
- Run a mass snowflake query to ensure records populated in table for a particular staging execution id:
-  select * from dev_staging.asw_config.DBO_ACTY where STAGINGEXECUTIONID='200376'  -- insert correct value for stagingexecutionid

**4. Last Altered Timestamp on Staging Tables:** 
- Run the following and verify the tables in a schema was populated/altered after each run
   
  ALTER SESSION SET TIMEZONE = 'America/New_York';

  select TABLE_CATALOG,TABLE_SCHEMA,TABLE_NAME,ROW_COUNT,BYTES,LAST_ALTERED  from "DEV_STAGING"."INFORMATION_SCHEMA"."TABLES" 
 where table_schema= 'ASW_CONFIG'


**Link to Video Tutorial:** https://ecoverys.sharepoint.com/sites/extEDWCloudMigration/Documents/Staging_Orchestration_Testing_Walkthrough.mp4
