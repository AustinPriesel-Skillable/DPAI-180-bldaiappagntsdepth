# Lab 4: Build Intelligent Knowledge Agents with MCP, Foundry IQ and Azure AI Search

**Scenario**

**Zava** is a home-improvement retailer that sells paints, primers,
tools, and repair kits, and offers a color-matching service. Its
customer-support and in-store teams answer the same kinds of questions
every day: *"Which Zava paint is best for a bathroom?"*, *"Does Zava
have exterior paint, and how much does it cost?"*, *"What products do I
need to repaint a room, and what will it cost?"*

The answers are spread across different places. Product details sit in
a product catalog, installation and care instructions are in product
manuals stored as documents, and general painting advice and industry
guidance are on the public web. A simple chatbot that searches one
source by keyword either misses information or makes up answers without
sources, which support teams cannot trust.

Zava's AI team wants to build **knowledge-grounded agents** that answer
every question from approved sources and show where each fact came
from. They choose **Foundry IQ** so that data connections and retrieval
live in a reusable **knowledge base**, while the agent focuses on the
conversation. Before connecting their own data, the team first validates
the approach on a public dataset, NASA's *Earth at Night* e-book, and
then builds the Zava multi-source knowledge base.

As a Zava AI engineer, you deploy the Azure resources, build knowledge
bases over indexed, blob, and web data, tune retrieval reasoning effort
for cost and quality, and connect a Foundry agent that answers with
citations and says *"I don't know"* when the knowledge base has no
answer.

**Introduction**

In this lab, you use **Foundry IQ**, the knowledge layer of **Microsoft
Foundry**, to give AI agents trusted, cited answers from your own data.
Foundry IQ is built on **Azure AI Search**. It lets you connect data as
**knowledge sources**, combine them into a **knowledge base**, and query
that knowledge base with **agentic retrieval**: an LLM plans the query,
splits it into focused subqueries, searches the right sources in
parallel, reranks the results, and synthesizes a grounded answer with
citations. You then expose the knowledge base to a **Foundry Agent
Service** agent through its **MCP (Model Context Protocol)** endpoint.

You work through three hands-on Jupyter notebooks (cookbooks) in Visual
Studio Code:

1.  **Unlocking Knowledge for Agents** - create a search index,
    knowledge source, and knowledge base, query it, and connect it to a
    Foundry agent.

2.  **Building the Data Pipeline with Knowledge Sources** - combine an
    indexed source, an Azure Blob Storage source, and a web source in
    one knowledge base.

3.  **Querying Multi-Source AI Knowledge Bases** - compare minimal, low,
    and medium retrieval reasoning effort and customize answer
    synthesis.

**Architecture**

| **Component** | **Role in the lab** |
|----|----|
| Azure AI Search iqs1-search-\<suffix\> | Hosts the search indexes, knowledge sources, and knowledge bases (Foundry IQ) |
| Azure OpenAI iqs1-openai-\<suffix\> | text-embedding-3-large (vectors) and gpt-5.4 (query planning and answer synthesis) |
| Microsoft Foundry iqs1-ai-\<suffix\> / project iqs1-project | Foundry Agent Service that hosts the earth-at-night-agent |
| Storage account iqs1stor\<suffix\> | Blob container product-manuals for the Blob knowledge source |
| Deployment script iqs1-seed-data | Seeds sample data during deployment |
| Knowledge base MCP endpoint | Lets the agent (or any MCP client) call the knowledge base as a tool |
| Visual Studio Code + Jupyter | Runs the three Foundry IQ cookbooks on the lab VM |

**Objectives**:

- Deploy the Foundry IQ lab resources (Azure AI Search, Azure OpenAI,
  Microsoft Foundry, and Azure Storage) using an ARM template.

- Create a search index, knowledge sources, and a knowledge base with
  Foundry IQ.

- Query a knowledge base with agentic retrieval and inspect the query
  plan, activity, and references.

- Connect a knowledge base to a Foundry Agent Service agent using an MCP
  tool.

- Combine indexed, Azure Blob Storage, and web knowledge sources in one
  knowledge base.

- Compare minimal, low, and medium retrieval reasoning effort and
  configure answer synthesis.

**Prerequisites**

- Azure subscription credentials provided with the lab (Owner access on
  the **Cloud-Native** resource group).

- Lab VM with Visual Studio Code, Python 3.11+, the Jupyter extension,
  and the Azure CLI. The lab files are in **C:\Labfiles\Foundry-IQ**.

