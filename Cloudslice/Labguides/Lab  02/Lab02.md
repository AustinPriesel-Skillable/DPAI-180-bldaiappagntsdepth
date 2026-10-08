# Lab 2: Deploy and Manage a Secure, Scalable Container App with Azure Container Apps

## Scenario

**Contoso Ltd.** is modernizing its line-of-business services by moving them to containers. The development team has built a new ASP.NET Web API that will publish order events to **Azure Service Bus**, and the operations team wants to run it on a fully managed platform without operating Kubernetes clusters.

Today, deployments are manual, container images are pulled with shared admin credentials, and every release replaces the previous version in one step. This makes releases risky and difficult to audit, and the security team has asked for private network access and least-privilege identities for all workloads.

To address these challenges, Contoso has chosen **Azure Container Apps**. The platform team wants images stored in a private **Azure Container Registry**, pulled through a **user-assigned managed identity** over a **private endpoint**, and deployed automatically through **Azure Pipelines**. New versions must be rolled out gradually using **revisions and traffic splitting**.

As a Cloud Engineer, your responsibility is to provision the Azure infrastructure, containerize the Web API, secure access to the container registry, deploy the container app into a virtual network, connect it to Service Bus, configure autoscaling, automate deployments with Azure DevOps, and manage revisions for controlled rollouts.

## Introduction

In this lab, you will learn how to deploy, secure, scale, and manage containerized applications using **Azure Container Apps**. You will provision the required Azure infrastructure, including Virtual Networks, Azure Container Registry, Service Bus, Managed Identities, and Azure DevOps resources. You will then deploy a containerized ASP.NET application, configure secure access using managed identities and private endpoints, implement autoscaling, automate deployments through Azure Pipelines, and manage application revisions for controlled traffic distribution.

### Azure resources used in this lab

| **Resource** | **Name** | **Role in the lab** |
|----|----|----|
| Resource group | @lab.CloudResourceGroup(ResourceGroup1).Name | Holds all lab resources |
| Virtual network | VNET1 (PESubnet, ACASubnet) | Private networking for the registry and container app |
| Service Bus namespace | sb-az@lab.LabInstance.Id | Messaging service the app connects to |
| Azure Container Registry (Premium) | acraz@lab.LabInstance.Id| Stores the aspnetcorecontainer image |
| User-assigned managed identity | uai-az@lab.LabInstance.Id | Identity used to pull images and connect to Service Bus |
| Private endpoint | pe-acr-az@lab.LabInstance.Id | Private access to the registry from VNET1 |
| Container App | aca-az@lab.LabInstance.Id | Runs the ASP.NET Web API |
| Azure DevOps project | Project1 / Pipeline1 | CI/CD pipeline and self-hosted agent |

### Objectives

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


### Prerequisites

- **GitHub Account**: You are expected to have your own GitHub login
  credentials. If you do not have an account, create one by visiting: +++https://github.com/signup+++


## Exercise 1: Configure deployment tools and Azure resources

Before deploying the container app, you need to prepare the Azure environment and your development tools. In this exercise, you will create the resource group, virtual network, Service Bus and Container Registry, build and push the container image, and configure Azure DevOps with a starter pipeline and a self-hosted agent.

### Task 1: Configure a resource group for your Azure resources

1. Open your browser, navigate to the address bar, and type or paste the following URL: +++https://portal.azure.com/+++ then press the **Enter** button. Sign in with the following credentials.

    | **Username** | +++@lab.CloudPortalCredential(User1).Username+++ |
    |----|----|
    | **Password** | +++@lab.CloudPortalCredential(User1).AccessToken+++ |

1. On the top search bar of the Azure portal, in the Search textbox, enter +++resource group+++

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image1.png)

1. In the search results, select **Resource groups**, and then click on **+ Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image2.png)

1. On the **Basics** tab, configure the resource group as follows:

    | **Subscription** | Select the Azure subscription that you're using for this lab |
    |----|----|
    | **Resource group** | @lab.CloudResourceGroup(ResourceGroup1).Name |
    | **Region** | **Central US** |

1. Click on **Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image3.png)

1. Once validation has passed, click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image4.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image5.png)


### Task 2: Configure a virtual network and subnets

1. Ensure that you have your Azure portal open in a browser window.

1. On the top search bar of the Azure portal, in the Search textbox, enter +++virtual network+++. In the search results, select **Virtual networks**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image6.png)

1. Click on **+ Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image7.png)

1. On the **Basics** tab, configure your virtual network as follows, and then click on **Next**.

    | **Subscription** | Select the Azure subscription that you're using for this lab |
    |----|----|
    | **Resource group** | **@lab.CloudResourceGroup(ResourceGroup1).Name** |
    | **Virtual network name** | +++VNET1+++ |
    | **Region** | **Central US** |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image8.png)

1. On the **Security** tab, keep the default settings, and then click on **Next**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image9.png)

1. Select the **IP addresses** tab.

1. On the **IP addresses** tab, under **Subnets**, select **default**.

1. On the **Edit subnet** page, configure the subnet as follows, and then click on **Save**.

    | **Name** | +++PESubnet+++ |
    |----|----|
    | **Starting address** | **10.0.0.0** |
    | **Subnet size** | **/24 (256 addresses)** |

1. On the **IP addresses** tab, click on **+ Add a subnet**.

1. On the **Add a subnet** page, configure the subnet as follows, and then click on **Add**.

    | **Name** | +++ACASubnet+++ |
    |----|----|
    | **Starting address** | **10.0.4.0** |
    | **Subnet size** | **/23 (512 addresses)** |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image10.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image11.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image12.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image13.png)

1. Click on **Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image14.png)

1. Once validation has passed, click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image15.png)

1. Wait for the deployment to complete.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image16.png)

1. After the deployment completes successfully, click on **Go to resource** to open the newly created virtual network and review its configuration.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image17.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image18.png)


### Task 3: Configure Service Bus

1. In the **Azure portal** search bar, enter `Service Bus`, and then select **Service Bus** from the search results.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image19.png)

1. Click on **Create service bus namespace**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image20.png)

1. On the **Basics** tab, configure your Service Bus namespace as follows:

    | **Subscription** | Select the Azure subscription that you're using for this lab |
    |----|----|
    | **Resource group** | **@lab.CloudResourceGroup(ResourceGroup1).Name** |
    | **Namespace name** | +++sb-az@lab.LabInstance.Id+++|
    | **Location** | **Central US** |
    | **Pricing tier** | **Basic** |

1. Click on **Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image21.png)

1. Once the **Validation succeeded** message appears, click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image22.png)

1. Wait for the deployment to complete.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image23.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image24.png)


### Task 4: Configure Azure Container Registry

1. On the top search bar of the Azure portal, in the Search textbox, enter +++container registry+++

1. In the search results, select **Container registries**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image25.png)

1. On the **Container registries** page, click on **Create container registry** or **+ Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image26.png)

