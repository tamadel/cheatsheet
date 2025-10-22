# ERP-Centric Data Engineering Roadmap: From ETL Design to Secure Analytics on Azure

====================================================================
1- Central Data Platform
(construction layer: storage, ingestion, contracts, error handling)
====================================================================
==============================================
1.1 Build and maintain a central data platform
==============================================
Definition:
A data platform is the unified layer that stores, cleans, and serves business data from all operational systems (ERP, CRM, logistics).

Why:
It guarantees one consistent data source for BI, reporting, and automation.

When:
Always when you have multiple systems feeding analytics or AI.


- Python Example:

#------------------------------------------------------------
import requests
import pandas as pd

#ingest ERP sales via API to central data lake
r = requests.get(ERP_URL, params= {"since":"2025-10-01"})
df = pd.DataFrame(r.json())
df.to_parquet("bronze/sales_202510.parquet")
#------------------------------------------------------------

- ADF equivalent:

#------------------------------------------------------------
{
  "pipeline": "ERP_Ingest_Sales",
  "activities": [
    {"type": "Copy", "source": "Dynamics365_API", "sink": "ADLS_Bronze"}
  ]
}
#------------------------------------------------------------


- Practical note

Bronze: raw immutable files.
Silver: cleaned, typed tables.
Gold: curated marts.


Use data contracts (JSON schemas) to lock field names and types. >> (platform stops accepting nonsense)
Automate SLA alerts (Success Time, Row Count, Cost). >> (screams loudly when thresholds break, and gives you receipts for every run.)


==============================================
1.2 Error Handling and Retry Logic
==============================================
Definition:
Error handling ensures data jobs fail safely, log precisely, and recover automatically.

Why:
APIs, files, and databases can fail without warning. Without structured handling, we risk data loss, duplicates, or silent corruption.

When:
Every data ingestion, transformation, or API integration.


- Python Example:

#------------------------------------------------------------
import requests, pandas as pd, time, logging

logging.basicConfig(filename="etl.log", level=logging.INFO)

for attempt in range(3):  # retry logic
    try:
        r = requests.get(ERP_URL, params={"since": "2025-10-01"}, timeout=30)
        r.raise_for_status()
        df = pd.DataFrame(r.json())
        df.to_parquet("bronze/sales_202510.parquet")
        logging.info(f"Run success: {len(df)} rows ingested.")
        break
    except (requests.exceptions.Timeout, requests.exceptions.ConnectionError) as e:
        logging.warning(f"Attempt {attempt+1}: {e}")
        time.sleep(5)
    except Exception as e:
        logging.error(f"Fatal: {e}")
        raise
#------------------------------------------------------------

- ADF equivalent:

#------------------------------------------------------------
{
  "pipeline": "ERP_Ingest_Sales",
  "activities": [
    {"type": "Copy", "source": "Dynamics365_API", "sink": "ADLS_Bronze", "policy": {"retry": 3, "timeout": "00:30:00"}}
  ],
  "errorHandling": {
    "onFailure": "Send_Email_Alert"
  }
}
#------------------------------------------------------------


- Practical note

Always include retry and timeout parameters.
Log both success and failure counts.
Never suppress errors silently.




====================================================================
2- ETL Pipelines
This handles extraction, transformation, loading, and job control.
(Metadata, lineage, testing, CI/CD)
====================================================================

=======================================================================
2.1 Develop and monitor ETL pipelines between ERP, BI and side systems
=======================================================================
Definition:
ETL = Extract Transform Load. Moves data from sources to targets in a repeatable, traceable process.

Why:
To automate data refreshes and guarantee that ERP data reaches BI tools accurately.

When:
Whenever ERP updates must flow daily or hourly to reporting. 

- Python Example:

#----------------------------------------------------------------------------------
import sqlalchemy as sa

#extract ERP orders, transform, load into SQL Server
orders = pd.read_json("bronze/sales_202510.parquet")

orders["OrderDate"] = pd.to_datetime(orders["OrderDate"])

orders["NetAmountCHF"] = orders["Qty"] * orders["UnitPrice"]

orders.to_sql("stg_sales", con = sql_engine, if_exists="replace", index=False)
#----------------------------------------------------------------------------------

- ADF equivalent:

#----------------------------------------------------------------------------------
{
  "pipeline": "ETL_Sales",
  "activities": [
    {"type": "Copy", "source": "ADLS_Bronze", "sink": "SQL_Stage"},
    {"type": "DataFlow", "name": "Transform_Sales"},
    {"type": "StoredProc", "name": "Merge_Into_Gold"}
  ]
}
#----------------------------------------------------------------------------------

