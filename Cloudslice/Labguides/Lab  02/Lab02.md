# Lab 2: Deploy and Manage a Secure, Scalable Container App with Azure Container Apps

**Scenario**

**Contoso Ltd.** is modernizing its line-of-business services by moving
them to containers. The development team has built a new ASP.NET Web API
that will publish order events to **Azure Service Bus**, and the
operations team wants to run it on a fully managed platform without
operating Kubernetes clusters.

Today, deployments are manual, container images are pulled with shared
admin credentials, and every release replaces the previous version in
one step. This makes releases risky and difficult to audit, and the
security team has asked for private network access and least-privilege
identities for all workloads.

To address these challenges, Contoso has chosen **Azure Container
Apps**. The platform team wants images stored in a private **Azure
Container Registry**, pulled through a **user-assigned managed
identity** over a **private endpoint**, and deployed automatically
through **Azure Pipelines**. New versions must be rolled out
gradually using **revisions and traffic splitting**.

As a Cloud Engineer, your responsibility is to provision the Azure
infrastructure, containerize the Web API, secure access to the container
registry, deploy the container app into a virtual network, connect it to
Service Bus, configure autoscaling, automate deployments with Azure
DevOps, and manage revisions for controlled rollouts.

**Introduction**

In this lab, you will learn how to deploy, secure, scale, and manage
containerized applications using **Azure Container Apps**. You will
provision the required Azure infrastructure, including Virtual Networks,
Azure Container Registry, Service Bus, Managed Identities, and Azure
DevOps resources. You will then deploy a containerized ASP.NET
application, configure secure access using managed identities and
private endpoints, implement autoscaling, automate deployments through
Azure Pipelines, and manage application revisions for controlled traffic
distribution.

**Azure resources used in this lab**

| **Resource** | **Name** | **Role in the lab** |
|----|----|----|
| Resource group | lab2-RG | Holds all lab resources |
| Virtual network | VNET1 (PESubnet, ACASubnet) | Private networking for the registry and container app |
| Service Bus namespace | sb-az2003-\<initials\> | Messaging service the app connects to |
| Azure Container Registry (Premium) | acraz2003\<initials\> | Stores the aspnetcorecontainer image |
| User-assigned managed identity | uai-az2003 | Identity used to pull images and connect to Service Bus |
| Private endpoint | pe-acr-az2003 | Private access to the registry from VNET1 |
| Container App | aca-az2003 | Runs the ASP.NET Web API |
| Azure DevOps project | Project1 / Pipeline1 | CI/CD pipeline and self-hosted agent |

**Objectives**:

- Create and configure Azure resources required for Azure Container Apps
  deployments.

- Build a containerized ASP.NET Web API application and publish it to
  Azure Container Registry.

- Configure Azure DevOps projects, repositories, pipelines, and
  self-hosted agents.

- Secure container image access using user-assigned managed identities
  and Azure RBAC.

- Configure private endpoint connectivity for Azure Container Registry.

- Deploy applications to Azure Container Apps using container images
  stored in Azure Container Registry.

- Integrate Azure Container Apps with Azure Service Bus using managed
  identities.

- Configure autoscaling rules and replica settings for containerized
  applications.

- Implement continuous integration and deployment (CI/CD) using Azure
  Pipelines.

- Deploy application updates using Azure Container Apps revisions.

- Configure revision labels and traffic splitting to support controlled
  application rollouts and testing.

**Prerequisites**

- **GitHub Account**: You are expected to have your own GitHub login
  credentials. If you do not have an account, create one by visiting:
  +++https://github.com/signup+++

## Exercise 1: Configure deployment tools and Azure resources

Before deploying the container app, you need to prepare the Azure
environment and your development tools. In this exercise, you will
create the resource group, virtual network, Service Bus and Container
Registry, build and push the container image, and configure Azure DevOps
with a starter pipeline and a self-hosted agent.

### Task 1: Configure a resource group for your Azure resources

1.  Open your browser, navigate to the address bar, and type or paste
    the following URL: +++https://portal.azure.com/+++ then press the
    **Enter** button. Sign in with the following credentials.

| **Username** | **+++@lab.CloudPortalCredential(User1).Username+++** |
|----|----|
| **Password** | **+++@lab.CloudPortalCredential(User1).AccessToken+++** |

2.  On the top search bar of the Azure portal, in the Search textbox,
    enter +++resource group+++

![](./media/image1.png)

3.  In the search results, select **Resource groups**, and then click on
    **+ Create**.

![](./media/image2.png)

4.  On the **Basics** tab, configure the resource group as follows:

| **Subscription** | Select the Azure subscription that you're using for this lab |
|----|----|
| **Resource group** | +++lab2-RG+++ |
| **Region** | **Central US** |

5.  Click on **Review + create**.

![](./media/image3.png)

6.  Once validation has passed, click on **Create**.

![](./media/image4.png)

![](./media/image5.png)

### Task 2: Configure a virtual network and subnets

1.  Ensure that you have your Azure portal open in a browser window.

2.  On the top search bar of the Azure portal, in the Search textbox,
    enter +++virtual network+++. In the search results, select **Virtual
    networks**.

![](./media/image6.png)

