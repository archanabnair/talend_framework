[[_TOC_]]
#General
All Source Code is kept within Azure DevOps:

[https://dev.azure.com/coverysit/DataWarehouse]()

Code is split into two origin branches:
1. Development
1. Master

Developers will create local branches to do thier work, and push up to the
origin/Development branch once work is complete and ready for regression testing.

Two seperate processes are required to maintain the code base. One for Snowflake
SQL code, and another for Talend processes.

#Snowflake
Deployment process will flow from azure-edw-dev -> Check-in to development branch

SnowSQL Deployments will be pushed to the following locations:

1. azure-edw-dev

    F:\SnowSQL\Staging

   F:\SnowSQL\Transform

1. azure-edw-uat
 
   F:\SnowSQL\Staging
   
   F:\SnowSQL\Transform

1. azure-edw-prod

   F:\SnowSQL\Staging
   
   F:\SnowSQL\Transform

1. azure-edw-dr

   F:\SnowSQL\Staging

   F:\SnowSQL\Transform

The origin for the push is:
https://dev.azure.com/coverysit/DataWarehouse/Repos/Files/SnowSQL

All code will be merged to Development and deployed to Production

Once Production Validation has been completed the origin/development branch will be
mergee into master

#Talend
The Talend Cloud Management Console handles code deployment slightly different
than a typical Git installation, providing code promotion tools internally.

Once Developers have merged thier work into the orgin/development branch, they will
Publish their code to the Cloud Management Console through the Talend Desktop Tool

![image.png](/.attachments/image-d307f20d-96b1-4c99-b23f-0cf76810d359.png)

The code then flows through the Talend environments using the Talend Management
Console (https://portal.us.cloud.talend.com/)

![image.png](/.attachments/image-941113b4-cb94-498b-895f-0ae05a6363fb.png)

To promote the code through each environment. A designated deployment manager
will use Promotions created within the Talend Management Console to migrate code
throughout the Software Development Lifecycle.

![image.png](/.attachments/image-95bf00bb-91f5-4fde-95b7-2662a603b4ac.png)

All code is maintained in Azure DevOps Git:
[[https://dev.azure.com/coverysit/DataWarehouse/COVERYS_EDW]()]()

All code will be merged to Development and deployed to Production
Once Production Validation has been completed the origin/development branch will be
mergee into master