## Azure Setup

**1. Create a Resource Group**

Create a single resource group to hold all project resources.

- Go to the [Azure Portal](https://portal.azure.com) → **Resource groups** → **Create**
- Choose a subscription, region, and name (e.g. `rg-ride-hailing`)

---

**2. Create Azure Data Lake Storage Gen2**

Used to store historical ride files and mapping data.

- Go to **Storage accounts** → **Create**
- Select the resource group created above
- Under **Advanced**, enable **Hierarchical namespace** (this enables ADLS Gen2)
- Once created, go to the storage account → **Containers** → create the following containers:
  - `historical-data` — for historical ride CSV/JSON files
  - `mapping-data` — for province, ride option, and payment method JSON files

---

**3. Create Azure Event Hubs**

Used to receive real-time ride events from the data generator.

- Go to **Event Hubs** → **Create** → create a **Namespace**
- Select the resource group and choose a pricing tier (Basic or Standard)
- Inside the namespace, go to **Event Hubs** → **+ Event Hub** → create one named `rides`

---

**4. Create Azure Databricks**

Used to run the Bronze, Silver, and Gold pipeline notebooks.

- Go to **Azure Databricks** → **Create**
- Select the resource group and a pricing tier (Standard or Premium)
- Once deployed, click **Launch Workspace**
- Inside the workspace, create a cluster to run the pipelines
- Connect the Databricks workspace to ADLS Gen2 and Event Hubs by storing credentials as Databricks secrets:

  **Install the Databricks CLI and authenticate**
  ```bash
  pip install databricks-cli
  databricks configure --token
  # Enter your Databricks workspace URL and a personal access token
  ```

  **Create a secret scope**
  ```bash
  databricks secrets create-scope --scope ride-hailing
  ```

  **Add secrets to the scope**
  ```bash
  databricks secrets put --scope ride-hailing --key ADLS_ACCOUNT_NAME
  databricks secrets put --scope ride-hailing --key ADLS_ACCOUNT_KEY
  databricks secrets put --scope ride-hailing --key EVENTHUB_CONNECTION_STRING
  ```

  Secrets can then be read inside Databricks notebooks with:
  ```python
  dbutils.secrets.get(scope="ride-hailing", key="ADLS_ACCOUNT_KEY")
  ```

---

## Local Setup

**1. Clone the repository**
```bash
git clone <repo-url>
cd ride-hailing-project
```

**2. Create virtual environment**
```bash
python -m venv .venv
.venv\Scripts\activate       # Windows
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Set up environment variables**
```bash
cp .env.example .env
# Fill in your Azure credentials
```

**5. Start Nominatim Docker**
```bash
docker run -p 8080:8080 mediagis/nominatim
```

**6. Generate pools**
```bash
python generate_pools.py
```

**7. Generate data**
```bash
python data_generator.py historical --count 5000 --format csv --duration 2026-01-01:2026-02-01
```

**8. Upload to Azure**
```bash
python upload_historical.py
```