**Important:** The resource names in the screenshots (for example,
**iqs1-search-bdruy63orelkw** and **iqs1-ai-bdruy63orelkw**) are
examples. The suffix is generated for each deployment. Always use the
names, endpoints, and keys from **your own** deployment.

## Exercise 1: Deploy the Foundry IQ resources

In this exercise, you deploy the Azure AI Search, Azure OpenAI,
Microsoft Foundry and Azure Storage resources with an ARM template, and
collect the values you need to configure the cookbooks.

### Task 1: Get your user object ID

The deployment template grants your user account access to Azure AI
Search, Azure OpenAI, and Foundry. It needs your Microsoft Entra **user
object ID**.

1.  Open your browser, navigate to the address bar, and type or paste
    the following URL: +++https://portal.azure.com/+++ then press the
    **Enter** button. Sign in with the following credentials.

| **Username** | **+++@lab.CloudPortalCredential(User1).Username+++** |
|----|----|
| **Password** | **+++@lab.CloudPortalCredential(User1).AccessToken+++** |

2.  In the Azure portal, select the **Cloud Shell** icon from the top
    navigation bar.

![](./media/image1.png)

3.  In the **Welcome to Azure Cloud Shell** dialog, select **Bash**.

![](./media/image2.png)

4.  Choose **No storage account required**, select your subscription,
    and then click on **Apply**.

![](./media/image3.png)

**Tip:** Cloud Shell is already signed in with your portal account.
Steps 5-9 (**az login**) are only required if the command in step 10
returns an error.

5.  In Cloud Shell, run the following command. Then open
    +++https://login.microsoft.com/device+++ in a new browser tab, and
    note the **code** shown in the terminal.

> +++az login+++

![](./media/image4.png)

6.  Enter the code and click on **Next**.

![](./media/image5.png)

7.  Select your lab account.

![](./media/image6.png)

8.  Click on **Continue** to sign in to the Microsoft Azure CLI.

![](./media/image7.png)

9.  Back in Cloud Shell, type **1** to select the subscription and
    tenant, and then press **Enter**.

![](./media/image8.png)

10. Run the following command to get your user object ID.

> +++az ad signed-in-user show --query id -o tsv+++

![](./media/image9.png)

11. Copy the GUID that is returned (for example,
    **c92f6c6d-ad65-4768-80fc-3ce999842353**) and save it in Notepad.
    You need it in the next task and in Exercise 2.

### Task 2: Deploy the resources with the ARM template

1.  Open a new browser tab and go to
    +++https://aka.ms/iq-series/deploytoazure+++. The **Custom
    deployment** page opens. Click on **Edit template**.

![](./media/image10.png)

2.  In the template editor, under **parameters**, verify that the
    **location** parameter has **"defaultValue": "eastus"**.

![](./media/image11.png)

**Note:** The location must be a region that supports **agentic
retrieval** in Azure AI Search. The resources are created in this
location, even if you select a different **Region** on the Basics tab.

3.  Scroll to the **chatModelName** and **chatModelVersion** parameters
    and update them as follows:

| **chatModelName** | "defaultValue": +++gpt-5.4+++ |
|----|----|
| **chatModelVersion** | "defaultValue": +++2026-03-05+++ |

![](./media/image12.png)

4.  Scroll down to **variables** \> **names** and set
    **"chatDeployment"** to +++gpt-5.4+++.

![](./media/image13.png)

5.  Click on **Save**.

![](./media/image14.png)

**Important:** The chat model name and version must be available for
your subscription and region. If the deployment fails with a *model not
supported* or *quota* error, use a chat model and version that is
available in your region, and use the same deployment name later in the
**.env** files.

6.  On the **Basics** tab, enter the following details and click on
    **Review + create**.

| **Subscription** | Keep the default subscription |
|----|----|
| **Resource group** | **Cloud-Native** |
| **Region** | Keep the default |
| **User Object Id** | Paste the object ID you copied in Task 1 |
| **Resource Prefix** | Keep **iqs** or enter a short prefix such as +++iqs1+++ |

![](./media/image15.png)

7.  Click on **Create**.

![](./media/image16.png)

8.  Wait for the deployment to complete. This takes about **10-15
    minutes**, because a deployment script waits for role assignments to
    propagate and then seeds the sample data.

![](./media/image17.png)

9.  When the deployment is complete, click on **Go to resource group**.

![](./media/image18.png)

