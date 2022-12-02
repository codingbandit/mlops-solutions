# Challenge 2: Trigger the Azure Machine Learning job with GitHub Actions

## Create a service principal

Replace subscription id tokens:

```bash
az ad sp create-for-rbac --name "diabetes-dev-svcid" --role contributor \
                              --scopes /subscriptions/<subscription-id>/resourceGroups/diabetes-dev-rg \
                              --sdk-auth
```

This will yield a similar output to the following - this entire message is what will be used in the AZURE_CREDENTIALS secret in GitHub - record this value for the next step and future use in this lab:

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

## Create GitHub Secret

Access the remote repository, select **Settings** then choose **Secrets** from the left menu.

Select to create a new **Repository secret**, name it **AZURE_CREDENTIALS**, and paste the response from the service principal creation output. Save the value.

## Create a compute cluster

A service principal is only able to submit jobs that use a compute cluster, not a compute instance. Look at the YAML for the definition. See [challenge2_cluster.yml](challenge2_cluster.yml) for the compute definition.

```bash
az ml compute create -f challenge2_cluster.yml
```

## Modify the job.yml to use the compute cluster instead of the instance

```yml
compute: azureml:diabetes-cluster-cep
```

## Modify the GitHub workflow to execute the job from Challenge 1

Locate the job yaml file by navigating to: `.github\workflows\02-manual-trigger-job.yml`

Add the following steps to execute the job, need to add the resource group and workspace name:

```yml
    - name: Submit ML job
      run: az ml job create --file src/job.yml --resource-group diabetes-dev-rg --workspace-name aml-diabetes-dev
```

## Check in code and run workflow

Workflow must be triggered manually through the actions menu, selecting the submit job workflow.

Access the Machine Learning workspace, the experiment/job name is the same, so dive into it and you'll see a new run queued. Watch it for successful completion. Note that it runs under the Service principal rather than the user's name.
