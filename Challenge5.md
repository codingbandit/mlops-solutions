# Challenge 5: Work with environments

## Create prod environment with cluster and service principal for prod

This approach will use the CLIv2 to create workspace assets. If you want to reuse another environment, modify to fit your context. Ensure you've logged in using `az login` and set the default subscription `az account set -s {SUBSCRIPTION_ID}`.

1. Create a resource group and make it the default:

   ```bash
    az group create --name "diabetes-prod-rg" --location "eastus"
    az configure --defaults group="diabetes-prod-rg"
    ```

2. Create the AML workspace default (so you don't have to specify it on each command):

    ```bash
    az ml workspace create --name "aml-diabetes-prod"
    az configure --defaults workspace="aml-diabetes-prod"
    ```

3. Create compute cluster (see [challenge5_cluster.yml](challenge5_cluster.yml) for the cluster definition):

    ```bash
    az ml compute create -f challenge5_cluster.yml
    ```

4. Create service principal for production, retain a copy of the output for future use:

   ```bash
    az ad sp create-for-rbac --name "diabetes-prod-svcid" --role contributor \
                              --scopes /subscriptions/<subscription-id>/resourceGroups/diabetes-prod-rg \
                              --sdk-auth
    ```

    ```json
    {
        "clientId": "<REDACTED>",
        "clientSecret": "<REDACTED>",
        "subscriptionId": "<REDACTED>",
        "tenantId": "<REDACTED>",
        "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
        "resourceManagerEndpointUrl": "https://management.azure.com/",
        "activeDirectoryGraphResourceId": "https://graph.windows.net/",
        "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
        "galleryEndpointUrl": "https://gallery.azure.com/",
        "managementEndpointUrl": "https://management.core.windows.net/"
    }
    ```

## Create production dataset

Registering the dataset using the CLI pointing to a local file will automatically upload it to Azure Machine Learning storage.

1. Create the [challenge5_dataset.yml](challenge5_dataset.yml) file (source in-file). Note the name and the location points to the folder.

2. Register using the CLI:

    ```bash
    az ml data create --file challenge5_dataset.yml
    ```

## Add Dev and Prod environment configs on GitHub Repo

Before adding and configuring environments, we must remove the "global" azure credentials secret. Big Note! The repository must be set to public in order to see the **Environment protection rules**.

1. On the GitHub repo page, go to **Settings**.
2. Expand **Secrets** (beneath the **Security** heading), and choose **Actions**.
3. Delete **AZURE_CREDENTIALS**.

Add a dev environment.

1. Remaining on the Settings page, select **Environments** (beneath the **Code and automation heading).
2. Select **New environment**.
3. Name it **Dev**.
4. Select **Configure environment**.
5. Beneath **Environment secrets**, select **Add secret**.
6. Name the secret **AZURE_CREDENTIALS** and populate with the original credentials created in Challenge 2.

Add a prod environment.

1. Remaining on the Settings page, select **Environments** (beneath the **Code and automation heading).
2. Select **New environment**.
3. Name it **Prod**.
4. Select **Configure environment**.
5. Beneath **Environment secrets**, select **Add secret**.
6. Name the secret **AZURE_CREDENTIALS** and populate with the original credentials created in this Challenge (5).

Add Gatecheck reviewers

1. Remaining on the **Settings** page, select **Environments**.
2. Under **Environment protection rules**, check **Required reviewers**.
3. Search for and select your GitHub Id.
4. Select **Save protection rules**.

## Create production job YAML file

1. Add new file **src/prod_job.yml** configured with production environment settings.

    ```yml
    $schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
    code: model
    command: >-
      python train.py
      --training_data ${{inputs.training_data}}
    inputs:
      training_data: 
        path: azureml:diabetes-prod-folder:1 
        mode: ro_mount
      reg_rate: 0.01
    environment: azureml:AzureML-sklearn-0.24-ubuntu18.04-py37-cpu@latest
    compute: azureml:diabetes-cluster-cep-prod
    experiment_name: challenge-5-experiment
    description: challenge 5 experiment
    ```

## Create multi-environment action/workflow

The workflow contains two jobs, one for dev and one for production. The production should not run unless dev was successful. The `--stream` value was added to ensure the CLI doesn't return until the job completes, and the `needs` is added to the production job so that it isn't executed unless the dev job completes successfully. Waiting for the job to queue and run will take some time, do not cancel the action. You can go into the dev or prod AML workspaces to see it's status.

Create the file `06-multi-environment.yml` in the **.github\workflows** folder.

```yml
name: Train model

on:
  push:
    branches: [ main ]

jobs:
  buildDev:
    runs-on: ubuntu-latest
    environment:
        name: dev 
    steps:
    - name: Check out repo
      uses: actions/checkout@main
    - name: Install az ml extension
      run: az extension add -n ml -y
    - name: Azure login
      uses: azure/login@v1
      with:
        creds: ${{secrets.AZURE_CREDENTIALS}}
    - name: Submit ML job
      run: az ml job create --file src/job.yml --resource-group diabetes-dev-rg --workspace-name aml-diabetes-dev --stream
  buildProd:
    runs-on: ubuntu-latest
    environment:
        name: prod
    needs: [ buildDev ]
    steps:
    - name: Check out repo
      uses: actions/checkout@main
    - name: Install az ml extension
      run: az extension add -n ml -y
    - name: Azure login
      uses: azure/login@v1
      with:
        creds: ${{secrets.AZURE_CREDENTIALS}}
    - name: Submit ML job
      run: az ml job create --file src/prod_job.yml --resource-group diabetes-prod-rg --workspace-name aml-diabetes-prod --stream
```