- Practical note

Make jobs idempotent (re-runs give same result).	>> (Safe to run again) (Recover from failure without duplicating or breaking data.)
Store watermark timestamps.				>> (Last timestamp processed.) (Load only new or changed data.)
Log rows in/out + error count.
Use ADF monitoring + email alerts on failure.

These two rules make pipelines robust:
(A):	job idempotent
	-----------------------------------
			>> What it means:
					A job is idempotent when running it multiple times produces the same final result — no duplicates, no corruption.
			>> Why:
					ETL jobs sometimes fail halfway. You want to re-run safely without double-inserting or deleting good data.
		       			Idempotency means you can restart anytime and always end with a clean, correct state.
			>> How:
					1- Write data to a staging area first.
			   		2- Use a MERGE or UPSERT operation to update or insert based on keys.
			   		3- Commit only after full success.
			   		4- Avoid INSERT without deduplication.	


			- SQL Server:	 
			/*---------------------------------------------------------------------------------------------------------
			   	MERGE 
					dbo.SalesFact 		AS target
				USING 
					dbo.SalesFact_Stage 	AS source
				ON 
					target.OrderId = source.OrderId 
					AND 
					target.LineId = source.LineId
				WHEN MATCHED THEN 
  					UPDATE SET target.NetAmount = source.NetAmount
				WHEN NOT MATCHED BY TARGET THEN 
  					INSERT (OrderId, LineId, NetAmount) VALUES (source.OrderId, source.LineId, source.NetAmount);
			-----------------------------------------------------------------------------------------------------------*/
			- Python equiv.
			#-----------------------------------------------------------------------------------------------------------
				staging = pd.read_sql("SELECT OrderId, LineId FROM dbo.SalesFact", conn)
				new = pd.read_csv("sales_stage.csv")
				merged = pd.concat([staging, new]).drop_duplicates(subset=["OrderId","LineId"], keep="last")
				merged.to_sql("SalesFact", conn, if_exists="replace", index=False)
			#-----------------------------------------------------------------------------------------------------------


			>> Typical tools	
						- SQL MERGE or UPSERT
			   			- ADF Data Flow with “Alter Row” policy
			   			- Python logic with pandas merge and dedup




(B):	Watermark
	-----------------------------------
			>> What it means:
					A watermark is the “last-processed point in time.” It tells your next job where to start reading data again.
			>> Why:
					ERP tables and APIs often store a "LastModified" or "UpdatedAt" column.
			   		Instead of reloading all records each run, you load only the rows changed after the previous watermark.
			   		This makes incremental loads faster and cheaper.
			>> When:
					In every pipeline that reads data from a system which supports timestamps or sequence numbers.
			>> How:
					1- Store the last processed timestamp in a control table.
			   		2- On the next run, read only rows newer than that timestamp.
			   		3- Update the watermark when the run succeeds.
						
			- SQL Server	
			/*---------------------------------------------------------------------------------------------------------
				DECLARE 
					@watermark datetime2
				;

				SELECT 
					@watermark = LastRunTime 
				FROM 
					dbo.etl_watermark 
				WHERE 
					TableName='SalesHeader'
				;

				SELECT 
					* 
				FROM 
					dbo.ERP_SalesHeader
				WHERE 
					LastModified > @watermark
				;


				UPDATE 
					dbo.etl_watermark
				SET 
					LastRunTime = SYSUTCDATETIME()
				WHERE 
					TableName='SalesHeader'
				;
			-----------------------------------------------------------------------------------------------------------*/
	
			- Typical tools:
					- Table or file in the control schema (etl_watermark)
			   		- ADF pipeline parameter or variable called last_watermark
			   		- Airflow Variable if using Python orchestration




(C):	Metadata and Lineage Management
	-----------------------------------
			>> What it means:
					Metadata describes data assets. Lineage tracks their flow from source to destination.
			>> Why:
					Without metadata and lineage, we lose traceability, auditability, and business trust.
			>> When:
					Across all ingestion and transformation steps.
						
			- Python Example:

			#----------------------------------------------------------------------------------
			cursor.execute("""
					INSERT INTO 
						etl_job_log(job_name, source, target, start_time, row_count)
					VALUES 
						(?, ?, ?, SYSDATETIME(), ?)
					"""
					,("ERP_Ingest_Sales", "ERP_API", "ADLS_Bronze", len(df))
			)
			#----------------------------------------------------------------------------------

			- Typical Tools:
					- Azure Purview (catalog + lineage)
					- SQL metadata tables (etl_job_log, etl_column_map)
					- ADF integration runtime lineage tracking


			- Practical note:
					- Maintain etl_job_log and etl_watermark tables for traceability.
					- Register datasets and schemas in Purview.
					- Capture column lineage at transformation level.
					- Every BI report must trace back to its raw origin.