3.  Click on **+ Create**.

![](./media/image7.png)

4.  On the **Basics** tab, configure your virtual network as follows,
    and then click on **Next**.

| **Subscription** | Select the Azure subscription that you're using for this lab |
|----|----|
| **Resource group** | **lab2-RG** |
| **Virtual network name** | +++VNET1+++ |
| **Region** | **Central US** |

![](./media/image8.png)

5.  On the **Security** tab, keep the default settings, and then click
    on **Next**.

![](./media/image9.png)

6.  Select the **IP addresses** tab.

7.  On the **IP addresses** tab, under **Subnets**, select **default**.

8.  On the **Edit subnet** page, configure the subnet as follows, and
    then click on **Save**.

| **Name** | +++PESubnet+++ |
|----|----|
| **Starting address** | **10.0.0.0** |
| **Subnet size** | **/24 (256 addresses)** |

9.  On the **IP addresses** tab, click on **+ Add a subnet**.

10. On the **Add a subnet** page, configure the subnet as follows, and
    then click on **Add**.

| **Name** | +++ACASubnet+++ |
|----|----|
| **Starting address** | **10.0.4.0** |
| **Subnet size** | **/23 (512 addresses)** |

![](./media/image10.png)

![](./media/image11.png)

![](./media/image12.png)

![](./media/image13.png)

11. Click on **Review + create**.

![](./media/image14.png)

12. Once validation has passed, click on **Create**.

![](./media/image15.png)

13. Wait for the deployment to complete.

![](./media/image16.png)

14. After the deployment completes successfully, click on **Go to
    resource** to open the newly created virtual network and review its
    configuration.

![](./media/image17.png)

![](./media/image18.png)

### Task 3: Configure Service Bus

1.  In the **Azure portal** search bar, enter +++Service Bus+++, and
    then select **Service Bus** from the search results.

![](./media/image19.png)

2.  Click on **Create service bus namespace**.

![](./media/image20.png)

3.  On the **Basics** tab, configure your Service Bus namespace as
    follows:

| **Subscription** | Select the Azure subscription that you're using for this lab |
|----|----|
| **Resource group** | **lab2-RG** |
| **Namespace name** | +++sb-az2003-+++ followed by your name or initials. For example: **sb-az2003-cah** |
| **Location** | **Central US** |
| **Pricing tier** | **Basic** |

4.  Click on **Review + create**.

![](./media/image21.png)

5.  Once the **Validation succeeded** message appears, click on
    **Create**.

![](./media/image22.png)

6.  Wait for the deployment to complete.

![](./media/image23.png)

![](./media/image24.png)

### Task 4: Configure Azure Container Registry

1.  On the top search bar of the Azure portal, in the Search textbox,
    enter +++container registry+++

2.  In the search results, select **Container registries**.

![](./media/image25.png)

3.  On the **Container registries** page, click on **Create container
    registry** or **+ Create**.

![](./media/image26.png)

4.  On the **Basics** tab of the **Create container registry** page,
    specify the following information:

**Note:** The name of your registry must be unique. Also, the
**Premium** tier is required for private link with private endpoints.

| **Subscription** | Select the Azure subscription that you're using for this lab |
|----|----|
| **Resource group** | **lab2-RG** |
| **Registry name** | +++acraz2003+++ followed by your initials and date. For example: **acraz2003cah25oct** |
| **Location** | **Central US** |
| **SKU** | **Premium** |

5.  Click on **Review + create**.

![](./media/image27.png)

6.  Click on **Create**.

![](./media/image28.png)

7.  After the deployment has completed, open the deployed resource.

![](./media/image29.png)

8.  On the left-side menu, under **Settings**, select **Networking**.

9.  On the **Networking** page, on the **Public access** tab, ensure
    that **All networks** is selected.

![](./media/image30.png)

10. On the left-side menu, under **Settings**, select **Properties**.

11. On the **Properties** page, select **Admin user**, and then click on
    **Save**.

![](./media/image31.png)

### Task 5: Create a Web API app and publish to a GitHub repository

1.  Open **Visual Studio Code**.

2.  On the **File** menu, select **Open Folder**.

![](./media/image32.png)

3.  Create a new folder named +++AZ2003+++ in a location that is easy to
    find, for example on the Windows Desktop, and open it.

![](./media/image33.png)

4.  On the **Terminal** menu, select **New Terminal**.

![](./media/image34.png)

5.  At the terminal command prompt, run the following command to create
    a new ASP.NET Web API project.

> +++dotnet new webapi --no-https+++

![](./media/image35.png)

![](./media/image36.png)

6.  At the terminal command prompt, run the following command to build
    the project.

> +++dotnet build+++

![](./media/image37.png)

7.  On the **View** menu, select **Command Palette**, and then run the
    following command: **.NET: Generate Assets for Build and Debug**.

![](./media/image38.png)

**Note:** If the command generates an error message, select **OK**, and
then run the command again. Alternatively, open **Program.cs** and press
**F5** — VS Code detects that there is no debug configuration yet and
offers to create it for you, which gives the same result.

![](./media/image39.png)

8.  In the root project folder, create a **.gitignore** file that
    contains the following information.