1. On the **Basics** tab of the **Create container registry** page, specify the following information:

    >[!Note] The name of your registry must be unique. Also, the **Premium** tier is required for private link with private endpoints.

    | **Subscription** | Select the Azure subscription that you're using for this lab |
    |----|----|
    | **Resource group** | **@lab.CloudResourceGroup(ResourceGroup1).Name** |
    | **Registry name** | +++acraz@lab.LabInstance.Id+++ |
    | **Location** | **Central US** |
    | **SKU** | **Premium** |

1. Click on **Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image27.png)

1. Click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image28.png)

1. After the deployment has completed, open the deployed resource.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image29.png)

1. On the left-side menu, under **Settings**, select **Networking**.

1. On the **Networking** page, on the **Public access** tab, ensure that **All networks** is selected.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image30.png)

1. On the left-side menu, under **Settings**, select **Properties**.

1. On the **Properties** page, select **Admin user**, and then click on **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image31.png)


### Task 5: Create a Web API app and publish to a GitHub repository

1. Open **Visual Studio Code**.

1. On the **File** menu, select **Open Folder**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image32.png)

1. Create a new folder named +++AZ@lab.LabInstance.Id+++ in a location that is easy to find, for example on the Windows Desktop, and open it.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image33.png)

1. On the **Terminal** menu, select **New Terminal**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image34.png)

1. At the terminal command prompt, run the following command to create a new ASP.NET Web API project.

    `dotnet new webapi --no-https`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image35.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image36.png)

1. At the terminal command prompt, run the following command to build the project.

    `dotnet build`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image37.png)

1. On the **View** menu, select **Command Palette**, and then run the following command: **.NET: Generate Assets for Build and Debug**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image38.png)

    >[!Note] If the command generates an error message, select **OK**, and then run the command again. Alternatively, open **Program.cs** and press **F5** - VS Code detects that there is no debug configuration yet and offers to create it for you, which gives the same result.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image39.png)