==============================================
2.2 Testing and CI/CD
==============================================
Definition:
Testing and CI/CD guarantee pipelines deploy safely and transformations behave predictably.

Why:
Without automated testing, errors appear late in production.
Without CI/CD, changes drift and rollback is painful.

When:
Every pipeline update or release.

- Python Example (pytest + Great Expectations):

#------------------------------------------------------------
import pandas as pd
def test_sales_not_empty():
    df = pd.read_parquet("silver/sales.parquet")
    assert len(df) > 0
#------------------------------------------------------------

- CI/CD Example:

#-----------------------------------------------------------------------------------------------------------------------
#azure-pipelines.yml
stages:
  - stage: Test
    jobs:
      - job: RunPyTests
        steps:
          - script: pytest tests/
  - stage: Deploy
    jobs:
      - job: DeployADF
        steps:
          - script: az datafactory pipeline create --factory-name weita-df --name ETL_Sales --pipeline "@pipeline.json"

#------------------------------------------------------------------------------------------------------------------------


- Practical note

Validate data after each ETL step.
Use separate Dev, Test, and Prod environments.
Deploy ADF via ARM templates or YAML pipelines.
Track pipeline version in metadata.




======================================================
3- AI Automation
This logically follows once the ETL layer is stable.
Security references
======================================================
==========================================
3.1 Implement AI and LLM-based automation
==========================================
Definition:
Adding model-driven logic (text extraction, classification, summarization) to data workflows.

Why:
To cut manual data preparation or categorize unstructured ERP inputs (e.g. supplier emails).

When:
After core ETL is stable; use on low-risk, high-volume text data.


- Python Example:

#----------------------------------------------------------------------------------
#categorize supplier message using OpenAI API
msg = "Order confirmation for item 4711"
prompt = f"Extract supplier name and intent: {msg}"
response = client.chat.completions.create(model= "gpt-4o-mini", messages=[{"role":"user", "content":prompt}])
print(response.choices[0].message.content)
#----------------------------------------------------------------------------------

- ADF equivalent:

#----------------------------------------------------------------------------------
Use a Web Activity to call an Azure Function that wraps the LLM API, logs input/output in SQL.
#----------------------------------------------------------------------------------

- Practical note:


Add regex validators for PO numbers.
Reject low confidence results.
Keep audit log of input and output.
Start with classification or routing tasks (cheap, safe).


This points matters in the AI/LLM automation part of our pipeline.
These four rules make AI in pipelines safe, testable, and auditable:

(A):	Add regex validators for PO numbers
	-----------------------------------
			>> What it means:
						Use regular expressions (regex) to verify that extracted Purchase Order (PO) numbers follow the expected pattern before you trust an LLM output.

			>> Why:
						LLMs sometimes hallucinate — they generate strings that look like data but aren’t real. Regex ensures the extracted value fits the company’s real format.

			>> When:
						Every time an LLM extracts structured fields (IDs, numbers, emails, dates).

			Example:
			PO numbers often follow patterns like PO-2025-12345.
			You validate it before inserting into SQL.		
						
			- Python Example:

			#----------------------------------------------------------------------------------
			import re
			po = "PO-2025-12345"
			if not re.match(r"^PO-\d{4}-\d{5}$", po):
    				raise ValueError("Invalid PO number")
			#----------------------------------------------------------------------------------




(B):	Reject low confidence results
	-----------------------------
			>> What it means:
						You set a confidence threshold. If the AI model’s certainty score is too low (or missing required info), you discard or route the record for manual review.

			>> Why:
						You can’t risk wrong automation on business-critical data like invoices or orders.
						It’s better to skip uncertain entries than store false information.

			>> When:
						Always when your AI or LLM returns probabilistic or heuristic results.


			- Python Example:

			#----------------------------------------------------------------------------------
			if response['confidence'] < 0.8:
    				log_to_review_queue(record_id, response)
    				continue
			#----------------------------------------------------------------------------------