![](./media/image40.png)

![](./media/image41.png)

```text
[Bb]in/
[Oo]bj/
```

![](./media/image42.png)

9.  On the **File** menu, select **Save All**.

![](./media/image43.png)

10. In Visual Studio Code, click on **Sign In** in the upper-right
    corner, and then sign in with your GitHub account to enable source
    control and repository operations.

![](./media/image44.png)

![](./media/image45.png)

![](./media/image46.png)

11. Open the **Source Control** view, and then click on **Publish to
    GitHub**.

![](./media/image47.png)

12. If prompted to enable the GitHub extension to sign in using GitHub,
    click on **Allow**, and then provide authorization in GitHub.

![](./media/image48.png)

![](./media/image49.png)

13. In Visual Studio Code, select **Publish to GitHub public
    repository**.

![](./media/image50.png)

14. Ensure that the **bin** and **obj** folders are not included in the
    repository.

### Task 6: Create a Docker image and push it to Azure Container Registry

1.  Ensure that you have your **AZ2003** code project open in Visual
    Studio Code.

2.  To create a Dockerfile, run the following command in the **Command
    Palette**: **Docker: Add Docker Files to Workspace**.

![](./media/image51.png)

![](./media/image52.png)

3.  When prompted, specify the following information:

| **Application Platform** | **.NET: ASP.NET Core** |
|----|----|
| **Operating System** | **Linux** |
| **Ports** | +++5000+++ |
| **Docker Compose files** | **No** |

![](./media/image53.png)

![](./media/image54.png)

![](./media/image55.png)

![](./media/image56.png)

4.  At a terminal command prompt, run the following Docker CLI command.

> +++docker build --tag aspnetcorecontainer:latest .+++

![](./media/image57.png)

**Note:** The syntax for the build command is **docker build --tag
\<image name\>:\<image tag\> .** This command builds a container image
that is hosted by Docker and accessible using the Docker extension for
VS Code.

5.  Wait for the Docker build command to complete.

![](./media/image58.png)

6.  Open the Visual Studio Code **Command Palette**, and then run the
    following command: **Docker Images: Push**.

![](./media/image59.png)

![](./media/image60.png)

7.  When the command runs, select the Docker image name that you
    created: **aspnetcorecontainer**

![](./media/image61.png)

8.  Select the image tag that you created: **latest**

![](./media/image62.png)

9.  If you see a message stating that no registry is connected, click on
    **Connect Registry**, and then enter the following information:

- **Registry provider**: Select **Azure**. Follow the online
  instructions to verify your Azure account if needed.

- **Azure subscription**: Select the Azure subscription that you're
  using for this lab.

- Select your Azure Container Registry resource. For example:
  **acraz2003cah25oct**

![](./media/image63.png)

![](./media/image64.png)

![](./media/image65.png)

![](./media/image66.png)

![](./media/image67.png)

![](./media/image68.png)

![](./media/image69.png)

10. An image tag is generated, for example
    **acraz2003cah25oct.azurecr.io/aspnetcorecontainer:latest**. Press
    **Enter** to push the image to your container registry.

![](./media/image70.png)

The following Docker command is executed:

> docker image push \<your-registry\>.azurecr.io/aspnetcorecontainer:latest

11. Wait for the image to be pushed to your Azure Container Registry.

![](./media/image71.png)

![](./media/image72.png)

12. Open the **Source Control** view, and then **Commit** and **Sync**
    your file updates.

![](./media/image73.png)

![](./media/image74.png)

![](./media/image75.png)

### Task 7: Configure Azure DevOps and a starter pipeline

1.  In the Azure portal, on the top search bar, enter +++devops+++

2.  In the search results, select **Azure DevOps organizations**.

![](./media/image76.png)

3.  Click on **View my organizations**.

![](./media/image77.png)

![](./media/image78.png)

4.  If you haven't created an organization, click on **Create new
    organization**.

![](./media/image79.png)

![](./media/image80.png)

![](./media/image81.png)

5.  On the home page of your organization, in the lower-left corner of
    the page, select **Organization settings**.

![](./media/image82.png)

6.  On the left-side menu under **Security**, select **Policies**.

![](./media/image83.png)

7.  Ensure that the **Allow public projects** policy is set to **On**.

8.  Return to the home page of your organization, and click on **New
    project**.

**Note:** If you created a new organization, you may see the **Create a
project to get started** page.

9.  On the **Create new project** page, specify the following
    information, and then click on **Create project**.

| **Project name** | +++Project1+++ |
|----|----|
| **Description** | +++AZ-2003 project+++ |
| **Visibility** | **Public** |

![](./media/image84.png)

![](./media/image85.png)

10. On the left-side menu, select **Repos**.

![](./media/image86.png)

11. Under **Import a repository**, click on **Import**.

![](./media/image87.png)

![](./media/image88.png)

12. On your GitHub repository page, click on **Code**, and then select
    the **Copy URL** icon to copy the repository HTTPS URL.

![](./media/image89.png)

![](./media/image90.png)

13. In **GitHub**, select your profile icon, choose **Settings \>
    Developer settings \> Personal access tokens \> Tokens (classic)**,
    click on **Generate new token**, and then choose **Generate new
    token (classic)**.

