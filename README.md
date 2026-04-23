<!--
---
name: Durable Functions Python Fan-Out/Fan-In (Durable Task Scheduler)
description: This repository contains a Durable Functions quickstart written in Python demonstrating the fan-out/fan-in pattern. It uses the Azure Durable Task Scheduler (DTS) backend and is deployed to Azure Functions Flex Consumption via the Azure Developer CLI (azd).
page_type: sample
products:
- azure-functions
- azure
- entra-id
urlFragment: starter-durable-fan-out-fan-in-python
languages:
- python
- bicep
- azdeveloper
---
-->

# Durable Functions Fan-Out/Fan-In quickstart - Python (Durable Task Scheduler backend)

This template repository contains a Durable Functions sample demonstrating the fan-out/fan-in pattern in Python (v2 programming model), backed by the **Azure Durable Task Scheduler (DTS)**. The sample can be easily deployed to Azure using the Azure Developer CLI (`azd`). It uses a user-assigned managed identity and can optionally deploy a virtual network.

[Durable Functions](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-overview) orchestrates stateful, long-running, multi-step logic with *durable execution*. State is persisted by a [backend provider](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-storage-providers). This sample uses the **[Azure Durable Task Scheduler](https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler)** provider, a fully managed backend purpose-built for Durable Functions and the Durable Task Framework. It replaces the Azure Storage backend and provides a dedicated dashboard for monitoring orchestrations.

> DTS is currently in preview. The sample uses the preview extension bundle `Microsoft.Azure.Functions.ExtensionBundle.Preview`.

## Prerequisites

