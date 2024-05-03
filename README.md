# How to create a new secretless Function App using managed identity

To improve its overall security posture, your organization may have a policy to disable storage keys and use managed identity. Since the Consumption and Premium plans of Azure Functions takes a dependency on Azure Files, which does not yet support managed identity, this E2E tutorial will teach you how to create a **new** Function app without Azure Files and perform deployments accordingly.

## Some considerations before proceeding
- **Only Linux Consumption supports re-configuring an existing** app to remove its dependency on Azure Files. Extending this to other SKUs is actively being invested in.
- There is a known bug where you might see a diagnostic event in the Portal saying, "Storage is not configured properly, Function scaling will be limited. Click to learn more." If you follow the deployment guidance in this tutorial, cross-stamp scaling will **not** be affected.
- Without Azure Files, we recommend you to deploy via the `WEBSITE_RUN_FROM_PACKAGE = URL` [method](https://learn.microsoft.com/azure/azure-functions/run-functions-from-deployment-package#using-website_run_from_package--url). This has a negative impact on cold start.
  - If you are creating a Premium app, cold start will be mitigated because there will be [always ready instances](https://learn.microsoft.com/azure/azure-functions/functions-premium-plan?tabs=portal#always-ready-instances) and [prewarmed instances](https://learn.microsoft.com/en-us/azure/azure-functions/functions-premium-plan?tabs=portal#prewarmed-instances).
  - If you are creating a Windows Consumption app, you will experience cold starts proportional to the size of the package you deploy.
- The `WEBSITE_RUN_FROM_PACKAGE = URL` requires you to take care of deployment, manually. Consequently, you cannot use client tooling that rely on `WEBSITE_RUN_FROM_PACKAGE = 1`.
  - You cannot use Core Tools' `func azure functionapp publish ...` command.
  - You cannot use AZ CLI's `az functionapp deployment source config-zip ...`
  - You cannot use VS/VS Code's UI to deploy.
  - You cannot use the ADO Task `AzureFunctionApp@1/2` (as of May 2, 2024)
  - You cannot use the GH Action, `Azure/functions-actions@v1` (as of May 2, 2024)
> If these considerations prove this method to be infeasible, you will have to wait until migrating an existing Function app off of Azure Files is supported.

## Creating a new Function App without Azure Files through the Portal
The following steps will show you how to create a Function app with no dependency on Azure Files.
1. In the **Portal**, enter the UX to create a new Function app.
2. In the **Basics** tab, choose either Windows Consumption or Windows/Linux Premium as your **Operating system** and **Hosting plan**.
3. For the rest of the creation flow, until the **Review + create** tab, configure your function app as desired.


Now with a standard template, we will modify it to remove a dependency on Azure Files.

4. In the **Review + create** tab, select **Download a template for automation**.
5. In the **Custom deployment** page, select **Edit template**.
6. In the **template editor**, make the following modifications:

   a. Under the **Microsoft.Web/sites** resource, add the following item within "propertes": `"vnetContentShareEnabled": true`.
   
   b. Under the same resource, under "appSettings", delete the entries for `WEBSITE_CONTENTAZUREFILECONNECTIONSTRING` and `WEBSITE_CONTNETSHARE`.
   
7. Select **Save** at the bottom of the template editor page.
8. In the **Custom deployment** page, keep the default paramter values.
9. Select **Review + create** at the bottom of the page.
10. Once the validation succeeds, select **Create**.

Your new Function app will now use managed files as for its root directory.

## Configure your Function app and storage account with managed identity
The first step is to associate your new Function app with an identity. You can either use system assigned or user assigned identity. View this article to [learn more about their differences](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/managed-identity-best-practice-recommendations#choosing-system-or-user-assigned-managed-identities). 

1. In the **Function app resource**, navigate to the **Identity** blade.
2. If you desire to use System assigned managed identity, toggle the **Status pill** to **On**.
3. If you desire to use User assigned managed identity, select the **User assigned** tab and select **Add**.
4. Continue through the UX to select your identity of choice.

Next, you'll grant the identity the right roles to connect with storage. This section follows the guidance in this [doc](https://learn.microsoft.com/azure/azure-functions/functions-reference?tabs=blob&pivots=programming-language-csharp#connecting-to-host-storage-with-an-identity).

5. Navigate to the storage account resource specified in your `AzureWebJobsStorage` app setting.
6. In the Access Control (IAMs) blade, select the Add option to add a new role assignment.

   a. Search for `Storage Blob Data Owner` and select it.
   
   b. For the **Assign access to** field, select **Managed identity**.
   
   c. Click + Select Members to bring up the context pane UX.
   
   d. Proceed throught the UX to select either the system assigned or user assigned managed identity you configured in the previous section.
   
   e. Click Select to confirm your choice and click Review + assign, twice.
   
>    **Note**: The `Storage Blob Data Owner` role enables the Functions host to connect with your storage account via identity. If you intend to use this storage account for working with triggers and extensions, please add the necessary roles specified in the last table of [Connecting to host storage with an identity](https://learn.microsoft.com/azure/azure-functions/functions-reference?tabs=eventhubs&pivots=programming-language-csharp#connecting-to-host-storage-with-an-identity). 


At this point, you can disable keys on your storage account, if you haven't done so already.

7. In the **Configuration** blade, select **Disabled** for **Allow storage account key access**.

Next, you will specify your Function app to make host connections to the storage account with managed identity.

8. In the **Function app** resource, navigate to the **Environment variables** blade.
9. Select the **Advanced edit** option.
10. In the context pane UX, delete the entry for `AzureWebJobsStorage`. Instead, replace it with the following app settings:

    a. App setting name: `AzureWebJobsStorage__blobServiceUri` (two underscores); value: `https://<storageAccountName>.blob.core.windows.net`

    b. App setting name: `AzureWebJobsStorage__queueServiceUri` (two underscores); value: `https://<storageAccountName>.queue.core.windows.net`

    c. App setting name: `AzureWebJobsStorage__tableServiceUri` (two underscores); value: `https://<storageAccountName>.table.core.windows.net`

11. While you're here, create the following app setting to set up for deploying with managed identity:

    a. If your app is using **system assigned managed identity**. App setting name: `WEBSITE_RUN_FROM_PACKAGE_BLOB_MI_RESOURCE_ID`; value: `SystemAssigned`.

    b. If your app is using **user assigned managed identity**. App setting name: `WEBSITE_RUN_FROM_PACKAGE_BLOB_MI_RESOURCE_ID`; value: `SystemAssigned`.

Now your Function app's host will connect to storage using managed identity. The next step is to configure your app to fetch the packages you deployed to it using managed identity.

## Complete your first deployment using managed identity
Function apps using managed identity are recommended to the `WEBSITE_RUN_FROM_PACKAGE = URL` [deployment method](https://learn.microsoft.com/azure/azure-functions/run-functions-from-deployment-package#using-website_run_from_package--url). This section will show you how to set up and perform this deployment method using managed identity.

The first thing we'll do is create the blob container that will store the packages you deploy to your app.

1. Navigate to the storage account pointed to by your `AzureWebJobsStorage__blobServiceUri` app setting.
2. Navigate to the **Containers** blade. Select the **+ Container** option to create a new container with default settings. You can name it whatever you wish.
3. Navigate into the newly created **container**. In the **overview** page, if you *do not see* an error banner stating that you do not have permission to list the contents of the container, you may skip to the next step.

   a. You will need to grant yourself a `Storage Data Blob Owner` role. You can do so by going to the **container**'s **Access Control (IAMs)** blade.


Now with the infrastructure set up, let's perform your first deployment using managed identity!

4. In your editor of choice, **build** your functions code and **generate** a zip file. Save it to a file path you can access later. This is the package we'll be deploying.
5. Navigate to the container you created to store deployment packages. In the **Overview** page, select the **Upload** option and select the zip package you just created.

> **Note**: You might have a policy preventing you from uploading files in the Portal. If so, upload the zip package using this [AZ CLI](https://learn.microsoft.com/cli/azure/storage/blob?view=azure-cli-latest#az-storage-blob-upload) command: `az storage blob upload -f <local file path to zip> --account-name <storage account> -c <container name> -n <your desired blob file name> --overwrite true --auth-mode login`
 
6. There should now be a new blob for the zip file. **Right click** on it or **select the ellipses icon** at the very right of the entry.

   a. Select **Properties** from the dropdown and **copy the URL**.


Once your blob is uploaded to the container, what's left is to notify your Function app that a package has been deployed.

7. In your **Function app** resource, navigate to the **Environment variables** blade.
8. Select the **Add application setting** option. Add the following settings:
    
    a. App setting name: `WEBSITE_RUN_FROM_PACAKGE_BLOB_MI_RESOURCE_ID`; value: `"SystemAssigned"`
    
    b. App setting name: `WEBSITE_RUN_FROM_PACKAGE`; value: `"https://<storageAccountName>.blob.core.windows.net/<containerName>/<fileName>.zip"`, or the URL blob file path you copied.

9. Select **Apply** in the pane to confirm your input.
10. Select **Apply** in the blade to confirm your changes.
11. Updating your app settings will cause your app to restart, loading your functions from the zip file and calling SyncTriggers.

Congratulations! You completed your first deployment to your new function app. Every subsequent deployment will be similar, but there are important, differing details.

## How to complete subsequent deployments with managed identity
The major difference for subsequent deployments is that you'll need to notify your Function app that a new package has been uploaded to your container for deployments. This notifying gesture is called [SyncTriggers](https://learn.microsoft.com/azure/azure-functions/functions-deployment-technologies?tabs=windows#trigger-syncing), a gesture typically done automatically in tooling, which use `WEBSITE_RUN_FROM_PACKAGE = 1` as the deployment method.

1. In your editor of choice, **build** your functions code and **generate** a zip file. Save it to a file path you can access later. This is the package we'll be deploying.
2. Navigate to the container specified in the `WEBSITE_RUN_FROM_PACKAGE` app setting. 
3. Select the **Upload** option and select the zip package you generated in step 1.

   a. You might have a policy preventing you from uploading files in the Portal. If so, upload the zip package using this [AZ CLI](https://learn.microsoft.com/cli/azure/storage/blob?view=azure-cli-latest#az-storage-blob-upload) command: `az storage blob upload -f <local file path to zip> --account-name <storage account> -c <container name> -n <your desired blob file name> --overwrite true --auth-mode login`

4. If you intend to use the same zip file name, you may overwrite the existing blob.

If you used the *same* zip file name, complete the deployment with the following steps. If you used a different name, skip to the next section.

5. Navigate to your **Function app resource**. 
6. In the **Overview** blade, select **Restart**. This will cause your function app to fetch the zip package, load your functions, and perform a SyncTriggers call. Like all deployments, your function app will be down for the duration of the restart.

   a. Alternatively, you can call SyncTriggers through the AZ CLI: `az rest --method post --url https://management.azure.com/subscriptions/<subID>/resourceGroups/<resourceGroup>/providers/Microsoft.Web/sites/<functionAppName>/syncfunctiontriggers?api-version=2016-08-01`

7. Once your app restarts, the deployment is completed.

If you used a *different* zip file name, complete the deployment with the following steps.

5. In the container’s **Overview** blade, **right click** on the new blob you uploaded or **select the ellipses icon** on the far right.
   
   a. Select **Properties** from the dropdown and **copy the URL**.

6. Navigate to your **Function app resource**.
7. In the **Environment variables** blade, select **Advanced edit**.
8. Update the value of the app setting `WEBSITE_RUN_FROM_PACKGE` with the new blob file path you cpied.
9. Select **Ok** in the pane to confirm your input.
10. Select **Apply** in the blade to confirm your changes. Updating your app settings will cause your app to restart, fetching the zip package, loading your functions, and performing a SyncTriggers call.
11. Once your app restarts, the deployment is completed.

> **Note**: you will need to repeat this process for every subsequent deployment. A CI/CD script is being worked on.
