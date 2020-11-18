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

### Add data classification and dynamic data masking

### Create KeyVault access policy to grant permission to current user to access all secrets

### Create new SP and grant reader role to the hands-on-lab resource group

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

### Grant permission to newly created service principle to read secrets:

    az keyvault set-policy -n <your-key-vault-name> --spn http://contoso-apps --secret-permissions get list

### Add new KeyVault secret for SQL Connection string

Key name **SqlConnectionString**

### Azure Database Migration Service

Note: Do not try to restart the DMS service. Because DMS service takes around 30 mins to restart the service.

### Deploy the the API app service to Azure:

**Login to LabVM**

Make the required changes in program.cs and startup.cs

**Changes in Program.cs**

    config.AddAzureKeyVault(KeyVaultConfig.GetKeyVaultEndpoint(buildConfig["KeyVaultName"]),
    buildConfig["KeyVaultClientId"],
    buildConfig["KeyVaultClientSecret"]);

**Changes in startup.cs**

    services.AddDbContext<ContosoDbContext>(
                options => options.UseSqlServer(Configuration["SqlConnectionString"])); // Add the options to retrieve the database connection string from key vault)

**Make changes in API APP service to add the following configurations from advanced editor:**

    [{
        "name": "KeyVaultName",
        "value": "contoso-kv-khbt5achzc2wa"
    },
    {
        "name": "KeyVaultClientId",
        "value": "1cd3bf35-97c8-4528-b228-16353810e01b"
    },
    {
        "name": "KeyVaultClientSecret",
        "value": "jTAQZPRfepm9v2tNCDa89JU_7Pxlyj6w-t"
    }]

**Use visual studio to publish the API**

Once the API is published successfully, the API app is opened in browser. Append **/swagger** at the end of the URL to test the API. From swagger documentation, test the dependents API. If everything are wired up correctly, it should fetch the list of dependents.

# Deploy web application

### Add API url configuration

**List all web app services within the resource group**

    az webapp list -g $rg -o table
    $webAppName = "contoso-web-khbt5achzc2wa"
    $defaultHostName = "contoso-api-khbt5achzc2wa.azurewebsites.net"
    az webapp config appsettings set -n $webAppName -g $rg --settings "ApiUrl=https://$defaultHostName"

### Deploy the web app and test using following credential:

**demouser** **Password.1!!**

# Create storage container and SAS token and take note of those values:

**Note:** For SAS token mention the start date mention 1 day past date, otherwise you may face unauthorise access issue due to time zone conflict.

Bulk upload policy documents using [azcopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy)

In this task, you download and install AzCopy. You then use AzCopy to copy the PDF files from the "on-premises" location into the policies container in Azure storage.

If you use the updated version of azcopy .exe it is not installed but rather just used from cmd/PowerShell. Then Use the following syntax:

    azcopy cp "E:\sources\repos\git-samples\.NET-Modernization-In-a-Day\Hands-on lab\lab-files\policy-documents\*" "https://contosokhbt5achzc2wa.blob.core.windows.net/policies?[sas token]" --recursive=false

# Update function app configuration value

    $resourceGroup = "hands-on-lab"
    $functionAppName = "contoso-func-khbt5achzc2wa"
    $storageUrl = "https://contosokhbt5achzc2wa.blob.core.windows.net/policies"
    $storageSas = "?sv=2019-12-12&ss=b&srt=sco&sp=rwdlacx&se=2020-11-29T21:35:02Z&st=2020-11-06T13:35:02Z&spr=https&sig=%2B%2BbVVx5QtAp9gFA0%2Fflfxt1%2B%2Fpc0wGDJe10EzMYMDXc%3D"
    az functionapp config appsettings set -n $functionAppName -g $resourceGroup --settings "PolicyStorageUrl=$storageUrl" "PolicyStorageSas=$storageSas"
