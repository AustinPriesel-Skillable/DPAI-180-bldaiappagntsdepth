# Lab 3: Create advanced Postgres-powered agentic apps with Azure HorizonDB

**Scenario**

**HorizonShip** is a global logistics company that moves cargo for
retailers, hospitals, manufacturers and relief organizations. It tracks
shipments across ocean, air and land routes. The operations team handles
hundreds of shipment questions every day, such as *"Where are the
medical supplies for the clinics?"*, *"Which shipments are delayed?"*
and *"Which ship is carrying cold-chain vaccines?"*

Today the answers come from several separate systems. Shipment records
sit in a relational database, locations in a separate mapping tool, and
search depends on exact keywords and filters. A coordinator who asks for
"hospital equipment" won't find a shipment described as "medical imaging
devices for a new clinic." Answering one question can mean running
several queries and checking a map by hand, which slows the response to
delays and exceptions.

HorizonShip wants one intelligent operations platform where coordinators
can ask questions in plain language and get accurate answers based on
live shipment data. The engineering team chose **Azure HorizonDB**, a
PostgreSQL-compatible, cloud-native database, as the single data
platform. Shipment data, locations and AI embeddings all live in one
database:

- **PostGIS** stores and queries each shipment's current location and
  route.

- **azure_ai** calls Azure OpenAI from inside the database to generate
  an embedding for each shipment description.

- **pgvector** and **DiskANN** store those embeddings and run fast
  similarity searches, so a query can find shipments by meaning, not
  just keywords.

- A **Shipment assistant** agent built with the **Microsoft Agent
  Framework** uses the **gpt-5.4** model. It calls a shipment-search
  tool against HorizonDB and answers questions from the matching
  shipment data.

As a HorizonShip cloud engineer, you will set up the Azure resources,
enable the required database extensions, load the fleet data and
embeddings, and run the HorizonShip Global Operations app. When you
finish, operations coordinators can filter the fleet on a live map and
ask the assistant questions such as *"Which shipments are going to
Europe?"* and get ranked, explained answers in seconds.

**Introduction**

In this lab, you build and run **HorizonShip**, a global
shipment-tracking application powered by **Azure HorizonDB (Preview)**,
a PostgreSQL-compatible, cloud-native database service. The app combines
relational data, geospatial queries (**PostGIS**), vector embeddings
(**pgvector** + **DiskANN**), and in-database AI (**azure_ai**) with an
AI agent built on the **Microsoft Agent Framework** and **Azure
OpenAI**. You will provision the database and AI resources in the Azure
portal, configure the extensions the app needs, seed the database with
shipment data and embeddings, and then chat with a shipment assistant
that answers questions using semantic (vector) search.

**Architecture**