1. In the root project folder, create a **.gitignore** file that contains the following information.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image40.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image41.png)

    ```text
    [Bb]in/
    [Oo]bj/
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image42.png)

1. On the **File** menu, select **Save All**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image43.png)

1. In Visual Studio Code, click on **Sign In** in the upper-right corner, and then sign in with your GitHub account to enable source control and repository operations.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image44.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image45.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image46.png)

1. Open the **Source Control** view, and then click on **Publish to GitHub**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image47.png)

1. If prompted to enable the GitHub extension to sign in using GitHub, click on **Allow**, and then provide authorization in GitHub.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image48.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image49.png)

1. In Visual Studio Code, select **Publish to GitHub public repository**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image50.png)

1. Ensure that the **bin** and **obj** folders are not included in the repository.


### Task 6: Create a Docker image and push it to Azure Container Registry

1. Ensure that you have your **AZ@lab.LabInstance.Id** code project open in Visual Studio Code.

1. To create a Dockerfile, run the following command in the **Command Palette**: +++> Docker: Add Docker Files to Workspace+++.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image51.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image52.png)

1. When prompted, specify the following information:

    | **Application Platform** | **.NET: ASP.NET Core** |
    |----|----|
    | **Operating System** | **Linux** |
    | **Ports** | +++5000+++ |
    | **Docker Compose files** | **No** |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image53.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image54.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image55.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image56.png)

1. At a terminal command prompt, run the following Docker CLI command.

    `docker build --tag aspnetcorecontainer:latest .`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image57.png)

    >[!Note] The syntax for the build command is **docker build --tag *{image name}*:*{image tag}* .** This command builds a container image that is hosted by Docker and accessible using the Docker extension for VS Code.

1. Wait for the Docker build command to complete.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image58.png)

1. Open the Visual Studio Code **Command Palette**, and then run the following command: +++> Docker Images: Push+++.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image59.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image60.png)

1. When the command runs, select the Docker image name that you created: +++aspnetcorecontainer+++

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image61.png)

1. Select the image tag that you created: **latest**

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image62.png)

1. If you see a message stating that no registry is connected, click on **Connect Registry**, and then enter the following information:

    - **Registry provider**: Select **Azure**. Follow the online
    instructions to verify your Azure account if needed.

    - **Azure subscription**: Select the Azure subscription that you're
    using for this lab.

    - Select your Azure Container Registry resource. For example:
    **acraz@lab.LabInstance.Id**

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image63.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image64.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image65.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image66.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image67.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image68.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image69.png)


1. An image tag is generated, for example **acraz@lab.LabInstance.Id.azurecr.io/aspnetcorecontainer:latest**. Press **Enter** to push the image to your container registry.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image70.png)

    The following Docker command is executed:

    > docker image push *{your-registry}*.azurecr.io/aspnetcorecontainer:latest

1. Wait for the image to be pushed to your Azure Container Registry.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image71.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image72.png)

1. Open the **Source Control** view, and then **Commit** and **Sync** your file updates.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image73.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image74.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image75.png)


### Task 7: Configure Azure DevOps and a starter pipeline

1. In the Azure portal, on the top search bar, enter +++devops+++

1. In the search results, select **Azure DevOps organizations**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image76.png)

1. Click on **View my organizations**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image77.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image78.png)

1. If you haven't created an organization, click on **Create new organization**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image79.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image80.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image81.png)

1. On the home page of your organization, in the lower-left corner of the page, select **Organization settings**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image82.png)

1. On the left-side menu under **Security**, select **Policies**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image83.png)

1. Ensure that the **Allow public projects** policy is set to **On**.

1. Return to the home page of your organization, and click on **New project**.

    >[!Note] If you created a new organization, you may see the **Create a project to get started** page.

1. On the **Create new project** page, specify the following information, and then click on **Create project**.

    | **Project name** | +++Project1+++ |
    |----|----|
    | **Description** | `AZ-@lab.LabInstance.Id project` |
    | **Visibility** | **Public** |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image84.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image85.png)

1. On the left-side menu, select **Repos**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image86.png)

1. Under **Import a repository**, click on **Import**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image87.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image88.png)

1. On your GitHub repository page, click on **Code**, and then select the **Copy URL** icon to copy the repository HTTPS URL.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image89.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image90.png)

1. In **GitHub**, select your profile icon, choose **Settings \> Developer settings \> Personal access tokens \> Tokens (classic)**, click on **Generate new token**, and then choose **Generate new token (classic)**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image91.png)

1. On the **New personal access token (classic)** page, enter +++AZ@lab.LabInstance.Id DevOps import+++ in the **Note** field, select the required expiration period, enable the **repo** scope, and then scroll down and click on **Generate token**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image92.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image93.png)

1. After the token is generated, select the **Copy** icon to copy the personal access token, and save it in a secure location, as you will not be able to view the token again.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image94.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image95.png)

1. On the **Import a Git repository** page, enter the URL of the GitHub repository you created for your code project, provide the personal access token if required, and then click on **Import**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image96.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image97.png)

    >[!Note] Your repository URL should be similar to **https://github.com/*{your account}*/AZ@lab.LabInstance.Id**

1. On the left-side menu, select **Pipelines**, and then click on **Create Pipeline**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image98.png)

1. Select **Azure Repos Git**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image99.png)

1. On the **Select a repository** page, select **Project1**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image100.png)

1. Select **Starter pipeline**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image101.png)

1. Under **Save and run**, select **Save**, and then click on **Save** again.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image102.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image103.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image104.png)

1. To rename the pipeline, on the left-side menu, select **Pipelines**. To the right of the **Project1** pipeline, select **More options**, and then select **Rename/move**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image105.png)

1. In the **Rename/move pipeline** dialog, under **Name**, enter +++Pipeline1+++ and then click on **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image106.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image107.png)

    >[!Note] Don't run the pipeline now. You will configure this pipeline later in the lab.


### Task 8: Deploy a self-hosted Windows agent

For an Azure Pipeline to build and deploy Windows, Azure, and other Visual Studio solutions, you need at least one Windows agent in the host environment.

1. Ensure that you're signed in to Azure DevOps with the user account you're using for your Azure DevOps organization, for example **https://dev.azure.com/{your organization}**

1. From the home page of your organization, open your **User settings**, and then select **Personal access tokens**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image108.png)

1. Click on **+ New Token**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image109.png)

1. Under **Name**, enter +++AZ@lab.LabInstance.Id+++.

1. At the bottom of the **Create a new personal access token** window, click on **Show all scopes**.

1. For the custom defined scope, select **Agent Pools (Read & manage)** and **Deployment Groups (Read & manage)**. Ensure that all the other boxes are cleared.

1. Click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image110.png)

1. On the **Success** page, click on **Copy to clipboard** to copy the token. You use this token when you configure the agent.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image111.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image112.png)

1. Ensure that you're signed in to Azure DevOps as the organization owner. Select your DevOps organization, and then select **Organization settings**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image113.png)

1. On the left-side menu under **Pipelines**, select **Agent pools**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image114.png)

1. If a list of agent pools is displayed, select **Default**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image115.png)

1. Select the **Agents** tab, and then click on **New agent** to begin configuring a new self-hosted agent.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image116.png)

1. In the **Get the agent** pane, ensure **Windows (x64)** is selected, and then click on **Download** to download the Azure DevOps agent package.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image117.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image118.png)

1. In **Windows PowerShell**, navigate to **C:\LabFiles**, create a folder named +++agent+++, switch to it, and then extract the agent files into the folder.

    - `cd C:\LabFiles`
    - `mkdir agent ; cd agent`

    `Add-Type -AssemblyName System.IO.Compression.FileSystem ; [System.IO.Compression.ZipFile]::ExtractToDirectory("$HOME\Downloads\vsts-agent-win-x64-5.279.0.zip", "$PWD")`

    >[!Note] Replace **vsts-agent-win-x64-5.279.0.zip** with the name of the file you downloaded if the version is different.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image119.png)


1. Run the following command to configure the agent.

    `.\config.cmd`

    When prompted, enter the following:
    - **Server URL**: Your Azure DevOps organization URL, for example
    **https://dev.azure.com/*{your organization}***

    - **Authentication type**: Choose **PAT**, then paste the personal
    access token you created earlier in this task

    - **Agent pool**: Press **Enter** to accept **Default**
    - **Agent name**: Press **Enter** to accept the default, or enter a name
    - **Run as a service**: Enter +++Y+++ to run the agent automatically, or
    **N** to run it interactively

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image120.png)


1. In Azure DevOps, open the **Default** agent pool. Verify that the agent is displayed and that it started successfully.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image121.png)


## Exercise 2: Configure Azure Container Registry for a secure connection with Azure Container Apps

In this exercise, you configure the container registry for a secure connection from a container app. The following Azure resources must be available in your resource group **@lab.CloudResourceGroup(ResourceGroup1).Name** (created in Exercise 1):
- A Container Registry instance that contains one image
- A Virtual Network with subnets
- A Service Bus namespace


You've been asked to configure your Azure resources to meet the following requirements:
- Your resource group must include a user-assigned managed identity.
- Your container registry must be able to use the managed identity to
  pull artifacts.

- Access for the managed identity must be limited using the principle
  of least privilege.

- Your container registry must be accessible from a private endpoint on
  **VNET1/PESubnet**.


### Task 1: Configure a user-assigned managed identity

1. In the **Azure portal** search bar, enter +++Managed Identities+++, and then select **Managed Identities** from the search results.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image122.png)

1. On the **Managed Identities** page, click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image123.png)

1. On the **Create User Assigned Managed Identity** page, specify the following information:

    | **Subscription** | Select the Azure subscription that you're using for this lab |
    |----|----|
    | **Resource group** | **@lab.CloudResourceGroup(ResourceGroup1).Name** |
    | **Region** | **Central US** |
    | **Name** | +++uai-az@lab.LabInstance.Id+++ |

1. Click on **Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image124.png)

1. Click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image125.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image126.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image127.png)


### Task 2: Configure Container Registry with AcrPull permissions for the managed identity

1. In the Azure portal, open your **Container Registry** resource.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image128.png)

1. On the left-side menu, select **Access control (IAM)**.

1. On the **Access control (IAM)** page, click on **Add role assignment**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image129.png)

1. Search for the **AcrPull** role, select **AcrPull**, and then click on **Next**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image130.png)

1. On the **Members** tab, to the right of **Assign access to**, select **Managed identity**.

1. Click on **+ Select members**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image131.png)

1. On the **Select managed identities** page, under **Managed identity**, select **User-assigned managed identity**, and then select **uai-az@lab.LabInstance.Id**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image132.png)

1. Click on **Select**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image133.png)

1. On the **Members** tab of the **Add role assignment** page, click on **Review + assign**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image134.png)

1. On the **Review + assign** tab, click on **Review + assign**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image135.png)

1. Wait for the role assignment to be added.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image136.png)


### Task 3: Configure Container Registry with a private endpoint connection

1. Ensure that your **Container Registry** resource is open in the portal.

1. Under **Settings**, select **Networking**.

1. On the **Private access** tab, click on **+ Create a private endpoint connection**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image137.png)

1. On the **Basics** tab, under **Project details**, specify the following information, and then click on **Next: Resource**.

    | **Subscription** | Select the Azure subscription that you're using for this lab |
    |----|----|
    | **Resource group** | **@lab.CloudResourceGroup(ResourceGroup1).Name** |
    | **Name** | +++pe-acr-az@lab.LabInstance.Id+++ |
    | **Region** | **Central US** |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image138.png)

1. On the **Resource** tab, ensure the following information is displayed, and then click on **Next: Virtual Network**.

    | **Subscription** | The Azure subscription that you're using for this lab |
    |----|----|
    | **Resource type** | **Microsoft.ContainerRegistry/registries** |
    | **Resource** | The name of your registry |
    | **Target sub-resource** | **registry** |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image139.png)

1. On the **Virtual Network** tab, under **Networking**, ensure that **VNET1** is selected as the virtual network and **PESubnet** is selected as the subnet. Click on **Next: DNS**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image140.png)

1. On the **DNS** tab, under **Private DNS integration**, ensure that **Integrate with private DNS zone** is set to **Yes** and notice that **(new) privatelink.azurecr.io** is specified. Click on **Next: Tags**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image141.png)

1. Click on **Next: Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image142.png)

1. On the **Review + create** tab, when you see the **Validation passed** message, click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image143.png)

1. Wait for the deployment to complete.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image144.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image145.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image146.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image147.png)


## Exercise 3: Create and configure a container app in Azure Container Apps

In this exercise, you deploy a container app from an image in Azure Container Registry to the Azure Container Apps platform. The following Azure resources must be available in your resource group **@lab.CloudResourceGroup(ResourceGroup1).Name**:
- A Container Registry instance that contains one image
- A Virtual Network with subnets
- A Service Bus namespace
- A Managed Identity
- A Private endpoint


You've been asked to configure a container app that meets the following requirements:
- Is deployed to **VNET1/ACASubnet**.
- Pulls an image from the container registry.
- Authenticates using the user-assigned managed identity
  (**uai-az@lab.LabInstance.Id**).

- Connects to the Service Bus instance using the **.NET** client type.
- Runs up to two replicas that are added whenever there are 10,000
  concurrent HTTP requests.


### Task 1: Create a container app that uses an ACR image

1. On the top search bar of the Azure portal, enter +++container app+++

1. In the search results under **Services**, select **Container Apps**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image148.png)

1. Click on **+ Create** \> **+ Container App**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image149.png)

1. On the **Basics** tab, specify the following:

    | **Subscription** | Select the Azure subscription that you're using for this lab |
    |----|----|
    | **Resource group** | **@lab.CloudResourceGroup(ResourceGroup1).Name** |
    | **Container app name** | +++aca-az@lab.LabInstance.Id+++ |
    | **Region** | The region specified for **VNET1** (**Central US**) |
    | **Container Apps Environment** | Select **Create new** |

    >[!Note] The container app needs to be in the same region as the virtual network so you can choose VNET1 for the managed environment.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image150.png)

1. On the **Create Container Apps Environment** page, select the **Networking** tab, and then specify the following:

    | **Use your own virtual network** | **Yes** |
    |----|----|
    | **Virtual network** | **VNET1** |
    | **Infrastructure subnet** | **ACASubnet** |

    >[!Note] If the ACASubnet subnet is not listed, open your virtual network resource, adjust the subnet address range to **10.0.2.0/23**, and retry the steps to create the Container App.

1. On the **Create Container Apps Environment** page, click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image151.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image152.png)

1. On the **Create Container App** page, select the **Container** tab, and then specify the following:

    | **Use quickstart image** | Ensure this setting is **not** selected |
    |----|----|
    | **Name** | +++aca-az@lab.LabInstance.Id+++ |
    | **Image source** | **Azure Container Registry** |
    | **Registry** | Your container registry, for example **acraz@lab.LabInstance.Id.azurecr.io** |
    | **Image** | **aspnetcorecontainer** |
    | **Image tag** | **latest** |

1. Click on **Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image153.png)

1. Once validation has passed, click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image154.png)

1. Wait for the deployment to complete.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image155.png)

    >[!Note] This deployment normally takes 3-5 minutes to complete, but may take up to 10 minutes.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image156.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image157.png)


### Task 2: Configure the container app to authenticate using the user-assigned identity

1. In the Azure portal, open the **aca-az@lab.LabInstance.Id** Container App.

1. Under **Settings**, select **Identity**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image158.png)

1. Select the **User assigned** tab, and then click on **Add user assigned managed identity**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image159.png)

1. On the **Add user assigned managed identity** page, select **uai-az@lab.LabInstance.Id**, and then click on **Add**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image160.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image161.png)


### Task 3: Configure a connection between the container app and Service Bus

1. On the **aca-az@lab.LabInstance.Id** Container App overview page, select **Now you can manage this application in the Azure Container Apps portal**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image162.png)

1. In the Azure Container Apps portal, select **Container Apps**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image163.png)

1. Select **aca-az@lab.LabInstance.Id** from the **Container Apps** list to open the application details page and review its status, configuration, and runtime metrics.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image164.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image165.png)

1. In the Azure portal, select the **Cloud Shell** icon from the top navigation bar to open Azure Cloud Shell.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image166.png)

1. In the **Welcome to Azure Cloud Shell** dialog, select **Bash**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image167.png)

1. Choose **No storage account required**, select your subscription, and then click on **Apply**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image168.png)

1. In **Azure Cloud Shell**, run the following command to get the client ID of the managed identity. Copy the returned value and save it for the next step.

    `az identity show --resource-group @lab.CloudResourceGroup(ResourceGroup1).Name --name uai-az@lab.LabInstance.Id --query clientId -o tsv`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image169.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image170.png)

1. Run the following command to create the Service Bus connection. Replace ***{your-servicebus-namespace}*** with your namespace name (for example +++sb-az@lab.LabInstance.Id+++) and ***{managed-identity-client-id}*** with the client ID you copied.

    ```bash
    az containerapp connection create servicebus \
      --resource-group @lab.CloudResourceGroup(ResourceGroup1).Name \
      --name aca-az@lab.LabInstance.Id \
      --target-resource-group @lab.CloudResourceGroup(ResourceGroup1).Name \
      --namespace <your-servicebus-namespace> \
      --client-type dotnet \
      --user-identity client-id=<managed-identity-client-id>
    ```

1. When prompted for the **container name**, enter +++aca-az@lab.LabInstance.Id+++, and then press **Enter**. Verify that the Service Bus connection is created successfully.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image171.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image172.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image173.png)


### Task 4: Configure HTTP scale rules

1. Ensure that your Container App is open in the portal.

1. On the left-side menu under **Application**, select **Revisions and replicas**. Notice the name assigned to your active revision.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image174.png)

1. On the left-side menu under **Application**, select **Containers**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image175.png)

1. To the right of **Based on revision**, ensure that your active revision is selected.

1. At the top of the page, click on **Edit and deploy**.

1. At the bottom of the page, click on **Next : Scale**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image176.png)

1. Configure the replicas as follows:

    | **Min replicas** | +++0+++ |
    |----|----|
    | **Max replicas** | +++2+++ |

1. Under **Scale rule**, click on **+ Add**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image177.png)

1. On the **Add scale rule** page, specify the following, and then click on **Add scale rule**.

    | **Rule name** | +++scalerule-http+++ |
    |----|----|
    | **Type** | **HTTP scaling** |
    | **Concurrent requests** | +++10000+++ |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image178.png)

1. On the **Create and deploy new revision** page, click on **Save a new revision**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image179.png)

1. Ensure that your new scale rule is displayed.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image180.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image181.png)


## Exercise 4: Configure continuous integration by using Azure Pipelines

In this exercise, you configure Pipeline1 to deploy the container image from your container registry to your container app using the self-hosted agent pool. The following Azure resources must be available in your resource group **@lab.CloudResourceGroup(ResourceGroup1).Name**:
- A Container Registry instance that contains one image
- A Virtual Network with subnets
- A Service Bus namespace
- A Managed Identity
- A Private endpoint
- A Container App and a Container Apps Environment


You've been asked to configure a continuous integration environment for Container Apps that meets the following requirements:
- You need an Azure Container Apps deployment task in your Azure DevOps
  environment.

- Pipeline1 must deploy a container image from your container registry
  to your container app using a self-hosted agent pool.

- You must ensure that the pipeline successfully deploys the image at
  least once.


### Task 1: Configure Pipeline1 to use the self-hosted agent pool

1. Open a browser window, navigate to +++https://dev.azure.com+++, and then open your Azure DevOps organization.

1. Select **Project1**, and then in the left-side menu, select **Pipelines**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image182.png)

1. Select **Pipeline1**, and then click on **Edit**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image183.png)

1. To use the self-hosted agent pool, update the **azure-pipelines.yml** file as shown in the following example.

    ```yaml
    trigger:
    - main

    pool:
      name: default

    steps:
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image184.png)

    >[!Note] The **pool** section specifies the agent pool to use for the pipeline. The **name** property is **default**, which is the pool you configured with the self-hosted agent.

