# MongoDB-Databricks

## 1. Setup your Atlas cluster 

### Create Your MongoDB Atlas Cluster
If you already have a MongoDB Atlas account, log in and create a new Atlas cluster. If you do not have an account, you can set up a free cluster at the following URL: https://www.mongodb.com/cloud. Once your account is set up, you can create a new Atlas cluster by using the “+ New Cluster” dialog. MongoDB provides a free tier for Google Cloud.
![Create Cluster](/images/clustercreation.png)
Once you provide a cluster name and click on “create,” Atlas will take approximately five to seven minutes to create your Atlas cluster.

### Define Database Access
By default there are no users created in an Atlas cluster. To create an identity for our Spark cluster to connect to MongoDB Atlas, launch the “Add New Database User” dialog from the Database Access menu item.
![Database Access](/images/databaseaccess.png)

Notice that there are three options for authentication to MongoDB Atlas: Password, Certificate, and AWS IAM authentication. Select “Password,” and enter a username and password. Atlas provides granular access control: For example, you could restrict this user account to work only with a specific Atlas cluster or define the account as temporary and have Atlas expire within a specific time period.

### Defining Network Access
MongoDB Atlas does not allow any connection from the internet by default. You need to include MongoDB Atlas as part of a VPC peering or AWS PrivateLink configuration. If you do not have that set up with your cloud provider, you need to specify from which IP addresses Atlas can accept incoming connections. You can do this via the “Add IP Address” dialog in the Network Access menu. In this article, we will add “0.0.0.0,” allowing access from anywhere, because we don’t know specifically which IP our Databricks cluster will be running on.

![Network Access](/images/networkaccess.png)
MongoDB Atlas can also make this IP access list temporary, which is great for situations where you need to allow access from anywhere.

### Add Sample Data
Now that we have added our user account and allowed network access to our Atlas cluster, we need to add some sample data. Atlas provides several sample collections that are accessible from the menu item on the cluster.
![Load Data](/images/loaddata.png)


In this example, we will use the sales collection within the sample_supplies database.

![Connection String](/images/connstring.png)
Copy the MongoDB Atlas connection string by clicking on the Connect button and selecting “Connect your application.” Copy the contents of the connection string and note the placeholders for username and password. You will have to change those to your own credentials.

## 2. Analytics with MongoDB Aggregation Framework

Let us consider the collection sample_supplies.sales from the sample dataset. Sample document below:

![Aggregation Pipeline](/images/aggpipeline.png)

Our aggregation pipeline builder provides easy interface to build the aggregation pipelines stage by stage:

![Aggregation Pipeline Builder](/images/aggbuilder.png)

## 3. Visualize using MongoDB Charts

### Add the Data Collection as a Data Source
In MongoDB Atlas, click on Charts on the left side
1. Click the Data Sources tab
2. Click New Data Source
3. Select your Atlas Deployment in your project
4. Click Connect
5. Select the sample_mflix.movies collection
6. Click Set Permissions, leave the permissions as the default
7. Click Publish Data Source

### Create a New Dashboard
1. Click the Dashboards tab
2. Click the New Dashboard button
3. Enter the Title: Movie Details
4. Click Create

### Create Chart Showing Directors with the Most Awards
1. Click Add Chart
2. In the Data Source dropdown, select sample_supplies.sales
3. In the Chart Type dropdown, select Column.
4. Drag the storeLocation property from the Fields section of the Chart Builder view to the X Axis encoding channel. This tells MongoDB Charts to create a column for each storeLocation value in the dataset.
5. In the Fields section click the items field to expand the items object and view its properties.
6. Drag the items.price field to the Y Axis encoding channel. The Y Axis encoding channel determines which field to use for the chart's aggregation.
7. In the Array Reductions dropdown, select Unwind array.
8. In the Aggregate dropdown, select sum.

![Chart](/images/chart.png)

## 4. Analytics, AI/ML using Databricks Notebook

Provision a new Databricks workspace in your preferred cloud provider. 

Once your Databricks cluster is created, navigate to the Databricks cluster with the URL provided. Here you can create a new workspace.
![New Workspace](/images/databrickssetup1.png)

Once you’ve created your workspace, you will be able to launch it from the URL provided:

![Launch](/images/databrickssetup2.png)

Logging into your workspace brings up the following welcome screen:

![welcome](/images/welcome.png)

In this article, we will create a notebook to read data from MongoDB and use the PySpark libraries to perform the rolling average calculation. We can create our Databricks cluster by selecting the “+ Create Cluster” button from the Clusters menu.