10. Verify that the resource group contains the Foundry resource, the
    Foundry project, the Azure OpenAI resource, the Search service
    (Foundry IQ), a managed identity, a deployment script, and a storage
    account.

![](./media/image19.png)

### Task 3: Copy the deployment outputs

1.  In the **Cloud-Native** resource group, expand **Settings** and
    select **Deployments**.

![](./media/image20.png)

2.  Select the **CustomDeployment-\<timestamp\>** deployment.

![](./media/image21.png)

3.  Select **Outputs**.

![](./media/image22.png)

4.  Copy the following values to Notepad. You need them for the **.env**
    files in the next exercises.

| **Output** | **Used for** |
|----|----|
| searchEndpoint | SEARCH_ENDPOINT |
| openAiEndpoint | AOAI_ENDPOINT |
| foundryProjectEndpoint | FOUNDRY_PROJECT_ENDPOINT |
| searchConnectionName | AZURE_AI_SEARCH_CONNECTION_NAME |
| blobConnectionString | BLOB_CONNECTION_STRING |
| blobContainerName | BLOB_CONTAINER_NAME |
| embeddingDeploymentName / chatDeploymentName | Model deployment names |

![](./media/image23.png)

![](./media/image24.png)

**Important:** The **searchApiKey** and **blobConnectionString** outputs
contain secrets. Do not share them or commit them to GitHub. The
notebooks authenticate with **DefaultAzureCredential** (your az login),
so the search API key is not needed.

### Task 4: Copy the Foundry project resource ID

1.  In a new tab, go to +++https://ai.azure.com+++ and sign in. Under
    **All resources**, select **iqs1-project**.

![](./media/image25.png)

2.  The project **Home** page shows the **Project endpoint** and **Azure
    OpenAI endpoint**.

![](./media/image26.png)

3.  Select **Manage**, and under **Project (iqs1-project)** select
    **Project details**. Select the copy icon next to **Project ID** and
    save the value in Notepad. It has the format
    **/subscriptions/\<sub-id\>/resourceGroups/Cloud-Native/providers/Microsoft.CognitiveServices/accounts/\<ai-services-name\>/projects/iqs1-project**.

![](./media/image27.png)

**Note:** This value is the **FOUNDRY_PROJECT_RESOURCE_ID** in the .env
file. It is also the scope for the role assignment in Exercise 2.

## Exercise 2: Unlocking Knowledge for Agents

In this exercise, you use the first cookbook to create a search index,
knowledge source, and knowledge base over NASA's *Earth at Night* data,
query it with agentic retrieval, and connect it to a Foundry agent
through the MCP endpoint.

### Task 1: Open the lab folder and configure the .env file

1.  Open **Visual Studio Code**. Select **File** \> **Open Folder**.

![](./media/image28.png)

2.  Browse to **C:\Labfiles**, select the **Foundry-IQ** folder, and
    click on **Select folder**.

![](./media/image29.png)

**Note:** If prompted with **Do you trust the authors of the files in
this folder?**, select **Yes, I trust the authors**.

3.  In the **Explorer**, expand
    **1-Foundry-IQ-Unlocking-Knowledge-for-Agents**. Right-click the
    **cookbook** folder and select **New File**. Name the file
    +++.env+++.

![](./media/image30.png)

4.  Paste the following template into the **.env** file.

```text
SEARCH_ENDPOINT=https://<your-search-service>.search.windows.net
AOAI_ENDPOINT=https://<your-openai-resource>.openai.azure.com/
AOAI_EMBEDDING_MODEL=text-embedding-3-large
AOAI_EMBEDDING_DEPLOYMENT=text-embedding-3-large
AOAI_GPT_MODEL=gpt-5.4
AOAI_GPT_DEPLOYMENT=gpt-5.4
FOUNDRY_PROJECT_ENDPOINT=https://<your-ai-services>.services.ai.azure.com/api/projects/<your-project>
FOUNDRY_MODEL_DEPLOYMENT_NAME=gpt-5.4
AZURE_AI_SEARCH_CONNECTION_NAME=<your-search-connection-name>
FOUNDRY_PROJECT_RESOURCE_ID=/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<ai-services>/projects/<project>
```

![](./media/image31.png)

5.  Replace the placeholders with the values you saved in Exercise 1,
    and then save the file (**Ctrl+S**).

