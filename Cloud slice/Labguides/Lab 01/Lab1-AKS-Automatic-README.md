# Lab 1: Build and Deploy the Contoso Air AI Travel App on AKS Automatic

## Kubernetes the Easy Way with AKS Automatic

This workshop shows you how easy it is to deploy applications to AKS Automatic. AKS Automatic is a fully managed Kubernetes service that simplifies the deployment, management, and operations of Kubernetes clusters. Deploy a cluster with just a few steps in the Azure portal and focus on building your applications!

## Objectives

After completing this workshop, you will be able to:

- Deploy an application to an AKS Automatic cluster
- Troubleshoot application issues
- Integrate applications with Azure services
- Scale your cluster and applications
- Observe your cluster and applications

## Architecture

| Component | Role in the lab |
|---|---|
| GitHub fork of `contoso-air` | Source code and GitHub Actions workflow |
| Entra app registration `workflowapp-<number>` | Identity GitHub Actions uses to sign in to Azure |
| Azure Container Registry (ACR) | Stores the `contoso-air` container image |
| AKS Automatic cluster `myakscluster` | Runs the app in namespace `dev` |
| Managed identity `myidentity<xxx>` | Identity the app pod uses to call Azure OpenAI |
| Azure OpenAI `myopenai<xxx>` | LLM behind the AI travel assistant |
| Log Analytics / Azure Monitor workspace | Logs and Prometheus metrics |
| Application Insights `myappinsights<xxx>` | App performance telemetry |

## Prerequisites

- **Azure subscription** with permissions to create resources and app registrations.
- **GitHub account**: You need your own GitHub login credentials. If you do not have an account, create one at <https://github.com/signup>.

## Contents