![Create Cluster](/images/createcluster.png)

Note: For the purposes of this walkthrough we chose only one worker and preemptible instances; in a production environment you would want to include more workers and autoscaling.
Before we create our cluster, we have the option under Advanced Options to provide Spark configuration variables. One of the common settings for Spark config is to define spark.mongodb.output.uri and spark.mongodb.input.uri. Paste the URI you copied in the previous step (Setting up Atlas environment). 

Under Advanced Options in your Databricks workspace, paste the connection string for both the spark.mongodb.output.uri and spark.mongodb.input.uri variables. Note that you will need to update the credentials in the MongoDB Atlas connection string with those you defined previously. For simplicity in your PySpark code, change the default database in the connection string from MyFirstDatabase to sample_supplies. (This is optional, because you can always define the database name via Spark configuration options at runtime.)

![Advanced Options](/images/advancedoptions.png)

### Start the Databricks Cluster
Now that your Spark config is set, start the cluster.
Note: If the cluster fails to start, check the event log and view the JSON tab. This is an example error message you will receive if you forgot to increase the SSD storage quota:
![Start Cluster](/images/startcluster.png)

### Add MongoDB Spark Connector
Once the cluster is up and running, click on “Install New” from the Libraries menu.
![Add Spark Connector](/images/addconnector.png)

Here we have a variety of ways to create a library, including uploading a JAR file or downloading the Spark connector from Maven. In this example, we will use Maven and specify org.mongodb.spark:mongo-spark-connector_2.12:3.0.1 as the coordinates.

![Add Spark Connector](/images/addconnector1.png)

Click on “Install” to add our MongoDB Spark Connector library to the cluster.
Note: If you get the error message “Maven libraries are only supported on Databricks Runtime version 7.3 LTS, and versions >= 8.1,” you can download the MongoDB Spark Connector JAR file from https://repo1.maven.org/maven2/org/mongodb/spark/mongo-spark-connector_2.12/3.0.1/ and then upload it to Databricks by using the Upload menu option.

![Add Spark Connector](/images/addconnector2.png)

### Create a New Notebook
Click on the Databricks home icon from the menu and select “Create a blank notebook.”
![Create Notebook](/images/createnotebook.png)

Attach this new notebook to the cluster you created in the previous step.
![Attach Notebook](/images/attachnotebook.png)

Because we defined our MongoDB connection string as part of the Spark conf cluster configuration, your notebook already has the MongoDB Atlas connection context.
In the first cell, paste the following:

```
from pyspark.sql import SparkSession

pipeline="[{'$match': { 'items.name':'printer paper' }}, {'$unwind': { path: '$items' }}, {'$addFields': { totalSale: { \
	'$multiply': [ '$items.price', '$items.quantity' ] } }}, {'$project': { saleDate:1,totalSale:1,_id:0 }}]"

salesDF = 
spark.read.format("mongo").option("collection","sales").option("pipeline", pipeline).option("partitioner", "MongoSinglePartitioner").load()
```

Run the cell to make sure you can connect the Atlas cluster.
Note: If you get an error such as “MongoTimeoutException,” make sure your MongoDB Atlas cluster has the appropriate network access configured.

![Notebook Result](/images/notebookresult1.png)

The notebook gave us a schema view of what the data looks like. Although we could have continued to transform the data in the Mongo pipeline before it reached Spark, let’s use PySpark to transform it. Create a new cell and enter the following:

```
from pyspark.sql.window import Window

from pyspark.sql import functions as F

salesAgg=salesDF.withColumn('saleDate',
F.col('saleDate').cast('date')).groupBy("saleDate").sum("totalSale").orderBy("saleDate")

w = Window.orderBy('saleDate').rowsBetween(-7, 0)

df = salesAgg.withColumn('rolling_average', 
F.avg('sum(totalSale)').over(w))

df.show(truncate=False)
```
Once the code is executed, the notebook will display our new dataframe with the rolling averages column:

![Notebook Result 2](/images/notebookresult2.png)

It is this cell where we will provide some additional transformation of the data such as grouping the data by saleDate and provide a summation of the totalSale per day. Once the data is in our desired format, we define a window of time as the past seven entries and then add a column to our data frame that is a rolling average of the total sales data.
Once we have performed our analytics, we can write the data back to MongoDB for additional reporting, analytics, or archiving. In this scenario, we are writing the data back to a new collection called sales-averages:

```
df.write.format("mongo").option("collection","sales-averages").save()
```
![Notebook Result 3](/images/notebookresult3.png)









