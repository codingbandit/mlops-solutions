# Challenge 6: Deploy and test the model

## Create managed endpoint in production

This expects your CLI has the production resource group and workspaces set as default. We are creating this endpoint outside of the GitHub action for deployment because we want the resulting URL to remain the same for consumers of the service. We will issue the deployment to the endpoint from the GitHub action in the next step. Note: this command takes a few minutes to run.

1. Create the file [challenge6_managed_endpoint.yml](challenge6_managed_endpoint.yml) (source in-file).
2. Create the online managed endpoint:

    ```bash
    az ml online-endpoint create --name diabetes-prod -f challenge6_managed_endpoint.yml
    ```

## Create Deployment GitHub Action workflow

This GitHub Action will register the model from the latest job run and deploy it to the managed endpoint. Note: The deployment task can take a few minutes!

1. Create the file [challenge6_model_deployment.yml](challenge6_model_deployment.yml) (source in-file).
2. Create new file `06-deploy-latest-model.yml` in the **.github\workflows** folder.

    ```yml
    name: Deploy model
    
    on:
      workflow_dispatch:
    
    jobs:
      deployProd:
        runs-on: ubuntu-latest
        environment:
            name: Prod 
        steps:
        - name: Check out repo
          uses: actions/checkout@main
        - name: Install az ml extension
          run: az extension add -n ml -y
        - name: Azure login
          uses: azure/login@v1
          with:
            creds: ${{secrets.AZURE_CREDENTIALS}}
        - name: Set latest ML job name 
          run: echo "JOB_NAME=$(az ml job list --query '[0].name' --resource-group diabetes-prod-rg --workspace-name aml-diabetes-prod | tr -d '\"')" >> $GITHUB_ENV
        - name: Echo job name
          run: |
              echo "Latest job name is: $JOB_NAME"
        - name: Set model path
          run: echo 'PATH_URI="azureml://jobs/${{env.JOB_NAME}}/outputs/artifacts/model"' >> $GITHUB_ENV
        - name: Echo path URI
          run: |
              echo "Model path is: $PATH_URI"
        - name: Register model from latest job
          run: az ml model create --name "diabetes-model" --type "mlflow_model" --path ${{env.PATH_URI}} --resource-group diabetes-prod-rg --workspace-name aml-diabetes-prod
        - name: Create deployment for endpoint
          run: az ml online-deployment create --name diabetes-model-deployment --endpoint diabetes-prod -f challenge6_model_deployment.yml --resource-group diabetes-prod-rg --workspace-name aml-diabetes-prod --all-traffic
    ```

3. Commit the new workflow. Trigger this workflow manually.
4. Once deployment completes, go into Azure Machine Learning Studio and execute a test or two using data from the Challenge!

    Using AML Studio, go to the endpoint, select **Test**. Each of these should result in 1 responses for the first 3 and 0 for the last.

    ```json
    {
      "input_data": [
        [
          9,104,51,7,24,27.36983156,1.350472047,43
        ],
        [
          6,73,61,35,24,18.74367404,1.074147566,75
        ],
        [
          4,115,50,29,243,34.69215364,0.741159926,59
        ],
        [
          3,94,96,31,36,21.29447943,0.259020482,23
        ]
      ]
    }
    ```

    Select the **Consume** tag and you can see the key and REST endpoint as well as sample source code showing how to call them. If using tools such as Postman, the API key is the Bearer token. Ensure you also set the body to **raw** and **JSON**.