![](./media/image91.png)

14. On the **New personal access token (classic)** page, enter
    +++AZ2003 DevOps import+++ in the **Note** field, select the
    required expiration period, enable the **repo** scope, and then
    scroll down and click on **Generate token**.

![](./media/image92.png)

![](./media/image93.png)

15. After the token is generated, select the **Copy** icon to copy the
    personal access token, and save it in a secure location, as you will
    not be able to view the token again.

![](./media/image94.png)

![](./media/image95.png)

16. On the **Import a Git repository** page, enter the URL of the GitHub
    repository you created for your code project, provide the personal
    access token if required, and then click on **Import**.

![](./media/image96.png)

![](./media/image97.png)

**Note:** Your repository URL should be similar to
**https://github.com/\<your account\>/AZ2003**

17. On the left-side menu, select **Pipelines**, and then click on
    **Create Pipeline**.

![](./media/image98.png)

18. Select **Azure Repos Git**.

![](./media/image99.png)

19. On the **Select a repository** page, select **Project1**.

![](./media/image100.png)

20. Select **Starter pipeline**.

![](./media/image101.png)

21. Under **Save and run**, select **Save**, and then click on **Save**
    again.

![](./media/image102.png)

![](./media/image103.png)

![](./media/image104.png)

22. To rename the pipeline, on the left-side menu, select **Pipelines**.
    To the right of the **Project1** pipeline, select **More options**,
    and then select **Rename/move**.

![](./media/image105.png)

23. In the **Rename/move pipeline** dialog, under **Name**, enter
    +++Pipeline1+++ and then click on **Save**.

![](./media/image106.png)

![](./media/image107.png)

**Note:** Don't run the pipeline now. You will configure this pipeline
later in the lab.

### Task 8: Deploy a self-hosted Windows agent

For an Azure Pipeline to build and deploy Windows, Azure, and other
Visual Studio solutions, you need at least one Windows agent in the host
environment.

1.  Ensure that you're signed in to Azure DevOps with the user account
    you're using for your Azure DevOps organization, for example
    **https://dev.azure.com/{your organization}**

2.  From the home page of your organization, open your **User
    settings**, and then select **Personal access tokens**.

![](./media/image108.png)

3.  Click on **+ New Token**.

![](./media/image109.png)

4.  Under **Name**, enter +++AZ2003+++.

5.  At the bottom of the **Create a new personal access token** window,
    click on **Show all scopes**.

6.  For the custom defined scope, select **Agent Pools (Read & manage)**
    and **Deployment Groups (Read & manage)**. Ensure that all the other
    boxes are cleared.

7.  Click on **Create**.

![](./media/image110.png)

8.  On the **Success** page, click on **Copy to clipboard** to copy the
    token. You use this token when you configure the agent.

![](./media/image111.png)

![](./media/image112.png)

9.  Ensure that you're signed in to Azure DevOps as the organization
    owner. Select your DevOps organization, and then select
    **Organization settings**.

![](./media/image113.png)

10. On the left-side menu under **Pipelines**, select **Agent pools**.

![](./media/image114.png)

11. If a list of agent pools is displayed, select **Default**.

![](./media/image115.png)

12. Select the **Agents** tab, and then click on **New agent** to begin
    configuring a new self-hosted agent.

![](./media/image116.png)

13. In the **Get the agent** pane, ensure **Windows (x64)** is selected,
    and then click on **Download** to download the Azure DevOps agent
    package.

![](./media/image117.png)

![](./media/image118.png)

14. In **Windows PowerShell**, navigate to **C:\LabFiles**, create a
    folder named **agent**, switch to it, and then extract the agent
    files into the folder.