- [Exercise 1: Set up the lab environment](#exercise-1-set-up-the-lab-environment)
- [Exercise 2: Create the cluster and pipeline](#exercise-2-create-the-cluster-and-pipeline)
- [Exercise 3: Integrating apps with Azure services](#exercise-3-integrating-apps-with-azure-services)
- [Exercise 4: Observing your cluster and apps](#exercise-4-observing-your-cluster-and-apps)
- [Exercise 5: Scaling your cluster and apps](#exercise-5-scaling-your-cluster-and-apps)
- [Summary](#summary)

---

# Exercise 1: Set up the lab environment

## Task 1: Prepare the Azure environment and register providers

1. Open a browser and sign in to the Azure portal at <https://portal.azure.com/> with your credentials.

2. In the Azure portal, select the **Cloud Shell** icon from the top navigation bar to open Azure Cloud Shell and run commands directly from the browser.

   ![Open Cloud Shell](images/image1.png)

3. In the **Welcome to Azure Cloud Shell** dialog, select **Bash** to launch a Bash session.

   ![Select Bash](images/image2.png)

4. Choose **No storage account required**, select your subscription, and then select **Apply** to start Cloud Shell without creating a storage account.

   ![No storage account required](images/image3.png)

5. In Azure Cloud Shell, run the following command. Then open the displayed URL (<https://login.microsoftonline.com/device>) in a browser, enter the **device code** shown in the terminal, and complete the **sign-in** process with your Azure account.

   ```bash
   az login --use-device-code
   ```

   ![Device code login](images/image4.png)

   ![Enter device code](images/image5.png)

   ![Pick account](images/image6.png)

   ![Sign-in complete](images/image7.png)

6. Select the Azure subscription and tenant from the list, enter **1** to choose the displayed option, and then press **Enter** to continue.

   ![Select subscription](images/image8.png)

   > **Tip:** You can sign in to a different tenant by passing the `--tenant` flag with your tenant domain or tenant ID.

7. Add the AKS preview extension.

   ```bash
   az extension add --name aks-preview
   ```

   ![Add aks-preview extension](images/image9.png)

8. Register the Application Monitoring preview feature, and confirm the feature state shows **Registered** in the output.

   ```bash
   az feature register --namespace Microsoft.ContainerService --name AzureMonitorAppMonitoringPreview
   ```

   ![Register preview feature](images/image10.png)

9. Register the resource providers.

   ```bash
   az provider register --namespace Microsoft.DevHub
   az provider register --namespace Microsoft.Insights
   az provider register --namespace Microsoft.PolicyInsights
   az provider register --namespace Microsoft.ServiceLinker
   ```

   ![Register resource providers](images/image11.png)

## Task 2: Create the resource group

In this lab, you set environment variables for the resource group name and location. To keep the resource names unique, you use a random number as a suffix. This helps you avoid naming conflicts with other resources in your Azure subscription.

1. Generate a random number.

   ```bash
   RAND=$RANDOM
   export RAND
   echo "Random resource identifier will be: ${RAND}"
   ```

   ![Random number](images/image12.png)

2. Set the location to a region of your choice, for example `eastus` or `westeurope`. Make sure the region supports [availability zones](https://learn.microsoft.com/azure/aks/availability-zones-overview). Then create a resource group name using the random number.

   ```bash
   export LOCATION=eastus
   export RG_NAME=myresourcegroup$RAND
   ```

3. Create the resource group using the environment variables.

   ```bash
   az group create \
     --name ${RG_NAME} \
     --location ${LOCATION}
   ```

   ![Create resource group](images/image13.png)

## Task 3: Get your user object ID

1. The template gives your account access to Key Vault and Azure OpenAI, so it needs your user object ID. If this value is empty, the deployment fails with **InvalidPrincipalId**.

   ```bash
   export USER_ID=$(az ad signed-in-user show --query id -o tsv)
   echo "USER_ID=$USER_ID"
   ```

   ![Get user object ID](images/image14.png)

2. If nothing is printed, use this instead (it reads the ID from your sign-in token):

   ```bash
   export USER_ID=$(az account get-access-token --query accessToken -o tsv | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin)['oid'])")
   echo "USER_ID=$USER_ID"
   ```

   ![User ID from token](images/image15.png)

3. Save the environment variables to a file so you can reload them later, and verify the values.

   ```bash
   echo "export LOCATION=$LOCATION RG_NAME=$RG_NAME USER_ID=$USER_ID" > ~/aks-lab.env
   cat ~/aks-lab.env
   ```

   > **Tip:** If Cloud Shell times out, run `source ~/aks-lab.env` to restore the variables.

## Task 4: Deploy the resources

To keep the focus on AKS-specific features, this workshop requires several Azure resources to be pre-provisioned:

- Azure Log Analytics Workspace for container insights and application insights
- Azure Monitor Workspace for Prometheus metrics
- Azure Container Registry for storing container images
- Azure Key Vault for secrets management
- Azure User-Assigned Managed Identity for accessing Azure services via Workload Identity
- Azure Application Insights for application monitoring
- Azure OpenAI Service with a GPT model deployment for the AI chat feature

1. Make sure your user object ID is set.

   ```bash
   export USER_ID=$(az ad signed-in-user show --query id -o tsv)
   ```

2. Deploy the ARM template into the resource group.

   ```bash
   az deployment group create \
     --resource-group ${RG_NAME} \
     --name ${RG_NAME}-deployment \
     --template-uri https://raw.githubusercontent.com/azure-samples/aks-labs/refs/heads/main/docs/getting-started/aks-automatic/assets/main.json \
     --parameters userObjectId=${USER_ID}
   ```

3. If the deployment fails on the Application Insights resource (API version error), download the template, patch the Application Insights API version, and deploy the local copy instead:

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

   Wait until the resources are deployed. This can take around **15 minutes**.

   ![Deployment running](images/image16.png)

   ![Deployment output](images/image17.png)

   ![Deployment succeeded](images/image18.png)

4. In the Azure portal (<https://portal.azure.com/>), open your resource group.

5. Verify that all required resources were deployed successfully: Application Insights, Managed Identity, Key Vault, Log Analytics workspace, Azure OpenAI, Azure Monitor workspace, and Container Registry.

   ![Deployed resources](images/image19.png)

   > **Note:** Keep your terminal open, as you need it to run commands throughout the workshop.

## Task 5: Fork and clone the sample app

1. In the Bash shell, run the following command and follow the instructions printed in the terminal to complete the login process.

   ```bash
   gh auth login
   ```

   ![gh auth login](images/image20.png)

2. Choose **HTTPS** as the preferred protocol for Git operations, enter **Y** to authenticate with your GitHub credentials, and press **Enter**.

   ![Choose HTTPS](images/image21.png)

   ![Authenticate Git](images/image22.png)

3. Select **Login with a web browser**, and then press **Enter**.

   ![Login with a web browser](images/image23.png)

4. Copy the one-time device code, press **Enter** to open <https://github.com/login/device> in your browser, enter the code, and complete GitHub authentication.

   ![One-time code](images/image24.png)

   ![Device activation](images/image25.png)

   ![Enter code](images/image26.png)

   ![Authorize GitHub CLI](images/image27.png)

   ![Authentication complete](images/image28.png)

5. Fork the **contoso-air** repository to your GitHub account and clone it. Verify that the fork appears under your GitHub repositories.

   ```bash
   gh repo fork Azure-Samples/contoso-air --clone --default-branch-only
   ```

   ![Fork repository](images/image29.png)

6. Change into the `contoso-air` directory.

   ```bash
   cd contoso-air
   ```

7. Set the default repository to your fork.

   ```bash
   gh repo set-default
   ```

8. When prompted, select **your fork** of the repository and press **Enter**. Do **not** select the original **Azure-Samples/contoso-air** repository.

   ![Set default repo](images/image30.png)

9. Configure your Git identity, replacing the placeholders with your GitHub username and email address:

   ```bash
   git config user.name "<your GitHub user>"
   git config user.email "<your email>"
   ```

   ![Git config](images/image31.png)

10. Save your **GitHub username** and **email address**; you need them in later tasks.

## Task 6: Record your GitHub IDs

1. GitHub identifies your repository using numeric IDs. Run the following command (replace `<your-user>` with your GitHub username) and save the **Owner ID** and **Repo ID** values for later.

   ```bash
   gh api repos/<your-user>/contoso-air --jq '"Owner ID: \(.owner.id)  Repo ID: \(.id)"'
   ```

   ![GitHub IDs](images/image32.png)

---

# Exercise 2: Create the cluster and pipeline

## Task 1: Configure Automated Deployments for Azure Kubernetes Service (AKS)

1. Sign in to the Azure portal at <https://portal.azure.com/>.

2. Type **Kubernetes services** in the search box at the top of the page, and select **Kubernetes services** from the results.

   ![Search Kubernetes services](images/image33.png)

3. Select **+ Create** to view the available options, then select **Deploy application**.

   ![Deploy application](images/image34.png)

4. In the **Basics** tab, select the **Deploy your application** option, then select your Azure subscription and the resource group you created during setup.

   ![Basics tab](images/image35.png)

5. In the **Basics** tab, configure the application deployment settings:

   - **Subscription**: Select your Azure subscription.
   - **Resource group**: Select **myresourcegroupXXXX**.
   - **Region**: Select **(US) East US**, or the same region as your resource group.
   - **Workflow name**: Enter `contoso-air`.
   - **Repository location**: Verify that **GitHub** is selected.
   - Select **Authorize access** to grant Azure permission to create the deployment workflow in your GitHub repository.

   ![Basics settings](images/image36.png)

   ![Authorize access](images/image37.png)

   ![Authorize Azure app](images/image38.png)

   - **Repository source**: Select **My repositories**.
   - **Repository**: Select **contoso-air**.
   - **Branch**: Select **main**.
   - Select **Next**.

   ![Repository settings](images/image39.png)

6. In the **Application** tab, complete the **Image** section:

   - **Container configuration**: Select **Existing Dockerfile**.
   - **Dockerfile**: Select the **Select** link, browse to the `./src/web` directory, select the Dockerfile, then select **Select**.
   - **Dockerfile build context**: Enter `./src/web`.
   - **Azure Container Registry**: Select the Azure Container Registry in your resource group.
   - **Azure Container Registry image**: Select **Create new**, enter `contoso-air`, and select **Ok**.

   ![Image settings](images/image40.png)

   ![Select Dockerfile](images/image41.png)

   ![Container registry image](images/image42.png)

7. In the **Deployment configuration** section:

   - **Deployment options**: Select **Generate application deployment files**.
   - **Save files in repository**: Select the **Select** link, select the checkbox next to the **Root** folder, then select **Select**.
   - **Application port**: Enter `3000`.
   - Select **Next**.

   ![Deployment configuration](images/image43.png)

   ![Select root folder](images/image44.png)

   ![Application port](images/image45.png)

8. In the **Cluster configuration** section, make sure **Create Automatic Kubernetes cluster** is selected and enter `myakscluster` as the **Kubernetes cluster name**.

   ![Cluster configuration](images/image46.png)

9. For **Namespace**, select **Create new**, enter `dev`, select **OK**, and verify that the `dev` namespace is selected.

   ![Create namespace](images/image47.png)

10. Review the monitoring settings. Verify that **Container Logs**, **Prometheus metrics**, and the associated **Log Analytics Workspace** and **Azure Monitor Workspace** are configured correctly, then select **Next**.

    ![Monitoring settings](images/image48.png)

11. After validation passes, select **Deploy**.

    ![Validation passed](images/image49.png)

    > **Important:** The deployment can take **up to 20 minutes**. Do not close the browser window or navigate away until it finishes.

    ![Deployment in progress](images/image50.png)

    ![Deployment progress details](images/image51.png)

12. Verify that the cluster and application deployment completed successfully. The page shows an **Approve pull request** button for the GitHub Actions workflow that was created for the application.

    ![Deployment complete](images/image52.png)

13. **Do not approve or merge the pull request yet.** You first need to secure GitHub authentication in the next task.

    ![Pull request created](images/image53.png)

## Task 2: Secure GitHub authentication before merging changes

1. Sign in to the Azure portal at <https://portal.azure.com/>.

2. Type **App registrations** in the search box and select **App registrations** from the results.

   ![App registrations](images/image54.png)

3. Open **workflowapp-\<number\>** created today. Note its name.

   ![workflowapp registration](images/image55.png)

4. Under **Manage**, select **Certificates & secrets**, open the **Federated credentials** tab, and select **Add credential**.

   ![Add federated credential](images/image56.png)

5. For **Federated credential scenario**, select **GitHub Actions deploying Azure resources**, and enter the following values:

   | Field | Value |
   |---|---|
   | Issuer | Leave `https://token.actions.githubusercontent.com` |
   | Organization | Your GitHub username (saved in *Exercise 1 > Task 5 > Step 10*) |
   | Organization ID | Owner ID (saved in *Exercise 1 > Task 6*) |
   | Repository | `contoso-air` |
   | Repository ID | Repo ID (saved in *Exercise 1 > Task 6*) |
   | Entity type | Branch |
   | GitHub branch name | `main` |
   | Name | `github-main-immutable` |
   | Audience | Leave `api://AzureADTokenExchange` |

6. Scroll to **Subject identifier** (read-only). It must read:

   ```text
   repo:<user>@<ownerId>/contoso-air@<repoId>:ref:refs/heads/main
   ```

   Select **Add**.

   ![Credential details](images/image57.png)

   ![Subject identifier](images/image58.png)

   ![Credential added](images/image59.png)

## Task 3: Review, merge, and deploy application changes

1. Back on the Automated Deployments page, select **Approve pull request** (or open **Pull requests** in your GitHub fork).

   ![Approve pull request](images/image53.png)

2. In the pull request, select the **Files changed** tab and review the workflow and configuration updates.

   ![Files changed](images/image60.png)

3. Scroll down to the `manifests/deployment.yaml` file to review the generated Kubernetes deployment manifest. It includes the `SYS_PTRACE` capability, which should be removed for security hardening.

   ![deployment.yaml](images/image61.png)

   ![SYS_PTRACE capability](images/image62.png)

4. Hover over the line, select the **+** icon, choose **Add a suggestion**, delete the suggested line so the suggestion is empty, and then select **Comment**.

   ![Add a suggestion](images/image63.png)

   ![Empty suggestion](images/image64.png)

5. Select **Commit suggestions**.

   ![Commit suggestions](images/image65.png)

   ![Commit changes](images/image66.png)

6. Select the **Conversation** tab, review the comments and changes, and then select **Merge pull request**.

   ![Merge pull request](images/image67.png)

7. Select **Confirm merge**.

   ![Confirm merge](images/image68.png)

8. With the pull request merged, the changes are automatically deployed to your AKS cluster. Select the **Actions** tab in your GitHub repository to view the deployment logs.

   ![Actions tab](images/image69.png)

   ![Workflow list](images/image70.png)

9. In the **Actions** tab, select the running **Automated Deployments** workflow to view the logs.

   ![Workflow run](images/image71.png)

10. After 5–10 minutes, the workflow completes and you see green check marks next to the **buildImage** and **deploy** jobs. The application is now deployed to your AKS cluster.

    ![buildImage and deploy jobs](images/image72.png)

    ![Workflow complete](images/image73.png)

## Task 4: Verify the application deployment in the dev namespace

1. In the Azure portal, navigate to **myakscluster**.

   ![myakscluster](images/image74.png)

2. Select **Kubernetes resources** > **Services and ingresses** and verify that the application is running in the `dev` namespace.

   ![Services and ingresses](images/image75.png)

3. Locate the **contoso-air** service in the `dev` namespace, and select its **External IP** address to open the application.

   ![contoso-air service](images/image76.png)

   ![Contoso Air app](images/image77.png)

4. Test the chat functionality by selecting **Ask the AI travel assistant**.

   ![Ask the AI travel assistant](images/image78.png)

5. Try to interact with the AI assistant. You'll find that the **chat provider is not detected** — you fix this in the next exercise.

   ![Chat assistant](images/image79.png)

   ![Chat provider not detected](images/image80.png)

---

# Exercise 3: Integrating apps with Azure services

AKS Service Connector streamlines connecting applications to Azure resources like Azure OpenAI by automating the configuration of Workload Identity. It assigns identities to pods, enabling them to authenticate with Microsoft Entra ID and access Azure services securely without passwords.

## Task 1: Application configuration

A ConfigMap is a Kubernetes resource that stores non-confidential configuration data as key-value pairs. Applications running in pods can read these values as environment variables, making it easy to change application behavior without rebuilding the container image.

1. In the Azure portal, navigate to **myakscluster**, select **Kubernetes resources** > **Configuration**, and choose `dev` from the **Filter by namespace** list.

   ![Configuration](images/image81.png)

2. Select the **contoso-air-config** ConfigMap.

   ![contoso-air-config](images/image82.png)

3. Select the **YAML** tab and add the following configuration to the end of the file, then select **Review + save**.

   ```yaml
   data:
     AZURE_OPENAI_API_VERSION: 2024-12-01-preview
     AZURE_OPENAI_DEPLOYMENT: gpt-5.4-mini
     CHAT_PROVIDER: azure
     LOG_CHAT: 'true'
   ```

   ![Edit ConfigMap YAML](images/image83.png)

4. Review the changes, verify the Azure OpenAI settings are correct, select **Confirm manifest changes**, and then select **Save**.

   ![Confirm manifest changes](images/image84.png)

   ![ConfigMap saved](images/image85.png)

   > **Info:** This ConfigMap was created by the Automated Deployments workflow. The new settings tell the application to use Azure OpenAI as the chat provider and specify the model deployment name. The contoso-air application already reads these settings and injects them into its environment.

## Task 2: Service Connector setup

1. In the cluster menu, select **Settings** > **Service Connector**, then select **+ Create**.

   ![Service Connector](images/image86.png)

2. In the **Basics** tab, enter the following:

   - **Kubernetes namespace**: Enter `dev`.
   - **Service type**: Select **OpenAI Service**.
   - **OpenAI**: Select the Azure OpenAI account you created earlier.
   - Select **Next: Authentication**.

   ![Service Connector basics](images/image87.png)

3. In the **Authentication** tab, select **Workload Identity** and expand the **Advanced** section.

   ![Workload Identity](images/image88.png)

4. By default, the **Cognitive Services OpenAI Contributor** role is assigned, granting the workload identity permission to authenticate and interact with your LLM. You'll also see additional configuration that Service Connector sets as environment variables in the application. These variables are saved to a Kubernetes Secret used to configure the connection to Azure OpenAI.

5. Select **Next: Networking**, then **Next: Review + create**, and finally **Create**.

   ![Networking](images/image89.png)

   ![Review + create](images/image90.png)

   ![Create](images/image91.png)

   ![Connection created](images/image92.png)

## Task 3: Configure the application for Workload Identity

Now that the Service Connector is set up for your Azure OpenAI account, configure your application to use these connection details.

1. On the **Service Connector** page, select the checkbox next to the **OpenAI** connection and select **Yaml snippet**.

   ![Select connection](images/image93.png)

   ![YAML snippet](images/image94.png)

2. In the **YAML snippet** window, select **Kubernetes Workload** for **Resource type**, then select **contoso-air** for **Kubernetes Workload**.

   ![Kubernetes Workload](images/image95.png)

3. The YAML manifest for the contoso-air application appears, with the edits required to connect to Azure OpenAI via Workload Identity highlighted.

4. Scroll through the manifest to view all the changes highlighted in yellow, then select **Apply**. This redeploys the contoso-air application with the new connection details.

   ![Highlighted changes](images/image96.png)

   ![Apply](images/image97.png)

   ![Applied](images/image98.png)

   > **Note:** This applies changes directly to the deployment. Ideally, you'd commit these changes to your repository so they are versioned, tracked, and deployed by the Automated Deployments workflow you set up earlier.

5. Wait a minute or two for the new pod to roll out, then return to the application and chat with the AI assistant. It should now respond without errors.

   ![AI assistant working](images/image99.png)

   ![AI assistant response](images/image100.png)

---

# Exercise 4: Observing your cluster and apps

Monitoring and observability are key components of running applications in production. AKS Automatic provides many monitoring and observability features out of the box.

At the start of the workshop, you integrated the AKS Automatic cluster with Azure Log Analytics Workspace for logging and Azure Monitor Managed Workspace for metrics collection. AKS also provides built-in Grafana dashboards for data visualization. You can also enable Azure Monitor Application Insights for AKS to automatically instrument your applications.

## Task 1: App monitoring

Azure Monitor Application Insights is an Application Performance Management (APM) solution for real-time monitoring of your applications. It uses OpenTelemetry (OTel) to collect telemetry and stream it to Azure Monitor, helping you evaluate performance, pinpoint bottlenecks, and gain actionable insights. With AKS, the AutoInstrumentation feature collects telemetry without code changes.

1. Navigate to the **Monitor** blade of your AKS cluster and select **Monitor Settings**.

   ![Monitor Settings](images/image101.png)

2. Scroll to the **Application monitoring (preview)** section, select the **Enable** checkboxes for both auto-instrumentation and OpenTelemetry data collection, then select **Next**.

   ![Enable application monitoring](images/image102.png)

3. Review the changes and select **Enable**.

   ![Review and enable](images/image103.png)

4. Onboarding takes about 5–6 minutes while the components are set up in your cluster. Wait for it to complete before continuing.

   ![Onboarding](images/image104.png)

   ![Enable application monitoring progress](images/image105.png)

   ![Onboarding complete](images/image106.png)

   With this feature enabled, you can deploy an Instrumentation custom resource to automatically instrument your applications without modifying code.

5. Navigate to **Kubernetes resources** > **Workloads**, filter by the `dev` namespace, and select the **contoso-air** deployment.

   ![Workloads](images/image107.png)

6. Select the **Application Monitoring (Preview)** tab to configure AutoInstrumentation.

   ![Application Monitoring tab](images/image108.png)

   ![Configure](images/image109.png)

7. In the **Configure Application Monitoring** pane:

   - **Application Insights**: Select the Application Insights resource in your resource group.
   - **Application Language**: Select **NodeJS auto-instrumentation for all deployments**.
   - Select **Perform rollout restart of all deployments in the namespace to deploy changes immediately**.
   - Select **Configure**.

   ![Configure Application Monitoring](images/image110.png)

   ![Configuring](images/image111.png)

   > **Tip:** This example instruments an entire namespace. You can also instrument individual deployments by deploying custom resources into your cluster. See the [documentation](https://learn.microsoft.com/azure/azure-monitor/app/kubernetes-codeless#mixed-mode-onboarding) for details.

   Once the configuration is applied, the contoso-air deployment restarts to apply the instrumentation.

   ![Rollout restart](images/image112.png)

8. Verify that all monitoring steps completed successfully, then select **Close**.

   ![Application Monitoring Progress](images/image113.png)

   Return to the contoso-air application and chat with the AI assistant to generate some telemetry. Once it is collected by the OTel collector, you can view performance and usage metrics in the Azure portal.

9. Navigate to the **Application Insights** resource in your resource group.

   ![Application Insights](images/image114.png)

10. Select **Investigate** > **Application map** to view the application components, their dependencies, and the number of calls.

    ![Application map](images/image115.png)

    > **Note:** If the Azure OpenAI endpoint does not appear in the application map, chat with the AI assistant a bit more to generate data, then select **Refresh**. The map updates to show the application connected to the Azure OpenAI endpoint, along with request latency to the model.

    ![Application map with OpenAI](images/image116.png)

    ![Dependency latency](images/image117.png)

11. Select **Live Metrics** to view incoming and outgoing requests, response times, and exceptions in real time.

    ![Live Metrics](images/image118.png)

12. Select **Performance** to view the average response time, request rate, and failure rate.

    ![Performance](images/image119.png)

    Feel free to explore other Application Insights features.

## Task 2: Cluster monitoring

AKS Automatic simplifies cluster monitoring with [Container Insights](https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-overview). It gathers and analyzes logs, metrics, and events from your cluster and applications.

1. Navigate to **myakscluster**, select **Monitor (Insights)**, open the **More options (...)** menu, and select **Recommended alerts**.

   ![Recommended alerts](images/image120.png)

   The cluster is pre-configured with basic CPU and memory utilization alerts. You can also create additional alerts based on metrics in the Prometheus workspace.

2. Expand **Prometheus community alert rules (Preview)** to see the available Prometheus alert rules. Enable any of them using the toggle switch.

3. Select **Save** to enable the alerts.

   ![Prometheus alert rules](images/image121.png)

   ![Alerts saved](images/image122.png)

## Task 3: Workbooks and logs

With Container Insights enabled, you can query logs using Kusto Query Language (KQL) and use custom or pre-configured workbooks for visualization.

1. In the **Monitoring** section of the cluster menu, select **Workbooks**. The **Cluster Optimization** workbook is useful for identifying anomalies, detecting probe failures, and optimizing container requests and limits.

   ![Workbooks](images/image123.png)

   ![Cluster Optimization workbook](images/image124.png)

   > **Note:** Workbook visuals include a query button that shows the KQL query behind the visual — a great way to learn to write your own queries.

2. Select **Monitoring** > **Logs**. This gives access to logs gathered by the Azure Monitor agent on the cluster nodes.

   ![Logs](images/image125.png)

   > **Note:** If a **Welcome to Log Analytics** pop-up appears, close it.

3. Close the **Queries hub** pop-up, then switch the query editor from **Simple mode** to **KQL mode** using the drop-down in the top-right corner.

   ![KQL mode](images/image126.png)

   You should see log messages in the results table. Expand some entries to view message details generated by the application.

   > **Note:** Some queries might not have enough data to return results.

## Task 4: Dashboards with Grafana

If you prefer to visualize data using Grafana or run complex PromQL queries, use the **Dashboards with Grafana** feature built into AKS.

> **Info:** Dashboards with Grafana is a free Grafana experience integrated into the AKS portal. It doesn't require a separate Azure Managed Grafana resource and provides pre-configured dashboards and PromQL queries at no additional cost.

1. In the cluster menu, select **Monitoring** > **Dashboards with Grafana** to see the pre-configured dashboards.

   ![Dashboards with Grafana](images/image127.png)

2. Sign in to Grafana if prompted, then select **Dashboards** in the menu.

3. Expand the **Azure Managed Prometheus** folder and explore the dashboards. Each provides a different view of Prometheus metrics with filter controls.

4. Select the **Kubernetes / Compute Resources / Workload** dashboard.

   ![Workload dashboard](images/image128.png)

5. Set **namespace** to `dev`, **type** to `deployment`, and **workload** to `contoso-air` to see metrics for the contoso-air deployment.

   ![Dashboard filters](images/image129.png)

   ![contoso-air metrics](images/image130.png)

## Task 5: Querying metrics with PromQL

1. Select **Dashboards** to return to the list, then select the **Explore** tab.

   ![Explore](images/image131.png)

2. Select the data source drop-down and choose your Prometheus data source (**myprometheus\<xxx\>**).

   ![Prometheus data source](images/image132.png)

3. The query editor supports a graphical query builder and a text-based editor. The graphical builder is a great way to get started with PromQL: choose the metric, aggregation function, and any filters.

   ![Query builder](images/image133.png)

   Take some time to explore Grafana and PromQL.

   ![PromQL query](images/image134.png)

   ![Query results](images/image135.png)

---

# Exercise 5: Scaling your cluster and apps

Now that you've deployed and monitored your application, let's scale your cluster and application to handle workload demands.

Right now, the application runs a single pod. Under heavy load, it may not handle all requests. To scale deployments automatically, use **Kubernetes Event-driven Autoscaling (KEDA)**, which scales workloads based on utilization metrics, queue length, or a CRON schedule.

KEDA alone isn't enough: if the cluster runs out of resources, new pods stay **Pending**. With AKS Automatic, **Node Autoprovisioning (NAP)** replaces the traditional cluster autoscaler. NAP detects pending pods and automatically scales nodes to meet demand.

For the scheduler to place pods efficiently, set resource **requests** (the minimum CPU and memory a container needs) and **limits** (the maximum it can use). Automated Deployments added default values, but they may not be optimal — this is where the **Vertical Pod Autoscaler (VPA)** helps.

## Task 1: Vertical Pod Autoscaler (VPA) setup

VPA automatically adjusts CPU and memory requests and limits for your pods based on actual utilization. AKS Automatic comes with the VPA controller pre-installed, so you only need to deploy a VPA resource manifest.

> **Info:** The deployment manifest generated by Automated Deployments includes default requests and limits for the contoso-air pods. VPA can tune these values based on real usage.

1. In the cluster menu, select **Kubernetes resources** > **Custom resources**. Scroll to the bottom and select **Load more** to view all custom resources.

   ![Custom resources](images/image136.png)

2. Select the **VerticalPodAutoscaler** resource.

   ![VerticalPodAutoscaler](images/image137.png)

3. Select **+ Create** to open the **Apply with YAML** editor.

   ![Create VPA](images/image138.png)

   ![Apply with YAML](images/image139.png)

4. Select the text editor or press **Alt + I** to open the Copilot editor. In the **Draft with Copilot** box, enter the following prompt and press **Enter**:

   ```text
   Help me create a vertical pod autoscaler manifest for the contoso-air deployment in the dev namespace and set min and max cpu and memory to something typical for a nodejs app. Please apply the values for both requests and limits.
   ```

   ![Draft with Copilot](images/image140.png)

5. When the manifest is generated, select **Accept all** to accept the changes and create the VPA resource.

   ![Generated manifest](images/image141.png)

   ![Accept all](images/image142.png)

   > **Warning:** Microsoft Copilot in Azure may produce different results. If so, paste the following manifest into the **Apply with YAML** editor instead:

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

   VPA only updates requests and limits when the deployment has **more than 1 replica**. Pods are restarted when VPA updates them, so create Pod Disruption Budgets (PDBs) to prevent all pods from restarting at once.

   ![VPA created](images/image143.png)

## Task 2: KEDA scaler setup

AKS Automatic comes with the KEDA controller pre-installed, so you can deploy a KEDA scaler right away.

> **Info:** KEDA works with the Horizontal Pod Autoscaler (HPA). When you create a KEDA `ScaledObject`, KEDA automatically creates and owns an HPA. If an HPA already exists, you can transfer ownership to KEDA instead.

Automated Deployments created an HPA for contoso-air. Instead of transferring ownership, you'll delete it and let KEDA recreate it.

1. Navigate to **myakscluster**, expand **Kubernetes resources**, select **Run command**, enter the following command, and select **Run**:

   ```bash
   kubectl delete hpa contoso-air -n dev
   ```

   ![Delete HPA](images/image144.png)

2. Select **Settings** > **Application scaling**, then select **+ Create**.

   ![Application scaling](images/image145.png)

   ![Create scaler](images/image146.png)

3. In the **Basics** section, enter:

   - **Name**: `contoso-air-so`
   - **Namespace**: `dev`
   - **Target workload**: `contoso-air`
   - **Minimum replicas**: `3`

   ![Scaler basics](images/image147.png)

4. In the **Trigger details** section, set **Trigger type** to **CPU**. Leave the other fields at their defaults and select **Next**.

   ![Trigger details](images/image148.png)

5. In the **Review + create** tab, select **Customize with YAML** to view the generated `ScaledObject` manifest. You can add more configuration here if needed.

   ![Customize with YAML](images/image149.png)

6. Select **Save and create**.

   ![Save and create](images/image150.png)

   ![ScaledObject created](images/image151.png)

7. Go to **Kubernetes resources** > **Workloads** and filter by the `dev` namespace. The **contoso-air** deployment should now be running (or starting) **3 replicas**.

   ![Workloads](images/image152.png)

   ![3 replicas](images/image153.png)

   ![Pods running](images/image154.png)

   With more replicas, VPA can adjust CPU and memory requests and limits based on actual usage the next time it reconciles.

## Task 3: Clean up all the resources

1. In the Azure portal, go to **Resource groups** > your resource group.

2. Select all resources and select **Delete**. (**Do not delete** the resource group.)

   ![Select all and delete](images/image155.png)

3. Type the confirmation text in the box and select **Delete**.

   ![Confirm delete](images/image156.png)

   ![Resources deleted](images/image157.png)

---

# Summary

In this lab, you created an AKS Automatic cluster and deployed an application using Automated Deployments. You integrated the application with Azure OpenAI using the AKS Service Connector and Workload Identity. You enabled application monitoring with AutoInstrumentation using Azure Monitor Application Insights for deep visibility into performance without code changes. Finally, you configured resource-specific scaling with the Vertical Pod Autoscaler (VPA) and event-driven scaling with KEDA.
