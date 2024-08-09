---
title: projects
permalink: /projects/
classes: wide
---
## Azure DevOps Self-Hosted Ephemeral Agents

### Providing Ephemeral ADO Self-Hosted Agents running in OpenShift Docker Containers with Customer Metrics Autoscale (KEDA) for Azure Pipelines

| ADO | OpenShift | Docker | KEDA |
| :---: | :---: | :---: | :---: |
| ![image](assets/img/projects/ado.png) | ![image](assets/img/projects/redhat.png) | ![image](assets/img/projects/docker.png) | ![image](assets/img/projects/keda.png) |

The project is using the [KEDA](https://keda.sh/docs/2.14/scalers/azure-pipelines/) scaler for Azure DevOps Pipelines on OpenShift (Customer Metrics Autoscaler) to "link" an ADO Self-Hosted Agent Pool and onboard Self-Hosted ADO Agents when a new Pipeline Job is queued in that Pool.

After the Custom Metrics Autoscaler is enabled in the OpenShift cluster, the following objects need to be created and configured so that the Custom Metrics Autoscaler horizontally scales pods, and containers are onboarded as ADO Self-Hosted Agents in the linked Agent Pool.

- **Trigger Authentication:** The Trigger Authentication object is responsible to authenticate itself to an ADO Agent Pool, where it is using the ADO Job Request API (this is an undocumented API) to listen to any active queued Pipeline Jobs in that Agent Pool. Based on the configuration defined in the ScaledJob object, it will create a new ScaledJob, which in turn it will create a Pod and container, and that container will be onboarded as a Self-Hosted Agent in that Agent Pool. The onboarded Self-Hosted Agent is configured to run only one Pipeline Job (this configuration is defined in the ```start-once.sh``` script). After the Pipeline Job is completed (succeeded, failed, or cancelled), the Self-Hosted Agent is offboarded from the Agent Pool and the underlying container is terminated. The ScaledJob is marked as completed in OpenShift.

- **ScaledJob:** The ScaledJob object defines the template container spec of the linked docker container image (from the namespace's imagestream), the agent pool trigger requirements, and the environment variables passed to the container from the pod. It also defines any reserved cluster resources that the pod and/or the container will need.

OpenShift Pods are created using a Docker image built from the official Microsoft Hosted Agent [Ubuntu:22.04](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2204-Readme.md) and [Ubuntu:24.04](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md) GitHub repo. These images have been "translated" to be build with a Dockerfile in an ADO YAML Pipeline. Upon image built, the images are uploaded to an OpenShift Namespace imagestream where they are made available for use in ScaledJob container spec templates.

### Solution Benefits
- Efficiently update and maintain Azure DevOps container agent container images
- Rolling updates of image updates
- Autoscaling of Ephemeral ADO Agents based on Pipeline parallel jobs
- Auto Onboarding and Offbording Ephemeral ADO Agents per Pipeline Job
- Minimize Agent hydration time to 2 minutes
- OpenShift Cluster operation cost efficiencies
- Efficient utilization of Agent resources with container spec resource request and limits  

### Solution Architecture
The Ephemeral Agent workflow process starts from an ADO Pipeline triggered from within an ADO Project. The Pipeline YAML definition needs to point to the Agent Pool that is linked with the Ephemeral Agents OpenShift Autoscaler.

When an new Pipeline Job is queued in that Agent Pool, the OpenShift Trigger Authentication object will get notified over API. The Trigger Authentication will initiate a new OpenShift Job, from the ScaledJob object's definition, which in turn will create a new Pod and Container from the docker container image in the namespace's imagestream.

Once the new Container is created, it will run the ```start-once.sh``` script, defined as ENTRYPOINT in the Dockerfile definition. This script is responsible to download and export the Agent configuration software from the Microsoft repo, onboard the container as a Self-Hosted Agent in the Agent Pool, and initiate the agent service to listen for queued Pipeline Jobs.

When the Agent's status shows up as **<span style="color:green">"online"</span>**, the Agent will pickup the queued Pipeline Job to run its Tasks. Since the agent service on the container is configured to run only one Pipeline Job, the service will offboard the Agent from the Agent Pool once the queued Pipeline Job is completed (succeeded, failed, or canceled).

A Job Termination signal will also be sent to the container, which in turn will terminate itself and stop the pod.

After this operation is complete, then the OpenShift job will be marked as **<span style="color:green">"Completed"</span>**.

![ea](assets/img/projects/ephemeral-agents.gif)

## ADO & GHEC Log Management Solution

### Providing ADO & GHEC Organization-level Logs in PowerBi through CosmosDB Log Analytics Synapse Connection

| ADO | GHEC | Function App | CosmosDB | PowerBi |
| :---: | :---: | :---: | :---: | :---: |
| ![image](assets/img/projects/ado.png) | ![image](assets/img/projects/github.png) | ![image](assets/img/projects/function-app.png) | ![image](assets/img/projects/cosmos.png) | ![image](assets/img/projects/powerbi.png) |

This solution has been developed to supplement the out-of-the-box log retention and management service of Azure DevOps Services (ADO) and GitHub Enterprise Cloud (GHEC) platforms.

In ADO, Organization-level logs can only be maintained for up to 90 days. Logs dating more than 7 days must be extracted into a .csv or JSON file for further investigation. Project-level logs are not captured by the system.

In GHEC, Organization-level logs can only be maintained for up to 400 days. Displaying and filtering logs accross Organization is a challenging task.

The ADO & GHEC Log Monitor solution allows to overcome the above issues with logging as well as align with the InfoSec's log retainment directive of at least 12 months of Logs.

### Solution Benefits
- Organization, Project/Repo, and Agent/Runner level logs in one place
- Easier retrieval of logs for trends and troubleshooting
- Function representation of logs in queries and dashboard for Admin and Managerial level of access
- Secondary location of log storage on Azure Log Analytics Workspaces
- Log retainment of 12+ months

### Solution Architecture

#### ADO
For retrieving weekly ADO usage statistics, a PowerShell Function with REST API input bindings to ADO Organizations and CosmosDB database output bindings to Document Collections, will run on ADO Project CI/CD Pipelines.  

The CosmosDB Document Collections will be enabled with the Azure Synapse Link services, which will generate T-SQL Views in a Synapse Analytics Workspace SQL Serverless Pool.  

The SQL Serverless Pool will in turn be queried by PowerBi defined Reports and Dashboards through a Synapse Analytics Workspace connection.

#### GHEC
GitHub Enterprise Cloud is capturing logs for all linked Organizations at the root Enterprise level.

For retrieving these logs, a new Audit Log stream of type Azure Event Hub needs to be configured under the Enterprise’s Log Streaming service. 

The Azure Event Hub will be bound to an Azure Event Hub Triggered Function App (Windows/PowerShell) which will be triggered whenever a new Enterprise-level JSON event message is added to the Event Hub. 

The Azure Function will analyze the event message and based on a PowerShell statement, will be sending (POST) each event message as a new JSON document to an out-bounded CosmosDB noSQL Document Database. 

The CosmosDB database will be configured with a Collection for Enterprise-level documents. 

In turn, all the CosmosDB Collection will be enabled with the Azure Synapse Link service, which will generate T-SQL Views in a Synapse Analytics Workspace SQL Serverless Pool.  

The SQL Serverless Pool will in turn be queried by PowerBi defined Reports and Dashboards through a Synapse Analytics Workspace connection.

![lm](assets/img/projects/log-management.png)

## ADO & GHEC Provisoning Solution

### Using Custom ADO Work Item CR to Provision ADO resources (Projects, Teams, Security Groups) and GHEC resources (Repos, Teams)

| ADO | Boards | PowerShell| Function App |
| :---: | :---: | :---: | :---: | 
| ![image](assets/img/projects/ado.png) | ![image](assets/img/projects/boards.png) | ![image](assets/img/projects/powershell.png) | ![image](assets/img/projects/function-app.png) |

The solution allows the provisioning of ADO and GHEC resources from the submission of an ADO Work Item (Change Request - CMMI teamplate).