> +++cd C:\LabFiles+++
>
> +++mkdir agent ; cd agent+++

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem ; [System.IO.Compression.ZipFile]::ExtractToDirectory("$HOME\Downloads\vsts-agent-win-x64-5.279.0.zip", "$PWD")
```

**Note:** Replace **vsts-agent-win-x64-5.279.0.zip** with the name of
the file you downloaded if the version is different.

![](./media/image119.png)

15. Run the following command to configure the agent.

> +++.\config.cmd+++

When prompted, enter the following:

- **Server URL**: Your Azure DevOps organization URL, for example
  **https://dev.azure.com/\<your organization\>**

- **Authentication type**: Choose **PAT**, then paste the personal
  access token you created earlier in this task

- **Agent pool**: Press **Enter** to accept **Default**

- **Agent name**: Press **Enter** to accept the default, or enter a name

- **Run as a service**: Enter **Y** to run the agent automatically, or
  **N** to run it interactively

![](./media/image120.png)

16. In Azure DevOps, open the **Default** agent pool. Verify that the
    agent is displayed and that it started successfully.

![](./media/image121.png)

## Exercise 2: Configure Azure Container Registry for a secure connection with Azure Container Apps

In this exercise, you configure the container registry for a secure
connection from a container app. The following Azure resources must be
available in your resource group **lab2-RG** (created in Exercise 1):

- A Container Registry instance that contains one image

- A Virtual Network with subnets

- A Service Bus namespace

You've been asked to configure your Azure resources to meet the
following requirements:

- Your resource group must include a user-assigned managed identity.

- Your container registry must be able to use the managed identity to
  pull artifacts.

- Access for the managed identity must be limited using the principle
  of least privilege.

- Your container registry must be accessible from a private endpoint on
  **VNET1/PESubnet**.

### Task 1: Configure a user-assigned managed identity

1.  In the **Azure portal** search bar, enter +++Managed Identities+++,
    and then select **Managed Identities** from the search results.

![](./media/image122.png)

2.  On the **Managed Identities** page, click on **Create**.

![](./media/image123.png)

3.  On the **Create User Assigned Managed Identity** page, specify the
    following information:

| **Subscription** | Select the Azure subscription that you're using for this lab |
|----|----|
| **Resource group** | **lab2-RG** |
| **Region** | **Central US** |
| **Name** | +++uai-az2003+++ |

4.  Click on **Review + create**.

![](./media/image124.png)

5.  Click on **Create**.

![](./media/image125.png)

![](./media/image126.png)

![](./media/image127.png)

### Task 2: Configure Container Registry with AcrPull permissions for the managed identity

1.  In the Azure portal, open your **Container Registry** resource.

![](./media/image128.png)

2.  On the left-side menu, select **Access control (IAM)**.

3.  On the **Access control (IAM)** page, click on **Add role
    assignment**.

![](./media/image129.png)

4.  Search for the +++AcrPull+++ role, select **AcrPull**, and then
    click on **Next**.

![](./media/image130.png)

5.  On the **Members** tab, to the right of **Assign access to**, select
    **Managed identity**.

6.  Click on **+ Select members**.

![](./media/image131.png)

7.  On the **Select managed identities** page, under **Managed
    identity**, select **User-assigned managed identity**, and then
    select **uai-az2003**.

![](./media/image132.png)

8.  Click on **Select**.

![](./media/image133.png)

9.  On the **Members** tab of the **Add role assignment** page, click on
    **Review + assign**.

![](./media/image134.png)

10. On the **Review + assign** tab, click on **Review + assign**.

![](./media/image135.png)

11. Wait for the role assignment to be added.

![](./media/image136.png)

### Task 3: Configure Container Registry with a private endpoint connection

1.  Ensure that your **Container Registry** resource is open in the
    portal.

2.  Under **Settings**, select **Networking**.

3.  On the **Private access** tab, click on **+ Create a private
    endpoint connection**.

![](./media/image137.png)

4.  On the **Basics** tab, under **Project details**, specify the
    following information, and then click on **Next: Resource**.

| **Subscription** | Select the Azure subscription that you're using for this lab |
|----|----|
| **Resource group** | **lab2-RG** |
| **Name** | +++pe-acr-az2003+++ |
| **Region** | **Central US** |

![](./media/image138.png)

5.  On the **Resource** tab, ensure the following information is
    displayed, and then click on **Next: Virtual Network**.

| **Subscription** | The Azure subscription that you're using for this lab |
|----|----|
| **Resource type** | **Microsoft.ContainerRegistry/registries** |
| **Resource** | The name of your registry |
| **Target sub-resource** | **registry** |

![](./media/image139.png)

6.  On the **Virtual Network** tab, under **Networking**, ensure that
    **VNET1** is selected as the virtual network and **PESubnet** is
    selected as the subnet. Click on **Next: DNS**.

![](./media/image140.png)

7.  On the **DNS** tab, under **Private DNS integration**, ensure that
    **Integrate with private DNS zone** is set to **Yes** and notice
    that **(new) privatelink.azurecr.io** is specified. Click on **Next:
    Tags**.

![](./media/image141.png)

8.  Click on **Next: Review + create**.

![](./media/image142.png)

9.  On the **Review + create** tab, when you see the **Validation
    passed** message, click on **Create**.

![](./media/image143.png)

10. Wait for the deployment to complete.

![](./media/image144.png)

![](./media/image145.png)

![](./media/image146.png)

![](./media/image147.png)

## Exercise 3: Create and configure a container app in Azure Container Apps

In this exercise, you deploy a container app from an image in Azure
Container Registry to the Azure Container Apps platform. The following
Azure resources must be available in your resource group **lab2-RG**:

- A Container Registry instance that contains one image

- A Virtual Network with subnets

- A Service Bus namespace

- A Managed Identity

- A Private endpoint

You've been asked to configure a container app that meets the following
requirements:

- Is deployed to **VNET1/ACASubnet**.

- Pulls an image from the container registry.

- Authenticates using the user-assigned managed identity
  (**uai-az2003**).

- Connects to the Service Bus instance using the **.NET** client type.

- Runs up to two replicas that are added whenever there are 10,000
  concurrent HTTP requests.

### Task 1: Create a container app that uses an ACR image

1.  On the top search bar of the Azure portal, enter +++container app+++

2.  In the search results under **Services**, select **Container Apps**.

![](./media/image148.png)

3.  Click on **+ Create** \> **+ Container App**.

![](./media/image149.png)

4.  On the **Basics** tab, specify the following:

| **Subscription** | Select the Azure subscription that you're using for this lab |
|----|----|
| **Resource group** | **lab2-RG** |
| **Container app name** | +++aca-az2003+++ |
| **Region** | The region specified for **VNET1** (**Central US**) |
| **Container Apps Environment** | Select **Create new** |

**Note:** The container app needs to be in the same region as the
virtual network so you can choose VNET1 for the managed environment.

![](./media/image150.png)

5.  On the **Create Container Apps Environment** page, select the
    **Networking** tab, and then specify the following:

| **Use your own virtual network** | **Yes** |
|----|----|
| **Virtual network** | **VNET1** |
| **Infrastructure subnet** | **ACASubnet** |

**Note:** If the ACASubnet subnet is not listed, open your virtual
network resource, adjust the subnet address range to **10.0.2.0/23**,
and retry the steps to create the Container App.

6.  On the **Create Container Apps Environment** page, click on
    **Create**.

![](./media/image151.png)

![](./media/image152.png)

7.  On the **Create Container App** page, select the **Container** tab,
    and then specify the following:

| **Use quickstart image** | Ensure this setting is **not** selected |
|----|----|
| **Name** | +++aca-az2003+++ |
| **Image source** | **Azure Container Registry** |
| **Registry** | Your container registry, for example **acraz2003cah25oct.azurecr.io** |
| **Image** | **aspnetcorecontainer** |
| **Image tag** | **latest** |

8.  Click on **Review + create**.

![](./media/image153.png)

9.  Once validation has passed, click on **Create**.

![](./media/image154.png)

10. Wait for the deployment to complete.

![](./media/image155.png)

**Note:** This deployment normally takes 3-5 minutes to complete, but
may take up to 10 minutes.

![](./media/image156.png)

![](./media/image157.png)

### Task 2: Configure the container app to authenticate using the user-assigned identity

1.  In the Azure portal, open the **aca-az2003** Container App.

2.  Under **Settings**, select **Identity**.

![](./media/image158.png)

3.  Select the **User assigned** tab, and then click on **Add user
    assigned managed identity**.

![](./media/image159.png)

4.  On the **Add user assigned managed identity** page, select
    **uai-az2003**, and then click on **Add**.

![](./media/image160.png)

![](./media/image161.png)

### Task 3: Configure a connection between the container app and Service Bus

1.  On the **aca-az2003** Container App overview page, select **Now you
    can manage this application in the Azure Container Apps portal**.

![](./media/image162.png)

2.  In the Azure Container Apps portal, select **Container Apps**.

![](./media/image163.png)

3.  Select **aca-az2003** from the **Container Apps** list to open the
    application details page and review its status, configuration, and
    runtime metrics.

![](./media/image164.png)

![](./media/image165.png)

4.  In the Azure portal, select the **Cloud Shell** icon from the top
    navigation bar to open Azure Cloud Shell.

![](./media/image166.png)

5.  In the **Welcome to Azure Cloud Shell** dialog, select **Bash**.

![](./media/image167.png)

6.  Choose **No storage account required**, select your subscription,
    and then click on **Apply**.

![](./media/image168.png)

7.  In **Azure Cloud Shell**, run the following command to get the
    client ID of the managed identity. Copy the returned value and save
    it for the next step.

> +++az identity show --resource-group lab2-RG --name uai-az2003 --query clientId -o tsv+++

![](./media/image169.png)

![](./media/image170.png)

8.  Run the following command to create the Service Bus connection.
    Replace **\<your-servicebus-namespace\>** with your namespace name
    (for example **sb-az2003-cah**) and
    **\<managed-identity-client-id\>** with the client ID you copied.

```bash
az containerapp connection create servicebus \
  --resource-group lab2-RG \
  --name aca-az2003 \
  --target-resource-group lab2-RG \
  --namespace <your-servicebus-namespace> \
  --client-type dotnet \
  --user-identity client-id=<managed-identity-client-id>