1. Under **Validate and save**, select **Save without validating**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image185.png)

1. Enter a commit message, and then click on **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image186.png)


### Task 2: Configure Pipeline1 with an Azure Container Apps deployment task

1. Ensure that you have **Pipeline1** open for editing.

1. On the right side under **Tasks**, in the **Search tasks** field, enter +++azure container+++

1. In the filtered list of tasks, select **Azure Container Apps Deploy**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image187.png)

1. Under **Azure Resource Manager connection**, select the subscription you're using, and then click on **Authorize**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image188.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image189.png)

1. In the Azure portal tab, open your Container App resource, and then open the **Containers** page. Note down the following three values exactly as shown:

    - **Registry**: for example **acraz@lab.LabInstance.Id.azurecr.io**
    - **Image**: **aspnetcorecontainer**
    - **Image tag**: **latest**

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image190.png)


1. Go back to your Azure DevOps tab, where the task panel is still open.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image191.png)

1. Leave **Application source path** empty (you're deploying an already-built image, not building from source code).

1. Leave the **Azure Container Registry name**, **username** and **password** fields empty.

1. Scroll down inside the task panel and configure the following fields:

    | **Docker Image to Deploy** | *{Registry}*/*{Image}*:*{Image tag}*, for example **acraz@lab.LabInstance.Id.azurecr.io/aspnetcorecontainer:latest** |
    |----|----|
    | **Azure Container App name** | +++aca-az@lab.LabInstance.Id+++ |
    | **Azure Resource group name** | @lab.CloudResourceGroup(ResourceGroup1).Name |

    >[!Note] If you need to verify the resource group name, you can find it on the **Overview** page of your Container App resource.

1. On the **Azure Container Apps Deploy** page, click on **Add**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image192.png)

1. The YAML file for your pipeline should now include the **AzureContainerApps** task as follows:

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
        containerAppName: 'aca-az@lab.LabInstance.Id'
        resourceGroup: '@lab.CloudResourceGroup(ResourceGroup1).Name'
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image193.png)

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
        imageToDeploy: 'acraz@lab.LabInstance.Id.azurecr.io/aspnetcorecontainer:latest'
        containerAppName: 'aca-az@lab.LabInstance.Id'
        resourceGroup: '@lab.CloudResourceGroup(ResourceGroup1).Name'
    ```

1. Click on **Validate and save**, and then click on **Save** again to commit directly to the main branch.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image194.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image195.png)

    >[!Note] The contents of the YAML file must be formatted correctly, including indentation. If you encounter an error, review the YAML file and correct any indentation issues.

1. Navigate back to the main page of your pipeline.


### Task 3: Run the Pipeline1 deployment task

1. Ensure that you have **Pipeline1** open in Azure DevOps.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image196.png)

1. On the **Runs** tab of the Pipeline1 page, click on **Run pipeline**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image197.png)

1. A **Run pipeline** page opens to display the associated job. Click on **Run**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image198.png)

1. The **Jobs** section displays the job status, which progresses from **Queued** to **Waiting**. It can take a couple of minutes for the status to change.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image199.png)

1. If a **Permission needed** message is displayed ("This pipeline needs permission to access 2 resources before this run can continue"), click on **View** and then provide the required permissions.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image200.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image201.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image202.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image203.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image204.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image205.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image206.png)

1. Monitor the status of the run and verify that the run is successful.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image207.png)


### Task 4: Verify the pipeline deployment

1. Ensure that you have **Project1** open in Azure DevOps.

1. On the left-side menu, select **Pipelines**, and then select **Pipeline1**. The **Runs** tab displays individual runs that can be selected to review details.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image208.png)

1. Open your Azure portal, and then open your Container App.

1. On the left-side menu, select **Activity log**.

1. Verify that a **Create or Update Container App** operation succeeded as a result of running your pipeline. Notice that the **Event initiated by** column shows your **Project1** as the source.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image209.png)


## Exercise 5: Manage revisions in Azure Container Apps

In this exercise, you deploy a new revision of your container app and configure traffic splitting between two labeled revisions.

You've been asked to configure traffic splitting for your Container Apps to meet the following requirements:
- You need to create a new revision of the container app that uses a
  suffix of **v2**.

- You must ensure that 25 percent of requests to your app are directed
  to the v2 revision.

- You must label the revisions **current** and **updated** and ensure
  that requests to the **updated** revision are directed to the v2 revision.


### Task 1: Set revision management to multiple

1. In the Azure portal, open your container app (**aca-az@lab.LabInstance.Id**).

1. Under **Application** in the left menu, select **Revisions and replicas**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image210.png)

1. In the top toolbar, click on **Deployment mode**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image211.png)

1. In the panel that opens on the right, select **Multiple revisions**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image212.png)

1. Click on **Go to ingress** (or **Confirm**, depending on what the panel shows) to save the change.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image213.png)

1. On the **Ingress** page, select the **Ingress** checkbox to enable ingress for the Container App.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image214.png)

1. Select **Accepting traffic from anywhere** for **Ingress traffic**, keep **HTTP** as the **Ingress type**, and then click on **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image215.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image216.png)


### Task 2: Create a new revision with a v2 suffix

1. Ensure that you have the **Revisions and replicas** page of your container app open.

1. At the top of the page, click on **+ Create new revision**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image217.png)

1. On the **Create and deploy new revision** page, enter +++v2+++ in **Name / suffix**, and under **Container image**, select your container image (for example **aca-az@lab.LabInstance.Id**).

1. Click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image218.png)

1. Wait for the deployment to be completed.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image219.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image220.png)


### Task 3: Configure labels on the revisions

1. On the left-side menu, under **Settings**, select **Ingress**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image221.png)

1. If ingress isn't enabled, select **Enabled**.

1. On the **Ingress** page, specify the following information:

    | **Ingress traffic** | **Accepting traffic from anywhere** |
    |----|----|
    | **Ingress type** | **HTTP** |
    | **Client certificate mode** | **Ignore** |
    | **Transport** | **Auto** |
    | **Insecure connections** | Ensure that **Allowed** is **NOT** checked |
    | **Target port** | +++5000+++ |
    | **IP Security Restrictions Mode** | **Allow all traffic** |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image222.png)

1. At the bottom of the **Ingress** page, click on **Save**, and then wait for the update to complete.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image223.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image224.png)

1. On the left-side menu, select **Revisions and replicas**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image225.png)

1. For the **v2** revision, under **Label**, enter +++updated+++

1. For the other revision, enter +++current+++

1. At the top of the page, click on **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image226.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image227.png)


### Task 4: Configure a traffic percentage on the revisions

1. Ensure that you have the **Revisions and replicas** page open.

1. For the **v2** revision, under **Traffic**, enter +++25+++ as the percentage.

1. For the other revision, under **Traffic**, enter +++75+++ as the percentage.

1. At the top of the page, click on **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image228.png)


## Summary

In this lab, you configured the Azure infrastructure needed to host a secure and scalable containerized application. You created and containerized an ASP.NET Web API, stored the image in Azure Container Registry, and deployed it to Azure Container Apps. You secured application resources using managed identities, RBAC permissions, and private endpoints. Additionally, you integrated the application with Azure Service Bus, configured autoscaling policies, and automated deployments through Azure DevOps Pipelines. Finally, you managed application revisions and implemented traffic splitting to support progressive deployment strategies. These tasks demonstrated key cloud-native deployment, security, automation, and operational management capabilities available in Azure Container Apps.

===

# Lab 1: Build and Deploy the Contoso Air AI Travel App on AKS Automatic

## Scenario

**Contoso Air** is a growing airline that wants to give its customers a modern, AI-powered travel experience. The company has built a web application that lets travelers search flights, manage bookings, and chat with an **AI travel assistant** to get personalized trip recommendations.

The development team currently spends significant time managing Kubernetes infrastructure - sizing node pools, configuring monitoring, wiring up autoscaling, and managing secrets for connecting to Azure services. These manual operations slow down releases and make it difficult for developers to focus on building features.

To simplify its platform, Contoso Air has adopted **Azure Kubernetes Service (AKS) Automatic**, a fully managed Kubernetes experience that comes with production-ready defaults for security, monitoring, and scaling. The team wants to use **AKS Automated Deployments** with **GitHub Actions** to build and ship the application, and connect it securely to **Azure OpenAI** without storing any passwords or keys.

As a Cloud/DevOps Engineer, your responsibility is to prepare the Azure environment, create an AKS Automatic cluster and a CI/CD pipeline, secure GitHub authentication with federated credentials, connect the application to Azure OpenAI using Service Connector and Workload Identity, enable monitoring with Application Insights, Container Insights and Grafana, and configure autoscaling with VPA and KEDA.

By implementing this solution, Contoso Air can ship features faster, run its AI travel assistant securely at scale, and gain full visibility into application and cluster health.

## Introduction

In this lab, you will deploy the **Contoso Air** application to an **AKS Automatic** cluster. AKS Automatic is a fully managed Kubernetes service that simplifies the deployment, management, and operations of Kubernetes clusters. You deploy a cluster with just a few steps in the Azure portal and focus on building your applications.

Throughout the lab, you will explore how AKS Automatic combines Automated Deployments, Service Connector, Workload Identity, Azure Monitor, Managed Prometheus, Grafana, Node Autoprovisioning, VPA and KEDA into a single, opinionated platform for running cloud-native and AI-enabled applications.

### Architecture

| **Component** | **Role in the lab** |
|----|----|
| GitHub fork of contoso-air | Source code and GitHub Actions workflow |
| Entra app registration workflowapp-*{number}* | Identity GitHub Actions uses to sign in to Azure |
| Azure Container Registry (ACR) | Stores the contoso-air container image |
| AKS Automatic cluster myakscluster | Runs the app in namespace dev |
| Managed identity myidentity*{xxx}* | Identity the app pod uses to call Azure OpenAI |
| Azure OpenAI myopenai*{xxx}* | LLM behind the AI travel assistant |
| Log Analytics / Azure Monitor workspace | Logs and Prometheus metrics |
| Application Insights myappinsights*{xxx}* | App performance telemetry |

### Objectives

- Prepare the Azure environment and deploy the prerequisite resources.
- Create an AKS Automatic cluster and a CI/CD pipeline with Automated
  Deployments.

- Secure GitHub Actions authentication with federated credentials.
- Integrate the application with Azure OpenAI using Service Connector
  and Workload Identity.

- Observe the application and cluster with Application Insights,
  Container Insights, Logs and Grafana.

- Scale the application with the Vertical Pod Autoscaler (VPA) and KEDA.


### Prerequisites

- **GitHub Account**: You are expected to have your own GitHub login
  credentials. If you do not have an account, create one by visiting: +++https://github.com/signup+++


## Exercise 1: Set up the lab environment

Before creating the cluster, you need to prepare the Azure environment. In this exercise, you will sign in to Azure Cloud Shell, register the required providers, deploy the prerequisite resources, and fork the sample application.

### Task 1: Prepare the Azure environment and register providers

1. Open your browser, navigate to the address bar, and type or paste the following URL: +++https://portal.azure.com/+++ then press the **Enter** button. Sign in with the following credentials.

    | **Username** | +++@lab.CloudPortalCredential(User1).Username+++ |
    |----|----|
    | **Password** | +++@lab.CloudPortalCredential(User1).AccessToken+++ |

1. In the Azure portal, select the **Cloud Shell** icon from the top navigation bar to open Azure Cloud Shell.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image1.png)

1. In the **Welcome to Azure Cloud Shell** dialog, select **Bash** to launch a Bash session.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image2.png)

1. Choose **No storage account required**, select your subscription, and then click on the **Apply** button.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image3.png)

1. In Azure Cloud Shell, run the following command. Then open the displayed URL (+++https://login.microsoftonline.com/device+++) in a browser, enter the **device code** shown in the terminal, and complete the **sign-in** process using your Azure account.

    `az login --use-device-code`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image4.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image5.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image6.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image7.png)

1. Select the Azure subscription and tenant from the list, enter +++1+++ to choose the displayed option, and then press **Enter**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image8.png)

    >[!Tip] You can log into a different tenant by passing the **--tenant** flag to specify your tenant domain or tenant ID.

1. Run the following command to add the AKS preview extension.

    `az extension add --name aks-preview --upgrade`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image9.png)

1. Run the following command to register the Application Monitoring preview feature, and confirm that the feature state is displayed as **Registered** in the output.

    `az feature register --namespace Microsoft.ContainerService --name AzureMonitorAppMonitoringPreview`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image10.png)

1. Run the following commands to register the resource providers.

    - `az provider register --namespace Microsoft.DevHub`
    - `az provider register --namespace Microsoft.Insights`
    - `az provider register --namespace Microsoft.PolicyInsights`
    - `az provider register --namespace Microsoft.ServiceLinker`
    - `az provider register --namespace Microsoft.ContainerService`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image11.png)


### Task 2: Set the resource group

In this task, you set environment variables for the resource group name and location. The resource group +++ResourceGroup1+++ has already been created for you.

1. Run the following commands to set the resource group name and location.

    ```bash
    export RG_NAME=ResourceGroup1
    export LOCATION=$(az group show --name ${RG_NAME} --query location -o tsv)
    echo "RG_NAME=$RG_NAME LOCATION=$LOCATION"
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image13.png)


### Task 3: Get your user object ID

1. The template gives your account access to Key Vault and Azure OpenAI, so it needs your user object ID. If this value is empty, the deployment fails with **InvalidPrincipalId**.

    ```bash
    export USER_ID=$(az ad signed-in-user show --query id -o tsv)
    echo "USER_ID=$USER_ID"
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image14.png)

1. If nothing is printed, use the following instead (it reads the ID from your sign-in token).

    ```bash
    export USER_ID=$(az account get-access-token --query accessToken -o tsv | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin)['oid'])")
    echo "USER_ID=$USER_ID"
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image15.png)

1. Save the environment variables to a file, and verify that they are available for the next commands.

    ```bash
    echo "export LOCATION=$LOCATION RG_NAME=$RG_NAME USER_ID=$USER_ID" > ~/aks-lab.env
    cat ~/aks-lab.env
    ```

    >[!Note] If Cloud Shell times out, run +++source ~/aks-lab.env+++ to restore the variables.


### Task 4: Deploy the resources

To keep the focus on AKS-specific features, this lab requires several Azure resources to be pre-provisioned, including:
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


1. Run the following command to save your user object ID to a variable.

    `export USER_ID=$(az ad signed-in-user show --query id -o tsv)`

1. Run the following command to deploy the ARM template into the resource group.

    ```bash
    az deployment group create \
      --resource-group ${RG_NAME} \
      --name ${RG_NAME}-deployment \
      --template-uri https://raw.githubusercontent.com/azure-samples/aks-labs/refs/heads/main/docs/getting-started/aks-automatic/assets/main.json \
      --parameters userObjectId=${USER_ID}
    ```

1. If the deployment fails on the Application Insights resource (API version error), download the template, patch the API version, and deploy the local copy instead.

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

1. Wait until the resources are deployed. This can take around **15 minutes**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image16.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image17.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image18.png)

1. In the Azure portal (+++https://portal.azure.com/+++), open **ResourceGroup1** and verify that all required resources have been deployed successfully, including Application Insights, Managed Identity, Key Vault, Log Analytics workspace, Azure OpenAI, Azure Monitor workspace, and Container Registry.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image19.png)

1. Run the following commands to look up the container registry and monitoring workspaces created by the template.

    ```bash
    export ACR=$(az deployment group show -g ${RG_NAME} -n ${RG_NAME}-deployment --query properties.outputs.containerRegistryUrl.value -o tsv | cut -d. -f1)
    export LOGS=$(az deployment group show -g ${RG_NAME} -n ${RG_NAME}-deployment --query properties.outputs.logWorkspaceId.value -o tsv)
    export PROM=$(az deployment group show -g ${RG_NAME} -n ${RG_NAME}-deployment --query properties.outputs.metricsWorkspaceId.value -o tsv)
    echo "ACR=$ACR"; echo "LOGS=$LOGS"; echo "PROM=$PROM"
    ```

    All three values must be displayed. Note the **ACR** name (for example, **myregistry*{xxx}***), as you select it later in the lab.

1. Run the following command to create the AKS Automatic cluster **myakscluster** and connect it to your container registry, Log Analytics workspace, and Azure Monitor workspace. This can take around **10-15 minutes**.

    ```bash
    az aks create -g ${RG_NAME} -n myakscluster \
      --location eastus2 \
      --sku automatic \
      --attach-acr ${ACR} \
      --workspace-resource-id ${LOGS} \
      --enable-azure-monitor-metrics --azure-monitor-workspace-resource-id ${PROM}
    ```

    >[!Note] If you see a capacity error such as **AKSCapacityHeavyUsage**, run `az aks delete -g ${RG_NAME} -n myakscluster --yes` and then run the command again with +++--location westus3+++.

1. Run the following command to give your account access to the Kubernetes resources in the cluster.

    ```bash
    az role assignment create \
      --assignee $(az ad signed-in-user show --query id -o tsv) \
      --role "Azure Kubernetes Service RBAC Cluster Admin" \
      --scope $(az aks show -g ${RG_NAME} -n myakscluster --query id -o tsv)
    ```

    >[!Note] Keep your terminal open as you will need it to run commands throughout the lab.


### Task 5: Fork and clone the sample app

1. In the Bash shell, run the following command, then follow the instructions printed in the terminal to complete the login process.

    `gh auth login`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image20.png)

1. Choose **HTTPS** as the preferred protocol for Git operations, enter +++Y+++ to authenticate with your GitHub credentials, and then press **Enter**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image21.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image22.png)

1. Select **Login with a web browser**, and then press **Enter**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image23.png)

1. Copy the one-time device code, press **Enter** to open +++https://github.com/login/device+++ in your browser, enter the code, and complete the GitHub authentication process.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image24.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image25.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image26.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image27.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image28.png)

1. Run the following command to fork the **contoso-air** repository to your GitHub account, and verify that the fork is created under your GitHub repositories.

    `gh repo fork Azure-Samples/contoso-air --clone --default-branch-only`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image29.png)

1. Change into the **contoso-air** directory.

    `cd contoso-air`

1. Set the default repository to your forked repository.

    `gh repo set-default`

1. When prompted, select **your fork** of the repository and press **Enter**. Do not select the original **Azure-Samples/contoso-air** repository.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image30.png)

1. Configure your Git identity by running the following commands, replacing the placeholder values with your GitHub username and email address.

    - `git config user.name "*{your GitHub user}*"`
    - `git config user.email "*{your email}*"`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image31.png)


1. Save your **GitHub username** and **email address**, as they will be required in later tasks.


### Task 6: Record your GitHub IDs

1. GitHub identifies your repository using numeric IDs. Run the following command (replace ***{your-user}*** with your GitHub username), then note and save the **Owner ID** and **Repo ID** values, as they will be required later in this lab.

    `gh api repos/$(gh api user --jq .login)/contoso-air --jq '"Owner ID: \(.owner.id)  Repo ID: \(.id)"'`

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image32.png)


## Exercise 2: Create the cluster and pipeline

In this exercise, you use AKS Automated Deployments to create a GitHub Actions pipeline that builds and deploys the Contoso Air application to your AKS Automatic cluster.

### Task 1: Configure Automated Deployments for Azure Kubernetes Service (AKS)

1. Open your browser and sign in to the Azure portal at +++https://portal.azure.com/+++.

1. Type +++Kubernetes services+++ in the search box at the top of the page, select **Kubernetes services** from the search results, and then select **myakscluster**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image33.png)

1. In the cluster menu, select **Automated deployments**, and then click on the **+ Create** button.

    >[!Alert] Start from **myakscluster**. If you start from **Kubernetes services > + Create > Deploy application**, the wizard reports that the registry must be connected to a cluster.

1. In the **Basics** tab, enter the following details to configure the application deployment settings.

    | **Subscription** | Select your Azure subscription |
    |----|----|
    | **Resource group** | Select **ResourceGroup1** |
    | **Workflow name** | +++contoso-air+++ |
    | **Repository location** | **GitHub** |

1. Click on **Authorize access** to grant Azure permission to create the deployment workflow in your GitHub repository.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image36.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image37.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image38.png)

1. Enter the following repository details, and then click on the **Next** button.

    | **Repository source** | **My repositories** |
    |----|----|
    | **Repository** | **contoso-air** |
    | **Branch** | **main** |

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image39.png)

1. In the **Application** tab, complete the **Image** section with the following details:

    - **Container configuration**: Select **Existing Dockerfile**
    - **Dockerfile**: Select the **Select** link, browse to the
    **./src/web** directory, select the Dockerfile, then click on the **Select** button

    - **Dockerfile build context**: Enter +++./src/web+++
    - **Azure Container Registry**: Select **myregistry*{xxx}***, the registry you noted in *Exercise 1 \> Task 4*
    - **Azure Container Registry image**: Select the **Create new** link,
    enter +++contoso-air+++ and click on **Ok**

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image40.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image41.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image42.png)


1. In the **Deployment configuration** section, enter the following details:

    - **Deployment options**: Select **Generate application deployment
    files**

    - **Save files in repository**: Select the **Select** link, select the
    checkbox next to the **Root** folder, then click on **Select**

    - **Application port**: Enter +++3000+++
    - Click on **Next**

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image43.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image44.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image45.png)


1. In the **Cluster configuration** section, select the existing cluster **myakscluster**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image46.png)

1. For **Namespace**, select **Create new**, enter +++dev+++ as the namespace name, click on **OK**, and verify that the **dev** namespace is selected.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image47.png)

1. If monitoring settings are displayed, verify that **Container Logs**, **Prometheus metrics**, and the associated **Log Analytics Workspace** and **Azure Monitor Workspace** are configured correctly, and then click on **Next**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image48.png)

1. After the validation passes, click on the **Deploy** button.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image49.png)

    >[!Important] The deployment can take **a few minutes** to complete. Do not close the browser window or navigate away from the page until the deployment finishes.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image50.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image51.png)

1. Verify that the application deployment completed successfully. The page shows an **Approve pull request** button for the GitHub Actions workflow created for the application.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image52.png)

1. **Do not approve or merge the pull request yet.** You first secure GitHub authentication in the next task.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image53.png)


### Task 2: Secure GitHub authentication before merging changes

1. In the Azure portal (+++https://portal.azure.com/+++), type +++App registrations+++ in the search box at the top of the page, and select **App registrations** from the search results.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image54.png)

1. Open **workflowapp-*{number}*** created today, and note its name.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image55.png)

1. Under **Manage**, select **Certificates & secrets**, open the **Federated credentials** tab, and then click on **Add credential**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image56.png)

1. For **Federated credential scenario**, select **GitHub Actions deploying Azure resources**, and enter the following details.

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

1. Scroll to **Subject identifier** (read-only). It must read **repo:*{user}*@*{ownerId}*/contoso-air@*{repoId}*:ref:refs/heads/main**. Click on the **Add** button.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image57.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image58.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image59.png)