| **Variable** | **Value** |
|----|----|
| SEARCH_ENDPOINT | searchEndpoint output |
| AOAI_ENDPOINT | openAiEndpoint output |
| FOUNDRY_PROJECT_ENDPOINT | foundryProjectEndpoint output |
| AZURE_AI_SEARCH_CONNECTION_NAME | searchConnectionName output (**iq-series-search-connection**) |
| FOUNDRY_PROJECT_RESOURCE_ID | Project ID from the Foundry portal |

![](./media/image32.png)

**Important:** The model values must match your **deployment names**.
The template in this lab deploys **text-embedding-3-large** and
**gpt-5.4**. If you keep the notebook default **gpt-4o-mini**, the
knowledge base and agent calls fail with a *DeploymentNotFound* error.

### Task 2: Create a Python virtual environment

1.  In Visual Studio Code, select the **More Actions (...)** menu,
    select **Terminal**, and then choose **New Terminal**.

![](./media/image33.png)

2.  Run the following commands, one at a time, to create and activate a
    virtual environment, install the required packages, and register a
    Jupyter kernel.

> +++cd C:\Labfiles\Foundry-IQ\1-Foundry-IQ-Unlocking-Knowledge-for-Agents\cookbook+++
>
> +++python -m venv .venv+++
>
> +++.\.venv\Scripts\Activate.ps1+++
>
> +++pip install --upgrade pip+++
>
> +++pip install azure-search-documents==12.1.0b1 azure-ai-projects azure-identity python-dotenv ipykernel+++
>
> +++python -m ipykernel install --user --name foundry-iq --display-name "Foundry-IQ (.venv)"+++

![](./media/image34.png)

![](./media/image35.png)

![](./media/image36.png)

**Note:** Foundry IQ knowledge bases need the preview SDK
**azure-search-documents==12.1.0b1**. A newer or older version may not
include the **KnowledgeBase** classes used in the notebooks.

**Tip:** If activation fails with *running scripts is disabled on this
system*, run
+++Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned+++
and then activate again.

### Task 3: Sign in to Azure from the terminal

The notebooks authenticate with **DefaultAzureCredential**, which uses
your Azure CLI sign-in. No API keys are stored in code.

1.  In the same terminal, run the following command.

> +++az login+++

2.  In the **Sign in** dialog, select **Work or school account**, and
    then click on **Continue**.

![](./media/image37.png)

3.  Enter your lab username and click on **Next**.

![](./media/image38.png)

4.  Enter the password and click on **Sign in**.

![](./media/image39.png)

5.  In the terminal, type **1** to select the lab subscription and press
    **Enter**.

![](./media/image40.png)

### Task 4: Open the notebook and run the setup cells

1.  In the **Explorer**, open
    **1-Foundry-IQ-Unlocking-Knowledge-for-Agents \> cookbook \>
    foundry-iq-cookbook.ipynb**. Review the introduction: in this
    cookbook you create a knowledge source, create a knowledge base,
    query it, and connect Foundry IQ to the Foundry Agent Service.

![](./media/image41.png)

2.  Scroll to **Setup** and select the **Run** icon next to the **%pip
    install** cell.

![](./media/image42.png)

3.  When prompted to select a kernel, select **Python Environments**.

![](./media/image43.png)

4.  Select the Python environment to use (the screenshots use **Python
    3.14.3**).

![](./media/image44.png)

**Note:** You can also select the **Foundry-IQ (.venv)** kernel that you
registered in Task 2. Use the same kernel for all cells of a notebook.

5.  Wait until the install cell shows a green check mark.

![](./media/image45.png)

6.  Run the next cell, which loads the **.env** file and creates the
    credential.

![](./media/image46.png)

7.  Verify that the output shows your **Search endpoint** and **Models:
    embedding=text-embedding-3-large, chat=gpt-5.4**.

![](./media/image47.png)

**Note:** If the output still shows **gpt-4o-mini**, the .env file was
not saved or is not in the **cookbook** folder. Save the file and run
the cell again.

### Task 5: Create a search index and upload documents

1.  Under **Step 1 - Create a Search Index and Upload Documents**, run
    the cell that defines the index. The index has text, vector, and
    semantic search capabilities. A **semantic_search** configuration
    with a **default_configuration_name** is required for agentic
    retrieval.

![](./media/image48.png)

2.  Verify the output **Index 'earth-at-night' ready**.

![](./media/image49.png)

3.  Run the next cell to download NASA's *Earth at Night* sample data
    and upload it to the index. Verify the output **194 documents
    uploaded to 'earth-at-night'**.