(C):	Keep audit log of input and output
	----------------------------------
			>> What it means:
						Store the prompt, the AI response, and metadata (timestamp, model, user, cost, success) in a table or file.
			>> Why:
						- Trace what the AI decided later (for compliance).
						- Reproduce or debug bad outputs.
						- Calculate accuracy and cost metrics.
			>> When:
						Every production LLM call. It’s non-negotiable for traceability.

			- SQL Example:

			#----------------------------------------------------------------------------------
			CREATE TABLE dbo.llm_audit (
  							RunId uniqueidentifier,
  							InputText nvarchar(max),
  							OutputText nvarchar(max),
  							Confidence float,
  							Model varchar(50),
  							CreatedAt datetime2 default sysutcdatetime()
			);
			#----------------------------------------------------------------------------------


			- Python Example:

			#----------------------------------------------------------------------------------
			cursor.execute(
  				"INSERT INTO dbo.llm_audit(InputText,OutputText,Confidence,Model) VALUES (?,?,?,?)",
  				(prompt, result, confidence, "gpt-4o-mini")
			)
			#----------------------------------------------------------------------------------

			>> Result:
				we have a record of every model action — what went in, what came out, and how sure it was.




(D):	Start with classification or routing tasks (cheap, safe)
	--------------------------------------------------------
			>> What it means:
						Begin using LLMs for simple categorization or routing decisions before attempting complex generation tasks.
			>> Why:
						- They’re predictable and easy to verify.
						- Output is short (low token cost).
						- Easy to benchmark accuracy.
						- Fewer business risks.
			>> When:
						First phase of any AI adoption — low-risk automation (e.g., tagging, sorting, triage).

			>> Example Task:
						- Classify supplier emails: invoice / order / complaint.
						- Route internal tickets to the right department.
						- Detect product language or category.

			- Python Example:

			#----------------------------------------------------------------------------------
			prompt = f"Classify this email: {email_text}"
			category = call_llm(prompt)
			if category not in ["invoice","order","other"]:
    				category = "review"
			#----------------------------------------------------------------------------------

			>> Result:
				we get measurable, controlled automation that doesn’t endanger business data.




==============================================
3.2 Security and Access Control
==============================================
Definition:
Security covers credentials, encryption, and data access policies.

Why:
Data pipelines often touch sensitive information (financials, customers).
Compromised secrets or open access breaches compliance.

When:
Always. Security is non-optional.

- Python Example:

#------------------------------------------------------------
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

kv = SecretClient(vault_url="https://tamvault.vault.azure.net", credential=DefaultAzureCredential())
SQL_PASS = kv.get_secret("sql-password").value
#------------------------------------------------------------

- Practical note

Store secrets in Azure Key Vault or environment variables.
Encrypt data at rest (ADLS, SQL TDE) and in transit (TLS).
Apply least privilege via role-based access control (RBAC).
Mask sensitive fields (CustomerEmail, IBAN) in non-prod.
Audit login and job access with SQL Server audit or Purview.




========================================================================
4- SQL Design
design → concurrency → monitoring → safe writes → performance scaling.
========================================================================
==================================================
4.1 Design and optimize SQL databases (SQL Server)
==================================================
Definition:
Database design defines structure; optimization makes queries fast and reliable.

Why:
ERP data is large and joins are heavy — poor design kills performance.

When:
From day one of data modeling and whenever queries slow down.


- Python Example:

#----------------------------------------------------------------------------------
#run parametrized query for monitoring
import sqlalchemy as sa
query = "SELECT COUNT(*) FROM dbo.SalesFact WHERE OrderDate >= GETDATE()-1"
count = pd.read_sql(query, sql_engine)
print(count)
#----------------------------------------------------------------------------------

- SQL Server core patterns

#----------------------------------------------------------------------------------
-- cluster by date for range queries
CREATE CLUSTERED INDEX CX_SalesFact ON dbo.SalesFact(OrderDateKey);

-- filtered index on active orders
CREATE INDEX IX_Order_Active ON dbo.Orders(Status) WHERE Status='Active';

-- statistics refresh
UPDATE STATISTICS dbo.SalesFact;

#----------------------------------------------------------------------------------

- ADF equivalent:

#-------------------------------------------------------------------------------------------------------------------------------------------
Schedule "Stored Procedure activities" to run these maintenance scripts weekly.

