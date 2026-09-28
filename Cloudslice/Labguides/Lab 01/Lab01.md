# Lab 1: Build and Deploy the Contoso Air AI Travel App on AKS Automatic

**Scenario**

**Contoso Air** is a growing airline that wants to give its customers a
modern, AI-powered travel experience. The company has built a web
application that lets travelers search flights, manage bookings, and
chat with an **AI travel assistant** to get personalized trip
recommendations.

The development team currently spends significant time managing
Kubernetes infrastructure — sizing node pools, configuring monitoring,
wiring up autoscaling, and managing secrets for connecting to Azure
services. These manual operations slow down releases and make it
difficult for developers to focus on building features.

To simplify its platform, Contoso Air has adopted **Azure Kubernetes
Service (AKS) Automatic**, a fully managed Kubernetes experience that
comes with production-ready defaults for security, monitoring, and
scaling. The team wants to use **AKS Automated Deployments** with
**GitHub Actions** to build and ship the application, and connect it
securely to **Azure OpenAI** without storing any passwords or keys.

As a Cloud/DevOps Engineer, your responsibility is to prepare the Azure
environment, create an AKS Automatic cluster and a CI/CD pipeline,
secure GitHub authentication with federated credentials, connect the
application to Azure OpenAI using Service Connector and Workload
Identity, enable monitoring with Application Insights, Container
Insights and Grafana, and configure autoscaling with VPA and KEDA.

By implementing this solution, Contoso Air can ship features faster,
run its AI travel assistant securely at scale, and gain full visibility
into application and cluster health.

**Introduction**

In this lab, you will deploy the **Contoso Air** application to an
**AKS Automatic** cluster. AKS Automatic is a fully managed Kubernetes
service that simplifies the deployment, management, and operations of
Kubernetes clusters. You deploy a cluster with just a few steps in the
Azure portal and focus on building your applications.

Throughout the lab, you will explore how AKS Automatic combines
Automated Deployments, Service Connector, Workload Identity, Azure
Monitor, Managed Prometheus, Grafana, Node Autoprovisioning, VPA and
KEDA into a single, opinionated platform for running cloud-native and
AI-enabled applications.

**Architecture**

| **Component** | **Role in the lab** |
|----|----|
| GitHub fork of contoso-air | Source code and GitHub Actions workflow |
| Entra app registration workflowapp-\<number\> | Identity GitHub Actions uses to sign in to Azure |
| Azure Container Registry (ACR) | Stores the contoso-air container image |
| AKS Automatic cluster myakscluster | Runs the app in namespace dev |
| Managed identity myidentity\<xxx\> | Identity the app pod uses to call Azure OpenAI |
| Azure OpenAI myopenai\<xxx\> | LLM behind the AI travel assistant |
| Log Analytics / Azure Monitor workspace | Logs and Prometheus metrics |
| Application Insights myappinsights\<xxx\> | App performance telemetry |

**Objectives**:

- Prepare the Azure environment and deploy the prerequisite resources.

- Create an AKS Automatic cluster and a CI/CD pipeline with Automated
  Deployments.

- Secure GitHub Actions authentication with federated credentials.

- Integrate the application with Azure OpenAI using Service Connector
  and Workload Identity.

- Observe the application and cluster with Application Insights,
  Container Insights, Logs and Grafana.

- Scale the application with the Vertical Pod Autoscaler (VPA) and KEDA.

**Prerequisites**

- **GitHub Account**: You are expected to have your own GitHub login
  credentials. If you do not have an account, create one by visiting:
  +++https://github.com/signup+++

## Exercise 1: Set up the lab environment

Before creating the cluster, you need to prepare the Azure environment.
In this exercise, you will sign in to Azure Cloud Shell, register the
required providers, deploy the prerequisite resources, and fork the
sample application.

### Task 1: Prepare the Azure environment and register providers

1.  Open your browser, navigate to the address bar, and type or paste
    the following URL: +++https://portal.azure.com/+++ then press the
    **Enter** button. Sign in with the following credentials.

| **Username** | **+++@lab.CloudPortalCredential(User1).Username+++** |
|----|----|
| **Password** | **+++@lab.CloudPortalCredential(User1).AccessToken+++** |

2.  In the Azure portal, select the **Cloud Shell** icon from the top
    navigation bar to open Azure Cloud Shell.

![](./media/image1.png)

3.  In the **Welcome to Azure Cloud Shell** dialog, select **Bash** to
    launch a Bash session.

![](./media/image2.png)

4.  Choose **No storage account required**, select your subscription,
    and then click on the **Apply** button.