![](./media/image50.png)

### Task 6: Create a knowledge source

1.  Under **Step 2 - Create a Knowledge Source**, run the cell. A
    knowledge source tells Foundry IQ *where* to find data. It points at
    the search index and declares which fields carry source metadata for
    citations.

2.  Verify the output **Knowledge source 'earth-knowledge-source'
    ready**.

![](./media/image51.png)

### Task 7: Create a knowledge base

1.  Under **Step 3 - Create a Knowledge Base**, run the cell. The
    knowledge base wraps the knowledge source with an LLM configuration
    (your gpt-5.4 deployment) and uses **ANSWER_SYNTHESIS** output mode,
    so it returns a natural-language answer with inline citations
    instead of raw chunks.

![](./media/image52.png)

2.  Verify the output **Knowledge base 'earth-knowledge-base' ready**.

![](./media/image53.png)

### Task 8: Query the knowledge base

1.  Under **Step 4 - Query the Knowledge Base**, review how agentic
    retrieval works: the knowledge base decomposes the query into
    focused subqueries, runs them in parallel, reranks the results with
    the semantic ranker, and synthesizes a grounded, cited answer.

2.  Run the cell. It sends the question *"What causes city lights to
    appear brighter from space during the holidays?"* with **low**
    retrieval reasoning effort.

![](./media/image54.png)

3.  Verify the output **Retrieval complete**.

![](./media/image55.png)

4.  Under **Inspect the answer**, run the cell. The response has three
    parts: the **Response** (synthesized answer), the **Activity** (query
    plan and subqueries), and the **References** (source documents).

![](./media/image56.png)

5.  Review the answer and the **Query plan**. The **modelQueryPlanning**
    step shows the gpt-5.4 tokens used to plan the query, and the
    **searchIndex** step shows the subquery that was sent to
    **earth-knowledge-source**.

![](./media/image57.png)

### Task 9: Use the knowledge base with Foundry Agent Service

In this task, you create an agent that uses the knowledge base's **MCP
endpoint** as a tool. The agent handles tool calls, citation formatting,
and multi-turn conversation.

1.  Under **Step 5 - Use with Foundry Agent Service**, run the first
    cell to build the knowledge base MCP endpoint URL. Verify that the
    output shows **MCP endpoint:
    https://\<search-service\>.search.windows.net/knowledgebases/earth-knowledge-base/mcp?api-version=...**

![](./media/image58.png)

2.  Under **Create a project connection for MCP authentication**, run
    the cell. It creates a **RemoteTool** project connection named
    **earth-kb-mcp-connection** that uses the project's managed identity
    to authenticate to the search service.

![](./media/image59.png)

3.  Verify the output **Project connection 'earth-kb-mcp-connection'
    created**.

![](./media/image60.png)

**Note:** If the cell fails with **401**, run **az login** again in the
terminal, complete MFA if prompted, and rerun the setup and connection
cells.

4.  Your account needs the **Foundry User** role on the Foundry project
    to create and run agents. In the Visual Studio Code terminal, run
    the following command. Replace **\<your-user-object-id\>** with the
    object ID from Exercise 1, Task 1, and
    **\<FOUNDRY_PROJECT_RESOURCE_ID\>** with the Project ID from
    Exercise 1, Task 4.

```powershell
az role assignment create `
  --assignee-object-id <your-user-object-id> `
  --assignee-principal-type User `
  --role "Foundry User" `
  --scope "<FOUNDRY_PROJECT_RESOURCE_ID>"
```

![](./media/image61.png)

**Important:** **Foundry User** is the new name of the **Azure AI User**
role. If your tenant still shows the old name, use +++Azure AI User+++
as the role. Role assignments can take a few minutes to take effect; if
the next cell fails with a permission error, wait 2-3 minutes and run it
again.

5.  Under **Create an agent with the MCP Knowledge Base tool**, review
    the agent instructions. The agent must always use the knowledge
    base, add citations, and respond with *"I don't know"* when the
    answer is not in the knowledge base. The **MCPTool** points at the
    MCP endpoint, allows only the **knowledge_base_retrieve** tool, and
    uses the project connection for authentication. Run the cell.

![](./media/image62.png)

6.  Verify the output **Agent 'earth-at-night-agent' created
    (version=1)**.

![](./media/image63.png)

7.  Under **Chat with the agent**, run the cell. It creates a
    conversation and asks *"Why is the Phoenix nighttime street grid so
    sharply visible from space?"*