(*) Automate SQL maintenance tasks > Weekly > Keeps DB fast & stable
	>> Meaning:
			ADF (Azure Data Factory) can run SQL Server maintenance scripts on a schedule.
			You use a "Stored Procedure" activity inside an "ADF pipeline" to execute tasks like:

			- index rebuilds
			- statistics updates
			- partition maintenance

	>> Why:
			Keeps the SQL database fast and consistent without manual intervention.

	>> When:
			Once a week or after heavy data loads (end of month, big ETL jobs).

	>> How:

			1. Create maintenance procedures inside SQL Server. Example:
			SQL:
			#----------------------------------------------------------------------------------
				CREATE OR ALTER PROCEDURE dbo.maint_rebuild_indexes 
				AS
				BEGIN
  				ALTER INDEX ALL ON dbo.SalesFact REBUILD;
  				UPDATE STATISTICS dbo.SalesFact;
				END;
			#----------------------------------------------------------------------------------

			2. In ADF, create a pipeline with one activity:
			#----------------------------------------------------------------------------------
				- Type: Stored Procedure
				- Linked service: your SQL Server
				- Procedure name: dbo.maint_rebuild_indexes
			#----------------------------------------------------------------------------------

			3. Use ADF Triggers to run it automatically every week (e.g., Sunday 02:00 CET).

	>> Outcome
			Regular maintenance → faster queries, fewer table locks, and predictable performance.
#--------------------------------------------------------------------------------------------------------------------------------------------

- Practical note:

Use partitioning by month for big tables.
Enable Read Committed Snapshot to reduce locks.
Monitor wait stats and query plans.


These four rules keep SQL Server healthy and our ETL/ADF jobs stable:
(A):	Use partitioning by month for big tables
	------------------------------------------
(*) Partitioning by month Reduce scan size for large tables > >10M rows or time-based data > Fast queries, easy archiving

			>> What it means:
					Split large fact tables (like SalesFact) into monthly chunks so SQL Server reads only what’s needed.
			>> Why:
					Improves query speed and simplifies data archiving.
					Example: queries for October read only October’s partition, not the entire table.
			>> When:
					Tables exceed ~10 million rows or when data naturally groups by time (sales, orders, invoices).

			>> How:
				1. Create a partition function and partition scheme:		
				#----------------------------------------------------------------------------------
				CREATE PARTITION FUNCTION pf_sales_date (int)
  				AS 
				RANGE RIGHT FOR VALUES (20240101,20240201,20240301,...);

				CREATE PARTITION SCHEME ps_sales_date
  				AS 
				PARTITION pf_sales_date ALL TO ([PRIMARY]);
				#----------------------------------------------------------------------------------

				2. Create or rebuild our table on that scheme:
				#----------------------------------------------------------------------------------
				CREATE TABLE dbo.SalesFact
				(OrderDateKey int, NetAmount decimal(12,2), ...)
				ON 
				ps_sales_date(OrderDateKey);
				#----------------------------------------------------------------------------------

				3. When we load data, SQL automatically writes to the right partition.

				4. We can switch out old partitions for archiving without affecting current data.

			>> Outcome:
					- Faster month-based queries
					- Easier maintenance and backup
					- Lower blocking during loads



	
(B):	Enable Read Committed Snapshot to reduce locks
	----------------------------------------------
(*) Read Committed Snapshot > Prevent ETL/report blocking > Apply when Concurrent reads & writes > No locks, stable queries

			>> What it means:
					SQL Server option that uses "row versioning" instead of "traditional locks" when reading data.
			>> Why:
					Stops readers (BI queries, reports) from blocking writers (ETL inserts/updates).
					Each query reads a consistent snapshot of the data as of the query start time.
			>> When:
					Databases with concurrent reads and writes — common in reporting systems or during ETL loads.

			>> How:
					Enable once per database:		
					#----------------------------------------------------------------------------------
					ALTER DATABASE WeitaDW SET READ_COMMITTED_SNAPSHOT ON;
					#----------------------------------------------------------------------------------

			>> Outcome:
					- ETL and BI can run in parallel.
					- No blocking or deadlocks between processes.
					- Query consistency guaranteed at minimal overhead.



	
(C):	Monitor wait stats and query plans
	----------------------------------------------
(*) Monitor wait stats & query plans > Detect root cause of slowness > Regular checks or slowdowns > Faster, predictable performance

			>> What it means:
					Check what your SQL Server is waiting on (I/O, locks, CPU) and how queries are executed.
			>> Why:
					Identifies performance bottlenecks precisely — whether it’s missing indexes, bad joins, or CPU overload.
			>> When:
					- After deployment of new ETL pipelines
					- When queries slow down unexpectedly
					- During weekly health checks
			>> How:
					1. Wait Stats – overall server delays		
					#----------------------------------------------------------------------------------
					SELECT 
						TOP 10 wait_type
						,wait_time_ms/1000 		AS seconds
					FROM 
						sys.dm_os_wait_stats
					ORDER BY 
						wait_time_ms DESC;
					/*
					If we see:
						"PAGEIOLATCH_*"	>> disk I/O issue
						"LCK_*"		>> lock contention
						"CXPACKET"	>> parallelism misbalance
					*/
					#----------------------------------------------------------------------------------

					2. Query Plan – individual slow query inspection
					#----------------------------------------------------------------------------------
					SET SHOWPLAN_XML ON;
					EXEC dbo.usp_slow_query_example;
					SET SHOWPLAN_XML OFF;

					-- OR 
					-- use SQL Server Management Studio > Query > Display Estimated Execution Plan.
					#----------------------------------------------------------------------------------

					3. Dynamic Management Views (DMVs) for recent queries
					#----------------------------------------------------------------------------------
					SELECT 
						TOP 10 total_elapsed_time/1000 		AS ms
						,execution_count
						,query_hash
					FROM 
						sys.dm_exec_query_stats
					ORDER BY 
						total_elapsed_time DESC;
					#----------------------------------------------------------------------------------					

			>> Outcome:
					We know why a job is slow — and we fix the cause, not the symptom.