```

9.  When prompted for the **container name**, enter +++aca-az2003+++,
    and then press **Enter**. Verify that the Service Bus connection is
    created successfully.

![](./media/image171.png)

![](./media/image172.png)

![](./media/image173.png)

### Task 4: Configure HTTP scale rules

1.  Ensure that your Container App is open in the portal.

2.  On the left-side menu under **Application**, select **Revisions and
    replicas**. Notice the name assigned to your active revision.

![](./media/image174.png)

3.  On the left-side menu under **Application**, select **Containers**.

![](./media/image175.png)

4.  To the right of **Based on revision**, ensure that your active
    revision is selected.

5.  At the top of the page, click on **Edit and deploy**.

6.  At the bottom of the page, click on **Next : Scale**.

![](./media/image176.png)

7.  Configure the replicas as follows:

| **Min replicas** | +++0+++ |
|----|----|
| **Max replicas** | +++2+++ |

8.  Under **Scale rule**, click on **+ Add**.

![](./media/image177.png)

9.  On the **Add scale rule** page, specify the following, and then
    click on **Add scale rule**.

| **Rule name** | +++scalerule-http+++ |
|----|----|
| **Type** | **HTTP scaling** |
| **Concurrent requests** | +++10000+++ |

![](./media/image178.png)

10. On the **Create and deploy new revision** page, click on **Save a
    new revision**.

![](./media/image179.png)

11. Ensure that your new scale rule is displayed.

![](./media/image180.png)

![](./media/image181.png)

## Exercise 4: Configure continuous integration by using Azure Pipelines

In this exercise, you configure Pipeline1 to deploy the container image
from your container registry to your container app using the self-hosted
agent pool. The following Azure resources must be available in your
resource group **lab2-RG**:

- A Container Registry instance that contains one image

- A Virtual Network with subnets

- A Service Bus namespace

- A Managed Identity

- A Private endpoint

- A Container App and a Container Apps Environment

You've been asked to configure a continuous integration environment for
Container Apps that meets the following requirements:

- You need an Azure Container Apps deployment task in your Azure DevOps
  environment.

- Pipeline1 must deploy a container image from your container registry
  to your container app using a self-hosted agent pool.

- You must ensure that the pipeline successfully deploys the image at
  least once.

### Task 1: Configure Pipeline1 to use the self-hosted agent pool

1.  Open a browser window, navigate to +++https://dev.azure.com+++, and
    then open your Azure DevOps organization.

2.  Select **Project1**, and then in the left-side menu, select
    **Pipelines**.

![](./media/image182.png)

3.  Select **Pipeline1**, and then click on **Edit**.

![](./media/image183.png)

4.  To use the self-hosted agent pool, update the **azure-pipelines.yml**
    file as shown in the following example.

```yaml
trigger:
- main