![](./media/image64.png)

8.  Review the agent response and the explanation of what happened: the
    agent invoked the **knowledge_base_retrieve** MCP tool, the
    knowledge base ran its agentic retrieval pipeline (query planning,
    hybrid search, semantic reranking), and the agent synthesized a
    cited answer.

![](./media/image65.png)

**Note:** The same MCP endpoint can be used by any MCP-compatible
client, such as GitHub Copilot, VS Code, or your own applications.

9.  Under **Inspect the full response**, run the cell to see the raw
    response, including the **mcp_call** output with the
    **knowledge-base** server label, the model, and the citations.

![](./media/image66.png)

10. Under **Clean up the agent and connection**, run the cell to delete
    the agent version and the project connection.

![](./media/image67.png)

## Exercise 3: Building the Data Pipeline with Knowledge Sources

In this exercise, you build Zava's product knowledge base from three
types of knowledge sources: an **indexed** product catalog, **product
manuals** in Azure Blob Storage, and the public **web**.

### Task 1: Configure the .env file and select the kernel

1.  In the **Explorer**, expand
    **2-Foundry-IQ-Building-the-Data-Pipeline-with-Knowledge-Sources**,
    right-click **cookbook**, select **New File**, and name it
    +++.env+++. Add the following values and save the file.

```text
SEARCH_ENDPOINT=<searchEndpoint output>
AOAI_ENDPOINT=<openAiEndpoint output>
AOAI_EMBEDDING_MODEL=text-embedding-3-large
AOAI_EMBEDDING_DEPLOYMENT=text-embedding-3-large
AOAI_GPT_MODEL=gpt-5.4
AOAI_GPT_DEPLOYMENT=gpt-5.4
BLOB_CONNECTION_STRING=<blobConnectionString output>
BLOB_CONTAINER_NAME=product-manuals
```

![](./media/image68.png)

2.  Open **foundry-iq-cookbook.ipynb** in the same **cookbook** folder,
    and run the **%pip install** cell.

![](./media/image69.png)

3.  When prompted, select the same Python kernel you used in Exercise 2.

![](./media/image70.png)

4.  Run the next cell and verify the output shows your **Search
    endpoint** and **Models: embedding=text-embedding-3-large,
    chat=gpt-5.4**. Note the resource names used in this cookbook:
    **product-catalog**, **product-index-source**,
    **product-docs-blob-source**, **web-source**, and
    **product-knowledge-base**.

![](./media/image71.png)

### Task 2: Create the product catalog index

1.  Under **Step 2 - Create a Search Index and Upload Documents**, run
    the index cell.

![](./media/image72.png)

2.  Verify the output **Index 'product-catalog' ready**.

![](./media/image73.png)

3.  Run the next cell to upload the sample Zava product catalog (paints,
    tools, repair kits, primer, and color matching service). Verify the
    output **8 documents uploaded to 'product-catalog'**.

![](./media/image74.png)

### Task 3: Create the knowledge sources

1.  Under **Step 3 - Create an Indexed Knowledge Source**, run the cell.
    When you wrap an existing Azure AI Search index as a knowledge
    source, you reuse the index you already have with no additional
    ingestion.

![](./media/image75.png)

2.  Verify the output **Indexed Knowledge Source 'product-index-source'
    ready**.

![](./media/image76.png)

3.  Under **Step 4 - Create a Blob Storage Knowledge Source**, review
    how Foundry IQ automates ingestion for documents in Blob Storage: it
    discovers the documents, chunks them, vectorizes each chunk with
    your embedding model, enriches metadata, and keeps the index fresh
    with scheduled indexers. Run the cell.

![](./media/image77.png)

4.  Verify the output **Blob Knowledge Source 'product-docs-blob-source'
    ready**.

![](./media/image78.png)

**Note:** If **BLOB_CONNECTION_STRING** is missing from the .env file,
the cell skips the Blob knowledge source and prints a warning. Add the
value from the deployment outputs and rerun the setup cell and this
cell.

5.  Under **Step 5 - Create a Web Knowledge Source**, run the cell. A
    web knowledge source brings in public information through managed
    Bing grounding. It is a *remote* source: no data is copied, and the
    web is queried at run time. Verify the output **Web Knowledge Source
    'web-source' ready**.

![](./media/image79.png)

### Task 4: Combine the sources in a knowledge base

