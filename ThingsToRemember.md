# Infrastructure provisioning

The ARM template: /lab-files/ARM-template/azure-deploy.json contains the definitions of all required Azure resources.

## To provision the infrastructure follow the below mentioned steps:

1.  To run the ARM template deployment, select the **Deploy to Azure** button below, which opens a custom deployment screen in the Azure portal.

    <a href ="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmicrosoft%2FMCW-App-modernization%2Fmaster%2FHands-on%20lab%2Flab-files%2FARM-template%2Fazure-deploy.json" target="_blank" title="Deploy to Azure">
    <img src="http://azuredeploy.net/deploybutton.png"/>
    </a>

### Manually deploy the ARM template in your subscription.

- Login to [Azure portal](https://portal.azure.com)
- From the Azure portal menu, in the search box, type **_deploy_**, and then select **_Deploy a custom template_**.
- Click on `Build your own template in the editor`
- Then click `Load file` and choose `azure-deploy.json` to upload into the editor
- ARM template editor will show all the resources, just click on `Save` to continue to the resource deployment.
- In the deployment form enter the following input:
  - Select `subscription`
  - Select `resource group` where all resources will be provisioned
  - SQL server name, default value is `contosoinsurance`, ARM template will add dynamic character to avoid conflict
  - Admin user name, default is `demouser`, this is applicable to all VMs and SQL server
  - Admin password, default is `Password.1!!`, this is applicable to all VMs and SQL server
  - API management email, it takes around **_30 mins_** to provision API management resource, Azure will send an email notification once this is provisioned.

## VM configurations

Once VMs are provisioned it contains all required tools like SSMS, Visual studio etc and also configured correctly.
Inside the SQL server 2008 VM, the insurance database is also attached.

# Hands on step by step

1. [Create new SQL login in SQL 2008 and setup full recovery](#create-new-sql-login-in-sql-2008-and-setup-full-recovery)
2. [Migrate database schema using DMA tool in SQL 2008 VM](#migrate-database-schema-using-dma-tool-in-sql-2008-vm)
3. [Get SQL 2008 VM public IP to connect to DMS](#get-sql-2008-vm-public-ip-to-connect-to-dms)

### Create new SQL login in SQL 2008 and setup full recovery

Copy and paste the SQL script below into the new query window:

    ```sql
    USE master;
    GO

    -- SET the sa password
    ALTER LOGIN [sa] WITH PASSWORD=N'Password.1!!';
    GO

    -- Enable Mixed Mode Authentication
    EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer', N'LoginMode', REG_DWORD, 2;
    GO

    -- Create a login and user named WorkshopUser
    CREATE LOGIN WorkshopUser WITH PASSWORD = N'Password.1!!';
    GO

    EXEC sp_addsrvrolemember
        @loginame = N'WorkshopUser',
        @rolename = N'sysadmin';
    GO

    USE ContosoInsurance;
    GO

    IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = N'WorkshopUser')
    BEGIN
        CREATE USER [WorkshopUser] FOR LOGIN [WorkshopUser]
        EXEC sp_addrolemember N'db_datareader', N'WorkshopUser'
    END;
    GO

    -- Update the recovery model of the database to FULL
    ALTER DATABASE ContosoInsurance SET RECOVERY FULL;
    GO
    ```

### Migrate database schema using DMA tool in SQL 2008 VM

### Get SQL 2008 VM public IP to complete the data migration using DMS:

- Connect az PowerShell session-
- Execute `az vm list-ip-addresses -g $resourceGroup -n SqlServer2008 --output table `
- Take note of public IP address
- Next run `az sql server list -g $resourceGroup` to list all the sql server and take note of the fully qualified SQL server name

### Create KeyVault access policy to grant permission to current user to access all secrets

### Create new SP and grant reader role to the hands-on-lab resource group

```ps
$rg = "hands-on-lab"

# Get the subscription id
az account list -o table
$subscriptionId ="<your subscription id>"
# Create new SP and grant permission
az ad sp create-for-rbac --name contoso-app --role reader --scopes subscription/$subscriptionId/resourceGroups/$rg
# Take note of the following details:
{
  "appId": "1cd3bf35-97c8-4528-b228-16353810e01b",
  "displayName": "contoso-app",
  "name": "http://contoso-app",
  "password": "jTAQZPRfepm9v2tNCDa89JU_7Pxlyj6w-t",
  "tenant": "2a1f4476-a400-xxxxxxxxxx"
}
```

### Azure Database Migration Service

Note: Do not try to restart the DMS service. Because DMS service takes around 30 mins to restart the service.