### Task 3: Review, merge, and deploy application changes

1. Back on the Automated Deployments page, click on **Approve pull request** (or open **Pull requests** in your GitHub fork).

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image53.png)

1. In the pull request, select the **Files changed** tab to review the workflow and configuration updates.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image60.png)

1. Scroll down to the **manifests/deployment.yaml** file to review the generated Kubernetes deployment manifest. This manifest includes the **SYS_PTRACE** capability, which should be removed for security hardening.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image61.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image62.png)

1. Hover over the line, select the **+** icon, choose **Add a suggestion**, delete the suggested line so that the suggestion is empty, and then click on **Comment**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image63.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image64.png)

1. Click on **Commit suggestions**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image65.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image66.png)

1. Select the **Conversation** tab, review the comments and changes, and then click on **Merge pull request**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image67.png)

1. Click on the **Confirm merge** button.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image68.png)

1. With the pull request merged, the changes are automatically deployed to your AKS cluster. Select the **Actions** tab in your GitHub repository to view the deployment logs.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image69.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image70.png)

1. In the **Actions** tab, select the running **Automated Deployments** workflow to view the logs.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image71.png)

1. After 5-10 minutes, the workflow completes and you will see green check marks next to the **buildImage** and **deploy** jobs. This means that the application has been successfully deployed to your AKS cluster.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image72.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image73.png)