pool:
  name: default

steps:
```

![](./media/image184.png)

**Note:** The **pool** section specifies the agent pool to use for the
pipeline. The **name** property is **default**, which is the pool you
configured with the self-hosted agent.

5.  Under **Validate and save**, select **Save without validating**.

![](./media/image185.png)

6.  Enter a commit message, and then click on **Save**.

![](./media/image186.png)

### Task 2: Configure Pipeline1 with an Azure Container Apps deployment task

1.  Ensure that you have **Pipeline1** open for editing.

2.  On the right side under **Tasks**, in the **Search tasks** field,
    enter +++azure container+++

3.  In the filtered list of tasks, select **Azure Container Apps
    Deploy**.

![](./media/image187.png)

4.  Under **Azure Resource Manager connection**, select the subscription
    you're using, and then click on **Authorize**.

![](./media/image188.png)

![](./media/image189.png)

5.  In the Azure portal tab, open your Container App resource, and then
    open the **Containers** page. Note down the following three values
    exactly as shown:

- **Registry**: for example **acraz2003cah25oct.azurecr.io**

- **Image**: **aspnetcorecontainer**

- **Image tag**: **latest**

![](./media/image190.png)

6.  Go back to your Azure DevOps tab, where the task panel is still
    open.

![](./media/image191.png)

7.  Leave **Application source path** empty (you're deploying an
    already-built image, not building from source code).

8.  Leave the **Azure Container Registry name**, **username** and
    **password** fields empty.

9.  Scroll down inside the task panel and configure the following
    fields:

| **Docker Image to Deploy** | \<Registry\>/\<Image\>:\<Image tag\>, for example **acraz2003cah25oct.azurecr.io/aspnetcorecontainer:latest** |
|----|----|
| **Azure Container App name** | +++aca-az2003+++ |
| **Azure Resource group name** | +++lab2-RG+++ |

**Note:** If you need to verify the resource group name, you can find it
on the **Overview** page of your Container App resource.

10. On the **Azure Container Apps Deploy** page, click on **Add**.

![](./media/image192.png)

11. The YAML file for your pipeline should now include the
    **AzureContainerApps** task as follows:

```yaml
trigger:
- main

pool:
  name: default

steps:
- task: AzureContainerApps@1
  inputs:
    azureSubscription: '<Subscription name>(<Subscription ID>)'
    imageToDeploy: '<Registry>/<Image>:<Image tag>'
    containerAppName: 'aca-az2003'
    resourceGroup: 'lab2-RG'
```

![](./media/image193.png)

Here's an example that shows a completed YAML configuration:

```yaml
trigger:
- main

pool:
  name: default

steps:
- task: AzureContainerApps@1
  inputs:
    azureSubscription: 'My Azure Subscription(00000000-0000-0000-0000-000000000000)'
    imageToDeploy: 'acraz2003cah25oct.azurecr.io/aspnetcorecontainer:latest'
    containerAppName: 'aca-az2003'
    resourceGroup: 'lab2-RG'
