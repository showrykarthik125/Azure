# Azure

## Resource Group: 
To create any resource within the Azure cloud, we should be having a Resource Group and in that resource group we create all the required resources, such as: **_Storage Account (ADLS)_**, **_Databricks_**, **_Azure Data Factory (ADF)_** etc

![image](https://github.com/user-attachments/assets/74c8aae0-4e47-4021-9c1d-367468f8f4e0)
![image](https://github.com/user-attachments/assets/5aa4cd8e-93bc-4bd7-b0bb-ba0877cf0a3e)

## Azure Data Lake Storage:
Azure Data Lake Storage is a cloud-based repository for storing large amounts of **structured** and **unstructured data**, optimized for analytics and big data processing.

![image](https://github.com/user-attachments/assets/2b38ffb3-b565-4003-9acd-3fa24a356c38)
![image](https://github.com/user-attachments/assets/b9d322d2-26b5-462e-8315-14756ef64f27)
### **_Enabling the option of hierarchical namespace will lets us use the features of Data Lake otherwise it would just be normal blob storage._**
![image](https://github.com/user-attachments/assets/003ddb83-d697-4be8-90a8-cc4373c2c38e)


## Databricks:
Databricks is a cloud-based data platform designed for big data analytics and machine learning. It's built around Apache Spark and offers a collaborative environment for data engineers, scientists, and analysts.

![image](https://github.com/user-attachments/assets/41d2d09a-ab76-4510-828a-cc4392de1fab)
![image](https://github.com/user-attachments/assets/5fd63c5a-a815-48b5-9c36-8ccc643a0564)

## Delta Lake:

Delta Lake is a Storage Layer which brings the feature of 
- ACID transactions,
- scalability and
- schema management
to the data in our data Lake, Delta Lake is built on top of the Data Lake, Delta Lake was created by Databricks, basically it takes the unstructured data from the data lake makes it as structured data, we can again keeps this refined data in our data lakes. Delta Lake Files are built using Apache Parquet.

## Parquet Files:

Parquet files stores the data in columnar format, which means it stores data by columns instead of rows.
For example, if we have some data as follows:


| Name | Age | Country |
|------|-----|---------|
| Ram  | 25  | India   |
| Sara | 30  | USA     |


When we store this data in parquet files, it will be stored as follows:
- Column: name    → ["Ram", "Sara"]
- Column: age     → [25, 30]
Column: country → ["India", "USA"]

Advantages:
- Compressed (small file size)
- Fast to read specific columns
- Commonly used for storage (data lakes)

# we have the raw data in our Data Lake (ADLS storage account) it can be in either csv, JSON or even in parquet format. we want to use this data in our databricks to refine this data to make some business decisions, but how can we access the data in our databricks workspace?



we can access the data in our data lake in the databricks workspace in two ways
# 1. Service Principle (mostly used to access the data as filesystem such as DBFS)
# 2. Managed Identity (Access connector for Databricks) (mostly used to access the data through unity catalog)



Both of them are by using the Microsoft Entra ID only 

## Microsoft Entra ID (Azure Active Directory):

**Microsoft Entra ID** is a cloud-based **Identity and Access management (IAM)** service provided by Microsoft. 
It helps organizations:

- Manage user identities and credentials
- Control access to applications and resources
- Enable secure sign-in across Microsoft services and third-party applications


# 1.1. **Service Principle**

## 1. Create a Service Principal in Azure
- Go to Microsoft Entra ID > App registrations > New Registration.
  ![image](https://github.com/user-attachments/assets/000d4937-60c4-4d9a-beff-368992246d73)
- Name it (e.g., `mychocloatesales_servicePrinciple`) and register.
- Note the Client ID and Tenant ID from the Overview tab.
  ![image](https://github.com/user-attachments/assets/b921a0ea-bb48-4628-8780-c0810cf516d8)
- Under Certificates & Secrets, create a new client secret.
  ![image](https://github.com/user-attachments/assets/b44c366e-3a7f-45c0-823a-455d262aa917)
- Save the secret value immediately (you won’t see it again).

## This secret key will let us access the data in our data lake when we assign it to the data lake at the later part, so we have to keep this securely, we can use Azure key vault and store it in there and access it whenever we need it.

## 2. Key-Vault:
This is used to store our secret keys  in our azure cloud and access them whenever and wherever securely.
- To create a Azure Key Vault
- Go to Key Vaults > Create.
  ![image](https://github.com/user-attachments/assets/74558492-8ff2-40ec-bfa5-fd0773b243f9)
- Set access policy mode to Vault access policy.
  ![image](https://github.com/user-attachments/assets/394ad74f-9a0b-4c55-9467-79f3587d5b16)
- After creation, assign yourself the Key Vault Administrator role in Access Control (IAM).
  ![image](https://github.com/user-attachments/assets/526d62ca-5bf2-445b-89fd-be71aa7ebd97)
- we will use the Vault URI and Resource ID in the properties, when we try to create a secret scope in the Databricks workspace
  ![image](https://github.com/user-attachments/assets/c7efb7bd-98c6-4925-82ce-021f2433c5ec)

## 3. Store the Secret in the secrets
- Go to Secrets > Generate/Import.
- Name the secret (e.g., appSecret) and paste the service principal secret value.
  ![image](https://github.com/user-attachments/assets/0dd83946-2d17-442a-90b1-446285a88653)

## 4. we need to give access to the Service Principleb that we created so that it can access the data in the Data Lake
- Go to your Storage Account > Container > Access Control (IAM).
  ![image](https://github.com/user-attachments/assets/1109608d-8e2b-4bd0-a22d-6a02d6fe3f66)
- Click Add Role Assignment.
- Role: Storage Blob Data Contributor
- Assign access to: User, group, or service principal
- Select the service principal you created and assign.
  ![image](https://github.com/user-attachments/assets/0b4fb533-16e8-447f-9b5e-0cc362b262a4)

## 5. Link this Key Vault with Databricks (By Creating Secret Scope)
- In our Databricks workspace, navigate to: `https://<databricks-instance>#secrets/createScope`
- Fill in:
- Scope Name: karthikScope
- DNS Name: Key Vault URI
- Resource ID: Key Vault Resource ID
  ![image](https://github.com/user-attachments/assets/288d3049-a66a-4823-ade9-a0a5202db5a0)

## 6. configuration in databricks notebook to access the data of data lake in databricks workspace
```python
configs = {
  "fs.azure.account.auth.type": "OAuth",
  "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
  "fs.azure.account.oauth2.client.id": client_id,
  "fs.azure.account.oauth2.client.secret": dbutils.secrets.get(scope="ScopeName", key="SecretName"),
  "fs.azure.account.oauth2.client.endpoint": "https://login.microsoftonline.com/<tenent_id>/oauth2/token"
}

dbutils.fs.mount(
  source = "abfss://<containerName>@<StorageAccountName>.dfs.core.windows.net/",
  mount_point = "/mnt/Sales",
  extra_configs = configs)
```


# 2. Managed Identity 

## To provide access to the data for us to use it the databricks using Managed Identity, First we create a **ACESS CONNECTOR** for Databricks

### 2.1 Create a Access Connector and assign the necessary permission to it:
- Go to Access Connector for Azure Databricks and click on Create and choose our resource group where our databricks is located.
  ![image](https://github.com/user-attachments/assets/af9e0294-0ff3-49c2-8d38-604ed5326d0a)
- we need to give access to the Access Connector that we created so that it can access the data in the Data Lake just like we did to the service Principle
- Go to your Storage Account > Container > Access Control (IAM).
- Click Add Role Assignment.
- Role: Storage Blob Data Contributor
- Assign access to: Managed Identity
- Select the Managed Identity you created (Access connector for databricks) and assign.
  ![image](https://github.com/user-attachments/assets/00c85392-0ab1-449a-bcba-6a34a97763c3)

## 2.2. Create a Metastore and Catalogs in that metastore and schemas and Tables within that unity Catalog:

### **Unity Catalog Metastore:**

- The Databricks **Unity Catalog Metastore** is a centralized storage system that holds metadata about our actual data — such as **catalogs**, **schemas**, **tables** and who has permission to our actual data.
A metastore is the top-level container in Unity Catalog. Within a metastore, It provides a 3-level namespace for organizing data: catalogs, schemas (also called databases), and tables / views.

- It stores metadata, not actual data.

### **DataBricks Unity Catalog:**

Unity Catalog is a data governance tool in Databricks. It helps us manage and control who can see, use, and change our data.
In the databricks, catalog acts as a top-level folder that has schemas, tables and views, it helps us organise our data in clean and structured way possible.

**Why is Unity Catalog useful?**
- Easy access: All our data is listed in one place.
- Security: You decide who can view or change the data.
- Audit: You can track who accessed what data and when.
- Works with all languages: Python, SQL, R, Scala—no problem!

Taking Library as an example:
Metastore = The librarian 
- Knows everything about the library.
- Keeps track of:
- All the sections (Unity Catalogs)
- What books (tables) are where
- Who is allowed to read or borrow which books
- Records who accessed what and when (audit)
Unity Catalog = A section of the library
- Like Science, Fiction, History, etc.
- Helps organize books into meaningful areas.

Schema = A bookshelf in the section
- A smaller group of books within a section
- Example: In the “Science” section, you might have bookshelves like “Biology,” “Physics,” etc.
Table/View = The actual book
- Holds the real content (data)
- Users come here to read (query) the data

Only one metastore per a region, each metastore can have multiple unity catalogs.
![image](https://github.com/user-attachments/assets/768915a6-f992-4889-afd3-cfe7ff95cef8)

## Steps to create a Metastore:
- Got to **https://accounts.azuredatabricks.net/** and login using the admin credentails and then under catalog create a metastore.
- Before creating a metastore, we can create a new container, which is used by the metastore where the metastore will store all the metadata (This is optional but useful)
- and also get the access connector resouyrce id which we created earlier
  ![image](https://github.com/user-attachments/assets/4ab0be59-6259-4e48-a0d6-695ce140cf28)
- After that we can assign the workspaces within that metastore

## Steps to create a Storage Credential and External Location:
  Since we have have our data in the ADLS and to access that data we have created a Managed Identity (using Access Connector for Databricks), we need to let unity
  catalog know these things by using Storage Credential (which will point to the Access connector which we created earlier) and External Location (Where our
  actual data), the databricks uses this storage credentail to access the data which is in the External Location.
- To create a Storage Credential and External Location follow  the below steps:
- Go the Databricks workspaece -> Go to the Catalog Section -> under the catalog section -> click on the settings Icon -> here we can set both the Storage
  Credentail and The External Location.
  ![image](https://github.com/user-attachments/assets/b47c6573-a5a9-4e18-88c6-7214c3c9e9a9)
- Firstly, lets create a Storage Credential as Follows:
- we need to know the Access Connector ID inorder to create the Storage Credential (Storage Credential is usually creaated by default when we create a metastore and provide the access connector id at the time of its creation we can use that).
  ![image](https://github.com/user-attachments/assets/74354ea7-7570-4558-812f-d10f8cc08bf4)
- To create an External Location, we need to mention where our data is wheter it is Data lake Storage or S3 or even our DBFS and the URL and we need to specify
  our Storage Credentail which we created earlier as follows:
  ![image](https://github.com/user-attachments/assets/75e5d647-b7f6-44f4-9130-3e97ac783eb2)
- we can even test the connection after creation
  ![image](https://github.com/user-attachments/assets/04bc2c0f-b330-416f-a342-b4e6f7841f50)
  ![image](https://github.com/user-attachments/assets/b68c57b5-c4f6-4b58-b4ed-67a6beeff8ea)
- This confirms that our databricks can access the data in our ADLS using storage credentials through External Location.



  

  


## Delta Files:

Delta is like a smart folder that stores your data files (as Parquet) and remembers every change you make — so you can go back in time, fix mistakes, and build reliable pipelines.

Delta = Parquet + Transaction Log
It keeps track of changes, like:
- Who added what?
- When did a row get deleted?
- What did the table look like last week?
 
Advantages:
- Time travel (see older versions)
- ACID transactions (no partial writes)
- Schema enforcement (data types must match)
- Fast updates, deletes (great for pipelines)

Under the hood:
•	It writes Parquet files for the actual data
•	It writes a **_delta_log** folder to track history

Format	Analogy
CSV	Basic notebook — easy to write, hard to manage
Parquet	Excel with filters and compression
Delta	Excel + Undo History + Audit Log + Rules

What is _delta_log?
- It’s a hidden folder inside every Delta table's directory (e.g., /mnt/data/sales_delta/_delta_log)
- Contains JSON and Parquet files
- Tracks every change made to the Delta table (writes, updates, schema changes, deletes, etc.)

![image](https://github.com/user-attachments/assets/8ace0ac5-f364-4f30-8429-dc835b636492)

 