| **Component** | **Role in the lab** |
|----|----|
| GitHub fork of horizondb-ship-agents | Source code for the backend (Python / FastAPI) and frontend (Vite / React) |
| Azure HorizonDB cluster horizondb\<number\> | PostgreSQL 17 database that stores shipments, locations, and embeddings |
| Parameter group aiextension | Allows the azure_ai, vector, pg_diskann, postgis, and uuid-ossp extensions on the cluster |
| Azure OpenAI horizonshipai\<number\> | Hosts the gpt-5.4 chat model and the text-embedding-3-small embedding model |
| Backend API (http://127.0.0.1:8000) | FastAPI service that exposes shipment, semantic search, and agent chat APIs |
| Frontend (http://localhost:5173) | HorizonShip web UI with a fleet map and the Shipment assistant |

**Objectives**:

- Create an Azure HorizonDB cluster and configure networking and
  authentication.

- Create an Azure OpenAI resource and deploy a chat model and an
  embedding model.

- Allow PostgreSQL extensions (azure_ai, vector, pg_diskann, postgis,
  uuid-ossp) using a HorizonDB parameter group.

- Seed a HorizonDB database with shipment data, vector embeddings, and
  a DiskANN index.

- Run a FastAPI backend and a Vite/React frontend locally and explore
  an agentic shipment assistant.

- Clean up the Azure resources created in the lab.

**Prerequisites**

- **GitHub Account**: You are expected to have your own GitHub login
  credentials. If you do not have an account, create one by visiting:
  +++https://github.com/signup+++

- **Lab VM tools**: Visual Studio Code, Git, Python 3.11+, Node.js, and
  npm (you verify these in Exercise 2).

**Important:** The resource names in the screenshots (for example,
**horizondb675489**, **horizondb98098**, **horizonshipai3675**, or
**horizonshipai7689**) are examples. Always use the names, endpoints,
and keys from **your own** lab environment.

## Exercise 1: Provision the Azure resources

In this exercise, you create the Azure HorizonDB cluster that stores the
shipment data, and an Azure OpenAI resource with the chat and embedding
models used by the app.

### Task 1: Create an Azure HorizonDB cluster

1.  Open your browser, navigate to the address bar, and type or paste
    the following URL: +++https://portal.azure.com/+++ then press the
    **Enter** button. Sign in with the following credentials.

| **Username** | **+++@lab.CloudPortalCredential(User1).Username+++** |
|----|----|
| **Password** | **+++@lab.CloudPortalCredential(User1).AccessToken+++** |

2.  In the Azure portal search bar, enter +++Azure HorizonDB+++, and then
    select **Azure HorizonDB (Preview)** from the search results.

![](./media/image1.png)

3.  Click on **+ Create**.

![](./media/image2.png)

4.  On the **Basics** tab, enter the following information, and then
    click on **Next**.

| **Resource group** | **Cloud-Native** |
|----|----|
| **Cluster name** | **horizondb\<unique number\>** (for example, **horizondb675489**) |
| **Region** | **(US) Central US** |
| **PostgreSQL version** | **17** |
| **Compute** | Keep the default settings (2 vCores, 16 GiB RAM, auto-scaling storage) |

![](./media/image3.png)

**Note:** The cluster name must be globally unique. If the name is
already taken, add a few more digits to it.

5.  Under **Authentication**, keep **PostgreSQL authentication** *only*
    selected, and enter the following details. Then click on **Next**.

| **Administrator login** | +++adminday2+++ |
|----|----|
| **Password** | +++password321!+++ |
| **Confirm password** | +++password321!+++ |

![](./media/image4.png)

**Important:** Note down the administrator login and password. You will
add them to the backend **.env** file in Exercise 2.

6.  On the **Networking** tab, select **Allow public access from any
    Azure service within Azure to this cluster**, and then click on
    **Review + create**.

![](./media/image5.png)

7.  Click on **Create**.

![](./media/image6.png)

8.  Deployment typically takes **5-10** minutes. When complete, click on
    **Go to resource**.

![](./media/image7.png)

![](./media/image8.png)

9.  In the **Cloud-Native** resource group, select your Azure HorizonDB
    cluster.

![](./media/image9.png)

10. The **Overview** page shows the host name as **Primary endpoint**.
    The value is truncated; click into the field or use the copy icon to
    get the full value, for example
    **horizondb675489.a3c5a14ee660.centralus.horizondb.azure.com**.

![](./media/image10.png)

11. Copy and save the **Cluster name** and **Primary endpoint** values,
    as you will need them in later tasks.

![](./media/image11.png)

### Task 2: Create an Azure OpenAI service and deploy the models

1.  From the Azure portal Home page, search for and select
    +++Microsoft Foundry+++.

![](./media/image12.png)

2.  In the **Microsoft Foundry** portal, select **Azure OpenAI** from
    the left navigation pane, click on **Create**, and then choose
    **Azure OpenAI** from the menu.

![](./media/image13.png)

3.  Enter the following details and click on **Next**.

| **Subscription** | **@lab.CloudSubscription.Name** |
|----|----|
| **Resource group** | **@lab.CloudResourceGroup(AgenticAI).Name** |
| **Region** | **@lab.CloudResourceGroup(AgenticAI).Location** |
| **Name** | +++horizonshipai@lab.LabInstance.Id+++ |
| **Pricing tier** | **Standard S0** |

![](./media/image14.png)

4.  Click on **Next** on the next two screens, and then click on
    **Create** on the **Review + submit** screen.

![](./media/image15.png)

![](./media/image16.png)

![](./media/image17.png)

5.  Click on **Go to resource** once the service is created.

![](./media/image18.png)

6.  Under **Resource Management**, select **Keys and Endpoint**, click
    on **Show Keys**, and then copy and save **KEY 1**,
    **Location/Region**, and **Endpoint**. These values are required in
    later tasks.

![](./media/image19.png)

**Important:** Treat **KEY 1** like a password. Do not share it or
commit it to GitHub.

7.  From the **Overview** page of the Azure OpenAI resource, click on
    **Go to Azure AI Foundry portal** to deploy the models.

![](./media/image20.png)

![](./media/image21.png)

8.  In Microsoft Foundry, locate your **horizonshipai\<number\>** Azure
    OpenAI resource, and then click on **Open in Foundry Classic**.

![](./media/image22.png)

9.  In **Model catalog**, enter +++gpt-5.4+++ in the search box, and
    then select the **gpt-5.4** model from the search results.

![](./media/image23.png)

10. Click on **Use this model**.

![](./media/image24.png)

11. Keep the deployment name as **gpt-5.4** and click on **Deploy**.

![](./media/image25.png)

![](./media/image26.png)

12. In **Model catalog**, enter +++text-embedding-3-small+++ in the
    search box, and then select the **text-embedding-3-small** model
    from the search results.

![](./media/image27.png)

13. Click on **Use this model**.

![](./media/image28.png)

14. Keep the deployment name as **text-embedding-3-small** and click on
    **Deploy**.

![](./media/image29.png)

![](./media/image30.png)

**Note:** The deployment names must match the values of
**AZURE_OPENAI_DEPLOYMENT** (gpt-5.4) and **AZURE_EMBED_DEPLOYMENT**
(text-embedding-3-small) in the backend **.env** file. If you change a
deployment name, update the .env file to match.

## Exercise 2: Set up the development environment

In this exercise, you fork and clone the HorizonShip repository, install
the backend dependencies, and configure the backend environment
variables.

### Task 1: Fork the GitHub repository

1.  Open your browser, navigate to the address bar, and type or paste
    the following URL:
    +++https://github.com/technofocus-pte/horizondb-ship-agents+++

![](./media/image31.png)

2.  Click on **Fork**, give a unique name to the repository, and then
    click on the **Create fork** button.

![](./media/image32.png)

![](./media/image33.png)

3.  In your forked repository, click on **Code**, verify that the
    **HTTPS** tab is selected, and then select the **Copy URL** icon to
    copy the repository URL for use in the next task.

![](./media/image34.png)

### Task 2: Clone the lab repository

1.  In the Windows search box, type +++Visual Studio Code+++, and then
    select **Visual Studio Code**.

2.  In Visual Studio Code, select the **More Actions (...)** menu,
    select **Terminal**, and then choose **New Terminal**.

![](./media/image35.png)

3.  Run the following commands to verify that Python, Node.js, and npm
    are installed.

> +++py --version+++
>
> +++node --version+++
>
> +++npm --version+++

![](./media/image36.png)

4.  Navigate to the **C:\LabFiles** directory.

> +++cd C:\LabFiles+++

5.  Create a new folder named **day2lab**.

> +++mkdir day2lab+++

![](./media/image37.png)

6.  Navigate to the **day2lab** folder, and then clone **your forked**
    repository. Replace **\<your-repo-URL\>** with the URL you copied in
    the previous task.

> +++cd day2lab+++
>
> +++git clone \<your-repo-URL\>+++

![](./media/image38.png)

**Note:** The screenshot shows an example fork URL. Use the URL of your
own fork.

7.  In Visual Studio Code, select **File \> Open Folder**, browse to the
    **C:\LabFiles\day2lab\horizondb-ship-agents** folder, and then click
    on **Select Folder**.

![](./media/image39.png)

![](./media/image40.png)

![](./media/image41.png)

**Note:** If prompted with **Do you trust the authors of the files in
this folder?**, select **Yes, I trust the authors**.

### Task 3: Install the backend dependencies

1.  In Visual Studio Code, select the **More Actions (...)** menu,
    select **Terminal**, and then choose **New Terminal**.

![](./media/image42.png)

2.  Navigate to the backend folder.

> +++Set-Location backend+++

3.  Create a Python virtual environment.

> +++py -m venv .venv+++

4.  Install the project dependencies in editable mode with development
    tools.

> +++.\.venv\Scripts\python -m pip install -e ".[dev]"+++

![](./media/image43.png)

**Note:** The installation takes a few minutes. You can ignore the **A
new release of pip is available** notice.

### Task 4: Configure the backend environment variables

1.  After the installation completes, create the **.env** file from the
    provided template.

> +++Copy-Item .env.example .env+++

![](./media/image44.png)

2.  In the **Explorer** pane, expand **backend** and open the **.env**
    file. Update the values with the details you saved in Exercise 1,
    and then save the file (**Ctrl+S**).

| **Variable** | **Value** |
|----|----|
| AZURE_PG_HOST | Primary endpoint of your HorizonDB cluster |
| AZURE_PG_NAME | +++postgres+++ |
| AZURE_PG_USER | +++adminday2+++ |
| AZURE_PG_PASSWORD | +++password321!+++ |
| AZURE_PG_PORT | +++5432+++ |
| AZURE_PG_SSLMODE | +++require+++ |
| AZURE_OPENAI_ENDPOINT | Endpoint of your Azure OpenAI resource |
| AZURE_OPENAI_KEY | KEY 1 of your Azure OpenAI resource |
| AZURE_OPENAI_DEPLOYMENT | +++gpt-5.4+++ |
| AZURE_EMBED_DEPLOYMENT | +++text-embedding-3-small+++ |
| AZURE_API_VERSION | +++2025-03-01-preview+++ |
| EMBEDDING_MODEL_ALIAS | +++horizonship-embedding+++ |

![](./media/image45.png)

**Important:** **AZURE_PG_NAME** is the **database name**, not the
cluster name. Set it to **postgres** (the default database). Using the
cluster name (for example, horizondb675489) causes a *database does not
exist* error.

3.  Switch to the Azure portal, open your HorizonDB cluster, and under
    **Settings** select **Networking**. Verify that **Allow public
    access from any Azure service within Azure to this cluster** is
    selected, and then click on **Save** if you made any changes.

![](./media/image46.png)

**Note:** If you run the app from a machine outside Azure (for example,
your own laptop), also select **+ Add current client IP address** and
then **Save**; otherwise the backend cannot connect to the cluster.

4.  Back in Visual Studio Code, verify that **AZURE_PG_NAME** is set to
    **postgres**, and then run the database setup script.

> +++.\.venv\Scripts\python -m app.setup_database+++

![](./media/image47.png)

**Note:** At this point, the script fails with a **HINT: to learn how to
allow an extension...** message. This is expected - HorizonDB only
allows extensions that are listed in the cluster's **azure.extensions**
parameter. You fix this in the next exercise.

![](./media/image48.png)

## Exercise 3: Allow extensions and seed the database

The HorizonShip schema needs five PostgreSQL extensions:

- **azure_ai** - calls Azure OpenAI from inside the database to generate
  embeddings

- **vector** - stores embeddings (pgvector)

- **pg_diskann** - builds a DiskANN index for fast approximate vector
  search

- **postgis** - stores and queries shipment locations

- **uuid-ossp** - generates UUIDs

On HorizonDB, parameters are managed through **parameter groups**. You
create a new parameter group that allows these extensions and then
connect it to your cluster.

### Task 1: Create a parameter group

1.  In the Azure portal, go to your HorizonDB cluster, expand
    **Settings**, and then select **Parameters**.

![](./media/image49.png)

2.  Next to **Parameter group**, select **(create)**.

![](./media/image50.png)

3.  On the **Create a parameter group** page, enter the following
    details and click on **Next**.

| **Subscription** | Keep the default subscription |
|----|----|
| **Resource group** | **Cloud-Native** |
| **Parameter group name** | +++aiextension+++ |
| **Region** | **(US) Central US** |
| **Parameter group** | **PostgreSQL 17** |

![](./media/image51.png)

4.  In the filter box, type +++azure.extensions+++. In the **Value**
    column of the **azure.extensions** row, open the drop-down list and
    select **azure_ai**.

![](./media/image52.png)

5.  Scroll down and select **pg_diskann**.

![](./media/image53.png)

6.  Continue scrolling and select **postgis**, **uuid-ossp**, and
    **vector**.

![](./media/image54.png)

7.  Verify that the **Value** shows **azure_ai, vector, pg_diskann,
    postgis, uuid-ossp**, and then click on **Create**.

![](./media/image55.png)

**Note:** Select all five extensions. If any extension is missing, the
database setup script fails when it tries to create that extension.

8.  Wait for the deployment to complete, and then click on **Go to
    resource**.

![](./media/image56.png)

![](./media/image57.png)

### Task 2: Connect the parameter group to the cluster

1.  Open the **Cloud-Native** resource group. You now see the
    **aiextension** parameter group, your HorizonDB cluster, and your
    Azure OpenAI resource.

![](./media/image58.png)

2.  Select your HorizonDB cluster.

![](./media/image59.png)

3.  Under **Settings**, select **Parameters**, and then next to
    **Parameter group: Default**, select **(change)**.

![](./media/image60.png)

4.  In the **Select parameter group** pane, select **aiextension**, and
    then click on **Save**.

![](./media/image61.png)

5.  Wait for the notification **Successfully connected parameter group
    aiextension to \<your cluster\>**.

![](./media/image62.png)

**Important:** **azure.extensions** is a **Static** parameter. If the
extensions are still not allowed after connecting the parameter group,
go to the cluster **Overview** page, click on **Restart**, and wait until
the status returns to **Succeeded** before continuing.

### Task 3: Verify the database connection

1.  Switch back to Visual Studio Code. In the terminal (in the
    **backend** folder), paste the following PowerShell block and press
    **Enter**. It creates a small script named **check_models.py** that
    connects to the database and lists the installed extensions.

```powershell
@'
import psycopg
from app.config import Settings

settings = Settings()
print("Connecting to database:", settings.azure_pg_name, "as", settings.azure_pg_user)

with psycopg.connect(settings.database_conninfo) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT current_database();")
        print("current_database():", cur.fetchone())

        cur.execute("SELECT extname, extversion FROM pg_extension ORDER BY extname;")
        print("Installed extensions:")
        for row in cur.fetchall():
            print(" ", row)

        cur.execute("SELECT nspname FROM pg_namespace WHERE nspname ILIKE '%%model%%' OR nspname ILIKE '%%azure%%';")
        print("Matching schemas:")
        for row in cur.fetchall():
            print(" ", row)
'@ | Out-File -Encoding utf8 check_models.py
```

![](./media/image63.png)

**Note:** Python is indentation-sensitive. Paste the block exactly as
shown, including the leading spaces. The closing **'@** must be at the
start of the line.

2.  Run the script.

> +++.\.venv\Scripts\python check_models.py+++

3.  Verify that the output shows **Connecting to database: postgres as
    adminday2** and **current_database(): ('postgres',)**. This confirms
    that the backend can reach your HorizonDB cluster.

4.  Check whether any Azure OpenAI variables are already set in the
    current terminal session.

> +++Get-ChildItem Env: | Where-Object Name -like "AZURE*"+++

![](./media/image64.png)

5.  If the command lists **AZURE_OPENAI_API_KEY** or
    **AZURE_OPENAI_ENDPOINT** (left over from another lab), remove them
    so that the values in your **.env** file are used.

> +++Remove-Item Env:AZURE_OPENAI_ENDPOINT -ErrorAction SilentlyContinue+++
>
> +++Remove-Item Env:AZURE_OPENAI_API_KEY -ErrorAction SilentlyContinue+++

6.  Run the following command again and confirm that nothing is
    returned.

> +++Get-ChildItem Env: | Where-Object Name -like "AZURE*"+++

![](./media/image65.png)

**Important:** Environment variables set in the terminal take
precedence over the .env file. If you skip this step, the app may call a
different Azure OpenAI resource and fail with an authentication or
*deployment not found* error.

### Task 4: Seed the database

1.  Run the database setup script again.

> +++.\.venv\Scripts\python -m app.setup_database+++

![](./media/image66.png)

2.  Verify that the output shows **HorizonShip ready: 24 shipments, 24
    Azure embeddings, primary DiskANN index: ready**.

![](./media/image67.png)

![](./media/image68.png)

![](./media/image69.png)

**Note:** The script creates the extensions and tables, loads 24 sample
shipments, generates an embedding for each shipment by calling the
**text-embedding-3-small** deployment through the **azure_ai**
extension, and builds a **DiskANN** index on the embeddings.

## Exercise 4: Run and explore the HorizonShip app

In this exercise, you start the backend API and the frontend, explore
the fleet dashboard, and chat with the Shipment assistant.

### Task 1: Start the backend API

1.  In the same terminal (in the **backend** folder), start the backend
    server.

> +++.\.venv\Scripts\python -m app.server+++

![](./media/image70.png)

2.  Wait until you see **Uvicorn running on http://127.0.0.1:8000**. You
    can ignore the **ExperimentalWarning** messages.

![](./media/image71.png)

**Important:** Keep this terminal open. The backend must keep running
while you use the frontend.

3.  Open a browser and go to +++http://127.0.0.1:8000+++. The response
    **{"detail":"Not Found"}** is expected because the API has no page
    at the root path.

![](./media/image72.png)

### Task 2: Start the frontend

1.  In Visual Studio Code, select the **More Actions (...)** menu,
    select **Terminal**, and then choose **New Terminal** to open a
    second terminal.

![](./media/image73.png)

2.  In the new terminal (at the repository root), run the following
    commands to install the frontend dependencies and start the
    development server.

> +++Set-Location frontend+++
>
> +++npm install+++
>
> +++npm run dev+++

![](./media/image74.png)

**Note:** Visual Studio Code may automatically activate the Python
virtual environment in the new terminal (you see **(.venv)** in the
prompt). This does not affect the frontend.

3.  When **VITE ready** and **Local: http://localhost:5173/** appear,
    hold **Ctrl** and select the link, or open
    +++http://localhost:5173+++ in your browser.

### Task 3: Explore the fleet dashboard

1.  The **HorizonShip - Global Operations** page opens. It shows the
    **Shipments** list on the left, a world map with 24 tracked
    shipments in the middle, and the **Shipment assistant** on the
    right.

![](./media/image75.png)

2.  In the **Shipments** list, select **SHIP-0001 Consumer
    Electronics**. The map draws the route, and a card shows the cargo,
    origin, destination, current location, and ETA.

![](./media/image76.png)

3.  Select the **All statuses** drop-down, and then select
    **Delivered**.

![](./media/image77.png)

![](./media/image78.png)

4.  The list is filtered to the delivered shipments (**SHIP-0003** and
    **SHIP-0013**) and the map zooms to their locations.

![](./media/image79.png)

5.  Select the status drop-down again and select **Delayed**.

![](./media/image80.png)

6.  The list shows the three delayed shipments (**SHIP-0005**,
    **SHIP-0011**, and **SHIP-0020**).

![](./media/image81.png)

### Task 4: Chat with the Shipment assistant

The Shipment assistant is an AI agent built with the Microsoft Agent
Framework. It uses the **gpt-5.4** model and a shipment-search tool that
runs a **vector (cosine similarity) search** against the DiskANN index
in HorizonDB.

1.  In the **Shipment assistant** pane, select the suggested prompt
    **Show medical supplies for clinics**.

![](./media/image82.png)

2.  Review the response. The assistant lists the best matches (for
    example, **SHIP-0002 Medical Supplies**, **SHIP-0014 Hospital
    Equipment**, and **SHIP-0021 Emergency Medical Kits**) with their
    status, route, current location, ETA, and why each one fits. The
    shipments list on the left shows a **similarity percentage** for
    each match.

![](./media/image83.png)

3.  With the **Delayed** filter selected, select the suggested prompt
    **Which shipments are going to Europe?**

![](./media/image84.png)

4.  Review the response. The assistant identifies **SHIP-0020
    (Shanghai, China to Rotterdam, Netherlands)** as the delayed
    shipment going to Europe.

![](./media/image85.png)

5.  In the **Ask about shipments** box, enter
    +++When did the ship reach Mumbai?+++ and select the **Send** icon.

![](./media/image86.png)

6.  Review the response and the matching shipment cards.

![](./media/image87.png)

**Note:** Occasionally the assistant replies that the shipment search
tool failed, as shown above. This is usually a transient error; resend
the question. Also note that the data only contains current location
and ETA, so the assistant can only answer from the returned shipment
data.

7.  Select the **Reset** icon at the top of the Shipment assistant to
    start a new conversation. Enter
    +++Which ship is transporting cold-chain vaccines?+++ and select the
    **Send** icon.

![](./media/image88.png)

8.  Review the response. The best match is **SHIP-0007 - Cold-Chain
    Vaccines** (Boston, USA to Nairobi, Kenya), with **SHIP-0017 -
    Pharmaceuticals** as a secondary match.

![](./media/image89.png)

9.  In the result cards, select **Locate** on a shipment to highlight it
    on the map and open its details card.

![](./media/image90.png)

### Task 5: Troubleshooting - Restart the backend and frontend

If the app stops responding (for example, the shipments list is empty or
the assistant returns errors), check the backend terminal.

1.  If the backend terminal shows an error such as
    **psycopg.OperationalError: consuming input failed: server closed
    the connection unexpectedly** followed by **Finished server
    process**, the backend has stopped.

![](./media/image91.png)

2.  Restart the backend in the **backend** terminal.

> +++.\.venv\Scripts\python -m app.server+++

![](./media/image92.png)

3.  If the frontend terminal has returned to the prompt, restart the
    frontend in the **frontend** terminal.

> +++npm run dev+++

![](./media/image93.png)

![](./media/image94.png)

4.  Open +++http://localhost:5173+++ again and refresh the page.

![](./media/image95.png)

**Note:** A dropped connection can happen when the HorizonDB cluster
restarts (for example, after connecting the parameter group) or when the
connection is idle for a long time. Restarting the backend creates a new
connection pool.

## Exercise 5: Clean up the resources

1.  In the Visual Studio Code terminals, press **Ctrl+C** to stop the
    frontend and backend servers.

2.  In the Azure portal, open the **Cloud-Native** resource group.

![](./media/image96.png)

3.  Select the checkbox next to **Name** to select all the resources
    (**aiextension**, your HorizonDB cluster, and your Azure OpenAI
    resource). Select **... (More)**, and then click on **Delete**.

![](./media/image97.png)

4.  In the **Delete Resources** pane, type +++delete+++ in the
    confirmation box, and then click on **Delete**. (**DO NOT DELETE**
    the resource group.)

![](./media/image98.png)

**Summary**

In this lab, you created an Azure HorizonDB cluster and an Azure OpenAI
resource with the gpt-5.4 chat model and the text-embedding-3-small
embedding model. You configured a parameter group to allow the azure_ai,
vector, pg_diskann, postgis, and uuid-ossp extensions, and seeded the
database with shipment data, in-database generated embeddings, and a
DiskANN vector index. You then ran the HorizonShip FastAPI backend and
React frontend, explored shipments on the map, and used an agentic
Shipment assistant built on the Microsoft Agent Framework to answer
natural-language questions using vector search over Postgres data.
Finally, you cleaned up the Azure resources.