+ [Python 3.11+](https://www.python.org/downloads/)
+ [Azure Functions Core Tools v4](https://learn.microsoft.com/azure/azure-functions/functions-run-local?pivots=programming-language-python#install-the-azure-functions-core-tools)
+ [Azure Developer CLI (`azd`)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
+ [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest)
+ [Docker](https://www.docker.com/) (only for the local DTS emulator)
+ To run/debug in Visual Studio Code:
  + [Visual Studio Code](https://code.visualstudio.com/)
  + [Azure Functions extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions)
  + [Python extension](https://marketplace.visualstudio.com/items?itemName=ms-python.python)

## Initialize the local project

You can initialize a project from this `azd` template in one of these ways:

+ Use `azd init` from an empty local folder:

    ```shell
    azd init --template durable-functions-quickstart-python-azd
    ```

+ Or clone directly:

    ```shell
    git clone https://github.com/Azure-Samples/durable-functions-quickstart-python-azd.git
    cd durable-functions-quickstart-python-azd
    ```

## Prepare your local environment

Copy the sample local settings file into place:

```shell
cp src/local.settings.json.sample src/local.settings.json
```

The resulting `src/local.settings.json` already points at the local DTS emulator:

```json
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "python",
        "DURABLE_TASK_SCHEDULER_CONNECTION_STRING": "Endpoint=http://localhost:8080;Authentication=None",
        "TASKHUB_NAME": "default"
    }
}
```

`AzureWebJobsStorage` is still used by the Functions host itself (not for Durable state) — the local storage emulator Azurite covers that.

### Install Python dependencies

From the `src` folder, create a virtual environment and install dependencies:

```bash
cd src
python -m venv .venv
source .venv/bin/activate  # On Windows use: .venv\Scripts\activate
pip install -r requirements.txt
cd ..
```

### Start the Durable Task Scheduler emulator

The DTS emulator runs in Docker. It exposes the DTS endpoint on port **8080** and the DTS dashboard on port **8082**:

```shell
docker run -p 8080:8080 -p 8082:8082 mcr.microsoft.com/dts/dts-emulator:latest
```

Open the dashboard at <http://localhost:8082> to watch orchestrations in real time.

### Start Azurite (for the Functions host)

In a separate terminal, start Azurite so `AzureWebJobsStorage` resolves:

```shell
npx azurite --skipApiVersionCheck --location ~/azurite-data
```

Or:

```shell
docker run -p 10000:10000 -p 10001:10001 -p 10002:10002 mcr.microsoft.com/azure-storage/azurite
```

## Run your app from the terminal

1. From the `src` folder, start the Functions host:

    ```shell
    func start
    ```

    You should see output similar to:

    ```
    Functions:

            http_start:  http://localhost:7071/api/orchestrators/{functionName}

            fetch_orchestration: orchestrationTrigger

            fetch_title: activityTrigger
    ```

1. In another terminal (or your browser), hit the HTTP trigger:
   <http://localhost:7071/api/orchestrators/fetch_orchestration>

    It returns the orchestration instance ID and status URLs. Watch the orchestration progress in the DTS dashboard at <http://localhost:8082>.

1. Press Ctrl+C to stop the Functions host when finished.

## Run your app using Visual Studio Code

1. Open the repository folder in VS Code (`code .`).
1. Ensure the DTS emulator (and Azurite) are running, as described above.
1. Press **Run/Debug (F5)** to start the app in the debugger.
1. Trigger the orchestration with an HTTP request to <http://localhost:7071/api/orchestrators/fetch_orchestration>.

## Source Code

Fanning out is easy with regular functions — just send multiple messages to a queue. Fanning back in is harder because you have to track completion and aggregate results. Durable Functions makes this simple:

```python
@myApp.orchestration_trigger(context_name="context")
def fetch_orchestration(context: df.DurableOrchestrationContext):
    """Orchestrator function that fans out to fetch article titles in parallel."""
    logger = logging.getLogger("fetch_orchestration")
    logger.info("Fetching data.")

    urls = [
        "https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-overview",
        "https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler",
        "https://learn.microsoft.com/azure/azure-functions/functions-scenarios",
        "https://learn.microsoft.com/azure/azure-functions/functions-create-ai-enabled-apps",
    ]

    tasks = [context.call_activity("fetch_title", url) for url in urls]
    results = yield context.task_all(tasks)
    return "; ".join(results)
```

## Deploy to Azure

Deactivate the virtual environment if it is still active:

```shell
deactivate
```

From the repo root, provision all resources (function app, storage, Log Analytics, Application Insights, user-assigned managed identity, **Durable Task Scheduler + task hub**, and role assignments) and deploy your code:

```shell
azd auth login
azd up
```

To skip the virtual-network prompt, pre-set the parameter:

```bash
azd env set VNET_ENABLED false
azd up
```

You'll be prompted for:

| Parameter | Description |
| ---- | ---- |
| _Environment name_ | Unique deployment context for your app. |
| _Azure subscription_ | Subscription where resources are created. |
| _Azure location_ | Region from the DTS-supported allowlist in `infra/main.bicep` (default: `northcentralus`). |

When deployment completes, `azd` prints the function app endpoints. In Azure, the Durable Task Scheduler is wired to the function app via the `DURABLE_TASK_SCHEDULER_CONNECTION_STRING` app setting, which uses the user-assigned managed identity (`Authentication=ManagedIdentity;ClientID=<uami-clientId>`).

## Test deployed app

```shell
az functionapp function list \
  --resource-group <resource-group-name> \
  --name <function-app-name> \
  --query "[].{name:name, url:invokeUrlTemplate}" \
  --output table
```

Then call:

```
https://<function-app-name>.azurewebsites.net/api/orchestrators/fetch_orchestration
```

## Monitor with the DTS dashboard

In Azure, open the deployed Durable Task Scheduler resource (type `Microsoft.DurableTask/schedulers`) in the Azure portal and follow the **Dashboard** link to inspect orchestrations, activities, history, and instance state. Locally, the dashboard is served by the emulator at <http://localhost:8082>.

To find the scheduler name quickly:

```shell
azd show
```

## Redeploy your code

Run `azd up` again any time to re-provision and redeploy. Deployed code is always overwritten by the latest deployment package.

## Clean up resources

```shell
azd down
```

## Troubleshooting

If you see the following transient error after `azd up`, rerun the command:

```
ERROR: error executing step command 'deploy --all': failed deploying service 'api': publishing zip file: deployment failed: [KuduSpecializer] Kudu has been restarted during deployment
```