(D):	Safe Table Updates (no full replacements)
	----------------------------------------------
(*) 
			>> What it means:
					Production tables must be updated safely through staging + MERGE, not by full replacement.
					(Replacing full tables is destructive. Always stage, merge, and commit.)
			>> Why:
					.to_sql(if_exists="replace") deletes structure and causes downtime.	
			>> When:
					Anytime new data is loaded into persistent tables.	

			>> How:
					1. Load data into staging table.
					2. MERGE into target by business keys.
					3. Commit after validation.
					4. python Example:
					#----------------------------------------------------------------------------------
					staging = pd.read_sql("SELECT * FROM dbo.SalesFact", conn)
					new = pd.read_parquet("silver/sales.parquet")
					merged = pd.concat([staging, new]).drop_duplicates(subset=["OrderId","LineId"], keep="last")
					merged.to_sql("SalesFact_Stage", conn, if_exists="replace", index=False)
					conn.execute("""
							MERGE 
								dbo.SalesFact AS tgt
							USING 
								dbo.SalesFact_Stage AS src
							ON 
								tgt.OrderId=src.OrderId 
								AND 
								tgt.LineId=src.LineId
							WHEN 
								MATCHED 
							THEN UPDATE SET 
								tgt.NetAmount=src.NetAmount
							WHEN NOT 
								MATCHED BY TARGET 
							THEN INSERT (...) VALUES (...)
							;
						""")
					#----------------------------------------------------------------------------------


			>> Practical note:

					Never use replace in production.
					Always apply staging and MERGE.
					Validate row counts before commit.



(E):	ADF Performance Optimization
	----------------------------------------------
(*) 
			>> What it means:
					ADF performance tuning ensures scalability for large datasets.					
			>> Why:
					Default Copy activity is fine for small volumes but bottlenecks at scale.
			>> When:
					Tables >10M rows or files >5 GB.
			>> How:
					- ADF Example:
					#----------------------------------------------------------------------------------
					{
  					  "type": "Copy",
  					  "source": {"type": "AzureSqlSource", "query": "SELECT * FROM dbo.SalesFact WHERE OrderDateKey>=20250101"},
  					  "sink": {"type": "AzureBlobSink", "enableStaging": true, "parallelCopies": 8}
					}
					#----------------------------------------------------------------------------------

			>> Practical note:
					Use PolyBase or Staging Copy for bulk loads.
					Enable parallel reads on partitioned data.
					Push transformations to Data Flows or SQL stored procedures.
					Avoid row-by-row logic in ADF.
					Monitor Integration Runtime (IR) performance and cost metrics.
			>> Outcome:
					Reduced runtime, predictable cost.



================================
5- Data Consolidation
Final lineage, exposure
The gold layer and consumption step.
================================
==================================================
5. Support data consolidation and analytics
==================================================
Definition:
Merging data from ERP and side systems into a consistent schema for BI and AI.

Why:
Business teams need one source of truth across sales, inventory, finance.

When:
After stable ETL and model alignment.

- Python Example:

#----------------------------------------------------------------------------------
#join ERP sales with supplier data for analysis
sales = pd.read_sql("SELECT * FROM dbo.SalesFact", sql_engine)
sup = pd.read_sql("SELECT * FROM dbo.DimSupplier", sql_engine)
merged = sales.merge(sup, on="SupplierId", how="left")
merged.to_sql("gold_sales_supplier", con=sql_engine, if_exists="replace", index=False)
#----------------------------------------------------------------------------------

- SQL Server equivalent

#----------------------------------------------------------------------------------
SELECT 
	s.OrderId
	,s.ProductId
	,p.Category
	,c.Region
FROM 
	dbo.SalesFact 	s