1.  Under **Step 6 - Combine Sources in a Knowledge Base**, run the
    cell. When an agent queries the knowledge base, Foundry IQ plans
    which sources to query, sends focused subqueries to each selected
    source in parallel, merges and reranks the results, and returns
    grounded results with citations.

![](./media/image80.png)

2.  Verify the output **Knowledge Base 'product-knowledge-base' ready
    with 3 source(s)** listing **product-index-source**,
    **product-docs-blob-source**, and **web-source**.

![](./media/image81.png)

### Task 5: Query across multiple sources

1.  Under **Step 7 - Query Across Multiple Sources**, run the cell. It
    asks *"What Zava paint is best for bathrooms and what general tips
    should I follow for bathroom painting?"*, a question that needs both
    product data and general web advice.

![](./media/image82.png)

2.  Verify the output **Retrieval complete**.

![](./media/image83.png)

3.  Run the next cell to show the **Activity log** and the references.
    Review how the query was planned and which knowledge sources (for
    example, **product-index-source** and **web-source**) contributed
    results.

![](./media/image84.png)

4.  Under **Clean Up**, run the cell to delete the knowledge base, the
    knowledge sources, and the **product-catalog** index.

![](./media/image85.png)

## Exercise 4: Querying Multi-Source AI Knowledge Bases

In this exercise, you use the third cookbook to compare **retrieval
reasoning effort** levels to balance speed, cost, and answer quality.

| **Effort** | **How it works** | **Best for** |
|----|----|----|
| Minimal | No LLM during retrieval. You pass explicit search intents, which are sent to every source. | Agents that already do their own reasoning; lowest cost and latency |
| Low | The LLM plans focused subqueries and selects the relevant sources. | Most conversational questions |
| Medium | Adds iterative retrieval: evaluates the first results and runs a second, refined pass if needed. | Complex, multi-part questions |

### Task 1: Configure the .env file and run the setup

1.  In the **Explorer**, expand
    **3-Foundry-IQ-Querying-the-Multi-Source-AI-Knowledge-Bases**,
    right-click **cookbook**, select **New File**, and name it
    +++.env+++. Add the following values and save the file.

```text
SEARCH_ENDPOINT=<searchEndpoint output>
AOAI_ENDPOINT=<openAiEndpoint output>
AOAI_EMBEDDING_MODEL=text-embedding-3-large
AOAI_EMBEDDING_DEPLOYMENT=text-embedding-3-large
AOAI_GPT_MODEL=gpt-5.4
AOAI_GPT_DEPLOYMENT=gpt-5.4
```

![](./media/image86.png)

2.  Open **foundry-iq-cookbook.ipynb** in the same **cookbook** folder,
    run the **%pip install** cell, and select the same Python kernel.

![](./media/image87.png)

3.  In the next cell, verify that the **GPT_MODEL** and
    **GPT_DEPLOYMENT** default values are **gpt-5.4**. If they show
    **gpt-4o-mini**, change them to +++gpt-5.4+++.

![](./media/image88.png)

4.  Run the cell and verify the output **Models:
    embedding=text-embedding-3-large, chat=gpt-5.4**. This cookbook uses
    the names **product-catalog-ep3**, **product-index-source-ep3**,
    **web-source-ep3**, and **product-kb-ep3**.

![](./media/image89.png)

### Task 2: Create the multi-source knowledge base

1.  Under **Step 1 - Create a Multi-Source Knowledge Base**, review the
    three core responsibilities of a knowledge base: query planning,
    knowledge sources, and output merging. Run the index cell.

![](./media/image90.png)

2.  Verify the output **Index 'product-catalog-ep3' ready**.

![](./media/image91.png)

3.  Run the next cell and verify **8 documents uploaded to
    'product-catalog-ep3'**.

![](./media/image92.png)

4.  Run the cell that creates the two knowledge sources: the indexed
    product catalog and the web source.

![](./media/image93.png)

5.  Verify the outputs **Indexed knowledge source
    'product-index-source-ep3' ready** and **Web knowledge source
    'web-source-ep3' ready**.

![](./media/image94.png)

6.  Run the knowledge base cell and verify **Knowledge base
    'product-kb-ep3' ready with 2 sources**.

![](./media/image95.png)

### Task 3: Query with minimal reasoning effort

1.  Under **Step 2 - Query with Minimal Reasoning Effort**, review the
    notes. With minimal effort you provide explicit **search_intents**,
    every intent is sent to every knowledge source, and no LLM tokens
    are consumed during retrieval. Run the cell.

![](./media/image96.png)