![](./media/image3.png)

5.  In Azure Cloud Shell, run the following command. Then open the
    displayed URL (+++https://login.microsoftonline.com/device+++) in a
    browser, enter the **device code** shown in the terminal, and
    complete the **sign-in** process using your Azure account.

> +++az login --use-device-code+++

![](./media/image4.png)

![](./media/image5.png)

![](./media/image6.png)

![](./media/image7.png)

6.  Select the Azure subscription and tenant from the list, enter **1**
    to choose the displayed option, and then press **Enter**.

![](./media/image8.png)

**Tip:** You can log into a different tenant by passing the
**--tenant** flag to specify your tenant domain or tenant ID.

7.  Run the following command to add the AKS preview extension.

> +++az extension add --name aks-preview+++

![](./media/image9.png)

8.  Run the following command to register the Application Monitoring
    preview feature, and confirm that the feature state is displayed as
    **Registered** in the output.

> +++az feature register --namespace Microsoft.ContainerService --name AzureMonitorAppMonitoringPreview+++

![](./media/image10.png)

9.  Run the following commands to register the resource providers.

> +++az provider register --namespace Microsoft.DevHub+++
>
> +++az provider register --namespace Microsoft.Insights+++
>
> +++az provider register --namespace Microsoft.PolicyInsights+++
>
> +++az provider register --namespace Microsoft.ServiceLinker+++

![](./media/image11.png)

### Task 2: Create the resource group

In this task, you set environment variables for the resource group name
and location. To keep the resource names unique, you use a random number
as a suffix. This helps you avoid naming conflicts with other resources
in your Azure subscription.

1.  Run the following commands to generate a random number.

```bash
RAND=$RANDOM
export RAND
echo "Random resource identifier will be: ${RAND}"
```

![](./media/image12.png)

2.  Set the location to a region of your choice, for example **eastus**
    or **westeurope**. Make sure the region supports [availability
    zones](https://learn.microsoft.com/azure/aks/availability-zones-overview).
    Then create a resource group name using the random number.

```bash
export LOCATION=eastus
export RG_NAME=myresourcegroup$RAND
```

3.  Run the following command to create the resource group.

```bash
az group create \
  --name ${RG_NAME} \
  --location ${LOCATION}
```

![](./media/image13.png)

### Task 3: Get your user object ID

1.  The template gives your account access to Key Vault and Azure
    OpenAI, so it needs your user object ID. If this value is empty, the
    deployment fails with **InvalidPrincipalId**.

```bash
export USER_ID=$(az ad signed-in-user show --query id -o tsv)
echo "USER_ID=$USER_ID"
```

![](./media/image14.png)

2.  If nothing is printed, use the following instead (it reads the ID
    from your sign-in token).

```bash
export USER_ID=$(az account get-access-token --query accessToken -o tsv | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin)['oid'])")
echo "USER_ID=$USER_ID"
```

![](./media/image15.png)

3.  Save the environment variables to a file, and verify that they are
    available for the next commands.

```bash
echo "export LOCATION=$LOCATION RG_NAME=$RG_NAME USER_ID=$USER_ID" > ~/aks-lab.env
cat ~/aks-lab.env
```

**Note**: If Cloud Shell times out, run +++source ~/aks-lab.env+++ to
restore the variables.

### Task 4: Deploy the resources

To keep the focus on AKS-specific features, this lab requires several
Azure resources to be pre-provisioned, including:

- Azure Log Analytics Workspace for container insights and application
  insights

- Azure Monitor Workspace for Prometheus metrics

- Azure Container Registry for storing container images

- Azure Key Vault for secrets management

- Azure User-Assigned Managed Identity for accessing Azure services via
  Workload Identity

- Azure Application Insights for application monitoring

- Azure OpenAI Service with a GPT model deployment for the AI chat
  feature

1.  Run the following command to save your user object ID to a
    variable.

> +++export USER_ID=$(az ad signed-in-user show --query id -o tsv)+++

2.  Run the following command to deploy the ARM template into the
    resource group.

```bash
az deployment group create \
  --resource-group ${RG_NAME} \
  --name ${RG_NAME}-deployment \
  --template-uri https://raw.githubusercontent.com/azure-samples/aks-labs/refs/heads/main/docs/getting-started/aks-automatic/assets/main.json \
  --parameters userObjectId=${USER_ID}
```

3.  If the deployment fails on the Application Insights resource (API
    version error), download the template, patch the API version, and
    deploy the local copy instead.

```bash
curl -o main.json https://raw.githubusercontent.com/azure-samples/aks-labs/refs/heads/main/docs/getting-started/aks-automatic/assets/main.json

python3 - <<'EOF'
import json
with open('main.json') as f:
    tpl = json.load(f)
for r in tpl['resources']:
    if r.get('type') == 'Microsoft.Insights/components':
        r['apiVersion'] = '2020-02-02'
with open('main.json', 'w') as f:
    json.dump(tpl, f, indent=2)
EOF

az deployment group create \
  --resource-group ${RG_NAME} \
  --name ${RG_NAME}-deployment \
  --template-file main.json \
  --parameters userObjectId=${USER_ID}
```

4.  Wait until the resources are deployed. This can take around **15
    minutes**.

![](./media/image16.png)

![](./media/image17.png)

![](./media/image18.png)

5.  In the Azure portal (+++https://portal.azure.com/+++), open your
    resource group and verify that all required resources have been
    deployed successfully, including Application Insights, Managed
    Identity, Key Vault, Log Analytics workspace, Azure OpenAI, Azure
    Monitor workspace, and Container Registry.

![](./media/image19.png)

**Note:** Keep your terminal open as you will need it to run commands
throughout the lab.

### Task 5: Fork and clone the sample app

1.  In the Bash shell, run the following command, then follow the
    instructions printed in the terminal to complete the login process.

> +++gh auth login+++

![](./media/image20.png)

2.  Choose **HTTPS** as the preferred protocol for Git operations, enter
    **Y** to authenticate with your GitHub credentials, and then press
    **Enter**.

![](./media/image21.png)

![](./media/image22.png)

3.  Select **Login with a web browser**, and then press **Enter**.

![](./media/image23.png)

4.  Copy the one-time device code, press **Enter** to open
    +++https://github.com/login/device+++ in your browser, enter the
    code, and complete the GitHub authentication process.

![](./media/image24.png)

![](./media/image25.png)

![](./media/image26.png)

![](./media/image27.png)

![](./media/image28.png)

5.  Run the following command to fork the **contoso-air** repository to
    your GitHub account, and verify that the fork is created under your
    GitHub repositories.

> +++gh repo fork Azure-Samples/contoso-air --clone --default-branch-only+++

![](./media/image29.png)

6.  Change into the **contoso-air** directory.

> +++cd contoso-air+++

7.  Set the default repository to your forked repository.

> +++gh repo set-default+++

8.  When prompted, select **your fork** of the repository and press
    **Enter**. Do not select the original **Azure-Samples/contoso-air**
    repository.

![](./media/image30.png)

9.  Configure your Git identity by running the following commands,
    replacing the placeholder values with your GitHub username and email
    address.

> +++git config user.name "\<your GitHub user\>"+++
>
> +++git config user.email "\<your email\>"+++

![](./media/image31.png)

10. Save your **GitHub username** and **email address**, as they will be
    required in later tasks.

### Task 6: Record your GitHub IDs

1.  GitHub identifies your repository using numeric IDs. Run the
    following command (replace **\<your-user\>** with your GitHub
    username), then note and save the **Owner ID** and **Repo ID**
    values, as they will be required later in this lab.

> +++gh api repos/\<your-user\>/contoso-air --jq '"Owner ID: \\(.owner.id)  Repo ID: \\(.id)"'+++

![](./media/image32.png)

## Exercise 2: Create the cluster and pipeline

In this exercise, you use AKS Automated Deployments to create an AKS
Automatic cluster and a GitHub Actions pipeline that builds and deploys
the Contoso Air application.

### Task 1: Configure Automated Deployments for Azure Kubernetes Service (AKS)

1.  Open your browser and sign in to the Azure portal at
    +++https://portal.azure.com/+++.

2.  Type +++Kubernetes services+++ in the search box at the top of the
    page, and select **Kubernetes services** from the search results.

![](./media/image33.png)

3.  In the upper left portion of the screen, click on the **+ Create**
    button, and then select the **Deploy application** option.

![](./media/image34.png)

4.  In the **Basics** tab, select the **Deploy your application**
    option, then select your Azure subscription and the resource group
    you created during the lab environment setup.

![](./media/image35.png)

5.  In the **Basics** tab, enter the following details to configure the
    application deployment settings.

| **Subscription** | Select your Azure subscription |
|----|----|
| **Resource group** | Select **myresourcegroupXXXX** |
| **Region** | **(US) East US** or the same region as your resource group |
| **Workflow name** | **+++contoso-air+++** |
| **Repository location** | **GitHub** |

6.  Click on **Authorize access** to grant Azure permission to create
    the deployment workflow in your GitHub repository.

![](./media/image36.png)

![](./media/image37.png)

![](./media/image38.png)

7.  Enter the following repository details, and then click on the
    **Next** button.

| **Repository source** | **My repositories** |
|----|----|
| **Repository** | **contoso-air** |
| **Branch** | **main** |

![](./media/image39.png)

8.  In the **Application** tab, complete the **Image** section with the
    following details:

- **Container configuration**: Select **Existing Dockerfile**

- **Dockerfile**: Select the **Select** link, browse to the
  **./src/web** directory, select the Dockerfile, then click on the
  **Select** button

- **Dockerfile build context**: Enter +++./src/web+++

- **Azure Container Registry**: Select the Azure Container Registry in
  your resource group

- **Azure Container Registry image**: Select the **Create new** link,
  enter +++contoso-air+++ and click on **Ok**

![](./media/image40.png)

![](./media/image41.png)

![](./media/image42.png)

9.  In the **Deployment configuration** section, enter the following
    details:

- **Deployment options**: Select **Generate application deployment
  files**

- **Save files in repository**: Select the **Select** link, select the
  checkbox next to the **Root** folder, then click on **Select**

- **Application port**: Enter +++3000+++

- Click on **Next**

![](./media/image43.png)

![](./media/image44.png)

![](./media/image45.png)

10. In the **Cluster configuration** section, ensure the **Create
    Automatic Kubernetes cluster** option is selected and enter
    +++myakscluster+++ as the **Kubernetes cluster name**.

![](./media/image46.png)

11. For **Namespace**, select **Create new**, enter +++dev+++ as the
    namespace name, click on **OK**, and verify that the **dev**
    namespace is selected.

![](./media/image47.png)

12. Review the monitoring settings, verify that **Container Logs**,
    **Prometheus metrics**, and the associated **Log Analytics
    Workspace** and **Azure Monitor Workspace** are configured correctly,
    and then click on **Next**.

![](./media/image48.png)

13. After the validation passes, click on the **Deploy** button.

![](./media/image49.png)

**Important:** The deployment can take **up to 20 minutes** to
complete. Do not close the browser window or navigate away from the page
until the deployment finishes.

![](./media/image50.png)

![](./media/image51.png)

14. Verify that the cluster and application deployment completed
    successfully. The page shows an **Approve pull request** button for
    the GitHub Actions workflow created for the application.

![](./media/image52.png)

15. **Do not approve or merge the pull request yet.** You first secure
    GitHub authentication in the next task.

![](./media/image53.png)

### Task 2: Secure GitHub authentication before merging changes

1.  In the Azure portal (+++https://portal.azure.com/+++), type
    +++App registrations+++ in the search box at the top of the page,
    and select **App registrations** from the search results.

![](./media/image54.png)

2.  Open **workflowapp-\<number\>** created today, and note its name.

![](./media/image55.png)

3.  Under **Manage**, select **Certificates & secrets**, open the
    **Federated credentials** tab, and then click on **Add credential**.

![](./media/image56.png)

4.  For **Federated credential scenario**, select **GitHub Actions
    deploying Azure resources**, and enter the following details.

| **Field** | **Value** |
|----|----|
| Issuer | Leave https://token.actions.githubusercontent.com |
| Organization | Your GitHub user name (saved in *Exercise 1 \> Task 5 \> Step 10*) |
| Organization ID | Owner ID (saved in *Exercise 1 \> Task 6*) |
| Repository | +++contoso-air+++ |
| Repository ID | Repo ID (saved in *Exercise 1 \> Task 6*) |
| Entity type | Branch |
| GitHub branch name | +++main+++ |
| Name | +++github-main-immutable+++ |
| Audience | Leave api://AzureADTokenExchange |

5.  Scroll to **Subject identifier** (read-only). It must read
    **repo:\<user\>@\<ownerId\>/contoso-air@\<repoId\>:ref:refs/heads/main**.
    Click on the **Add** button.

![](./media/image57.png)

![](./media/image58.png)

![](./media/image59.png)

### Task 3: Review, merge, and deploy application changes

1.  Back on the Automated Deployments page, click on **Approve pull
    request** (or open **Pull requests** in your GitHub fork).

![](./media/image53.png)

2.  In the pull request, select the **Files changed** tab to review the
    workflow and configuration updates.

![](./media/image60.png)

3.  Scroll down to the **manifests/deployment.yaml** file to review the
    generated Kubernetes deployment manifest. This manifest includes the
    **SYS_PTRACE** capability, which should be removed for security
    hardening.

![](./media/image61.png)

![](./media/image62.png)

4.  Hover over the line, select the **+** icon, choose **Add a
    suggestion**, delete the suggested line so that the suggestion is
    empty, and then click on **Comment**.

![](./media/image63.png)

![](./media/image64.png)

5.  Click on **Commit suggestions**.

![](./media/image65.png)

![](./media/image66.png)

6.  Select the **Conversation** tab, review the comments and changes,
    and then click on **Merge pull request**.

![](./media/image67.png)

7.  Click on the **Confirm merge** button.

![](./media/image68.png)

8.  With the pull request merged, the changes are automatically deployed
    to your AKS cluster. Select the **Actions** tab in your GitHub
    repository to view the deployment logs.

![](./media/image69.png)

![](./media/image70.png)

9.  In the **Actions** tab, select the running **Automated Deployments**
    workflow to view the logs.

![](./media/image71.png)

10. After 5-10 minutes, the workflow completes and you will see green
    check marks next to the **buildImage** and **deploy** jobs. This
    means that the application has been successfully deployed to your
    AKS cluster.

![](./media/image72.png)

![](./media/image73.png)

### Task 4: Verify the application deployment in the dev namespace

1.  In the **Azure portal**, navigate to **myakscluster**.

![](./media/image74.png)

2.  Select **Kubernetes resources** \> **Services and ingresses** to
    view the services exposed by the cluster, and verify that the
    application is running in the **dev** namespace.

![](./media/image75.png)

3.  Locate the **contoso-air** service in the **dev** namespace, and
    select its **External IP** address to open the deployed application.

![](./media/image76.png)

![](./media/image77.png)

4.  Test the application's chat functionality by clicking on the **Ask
    the AI travel assistant** button.

![](./media/image78.png)

5.  Attempt to interact with the AI assistant and you'll find that the
    **chat provider is not detected**. You fix this in the next
    exercise.

![](./media/image79.png)

![](./media/image80.png)

## Exercise 3: Integrating apps with Azure services

AKS Service Connector streamlines connecting applications to Azure
resources like Azure OpenAI by automating the configuration of Workload
Identity. It assigns identities to pods, enabling them to authenticate
with Microsoft Entra ID and access Azure services securely without
passwords.

### Task 1: Application configuration

A ConfigMap is a Kubernetes resource that stores non-confidential
configuration data as key-value pairs. Applications running in pods can
read these values as environment variables, making it easy to change
application behavior without rebuilding the container image.

1.  In the **Azure portal**, navigate to **myakscluster**, select
    **Kubernetes resources** \> **Configuration**, and choose +++dev+++
    from the **Filter by namespace** list.

![](./media/image81.png)

2.  Select the **contoso-air-config** config map.

![](./media/image82.png)

3.  Select the **YAML** tab, add the following configuration to the end
    of the file, and then click on **Review + save**.

```yaml
data:
  AZURE_OPENAI_API_VERSION: 2024-12-01-preview
  AZURE_OPENAI_DEPLOYMENT: gpt-5.4-mini
  CHAT_PROVIDER: azure
  LOG_CHAT: 'true'
```

![](./media/image83.png)

4.  Review the changes, verify that the Azure OpenAI settings are
    correct, select **Confirm manifest changes**, and then click on
    **Save**.

![](./media/image84.png)

![](./media/image85.png)

**Info:** This configmap was created by the Automated Deployments
workflow earlier. The new settings instruct the application to use Azure
OpenAI as the chat provider and specify the model deployment name. The
contoso-air application is already configured to read these settings
and inject them into the application environment.

### Task 2: Service Connector setup

1.  In the cluster menu, select **Service Connector** under
    **Settings**, then click on the **+ Create** button.

![](./media/image86.png)

2.  In the **Basics** tab, enter the following details, and then click
    on **Next: Authentication**.

| **Kubernetes namespace** | +++dev+++ |
|----|----|
| **Service type** | **OpenAI Service** |
| **OpenAI** | Select the Azure OpenAI account you created earlier |

![](./media/image87.png)

3.  In the **Authentication** tab, select the **Workload Identity**
    option and expand the **Advanced** section.

![](./media/image88.png)

4.  By default, the **Cognitive Services OpenAI Contributor** role is
    assigned, granting the workload identity permissions to authenticate
    and interact with your LLM. You'll also notice additional
    configuration that Service Connector sets as environment variables
    in the application. These variables are saved to a Kubernetes Secret
    which is used to configure the connection to the Azure OpenAI
    account.

5.  Click on **Next: Networking**, then **Next: Review + create**, and
    finally click on **Create**.

![](./media/image89.png)

![](./media/image90.png)

![](./media/image91.png)

![](./media/image92.png)

### Task 3: Configure the application for Workload Identity

Once you've set up the Service Connector for your Azure OpenAI account,
configure your application to use these connection details.

1.  In the **Service Connector** page, select the **checkbox** next to
    the **OpenAI** connection and click on the **Yaml snippet** button.

![](./media/image93.png)

![](./media/image94.png)

2.  In the **YAML snippet** window, select **Kubernetes Workload** for
    **Resource type**, then select **contoso-air** for **Kubernetes
    Workload**.

![](./media/image95.png)

3.  You will see the YAML manifest for the contoso-air application with
    the edits required to connect to Azure OpenAI via Workload Identity
    highlighted.

4.  Scroll through the YAML manifest to view all the changes highlighted
    in yellow, then click on **Apply**. This redeploys the contoso-air
    application with the new connection details.

![](./media/image96.png)

![](./media/image97.png)

![](./media/image98.png)

**Note:** This applies changes directly to the application deployment.
Ideally, you would commit these changes to your repository so that they
are versioned, tracked, and automatically deployed by the Automated
Deployments workflow you set up earlier.

5.  Wait a minute or two for the new pod to be rolled out, then navigate
    back to the application and interact with the AI assistant. You
    should now be able to chat with the AI assistant without any errors.

![](./media/image99.png)

![](./media/image100.png)

## Exercise 4: Observing your cluster and apps

Monitoring and observability are key components of running applications
in production. AKS Automatic provides many monitoring and observability
features out-of-the-box.

At the start of the lab, you integrated the AKS Automatic cluster with
Azure Log Analytics Workspace for logging and Azure Monitor Managed
Workspace for metrics collection. AKS also provides built-in Grafana
dashboards for data visualization. You can also enable the Azure Monitor
Application Insights for AKS feature to automatically instrument your
applications.

### Task 1: App monitoring

Azure Monitor Application Insights is an Application Performance
Management (APM) solution for real-time monitoring of your applications.
It uses OpenTelemetry (OTel) to collect telemetry data and stream it to
Azure Monitor. With AKS, the AutoInstrumentation feature collects
telemetry without requiring any code changes.

1.  Navigate to the **Monitor** blade of your AKS cluster in the Azure
    portal, and then select **Monitor Settings**.

![](./media/image101.png)

2.  Scroll down to the **Application monitoring (preview)** section,
    select the **Enable** checkboxes for both auto-instrumentation and
    OpenTelemetry data collection, and then click on the **Next**
    button.

![](./media/image102.png)

3.  Review the changes and click on the **Enable** button.

![](./media/image103.png)

4.  The onboarding process takes about 5-6 minutes to complete. Wait for
    the process to complete before moving on to the next steps.

![](./media/image104.png)

![](./media/image105.png)

![](./media/image106.png)

5.  Navigate to **Workloads** under **Kubernetes resources**. Filter by
    the +++dev+++ namespace and select the **contoso-air** deployment.

![](./media/image107.png)

6.  Select the **Application Monitoring (Preview)** tab to configure
    AutoInstrumentation for the contoso-air application.

![](./media/image108.png)

![](./media/image109.png)

7.  In the **Configure Application Monitoring** pane, enter the
    following details:

- **Application Insights**: Select the Application Insights resource in
  your resource group

- **Application Language**: Select **NodeJS auto-instrumentation for
  all deployments**

- Check the **Perform rollout restart of all deployments in the
  namespace to deploy changes immediately** checkbox

- Click on **Configure**

![](./media/image110.png)

![](./media/image111.png)

**Tip:** This is a simple example of instrumenting your application
across an entire namespace. You can also instrument individual
deployments by deploying custom resources into your cluster. See the
[documentation](https://learn.microsoft.com/azure/azure-monitor/app/kubernetes-codeless#mixed-mode-onboarding)
for more details.

![](./media/image112.png)

8.  Verify that the application monitoring deployment has completed
    successfully, and then click on **Close**.

![](./media/image113.png)

9.  Navigate back to the contoso-air application in your browser and
    chat with the AI assistant to generate some telemetry. Then navigate
    to the **Application Insights** resource in your resource group.

![](./media/image114.png)

10. Select **Application map** under **Investigate** to view a
    high-level overview of the application components, their
    dependencies, and number of calls.

![](./media/image115.png)

**Note:** If the Azure OpenAI endpoint does not appear in the
application map, return to the Contoso Air website and chat with the AI
assistant a bit more to generate data. Then, in the Application Map,
click on the **Refresh** button. The map should now display the
application connected to the Azure OpenAI endpoint, along with the
request latency to the model endpoint.

![](./media/image116.png)

![](./media/image117.png)

11. Select **Live Metrics** to view incoming and outgoing requests,
    response times, and exceptions in real-time.

![](./media/image118.png)

12. Select **Performance** to view the average response time, request
    rate, and failure rate for the application.

![](./media/image119.png)

### Task 2: Cluster monitoring

AKS Automatic simplifies monitoring your cluster using [Container
Insights](https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-overview).
It gathers and analyzes logs, metrics, and events from your cluster and
applications, providing insights into their performance and health.

1.  In the Azure portal, navigate to **myakscluster**, select **Monitor
    (Insights)**, open the **More options (...)** menu, and then select
    **Recommended alerts**.

![](./media/image120.png)

The AKS Automatic cluster is pre-configured with basic CPU and memory
utilization alerts. You can also create additional alerts based on the
metrics collected by the Prometheus workspace.

2.  Expand the **Prometheus community alert rules (Preview)** section to
    see the available Prometheus alert rules. You can enable any of these
    alerts by selecting the toggle switch.

3.  Click on **Save** to enable the alerts.

![](./media/image121.png)

![](./media/image122.png)

### Task 3: Workbooks and logs

With Container Insights enabled, you can query logs using Kusto Query
Language (KQL) and create custom or pre-configured workbooks for data
visualization.

1.  In the **Monitoring** section of the AKS cluster menu, select
    **Workbooks**. The **Cluster Optimization** workbook is particularly
    useful for identifying anomalies, detecting probe failures, and
    optimizing container resource requests and limits.

![](./media/image123.png)

![](./media/image124.png)

**Note:** The workbook visuals include a query button that you can
select to view the KQL query that powers the visual.

2.  Select **Logs** from the **Monitoring** section in the AKS cluster
    menu.

![](./media/image125.png)

**Note:** If you are presented with a **Welcome to Log Analytics**
pop-up, close it to access the query editor.

3.  Close the **Queries hub** pop-up, then change the query editor from
    **Simple mode** to **KQL mode** using the drop-down menu in the
    top-right corner.

![](./media/image126.png)

You should see log messages in the table below. Expand some of the logs
to view the message details generated by the application.

**Note:** Some of the queries might not have enough data to return
results.

### Task 4: Dashboards with Grafana

If you prefer to visualize the data using Grafana, or execute complex
queries using PromQL, you can use the **Dashboards with Grafana**
feature built into AKS.

**Info:** **Dashboards with Grafana** is a free Grafana experience
integrated directly into the AKS portal. It does not require a separate
Azure Managed Grafana resource.

1.  In the AKS cluster menu, select **Dashboards with Grafana** under
    the **Monitoring** section.

![](./media/image127.png)

2.  Sign in to the Grafana instance, then select the **Dashboards** link
    in the menu.

3.  In the **Dashboards** list, expand the **Azure Managed Prometheus**
    folder and explore the available dashboards.

4.  Select the **Kubernetes / Compute Resources / Workload** dashboard.

![](./media/image128.png)

5.  Filter the **namespace** to **dev**, the **type** to
    **deployment**, and the **workload** to **contoso-air**.

![](./media/image129.png)

![](./media/image130.png)

### Task 5: Querying metrics with PromQL

1.  Select **Dashboards** to navigate back to the dashboards list, and
    select the **Explore** tab at the top of the pane.

![](./media/image131.png)

2.  Select the data source dropdown, and choose the
    **myprometheus\<xxx\>** Prometheus data source.

![](./media/image132.png)

3.  The query editor supports a graphical query builder and a text-based
    query editor. Select the metric you want to query, the aggregation
    function, and any filters you want to apply.

![](./media/image133.png)

![](./media/image134.png)

![](./media/image135.png)

## Exercise 5: Scaling your cluster and apps

Right now, the application is running a single pod. When the web app is
under heavy load, it may not be able to handle the requests. To
automatically scale your deployments, use Kubernetes Event-driven
Autoscaling (KEDA). KEDA scales your workloads based on utilization
metrics, number of events in a queue, or a custom CRON schedule.

KEDA alone is not enough: if the cluster is out of resources, the new
pods remain in **Pending** status. With AKS Automatic, Node
Autoprovisioning (NAP) is enabled and replaces the traditional cluster
autoscaler. NAP detects pods pending scheduling and automatically scales
the nodes to meet demand.

For the scheduler to place pods efficiently, set resource requests and
limits. The Automated Deployment setup added some default values, but
they may not be optimal. This is where the Vertical Pod Autoscaler (VPA)
can help.

### Task 1: Vertical Pod Autoscaler (VPA) setup

VPA automatically adjusts the CPU and memory requests and limits for
your pods based on actual resource utilization. AKS Automatic comes with
the VPA controller pre-installed.

1.  Navigate to **Custom resources** under **Kubernetes resources** in
    the AKS cluster menu. Scroll down and click on **Load more** to view
    all the available custom resources.

![](./media/image136.png)

2.  Select the **VerticalPodAutoscaler** resource.

![](./media/image137.png)

3.  Click on the **+ Create** button to open the **Apply with YAML**
    editor.

![](./media/image138.png)

![](./media/image139.png)

4.  Select the text editor or use **Alt + I** to open the Copilot
    editor. In the **Draft with Copilot** text box, type the following
    prompt and press **Enter**.

> +++Help me create a vertical pod autoscaler manifest for the contoso-air deployment in the dev namespace and set min and max cpu and memory to something typical for a nodejs app. Please apply the values for both requests and limits.+++

![](./media/image140.png)

5.  When the VPA manifest is generated, click on **Accept all** to
    accept the changes and create the VPA resource.

![](./media/image141.png)

![](./media/image142.png)

**Warning:** Microsoft Copilot in Azure may provide different results.
If your results are different, copy the following VPA manifest and paste
it into the **Apply with YAML** editor.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: contoso-air-vpa
  namespace: dev
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: contoso-air
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
      - containerName: contoso-air
        minAllowed:
          cpu: 100m
          memory: 256Mi
        maxAllowed:
          cpu: 1
          memory: 512Mi
        controlledResources: ["cpu", "memory"]
```

**Note:** VPA only updates requests and limits when the number of
replicas is greater than 1. Pods are restarted when VPA updates them, so
create Pod Disruption Budgets (PDBs) to ensure pods are not restarted
all at once.

![](./media/image143.png)

### Task 2: KEDA scaler setup

AKS Automatic also comes with the KEDA controller pre-installed.

**Info:** KEDA works with the Horizontal Pod Autoscaler (HPA). It
automatically creates an HPA resource when you create a KEDA
ScaledObject and takes ownership of it. The Automated Deployment already
created an HPA for contoso-air, so you'll delete it and let KEDA
recreate it.

1.  Navigate to **myakscluster**, expand **Kubernetes resources**,
    select **Run command**, enter the following command, and click on
    **Run**.

> +++kubectl delete hpa contoso-air -n dev+++

![](./media/image144.png)

2.  Navigate to **Application scaling** under **Settings**, then click
    on the **+ Create** button.

![](./media/image145.png)

![](./media/image146.png)

3.  In the **Basics** section, enter the following details.

| **Name** | +++contoso-air-so+++ |
|----|----|
| **Namespace** | **dev** |
| **Target workload** | **contoso-air** |
| **Minimum replicas** | +++3+++ |

![](./media/image147.png)

4.  In the **Trigger details** section, select **CPU** as the **Trigger
    type**. Leave the rest of the fields as default and click on
    **Next**.

![](./media/image148.png)

5.  In the **Review + create** tab, select **Customize with YAML** to
    view the YAML manifest generated for the ScaledObject resource.

![](./media/image149.png)

6.  Click on **Save and create** to create the ScaledObject resource.

![](./media/image150.png)

![](./media/image151.png)

7.  Navigate to **Workloads** under **Kubernetes resources**, and select
    **dev** in the **Filter by namespace** drop-down list. The
    **contoso-air** deployment is now running (or starting) **3
    replicas**.

![](./media/image152.png)

![](./media/image153.png)

![](./media/image154.png)

## Exercise 6: Clean up resources

1.  In the Azure portal, navigate to **Resource groups** and select your
    resource group.

2.  Select all the resources and then click on **Delete**. (**DO NOT
    DELETE** the resource group.)

![](./media/image155.png)

3.  Type the confirmation text in the text box and click on **Delete**.

![](./media/image156.png)

![](./media/image157.png)

**Summary**

In this lab, you created an AKS Automatic cluster and deployed the
Contoso Air application using Automated Deployments and GitHub Actions.
You secured GitHub authentication with federated credentials, and
integrated the application with Azure OpenAI using the AKS Service
Connector and Workload Identity. You enabled application monitoring with
AutoInstrumentation using Azure Monitor Application Insights, explored
Container Insights, Logs and Grafana dashboards, and configured
resource-specific scaling with the Vertical Pod Autoscaler (VPA) and
event-driven scaling with KEDA. These skills provide the foundation for
running secure, observable and scalable AI-enabled applications on AKS
Automatic.