JOIN 
	dbo.DimProduct 	p 
ON 
	s.ProductKey=p.ProductKey
JOIN 
	dbo.DimCustomer c 
ON 
	s.CustomerKey=c.CustomerKey
;
#----------------------------------------------------------------------------------


- Practical note:

Keep conformed dimensions (Customer, Product, Date).
Use SCD2 for history tracking.
Publish views with clear grain (1 row per order line).
Expose data via Power BI datasets or APIs.
Catalog every Gold dataset in the metadata system.
Maintain lineage from raw to Gold for audit and reproducibility. 

These four rules make our Gold layer trustworthy, auditable, and consumable:

(A):	Keep conformed dimensions (Customer, Product, Date).
	-------------------------------------------------------
(*) 
			>> What it means:
					A conformed dimension is a shared reference table (e.g. Customer, Product, Date) 
					that is used consistently across all fact tables (Sales, Orders, Returns, etc.).
			>> Why:
					It guarantees all reports use the same meaning for key entities.
					Example: “Customer 123” means the same in SalesFact and InvoiceFact.					
					
			>> When:
					Whenever you integrate data from multiple ERP modules or side systems.
			>> How:
					- Create one "DimCustomer", "DimProduct", "DimDate", and reuse their keys across all fact tables.
					- Map ERP business keys (e.g., "CustomerNumber") to surrogate keys in these dimensions.
					- Apply consistent attributes (region, type, active flag).
					- SQL Example:
					#----------------------------------------------------------------------------------
					-- Conformed Customer Dimension
					CREATE TABLE dbo.DimCustomer (
  						CustomerKey 		int IDENTITY PRIMARY KEY
  						,CustomerNumber 	varchar(50)
  						,Name 			nvarchar(200)
  						,Region 		varchar(50)
  						,ValidFrom 		datetime2
  						,ValidTo 		datetime2
  						,CurrentFlag 		bit
					);		
					#----------------------------------------------------------------------------------
			>> Outcome:
					Cross-report comparisons (sales vs invoices vs returns) line up perfectly.

	
(B):	Use SCD2 for history tracking.
	-------------------------------------------------------
(*) 
			>> What it means:
					SCD2 = Slowly Changing Dimension Type 2.
					It keeps a full history of attribute changes (e.g. customer moved region).
			>> Why:
					Analysts need to know what was true at the time of the transaction, not only the latest state.
			>> When:
					For any dimension where attributes change over time: customer region, product category, supplier status.
			>> How:
					- Each change creates a new row with new "ValidFrom" and "ValidTo".
					- The current record has "ValidTo" = '9999-12-31' and "CurrentFlag" = 1.
					- SQL Example:
					#----------------------------------------------------------------------------------
					MERGE 
						dbo.DimCustomer 			AS tgt
					USING 
						(SELECT * FROM stg_DimCustomer) 	AS src
					ON 
						tgt.CustomerNumber = src.CustomerNumber 
						AND 
						tgt.CurrentFlag = 1
					WHEN 
						MATCHED AND (tgt.Name <> src.Name OR tgt.Region <> src.Region)
  					THEN UPDATE SET 
						tgt.ValidTo = SYSUTCDATETIME()
						,tgt.CurrentFlag = 0
					WHEN NOT 
						MATCHED BY TARGET
  					THEN INSERT 
						(CustomerNumber, Name, Region, ValidFrom, ValidTo, CurrentFlag)
       					VALUES 
						(src.CustomerNumber, src.Name, src.Region, SYSUTCDATETIME(), '9999-12-31', 1)
					;
					#----------------------------------------------------------------------------------
			>> Outcome:
					Historical queries stay correct — you can see sales under the old region vs new one.					

	
(C):	Publish views with clear grain (1 row per order line).
	-------------------------------------------------------
(*) 

			>> What it means:
					Grain means the exact level of detail each row represents.
					“1 row per order line” = each record corresponds to one ERP line item, not a header or summary.
			>> Why:
					Without a clear grain, aggregates double-count or undercount.
					Grain defines how facts relate to dimensions.
			>> When:
					For all fact tables and Power BI datasets.
			>> How:			
					- Decide the lowest business transaction unit (e.g., line item).
					- Join only once to each dimension via its surrogate key.
					- Build a SQL view that enforces this grain and hides complexity.
					- SQL Example:
					#----------------------------------------------------------------------------------
					CREATE VIEW vw_SalesFact 
					AS
					SELECT 
  						s.OrderId
  						,s.LineId
  						,s.CustomerKey
  						,s.ProductKey
  						,s.OrderDateKey
  						,s.Quantity
  						,s.NetAmount
					FROM 
						dbo.FactSales s
					;
					#----------------------------------------------------------------------------------
			>> Outcome:
					Each report knows what one record means; measures like SUM(NetAmount) aggregate cleanly.



