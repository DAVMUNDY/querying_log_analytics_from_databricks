# querying_log_analytics_from_databricks
Example of using the spark-kusto-connector in databricks to query Log Analytics

<br />
<br />

If you're looking to query Log Analytics (Azure Monitor Logs) from a Databricks notebook unfortuantely we cannot use an option similar to this Jupyter notebook example:<br />
<br />
https://github.com/Azure/azure-kusto-analytics-lib/blob/master/KqlMagic/Getting-Started-With-KqlMagic-on-ADX.ipynb <br />
<br />
...because KQL magic is not suppored in Azure Databricks notebooks :( <br />
So you could use tools such as Azure Notebook Service, Azure Data Studio.. however if you still want to use Databricks then we can run queries through the spark-kusto-connector.

<br />
<br />
<br />

### Steps ###

1) Create an ADX cluster and Database - https://docs.microsoft.com/en-us/azure/data-explorer/create-cluster-database-portal
2) Create a spark cluster and install the spark-kusto-connector - https://docs.microsoft.com/en-us/azure/data-explorer/spark-connector#spark-cluster-setup Note: the latest DB runtime worked fine for me
3) Make sure your SPN has read access to the ADX workspace and Log Analytics workspace
4) Set up your databricks commands - example below. 
    

```python
from pyspark.sql import SparkSession

# COMMAND ----------

# Optional:
sc._jvm.com.microsoft.kusto.spark.utils.KustoDataSourceUtils.setLoggingLevel("all")

# COMMAND ----------
pyKusto = SparkSession.builder.appName("kustoPySpark").getOrCreate()
kustoOptions = {"kustoCluster":"https://<ADX_workspace>.<region>.kusto.windows.net", "kustoDatabase" : "adxdb_name", "kustoQuery" : "cluster('https://ade.loganalytics.io/subscriptions/<subscription_id>/resourcegroups/<resourcegroup_name>/providers/microsoft.operationalinsights/workspaces/<loganalytics_workspace_name>).database('<loganalytics_workspace_name>').Heartbeat | summarize count() by OpType" , "kustoAadAppId":"<AppID>" , "kustoAadAppSecret":"<AppSecret>", "kustoAadAuthorityID":"<tenantID>"}

# COMMAND ----------


# Read the data from the kusto table with default reading mode
kustoDf  = pyKusto.read. \
            format("com.microsoft.kusto.spark.datasource"). \
            option("kustoCluster", kustoOptions["kustoCluster"]). \
            option("kustoDatabase", kustoOptions["kustoDatabase"]). \
            option("kustoQuery", kustoOptions["kustoQuery"]). \
            option("kustoAadAppId", kustoOptions["kustoAadAppId"]). \
            option("kustoAadAppSecret", kustoOptions["kustoAadAppSecret"]). \
            option("kustoAadAuthorityID", kustoOptions["kustoAadAuthorityID"]). \
            load()


kustoDf.show()
```


* We connect first to the ADX database.. (so that needs to be up and running!) but in the query we actually reference the log analytics workspace. Note the log analytics workspace and database will share the same name. If you want to test connectivity you can follow this example to connect to a log analytics database in the ADX UI https://docs.microsoft.com/en-us/azure/data-explorer/query-monitor-data 
* In my example I queried a Heartbeat table in log analytics.
* Although I haven't done in my example, please use secrets to protect your Azure AD APP ID and Key 


<br />
<br />


### Considerations ###
* I believe this feature is using the ADX proxy option to facilitate connection as mentioned here - https://docs.microsoft.com/en-us/azure/data-explorer/query-monitor-data This is currently in preview and not GA.
* For large data volumes it might be better to do an export of the data from Log Analytics into storage and then ingest into ADX..

<br />

### References ###
Examples - https://github.com/Azure/azure-kusto-spark/blob/master/samples/src/main/python/pyKusto.py <br />
Docs - https://docs.microsoft.com/en-us/azure/data-explorer/connect-from-databricks <br />
Docs - https://docs.microsoft.com/en-us/azure/data-explorer/spark-connector#azure-ad-application-authentication <br />