```

12. Click on **Validate and save**, and then click on **Save** again to
    commit directly to the main branch.

![](./media/image194.png)

![](./media/image195.png)

**Note:** The contents of the YAML file must be formatted correctly,
including indentation. If you encounter an error, review the YAML file
and correct any indentation issues.

13. Navigate back to the main page of your pipeline.

### Task 3: Run the Pipeline1 deployment task

1.  Ensure that you have **Pipeline1** open in Azure DevOps.

![](./media/image196.png)

2.  On the **Runs** tab of the Pipeline1 page, click on **Run
    pipeline**.

![](./media/image197.png)

3.  A **Run pipeline** page opens to display the associated job. Click
    on **Run**.

![](./media/image198.png)

4.  The **Jobs** section displays the job status, which progresses from
    **Queued** to **Waiting**. It can take a couple of minutes for the
    status to change.

![](./media/image199.png)

5.  If a **Permission needed** message is displayed ("This pipeline
    needs permission to access 2 resources before this run can
    continue"), click on **View** and then provide the required
    permissions.

![](./media/image200.png)

![](./media/image201.png)

![](./media/image202.png)

![](./media/image203.png)

![](./media/image204.png)

![](./media/image205.png)

![](./media/image206.png)

6.  Monitor the status of the run and verify that the run is
    successful.

![](./media/image207.png)

### Task 4: Verify the pipeline deployment

1.  Ensure that you have **Project1** open in Azure DevOps.

2.  On the left-side menu, select **Pipelines**, and then select
    **Pipeline1**. The **Runs** tab displays individual runs that can be
    selected to review details.

![](./media/image208.png)

3.  Open your Azure portal, and then open your Container App.

4.  On the left-side menu, select **Activity log**.

5.  Verify that a **Create or Update Container App** operation succeeded
    as a result of running your pipeline. Notice that the **Event
    initiated by** column shows your **Project1** as the source.

![](./media/image209.png)

## Exercise 5: Manage revisions in Azure Container Apps

In this exercise, you deploy a new revision of your container app and
configure traffic splitting between two labeled revisions.

You've been asked to configure traffic splitting for your Container
Apps to meet the following requirements:

- You need to create a new revision of the container app that uses a
  suffix of **v2**.

- You must ensure that 25 percent of requests to your app are directed
  to the v2 revision.

- You must label the revisions **current** and **updated** and ensure
  that requests to the **updated** revision are directed to the v2
  revision.

### Task 1: Set revision management to multiple

1.  In the Azure portal, open your container app (**aca-az2003**).

2.  Under **Application** in the left menu, select **Revisions and
    replicas**.

![](./media/image210.png)

3.  In the top toolbar, click on **Deployment mode**.

![](./media/image211.png)

4.  In the panel that opens on the right, select **Multiple
    revisions**.

![](./media/image212.png)

5.  Click on **Go to ingress** (or **Confirm**, depending on what the
    panel shows) to save the change.

![](./media/image213.png)

6.  On the **Ingress** page, select the **Ingress** checkbox to enable
    ingress for the Container App.

![](./media/image214.png)

7.  Select **Accepting traffic from anywhere** for **Ingress traffic**,
    keep **HTTP** as the **Ingress type**, and then click on **Save**.

![](./media/image215.png)

![](./media/image216.png)

### Task 2: Create a new revision with a v2 suffix

1.  Ensure that you have the **Revisions and replicas** page of your
    container app open.

2.  At the top of the page, click on **+ Create new revision**.

![](./media/image217.png)

3.  On the **Create and deploy new revision** page, enter +++v2+++ in
    **Name / suffix**, and under **Container image**, select your
    container image (for example **aca-az2003**).

4.  Click on **Create**.

![](./media/image218.png)

5.  Wait for the deployment to be completed.

![](./media/image219.png)

![](./media/image220.png)

### Task 3: Configure labels on the revisions

1.  On the left-side menu, under **Settings**, select **Ingress**.

![](./media/image221.png)

2.  If ingress isn't enabled, select **Enabled**.

3.  On the **Ingress** page, specify the following information:

| **Ingress traffic** | **Accepting traffic from anywhere** |
|----|----|
| **Ingress type** | **HTTP** |
| **Client certificate mode** | **Ignore** |
| **Transport** | **Auto** |
| **Insecure connections** | Ensure that **Allowed** is **NOT** checked |
| **Target port** | +++5000+++ |
| **IP Security Restrictions Mode** | **Allow all traffic** |

![](./media/image222.png)

4.  At the bottom of the **Ingress** page, click on **Save**, and then
    wait for the update to complete.

![](./media/image223.png)

![](./media/image224.png)

5.  On the left-side menu, select **Revisions and replicas**.

![](./media/image225.png)

6.  For the **v2** revision, under **Label**, enter +++updated+++

7.  For the other revision, enter +++current+++

8.  At the top of the page, click on **Save**.

![](./media/image226.png)

![](./media/image227.png)

### Task 4: Configure a traffic percentage on the revisions

1.  Ensure that you have the **Revisions and replicas** page open.

2.  For the **v2** revision, under **Traffic**, enter +++25+++ as the
    percentage.

3.  For the other revision, under **Traffic**, enter +++75+++ as the
    percentage.

4.  At the top of the page, click on **Save**.

![](./media/image228.png)

**Summary**

In this lab, you configured the Azure infrastructure needed to host a
secure and scalable containerized application. You created and
containerized an ASP.NET Web API, stored the image in Azure Container
Registry, and deployed it to Azure Container Apps. You secured
application resources using managed identities, RBAC permissions, and
private endpoints. Additionally, you integrated the application with
Azure Service Bus, configured autoscaling policies, and automated
deployments through Azure DevOps Pipelines. Finally, you managed
application revisions and implemented traffic splitting to support
progressive deployment strategies. These tasks demonstrated key
cloud-native deployment, security, automation, and operational
management capabilities available in Azure Container Apps.