(D):	Expose data via Power BI datasets or APIs.  
	-------------------------------------------------------
(*) 

			>> What it means:
					Make the cleaned, modeled data available for use — either in Power BI (semantic model) 
					or through APIs for external tools.
			>> Why:
					Final business consumption layer. Ensures analysts and systems access governed data, not raw tables.
			>> When:
					After the Gold layer is validated and approved.
			>> How:			
					- Power BI:
						- Connect Power BI Desktop to the Gold database or Fabric Warehouse.
						- Build a semantic model using star schema (facts + conformed dims).
						- Publish to Power BI Service > create dataset.
						- Manage access via Azure AD groups.
	
					- API / Data Service:
						- Use REST API or Azure Functions to serve JSON endpoints for system integrations.
						- Secure with Azure AD / Managed Identity.

			>> Outcome:
					- Data consumers use one controlled source:
						- Business users	> Power BI dashboards
						- Other systems 	> REST endpoints



(E):	Metadata and Lineage in the Gold Layer
	-------------------------------------------------------
			>> Every published dataset must include:
						- Table and view registration in the catalog (e.g., Azure Purview).
						- Source-to-target mapping in a lineage table (etl_lineage_map).
						- Version, owner, and last refresh timestamp.
						- Stored SQL view definitions for full traceability.

			>> Outcome:
					The Gold layer becomes auditable, reusable, and ready for enterprise analytics.
					Every report can trace its data back to origin, through all ETL and SQL steps.



==============================================
6. Security and Access Control
==============================================
Definition:
Security governs how credentials, data, and systems are protected across all stages of the data platform.

Why:
ETL and analytics pipelines handle sensitive data such as financial records and personal details.
Without strict security, you risk data leaks, compliance breaches, and unauthorized access.

When:
Always active — applied from ingestion to analytics.

Scope:
	Section 1: Protect API keys and connection strings for ingestion.
	Section 2: Secure ETL pipelines and data logs.
	Section 4: Enforce RBAC on SQL Server and ADF.
	Section 5: Mask or anonymize sensitive data before publishing analytics.

- Python Example:

#------------------------------------------------------------
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

vault = SecretClient(vault_url="https://tco-keyvault.vault.azure.net", credential=DefaultAzureCredential())
SQL_PASS = vault.get_secret("sql-password").value
#------------------------------------------------------------

- SQL Server Example (Encryption & Roles):

#------------------------------------------------------------
-- Enable Transparent Data Encryption
ALTER DATABASE WeitaDW SET ENCRYPTION ON;

-- Create and assign role
CREATE ROLE etl_runner;
GRANT SELECT, INSERT, UPDATE ON dbo.SalesFact TO etl_runner;
EXEC sp_addrolemember 'etl_runner', 'ADF_ServiceAccount';
#------------------------------------------------------------

- ADF Example (Key Vault Linked Service):

#------------------------------------------------------------
{
  "type": "AzureKeyVault",
  "typeProperties": {
    "baseUrl": "https://tco-keyvault.vault.azure.net/"
  }
}
#------------------------------------------------------------


- Practical note

Store credentials in Azure Key Vault or environment variables, never in code.
Encrypt data at rest (Transparent Data Encryption, ADLS encryption) and in transit (TLS/HTTPS).
Apply Role-Based Access Control (RBAC) at every service level (SQL, ADF, Power BI).
Limit access by principle of least privilege.
Mask or pseudonymize customer fields in non-production environments.
Log all access and configuration changes for audits.


- Outcome:
Credentials stay protected, sensitive data is never exposed, and every operation can be traced to a secure identity.
The platform becomes compliant, auditable, and resilient against unauthorized access.






==============================================
7. Appendix: Documentation and Standards
==============================================
End with formatting, naming, and version control conventions.


| Step | Layer                 | Embedded Topics                   |
| ---- | --------------------- | --------------------------------- |
| 0    | Platform Foundation   | Documentation, repo, standards    |
| 1    | Central Data Platform | Error handling & retries          |
| 2    | ETL Pipelines         | Metadata, lineage, testing, CI/CD |
| 3    | AI Automation         | Security references               |
| 4    | SQL Design            | Safe updates, ADF performance     |
| 5    | Data Consolidation    | Final lineage, exposure           |
| 6    | Security & Access     | Global policies                   |
| 7    | Appendix              | Formatting and governance docs    |