2.  Verify the output **Minimal effort retrieval complete**.

![](./media/image97.png)

3.  Run the next cell and review the **Activity log (minimal effort)**.
    There is no **modelQueryPlanning** step; the search intent *"What
    Zava paint is best for bathrooms?"* is sent directly to the index.

![](./media/image98.png)

### Task 4: Query with low reasoning effort

1.  Under **Step 3 - Query with Low Reasoning Effort**, run the cell. It
    asks *"Does Zava have exterior paint and how much does it cost?"*
    The LLM breaks the question into subqueries and selects only the
    relevant sources. Verify the output **Low effort retrieval
    complete**.

![](./media/image99.png)

2.  Run the next cell and review the **Activity log (low effort)**. It
    now includes a **modelQueryPlanning** step (gpt-5.4) followed by
    focused subqueries such as **exterior paint**.

![](./media/image100.png)

### Task 5: Query with medium reasoning effort

1.  Under **Step 4 - Query with Medium Reasoning Effort**, review how
    iterative retrieval works: a first pass, an evaluation of whether
    enough information was found, and a refined second pass if needed.
    Run the cell with the complex question *"Explain how to paint my
    house most efficiently, then give me a list of Zava products and
    prices..."*

![](./media/image101.png)

2.  Verify the output **Medium effort retrieval complete**.

![](./media/image102.png)

3.  Run the next cell and review the **Activity log (medium effort)**.
    Notice the **web** step that queries **web-source-ep3** in addition
    to the product index.

![](./media/image103.png)

**Note:** The web step may show a warning that some documents had their
title or content truncated to fit per-document size limits. This is
informational.

### Task 6: Compare effort levels side by side

1.  Under **Step 5 - Compare Effort Levels Side by Side**, run the cell.
    It sends *"What Zava products do I need to repaint a bathroom, and
    what will it cost?"* at all three effort levels.

![](./media/image104.png)

2.  Verify the output **All three effort levels retrieved**.

![](./media/image105.png)

3.  Run the next cell to print the comparison table of **Activity
    Steps** and **References** for each effort level.

![](./media/image106.png)

**Note:** Your numbers will differ from the screenshot. Higher effort
uses more LLM steps (cost and latency) in exchange for better-targeted
results; minimal effort is the fastest and cheapest option.

### Task 7: Customize answer synthesis

1.  Under **Step 6 - Answer Synthesis**, review the two output modes:
    **EXTRACTIVE_DATA** returns raw chunks and references for your agent
    to process, and **ANSWER_SYNTHESIS** returns a natural-language
    answer with inline citations. **ANSWER_SYNTHESIS** is required when
    a web knowledge source is included. Run the cell that updates the
    **answer_instructions**.

![](./media/image107.png)

2.  Verify the output **Knowledge Base updated with answer synthesis**.

![](./media/image108.png)

3.  Run the next cell. It asks *"What Zava paint should I use in my
    bathroom..."* and prints the synthesized answer.

![](./media/image109.png)

4.  Review the **Synthesized Answer**. It recommends **Zava Bathroom
    Moisture-Guard Paint** with inline citations such as
    **\[ref_id:0\]**, and reports how many references grounded the
    answer.

![](./media/image110.png)

5.  Scroll to the end of the notebook and run the remaining cells,
    including the **Clean Up** cell, to delete the knowledge base,
    knowledge sources, and index created in this cookbook.

## Exercise 5: Clean up the resources

1.  In the Azure portal, open the **Cloud-Native** resource group.

2.  Select all the resources created by the deployment, select **...
    (More)** \> **Delete**, type +++delete+++ to confirm, and then click
    on **Delete**. (**DO NOT DELETE** the resource group.)

**Note:** After you delete the Azure AI Search and Foundry resources,
the knowledge bases, knowledge sources, and agents are also removed.

**Summary**

In this lab, you deployed the Foundry IQ resources, including Azure AI
Search, Azure OpenAI, Microsoft Foundry, and Azure Storage, using an ARM
template. You created a search index, a knowledge source, and a
knowledge base, queried it with agentic retrieval, and inspected the
query plan and references. You connected the knowledge base to a Foundry
Agent Service agent through its MCP endpoint and received cited answers.
You then built Zava's multi-source knowledge base from indexed, Blob
Storage, and web knowledge sources, compared minimal, low, and medium
retrieval reasoning effort, and customized answer synthesis. Finally,
you cleaned up the Azure resources.