### Task 4: Verify the application deployment in the dev namespace

1. In the **Azure portal**, navigate to **myakscluster**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image74.png)

1. Select **Kubernetes resources** \> **Services and ingresses** to view the services exposed by the cluster, and verify that the application is running in the **dev** namespace.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image75.png)

1. Locate the **contoso-air** service in the **dev** namespace, and select its **External IP** address to open the deployed application.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image76.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image77.png)

1. Test the application's chat functionality by clicking on the **Ask the AI travel assistant** button.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image78.png)

1. Attempt to interact with the AI assistant and you'll find that the **chat provider is not detected**. You fix this in the next exercise.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image79.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image80.png)


## Exercise 3: Integrating apps with Azure services

AKS Service Connector streamlines connecting applications to Azure resources like Azure OpenAI by automating the configuration of Workload Identity. It assigns identities to pods, enabling them to authenticate with Microsoft Entra ID and access Azure services securely without passwords.

### Task 1: Application configuration

A ConfigMap is a Kubernetes resource that stores non-confidential configuration data as key-value pairs. Applications running in pods can read these values as environment variables, making it easy to change application behavior without rebuilding the container image.

1. In the **Azure portal**, navigate to **myakscluster**, select **Kubernetes resources** \> **Configuration**, and choose **dev** from the **Filter by namespace** list.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image81.png)

1. Select the **contoso-air-config** config map.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image82.png)

1. Select the **YAML** tab, add the following configuration to the end of the file, and then click on **Review + save**.

    ```yaml
    data:
      AZURE_OPENAI_API_VERSION: 2024-12-01-preview
      AZURE_OPENAI_DEPLOYMENT: gpt-5.4-mini
      CHAT_PROVIDER: azure
      LOG_CHAT: 'true'
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image83.png)

1. Review the changes, verify that the Azure OpenAI settings are correct, select **Confirm manifest changes**, and then click on **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image84.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/bldaiappagntsdepth/refs/heads/main/Cloudslice/Labguides/Lab%20%2002/media/image85.png)

    **Info:** This configmap was created by the Automated Deployments workflow earlier. The new settings instruct the application to use Azure OpenAI as the chat provider and specify the model deployment name. The contoso-air application is already configured to read these settings and inject them into the application environment.
