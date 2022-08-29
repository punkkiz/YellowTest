# YellowTest
Test task based on Yellow Cab dataset


This data engineering solution performs a few simple steps in order to make a usable dataset from a .parquet file, provided here: 
https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page  


Azure workspace template to create all resources can be found here: 


Synapse resources are linked to this repository 


Pre-Requisites: 

Azure account 
Synapse analytics workspace (template added) 
SQL Serverless Pool 
Apache pool
Azure storage blob

So what does this do and how it works? 

Overview: 

Synapse Analytics solution picks up the file, splits it by day and after performing some ETL steps stores it in user accessable views in SQL Serverless pool. However user experience will be the same as using SQL server 

Orchestration - in order to perform tasks in specific sequence additional layer of 'master pipeline' was addedd. It makes it easier to controll the data loading process. 

Step 1: 

Data file is manually loaded into a storage account:

1. Folders

![FilePickFolders](https://user-images.githubusercontent.com/112269784/187184213-732dd051-8e4e-4775-8451-01f4bfe5b8e7.PNG)

2. File

![FilePick](https://user-images.githubusercontent.com/112269784/187184143-dc9f7ee1-82dc-40bd-9315-d7214f9f1bef.PNG)


From here a "master' pipeline should be triggered (at the moment manually, however if I would have time for this - a trigger on file appearing in the incoming folder would be nice :) ) 

Step 2

Master package: 

![Msater](https://user-images.githubusercontent.com/112269784/187184513-302cebf2-26e5-475d-9d04-dc420547fcfb.PNG)


Master executes whole data loading process in sequence as follows: 

1. Split and Load

![SplitNLoadCapture](https://user-images.githubusercontent.com/112269784/187185831-962f1cbf-3198-432d-998d-338bc5d01144.PNG)

This executes python notebook which picks up the file and partitions it by required column, stores the files in 'staging' folder for later pickup.

IMPORTANT - code is far from ideal, filename is hardcoced. A for-each loop should be used here which would just itterate through the folder, picking any parquet files which meets the file structure/filename criteria

https://github.com/punkkiz/YellowTest/blob/main/notebook/Pick_and_partition.json 

python code:

%%pyspark
import pyspark.sql	
from pyspark.sql.functions  import *

df = spark.read.load('abfss://yellow@yellow2.dfs.core.windows.net/Incoming/yellow_tripdata_2022-04.parquet', format='parquet')

RR= df.withColumn("test", to_date(col("tpep_dropoff_Datetime"))).write.partitionBy('test').format("parquet").save('abfss://yellow@yellow2.dfs.core.windows.net/Staging', mode="append")



2. Load data

 ![LoadData](https://user-images.githubusercontent.com/112269784/187187313-4d3a7780-7bc7-4b82-b67d-527f49e073e9.PNG)

Uses Integrated Dataset, using ADSL Gen 2 storrage connector to pick up the 'Staging' folder, selects only needed columns (additional one used for file partitioning is dropped) 

IMPORTANT: due to lack of time no data quality solution was applied. However two variables are ready to use and push into additional table for statistics. 
If i would create such solution for real I would create a separate table in SQL Pool called 'stats' or sth, info to push there: 

package ID, name, start time, end time, run status, rows incoming, rows written



3. Dimensions 

Next 3 packages are almost identical, they load missing dimensions (hardcode) and assigns a surogate key for later use 

![DIM](https://user-images.githubusercontent.com/112269784/187188089-d8c44193-f3e0-48da-9947-17df2a579afc.PNG)


4. Fact table load 

![FactSTG2](https://user-images.githubusercontent.com/112269784/187188308-a3aa8c41-c11d-46ae-9b20-b40980fe0758.PNG)

In order to prepare the data model, we need to prepare some stuff. One of which is surogate keys. Data itself is quite tidy however, many data quality checks can be performed at this step: data conversion errors, row counts after each operation etc. Data quality checks not implemented due lack of time, and usually it's a secondary goal of the project :) 

However - this dataflow looks up the new surogate keys, mentioned earlier, and applies this data modeling techique onto the fact table.

5. Data is ready to use in SQL views: 

![image](https://user-images.githubusercontent.com/112269784/187189159-ae5e1572-b39e-48c5-874f-fcd3a6572528.png)




Database structure

Data loading has 2 'stages' 

1 stage - raw data. So facts come in, as they are in files, lands in dbo.fact. Same for dim tables, they are hardcoded to 'simulate' raw data feed into dbo schema

2 stage - after ETL - stg2 schema is used here. data is pretty much ready to use 

End user - views - views are way better user access solution as many operations as column names can be easily maintained without breaking too much stuff 




Improvements:

   As mentioned above, file pickup can be re-build to be more flexible and accept multiple files, store the files more efficiently. 
   Currently dimensions are hardcoded, ideally I would like to receive separate tables from source systems or managed master data solutions where dimension definitions are managed by key business users or sth.
   Data quality checks - not implemented. As mentioned above, separate dataflow/pipeline parameters can be outputed to a separate logging table to check row error counts, maymbe even re-direct errored rows to a separate table... all depends on the need. 
   Refresh scheduling - cureently everything is manual. Trigger on file arrival should work sort of OK in this scenario. If pickup script would be updated to pick only new files, and move files which are already in the DB to separate archive folder or something similar. 
   



