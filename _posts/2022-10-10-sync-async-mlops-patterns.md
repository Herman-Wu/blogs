---
title: "Sync-Async tasks pattern in MLOps pipeline"
layout: post
summary: "This article discusses how  ML training duration impacts MLOps pipeline and how to mitigate the impact using sync-async task patterns. It also includes code snippets when using Azure Pipeline and Azure Machine Learning Pipeline."
description: This article discusses how  ML training duration impacts MLOps pipeline and how to mitigate the impact using sync-async task patterns. It also includes code snippets when using Azure Pipeline and Azure Machine Learning Pipeline.
toc: false
comments: true
image: 
hide: false
search_exclude: false
categories: [mlops, machine leanring, devOps, Azure Pipeline, Azure Machine Learning]
---


# 

Machine learning operations (MLOps) process needs to [combine practices in DevOps and machine learning](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/ai-machine-learning-mlops#machine-learning-operations-vs-devops). In a software project, testing and validating pipelines usually take a few hours or less to run. Most software projects could complete their unit tests in a few hours. Pipeline tasks that run synchronously or in parallel are enough in most cases.
But in a machine learning project, the training and validation steps could take a long time to run, from a few hours to a few days. It is not practical to wait for the training to finish before moving on to the next step. So during the MLOps flow design, we need to take different approaches and find a way to combine synchronous and asynchronous steps in order to run the end-to-end training process efficiently.

This article will introduce different practices to implement Sync-Async tasks pattern in the MLOps pipeline using the [Azure pipeline](https://learn.microsoft.com/en-us/azure/devops/pipelines/get-started/what-is-azure-pipelines?view=azure-devops) and [Azure Machine Learning (AML) pipeline](https://learn.microsoft.com/en-us/azure/machine-learning/concept-ml-pipelines).

## Use synchronous task and toy dataset for ML build validation and unit test

During the build validation phase, we want to validate the code quality quickly and ensure we implement the data processing code and the ML algorithm correctly. The algorithm code should be able to train a model successfully using the provided dataset.
To speed up the process, we can use a small toy dataset to reduce the resource and time required for training the model. We can also reduce the training epoch and parameter range to reduce the training time further.
This approach uses synchronous pipeline tasks for preparing data and running the training. Because ML model training time is limited, the task can wait for the training to finish and then move on to the next step.

![synchronous task pattern]({{ site.url }}{{ site.baseurl }}/assets/img/2022-10-10-sync-async-mlops-patterns/ado-aml-sync-task-pattern.png)

Following is the code snippet that submits an ML training using [AML CLI v2](https://learn.microsoft.com/en-us/azure/machine-learning/concept-v2#azure-machine-learning-cli-v2) and waiting for the result in an ADO task.

```yaml
parameters:
  - name: amlJobExecutionScript
    type: string
  - name: amlJobSetCommand
    type: string

steps:
- task: AzureCLI@2
  displayName: Run Azure ML Pipeline and Wait for Results
  inputs: 
    azureSubscription: $(AZURE_RM_SVC_CONNECTION)
    scriptType: bash
    workingDirectory: $(System.DefaultWorkingDirectory)
    scriptLocation: inlineScript
    inlineScript: |
      export AZUREML_CURRENT_CLOUD="AzureCloud" #Choose a different value according to your cloud environement: AzureCloud, AzureChinaCloud, AzureUSGovernment, AzureGermanCloud
      run_id=$(az ml job create -f ${{ parameters.amlJobExecutionScript }} \
        ${{ parameters.amlJobSetCommand }})
      echo "RunID is $run_id"
      if [[ -z "$run_id" ]]
      then
        echo "Job creation failed"
        exit 3
      fi
      az ml job show -n $run_id --web
      status=$(az ml job show -n $run_id --query status -o tsv)
      if [[ -z "$status" ]]
      then
        echo "Status query failed"
        exit 4
      fi
      running=("NotStarted" "Queued" "Starting" "Preparing" "Running" "Finalizing")
      while [[ ${running[*]} =~ $status ]]
      do
        sleep 15 
        status=$(az ml job show -n $run_id --query status -o tsv)
        echo $status
      done
      if [[ "$status" != "Completed" ]]  
      then
        echo "Training Job failed"
        exit 3
      fi

```

Other additional parameters:

- `AZURE_RM_SVC_CONNECTION` - Azure DevOps service connection name.
- `amlJobExecutionScript` - Local path to the YAML file containing the Azure ML job specification.
- `amlJobSetCommand` - Additional Azure ML [Job parameters](https://learn.microsoft.com/en-us/cli/azure/ml/job?view=azure-cli-latest#az-ml-job-create-optional-parameters). For example, `--name training-object-detection` to specify the job name.

## Use asynchronous task for ML model training step in integration test pipeline and production

[MLOps pipeline usually includes multiple steps](https://learn.microsoft.com/en-us/azure/machine-learning/concept-model-management-and-deployment#mlops-in-machine-learning), such as data preprocessing, model training, model evaluation, model registration, and model deployment. Sometimes, we need to run ML training in the integration test and production environments. For example, a defect detection system might want to retrain an ML model using the existing algorithm with a newly updated dataset from a production line. To automate the process, we want to ensure the whole MLOps pipeline can pass the integration test and run correctly in the production environment. However, the model training step could take a long time to finish. We need to use asynchronous tasks to run the model training step and prevent the long waiting time in the main pipeline.
![asynchronous task pattern]({{ site.url }}{{ site.baseurl }}/assets/img/2022-10-10-sync-async-mlops-patterns/mlops-ado-aml-async-task.png)

In Azure DevOps, the Microsoft-hosted agent has a [job time-out limitation](https://learn.microsoft.com/en-us/azure/devops/pipelines/troubleshooting/troubleshooting?view=azure-devops#job-time-out). You can have a job running for the maximum 360 minutes (6 hours). The pipeline will fail if the model training step is longer than the time limitation. There are a few ways to implement an asynchronous pipeline task in Azure DevOps to prevent the problem.

### Use Azure Pipeline REST API task to invoke published Azure ML pipelines and wait for the post-back event

In this approach, you [publish your AML pipeline](https://learn.microsoft.com/en-us/azure/machine-learning/v1/how-to-deploy-pipelines) and get a REST endpoint for the pipeline. And then you can use Azure Pipeline [REST API task](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/http-rest-api?view=azure-devops) to invoke published Azure ML pipelines and wait for the post-back events. To wait for the post-back event, we need to set the `waitForCompletion` attribute of the REST API task to `true`.
![Use Azure Pipeline REST API task to invoke published Azure ML pipelines]({{ site.url }}{{ site.baseurl }}/assets/img/2022-10-10-sync-async-mlops-patterns/ado-aml-async-restapi.png)
The [Invoking Azure ML Pipeline From Azure DevOps](https://github.com/cse-labs/code-with-mlops/blob/main/docs/guidance-and-examples/azure-ml-tips-and-tricks/azure-ml-from-azdo.md) document in this playbook has more detail implementation introduction.

_Limitation: The latest AML CLI/SDK v2 doesn't support AML pipeline web API yet._

### Use an AML component to invoke REST API of another Azure Pipeline

An AML component is a self-contained piece of code that accomplish a task in a machine learning pipeline. It is the building block of am AML pipeline.

In this implementation, we use Azure ML CLI/SDK v2 to submit the AML pipeline job. And in the final step of pipeline job, use an [AML component](https://learn.microsoft.com/en-us/azure/machine-learning/concept-component) to invoke REST API of another [Azure Pipeline](https://learn.microsoft.com/en-us/rest/api/azure/devops/pipelines/runs/run-pipeline?view=azure-devops-rest-6.0) to trigger the next steps.  
![Use an AML component to invoke REST API of another Azure Pipeline]({{ site.url }}{{ site.baseurl }}/assets/img/2022-10-10-sync-async-mlops-patterns/mlops-ado-aml-async-components.png)

Following are the code snippets of an abbreviate reference implementation of the trigger Azure pipeline AML component.

- Trigger Azure Pipeline python code : `ado-pipeline-trigger.py`

```python
import requests
from requests.structures import CaseInsensitiveDict
import os
import argparse
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

def parse_args():
    """Parse input args"""
    # setup arg parser
    parser = argparse.ArgumentParser()

    # add arguments
    parser.add_argument("--modelpath", required=True, type=str, help="Path to input model")
    parser.add_argument("--modelname", required=True, type=str, help="Name of the registered model")
    parser.add_argument("--kvname", required=True, type=str, help="Key Vault Resource Name")
    parser.add_argument("--secretname", required=True, type=str, help="Secret name for ADO Personal Access Token")
    parser.add_argument("--org", required=True, type=str, help="ADO organization Name")
    parser.add_argument("--project", required=True, type=str, help="ADO project Name")
    parser.add_argument("--branch", required=True, type=str, help="ADO repo branch name")
    parser.add_argument("--apiversion", required=True, type=str, help="ADO restful api version")
    parser.add_argument("--pipelineid", required=True, type=str, help="ID of the pipeline you need to trigger")
    parser.add_argument("--pipelineversion", required=True, type=str, help="Pipeline version")

    # parse args
    args = parser.parse_args()
    # return args
    return args

def get_run_id(modelpath):
    """Read run_id from MLmodel"""
    mlmodel_path = os.path.join(modelpath, "MLmodel")
    run_id = ""
    with open(mlmodel_path, "r") as modelfile:
        while(True):
            line = modelfile.readline()
            if not line:
                break
            if "run_id" in line:
                run_id = line.split(':')[1].strip()
                break
    return run_id

def get_secret_value(kv_name, secret_name):
    """Get the secret value from keyvault"""
    kv_uri = f"https://{kv_name}.vault.azure.com"
    credential = DefaultAzureCredential()
    client = SecretClient(vault_url=kv_uri, credential=credential)
    print(f"Retrieving ADO personal access token {secret_name} from {kv_name}.")

    retrieved_secret = client.get_secret(secret_name)
    return retrieved_secret.value

def trigger_pipeline(args):
    """Trigger Azure Pipeline"""
    run_id = get_run_id(args.modelpath)
    secret_value = get_secret_value(args.kvname, args.secretname)
    headers = CaseInsensitiveDict()
    basic_auth_credentials = ('', secret_value)
    headers["Content-Type"] = "application/json"

    request_body = {
        "resources": {
            "repositories": {
                "self": {
                    "refName": args.branch
                }
            }
        },
        "variables": {
            "model_name": {
                "value": args.modelname
            },
            "run_id": {
                "value": run_id
            }
        }
    }

    url = "https://dev.azure.com/{}/{}/_apis/pipelines/{}/runs?pipelineVersion={}&api-version={}".format(
        args.org, args.project, args.pipelineid, args.pipelineversion, args.apiversion
    )
    print(f"url: {url}")
    resp = requests.post(url, auth=basic_auth_credentials, headers=headers, json=request_body)
    print(f"response code {resp.status_code}")
    resp.raise_for_status()


# run script
if __name__ == "__main__":
    # parse args
    args = parse_args()
    # trigger model registration pipeline
    trigger_pipeline(args)
```

- Trigger Azure Pipeline AML component: `component_pipeline_trigger.yaml`

```yaml
{% raw %}
$schema: https://azuremlschemas.azureedge.net/latest/commandComponent.schema.json
name: trigger_pipeline
display_name: trigger Azure pipeline
version: 1
type: command
inputs:
modelpath:
    type: mlflow_model
modelname:
    type: string
kvname:
    type: string
secretname:
    type: string
org:
    type: string
project:
    type: string
branch:
    type: string
apiversion:
    type: string
pipelineid:
    type: integer
pipelineversion:
    type: integer

code: ../../../src/pipeline_trigger/
environment: azureml:sklearn-jsonline-keyvault-env@latest
command: >-
python ado-pipeline-trigger.py
--modelpath ${{inputs.modelpath}}
--modelname ${{inputs.modelname}}
--kvname ${{inputs.kvname}}
--secretname ${{inputs.secretname}}
--org ${{inputs.org}}
--project ${{inputs.project}}
--branch ${{inputs.branch}}
--apiversion ${{inputs.apiversion}}
--pipelineid ${{inputs.pipelineid}}
--pipelineversion ${{inputs.pipelineversion}}
{% endraw %}
```

### Subscribe Azure ML Event Grid events, and use a supported event handler to  trigger another Azure Pipeline

Azure Machine Learning manages the entire lifecycle of machine learning process, during the lifecycle AML will [publish several status events](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-use-event-grid#event-types-for-azure-machine-learning) in Event Grid, such as a completion of training runs event or a registration and deployment of models event. We can use [supported event handler](https://learn.microsoft.com/en-us/azure/event-grid/event-handlers#supported-event-handlers) to subscribe these events and react to them.
![Subscribe Azure ML Event Grid events, and use a supported event handler to  trigger another Azure Pipeline]({{ site.url }}{{ site.baseurl }}/assets/img/2022-10-10-sync-async-mlops-patterns/mlops-ado-aml-async-eventgrid.png)

Here are the supported Azure ML events:

| Event type | Subject format |
| --- |  --- |
| `Microsoft.MachineLearningServices.RunCompleted` | `experiments/{ExperimentId}/runs/{RunId}` |
| `Microsoft.MachineLearningServices.ModelRegistered` | `models/{modelName}:{modelVersion}` |
| `Microsoft.MachineLearningServices.ModelDeployed` | `endpoints/{serviceId}` |
| `Microsoft.MachineLearningServices.DatasetDriftDetected` | `datadrift/{data.DataDriftId}/run/{data.RunId}` |
| `Microsoft.MachineLearningServices.RunStatusChanged` | `experiments/{ExperimentId}/runs/{RunId}` |

For example, to continue the MLOps pipeline when the ML training is finished, we will subscribe `RunCompleted` event and trigger another Azure  Pipeline when the event is published. To trigger another Azure Pipeline, we can use [Azure Automation runbooks](https://learn.microsoft.com/en-us/azure/automation/manage-runbooks),  [Logic Apps](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-overview), or [Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview) and implement the trigger next step code in one of them.

## Use Azure Pipeline Self-host agent to run the AML pipeline

In this approach, we will use [Azure Pipeline Self-host agent](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser#install) to run the AML pipeline . Because there is no  [job time-out limitation](https://learn.microsoft.com/en-us/azure/devops/pipelines/troubleshooting/troubleshooting?view=azure-devops#job-time-out) for self-hosted agent, it can trigger an AML training task and wait for the training to finish, then moving on to the next step.

![Use Azure Pipeline Self-host agent to run the AML pipeline]({{ site.url }}{{ site.baseurl }}/assets/img/2022-10-10-sync-async-mlops-patterns/mlops-ado-aml-selfhostagent.png)

This approach uses synchronized tasks. However, we will need to install the agent and maintain the environment that runs the agent.

## Summary

Some MLOps-related tasks have a different nature compared to DevOps focus tasks. It will impact the tool you could use and the decision of using which tool. This article highlights how the ML training duration affects the task pipeline in the MLOps process. This difference will be more obvious when you include traditional DevOps tasks such as unit testing and integration testing in your MLOps process. This article provides some strategies the operation team could use to mitigate the issue. It also includes code snippets when using Azure Pipeline and Azure Machine Learning Pipeline.



## Reference

- [MLOps for Python with Azure Machine Learning - Azure Architecture Center | Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/ai/mlops-python)

- [MLOps: Machine learning model management - Azure Machine Learning | Microsoft Learn](https://learn.microsoft.com/en-us/azure/machine-learning/concept-model-management-and-deployment)

- [System design patterns for machine learning](https://mercari.github.io/ml-system-design-pattern/)
