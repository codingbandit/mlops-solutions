# Challenge 1: Create an Azure Machine Learning job

This approach will use the CLIv2 to create workspace assets. If you want to reuse another environment, modify to fit your context. Ensure you've logged in using `az login` and set the default subscription `az account set -s {SUBSCRIPTION_ID}`.

## Setup CLI

1. Ensure the Azure CLI is installed, then install the extension, note to remove the azure-cli-ml extension as it conflicts with the new ml extension:

   ```bash
   az extension remove -n azure-cli-ml
   az extension remove -n ml
   az extension add -n ml -y
   ```

## Setup base resources

1. Create a resource group and make it the default:

   ```bash
    az group create --name "diabetes-dev-rg" --location "eastus"
    az configure --defaults group="diabetes-dev-rg"
    ```

2. Create the AML workspace default (so you don't have to specify it on each command):

    ```bash
    az ml workspace create --name "aml-diabetes-dev"
    az configure --defaults workspace="aml-diabetes-dev"
    ```

3. Create a compute instance for running code (--name must be unique in the region):

    ```bash
     az ml compute create --name "testdev-vm-cep" --size STANDARD_DS11_V2 --type ComputeInstance
    ```

## Register the dataset using CLI

Registering the dataset using the CLI pointing to a local file will automatically upload it to Azure Machine Learning storage.

1. Create the [challenge1_dataset.yml](challenge1_dataset.yml) file. Note the name and the location points to the folder.

2. Register using the CLI:

   ```bash
   az ml data create --file challenge1_dataset.yml
   ```

## Complete the src/job.yml file

Change out to the registered dataset name and version (vs folder URI). Set compute name as the one you created for this lab.

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
code: model
command: >-
  python train.py
  --training_data ${{inputs.training_data}}
inputs:
  training_data:
    path: azureml:diabetes-dev-folder:1 
    mode: ro_mount
  reg_rate: 0.01
environment: azureml:AzureML-sklearn-0.24-ubuntu18.04-py37-cpu@latest
compute: azureml:testdev-vm-cep
experiment_name: challenge-1-experiment
description: challenge 1 experiment
```

## Submit job through the CLI

```bash
az ml job create --file src/job.yml
```
