# ANT321 - Tiered Data Sets in Amazon Redshift

Amazon Redshift offers a common query interface against data stored in fast, local storage as well as data from high-capacity, inexpensive storage (S3). This workshop will cover the basics of this tiered storage model and outline the design patterns you can leverage to get the most from large volumes of data. You will build out your own Redshift cluster with multiple data sets to illustrate the trade-offs between the storage systems. By the time you leave, you’ll know how to distribute your data and design your DDL to deliver the best data warehouse for your business.


```
Matt Scaer
Principal DW Specialist SA
AWS

Karthik Odapally
Senior Solution Architect
AWS
```


## License Summary

This sample code is made available under a modified MIT license. See the LICENSE file.

## Agenda
* Introductions
* Account Logins and Cluster Spin-up (Via Cloudformation)
* Refresher on Amazon Redshift
* Workshop time
	* Learning Objectives
	* True-up against the website

## Why this session
* Data typically grows at 10x every 5 years.
* Average lifetime for an Analytics Platform is 15yrs. 
* Not just price and performance but also complexity. 

## Why you really need this session

* To learn how simple it is:
	* To query 2.87 billion rows (200Gb’s of data) in <5 seconds.
	* Query historical data residing on S3.
	* Plan for the future.

## Account Login and Redshift Cluster Spin-up 
* Log into AWS using the provided placeholder credentials, then switch the us-west-2.
* Create an IAM role to query S3 data, giving the role read-only access to all Amazon S3 buckets.

````
https://docs.aws.amazon.com/redshift/latest/dg/c-getting-started-using-spectrum-create-role.html 
````

* Use Redshift’s ‘Quick Create’ functionality (or “Classic”) to create a cluster and associate the IAM role with it.
	* Please use **2 compute nodes of DC2.Large**, using a cluster identifier, and master user of your choice.
	* **Do not pick an AZ**


<details><summary>How-to Screenshot</summary>
<p>
![GitHub Logo](/images/redshift_launch.png) 
</p>
</details>

## Refresher on Amazon Redshift

![GitHub Logo](/images/redshift_arch.png)

* Massively parallel, shared nothing columnar architecture
* Leader Node:
	* SQL Endpoint
	* Stores Metadata
	* Coordinates parallel SQL Processing
* Compute Node:
	* Local, columnar storage
	* Executes queries in parallel
	* Load, unload, backup, restore
* Amazon Redshift Spectrum nodes
	* Execute queries directly against 
	* Amazon Simple Storage Service (Amazon S3)

### Two Complimentary Usage Patterns

Amazon Redshift combines two usage patterns under a single, seamless service:

* Redshift (using direct-attached storage objects):
	* Billed hourly for the number and type of compute nodes in the cluster.
	* An “all you can eat” model.
* Redshift Spectrum (table data resides on S3):
	* Billed at $5 per TB of data scanned.
	* Both performance and cost-savings incent reducing the amount of data scanned through:
	* Compressing the data on S3.
	* Storing the data on S3 in a columnar format (eg. Parquet or ORC).
	* Partitioning the data on S3

**Amazon Redshift can be leveraged using the patterns either solely, or in combination.**

### Connecting to the Cluster

* Existing tools like SQL Workbench can be used. 
* Amazon Redshift has a built-in Query Editor via the AWS Management console. 

	![GitHub Logo](/images/query_editor.png)

	and you can type in the following query to get the ball-rolling

	````
	SELECT 'Hello Redshift Tiered-Storage workshop attendee!' AS welcome;
	````
* PgWeb -> Pgweb is a web-based database browser for PostgreSQL, written in Go and works on OSX, Linux and Windows machines. Feel free to download this runtime-client from here:

	````
	https://github.com/sosedoff/pgweb/releases
	````
* (optional) Command Line Interface (CLI) for Amazon Redshift.
	
	````
	https://docs.aws.amazon.com/redshift/latest/mgmt/setting-up-rs-cli.html
	````
## Workshop - Scenario #1: What happened in 2016?

* Assemble your toolset:
	* Choosing a SQL editor (SQL Workbench, PGWeb, psql, query from Console, etc.) 

* Load the Green company data for January 2016 into Redshift direct-attached storage (DAS) with COPY.
* Collect supporting/refuting evidence for the impact of the January, 2016 blizzard on taxi usage.
* The CSV data is by month on Amazon S3. Here's a quick screenshot via the CLI: 
	````
	$ aws s3 ls s3://us-west-2.serverless-analytics/NYC-Pub/green/ | grep 2016
	````
	
	![GitHub Logo](/images/s3_ls.png)

* Here's Sample data from the January File:

	````
	head -20 green_tripdata_2016-01.csv 
VendorID,lpep_pickup_datetime,Lpep_dropoff_datetime,Store_and_fwd_flag,RateCodeID,Pickup_longitude,Pickup_latitude,Dropoff_longitude,Dropoff_latitude,Passenger_count,Trip_distance,Fare_amount,Extra,MTA_tax,Tip_amount,Tolls_amount,Ehail_fee,improvement_surcharge,Total_amount,Payment_type,Trip_type 
	````

	![GitHub Logo](/images/jan_file_head.png)
	
### Build you DDL 
- Create a schema `workshop_das` and table `workshop_das.green_201601_csv` for tables that will reside on the Redshift compute nodes, AKA the Redshift direct-attached storage (DAS) tables.

	<details><summary>Hint</summary>
	<p>
	
	````
	CREATE SCHEMA workshop_das;

	CREATE TABLE workshop_das.green_201601_csv 
	(
	  vendorid                VARCHAR(4),
	  pickup_datetime         TIMESTAMP,
	  dropoff_datetime        TIMESTAMP,
	  store_and_fwd_flag      VARCHAR(1),
	  ratecode                INT,
	  pickup_longitude        FLOAT4,
	  pickup_latitude         FLOAT4,
	  dropoff_longitude       FLOAT4,
	  dropoff_latitude        FLOAT4,
	  passenger_count         INT,
	  trip_distance           FLOAT4,
	  fare_amount             FLOAT4,
	  extra                   FLOAT4,
	  mta_tax                 FLOAT4,
	  tip_amount              FLOAT4,
	  tolls_amount            FLOAT4,
	  ehail_fee               FLOAT4,
	  improvement_surcharge   FLOAT4,
	  total_amount            FLOAT4,
	  payment_type            VARCHAR(4),
	  trip_type               VARCHAR(4)
	)
	DISTSTYLE EVEN SORTKEY (passenger_count,pickup_datetime);

	````
	</p>
	</details>

### Build your Copy Command 

Build your copy command to copy the data from Amazon S3. This dataset has the number of taxi rides in the month of January 2016.

	`s3://us-west-2.serverless-analytics/NYC-Pub/green/green_tripdata_2016-01.csv`

<details><summary>Hint</summary>
<p>

````
COPY workshop_das.green_201601_csv
FROM 's3://us-west-2.serverless-analytics/NYC-Pub/green/green_tripdata_2016-01.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole'
DATEFORMAT 'auto'
IGNOREHEADER 1
DELIMITER ','
IGNOREBLANKLINES
;
````
**HINT HINT: The `XXXXXXXXXXXX` in the above command should be your AWS account number and Role information.**

</p>
</details>	

### Pin-point the Blizzard

In this month, there is a date which had the lowest number of taxi rides due to a blizzard. Can you find that date?

<details><summary>Hint</summary>
<p>

````
SELECT TO_CHAR(pickup_datetime, 'YYYY-MM-DD'),
COUNT(*)
FROM workshop_das. green_201601_csv
GROUP BY 1
ORDER BY 1;
````
</p>
</details>

> Matt S.: I didnt think adding the visualization here makes sense. maybe we will leave that in the PPT on the big screen ?? let me know your thoughts. 

## Workshop - Scenario #2: Go Back in Time

* Query historical data residing on S3:
	* Create external DB for Redshift Spectrum.
	* Create the external table on Spectrum.
	* Write a script or SQL statement to add partitions.
	* Create and populate a small number of dimension tables on Redshift DAS.
* Introspect the historical data, perhaps rolling-up the data in novel ways to see trends over time, or other dimensions.
* Enforce reasonable use of the cluster with Redshift Spectrum-specific Query Monitoring Rules (QMR).
	* Test the QMR setup by writing an excessive-use query.

**Note the partitioning scheme is Year, Month, Type (where Type is a taxi company). Here's a quick Screenshot:**

````
$ aws s3 ls s3://us-west-2.serverless-analytics/canonical/NY-Pub/
                           PRE year=2009/
                           PRE year=2010/
                           PRE year=2011/
                           PRE year=2012/
                           PRE year=2013/
                           PRE year=2014/
                           PRE year=2015/
                           PRE year=2016/

$ aws s3 ls s3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=1/
                           PRE type=fhv/
                           PRE type=green/
                           PRE type=yellow/

$ aws s3 ls s3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=1/type=green/
2017-05-18 19:43:22   18910771 part-r-00000-4c01b1ef-3419-40ba-908e-5b36b3556fa7.gz.parquet
````

### Create external schema (and DB) for Redshift Spectrum

* Create an external schema **ant321** from your database **spectrumdb**

	<details><summary>Hint</summary>
	<p>
	
	```python
	CREATE external SCHEMA ant321
	FROM data catalog DATABASE 'spectrumdb' 
	IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' 
	CREATE external DATABASE if not exists;
	```
	
	</p>
	</details>

### Create your Spectrum table DDL (or use this)

* Create your external table **ant321.NYTaxiRides** for
	`vendorid, pickup_datetime, dropoff_datetime, ratecode, passenger_count, trip_distance, fare_amount, total_amount, payment_type`
	stored in parquet format under location `s3://us-west-2.serverless-analytics/canonical/NY-Pub/`
	<details><summary>Hint</summary>
	<p>
	
	
	```python
	CREATE EXTERNAL TABLE ant321.NYTaxiRides (
    vendorid VARCHAR(6),
    pickup_datetime TIMESTAMP,
    dropoff_datetime TIMESTAMP,
    ratecode INT,
    passenger_count INT,
    trip_distance FLOAT8,
    fare_amount FLOAT8,
    total_amount FLOAT8,
    payment_type INT
    )
  PARTITIONED BY (YEAR INT, MONTH INT, "TYPE" CHAR(6))
  STORED AS PARQUET
  LOCATION 's3://us-west-2.serverless-analytics/canonical/NY-Pub/'
	;

	```
	</p>
	</details>
	
### Add the Partitions

```
WITH generate_smallint_series AS (select row_number() over () as n from workshop_das.green_201601_csv limit 65536)
, part_years AS (select n AS year_num from generate_smallint_series where n between 2009 and 2016)
, part_months AS (select n AS month_num from generate_smallint_series where n between 1 and 12)
, taxi_companies AS (SELECT 'fhv' taxi_vendor UNION ALL SELECT 'green' UNION ALL SELECT 'yellow’)

SELECT 'ALTER TABLE ant321.NYTaxiRides ADD PARTITION(year=' || year_num || ', month=' || month_num || ', type=\'' || taxi_vendor || '\') ' ||
'LOCATION \'s3://us-west-2.serverless-analytics/canonical/NY-Pub/year=' || year_num || '/month=' || month_num || '/type=' || taxi_vendor || '/\';'
FROM part_years, part_months, taxi_companies order by 1;

```

### Update the number of rows table property

1. Determine the number of rows in the table.
2. Save a copy of the explain plan for #1 above.
3. Set the (approximate or specific) number of rows using the TABLE PROPERTIES under ALTER EXTERNAL TABLE.
4. Rerun the explain plan for #1, noting the difference. When might this be impactful?

### Add a Redshift Spectrum Query Monitoring Rule to ensure reasonable use

In Amazon Redshift workload management (WLM), query monitoring rules define metrics-based performance boundaries for WLM queues and specify what action to take when a query goes beyond those boundaries. Setup a Query Monitoring Rule to ensure reasonable use.

```
https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html
```
Take a look at SVL_QUERY_METRICS_SUMMARY view shows the maximum values of metrics for completed queries. This view is derived from the STL_QUERY_METRICS system table. Use the values in this view as an aid to determine threshold values for defining query monitoring rules.

```
https://docs.aws.amazon.com/redshift/latest/dg/r_SVL_QUERY_METRICS_SUMMARY.html
```

### [Advanced Topic] Debug a Parquet/Redshift Spectrum datatype mismatch

1. Create a new Redshift Spectrum table, changing the datatype of column ‘trip_distance’ from FLOAT8 to FLOAT4. 
	* Add a single partition for testing.
2. Counts still work, but what about other operations (SELECT MIN(trip_distance) FROM, SELECT * FROM, CTAS)?
3. Instead of considering Apache Drill or other tool to help resolve the issue, consider Redshift system view SVL_S3LOG  
 
<details><summary>Hint</summary>
<p>


```python
https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-troubleshooting.html#spectrum-troubleshooting-incompatible-data-format
```

</p>
</details>


## Workshop - Scenario #3: Create a Single Version of Truth

### Create a view 

Create a view that covers both the January, 2016 Green company DAS table with the historical data residing on S3 to make a single table exclusively for the Green data scientists. Use CTAS to create a table with data from January, 2016 for the Green company. Compare the runtime to populate this with the COPY runtime earlier.


<details><summary>Hint</summary>
<p>


```
CREATE TABLE workshop_das.taxi_201601 AS SELECT * FROM ant321.NYTaxiRides WHERE year = 2016 AND month = 1 AND type = 'green';
```

</p>
</details>

### Complete populating the table 

Add to the January, 2016 table with an INSERT/SELECT statement for the other taxi companies.

<details><summary>Hint</summary>
<p>

```
INSERT INTO workshop_das.taxi_201601 (SELECT * FROM ant321.NYTaxiRides WHERE year = 2016 AND month = 1 AND type != 'green');

```

</p>
</details>

### Create a new Spectrum table 
Create a new Spectrum table **ant321.NYTaxiRides** (or simply drop the January, 2016 partitions).

<details><summary>Hint</summary>
<p>

```python
ALTER TABLE ant321.NYTaxiRides DROP PARTITION(year=2016, month=1, type='fhv'); 
ALTER TABLE ant321.NYTaxiRides DROP PARTITION(year=2016, month=1, type='green'); 
ALTER TABLE ant321.NYTaxiRides DROP PARTITION(year=2016, month=1, type='yellow'); 

```
</p>
</details>


### Create a view with no Schema Binding
Create a view **ant321_view_NYTaxiRides** from **workshop_das.taxi_201601** that allows seamless querying of the DAS and Spectrum data.

<details><summary>Hint</summary>
<p>

```python
CREATE VIEW ant321_view_NYTaxiRides AS
SELECT * FROM workshop_das.taxi_201601
UNION ALL 
SELECT * FROM ant321.NYTaxiRides
WITH NO SCHEMA BINDING
;

```

</p>
</details>

### Is it Surprising this is valid SQL?

- Note the use of the partition columns in the SELECT and WHERE clauses. Where were those columns in your Spectrum table definition?
- Note the filters being applied either at the partition or file levels in the Spectrum portion of the query (versus the Redshift DAS section).
- Does the runtime surprise you? Why or why not?

```
EXPLAIN SELECT year, month, type, COUNT(*) FROM ant321_view_NYTaxiRides WHERE year = 2016 AND month IN (1) AND passenger_count = 4 GROUP BY 1,2,3 ORDER BY 1,2,3;

```


```
QUERY PLAN 
XN Merge  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
  Merge Key: derived_col1, derived_col2, derived_col3
  ->  XN Network  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
        Send to leader
        ->  XN Sort  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
              Sort Key: derived_col1, derived_col2, derived_col3
              ->  XN HashAggregate  (cost=90025653.19..90025653.19 rows=2 width=48)
                    ->  XN Subquery Scan ant321_view_nytaxirides  (cost=25608.12..90025653.17 rows=2 width=48)
                          ->  XN Append  (cost=25608.12..90025653.15 rows=2 width=38)
                                ->  XN Subquery Scan "*SELECT* 1"  (cost=25608.12..25608.13 rows=1 width=18)
                                      ->  XN HashAggregate  (cost=25608.12..25608.12 rows=1 width=18)
                                            ->  XN Seq Scan on t201601_pqt  (cost=0.00..25292.49 rows=31563 width=18)
                                                  Filter: ((passenger_count = 4) AND ("month" = 1) AND ("year" = 2016))
                                ->  XN Subquery Scan "*SELECT* 2"  (cost=90000045.00..90000045.02 rows=1 width=38)
                                      ->  XN HashAggregate  (cost=90000045.00..90000045.01 rows=1 width=38)
                                            ->  XN Partition Loop  (cost=90000000.00..90000035.00 rows=1000 width=38)
                                                  ->  XN Seq Scan PartitionInfo of ant321.nytaxirides  (cost=0.00..15.00 rows=1 width=30)
                                                        Filter: (("month" = 1) AND ("year" = 2016))
                                                  ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45000010.00 rows=1000 width=8)
                                                        ->  S3 Aggregate  (cost=45000000.00..45000000.00 rows=1000 width=0)
                                                              ->  S3 Seq Scan ant321.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..37500000.00 rows=3000000000 width=0)
                                                                    Filter: (passenger_count = 4)

```

- Now include Spectrum data by adding a month whose data is in Spectrum

```
EXPLAIN SELECT year, month, type, COUNT(*) FROM ant321_view_NYTaxiRides WHERE year = 2016 AND month IN (1,2) AND passenger_count = 4 GROUP BY 1,2,3 ORDER BY 1,2,3;

```

```
QUERY PLAN
XN Merge  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
  Merge Key: derived_col1, derived_col2, derived_col3
  ->  XN Network  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
        Send to leader
        ->  XN Sort  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
              Sort Key: derived_col1, derived_col2, derived_col3
              ->  XN HashAggregate  (cost=90029268.90..90029268.90 rows=2 width=48)
                    ->  XN Subquery Scan ant321_view_nytaxirides  (cost=29221.33..90029268.88 rows=2 width=48)
                          ->  XN Append  (cost=29221.33..90029268.86 rows=2 width=38)
                                ->  XN Subquery Scan "*SELECT* 1"  (cost=29221.33..29221.34 rows=1 width=18)
                                      ->  XN HashAggregate  (cost=29221.33..29221.33 rows=1 width=18)
                                            ->  XN Seq Scan on t201601_pqt  (cost=0.00..28905.70 rows=31563 width=18)
                                                  Filter: ((passenger_count = 4) AND ("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                                ->  XN Subquery Scan "*SELECT* 2"  (cost=90000047.50..90000047.52 rows=1 width=38)
                                      ->  XN HashAggregate  (cost=90000047.50..90000047.51 rows=1 width=38)
                                            ->  XN Partition Loop  (cost=90000000.00..90000037.50 rows=1000 width=38)
                                                  ->  XN Seq Scan PartitionInfo of ant321.nytaxirides  (cost=0.00..17.50 rows=1 width=30)
                                                        Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                                                  ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45000010.00 rows=1000 width=8)
                                                        ->  S3 Aggregate  (cost=45000000.00..45000000.00 rows=1000 width=0)
                                                              ->  S3 Seq Scan ant321.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..37500000.00 rows=3000000000 width=0)
                                                                    Filter: (passenger_count = 4)

```

```
EXPLAIN SELECT passenger_count, COUNT(*) FROM ant321.NYTaxiRides WHERE year = 2016 AND month IN (1,2) GROUP BY 1 ORDER BY 1;

```

```
QUERY PLAN
XN Merge  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
  Merge Key: nytaxirides.derived_col1
  ->  XN Network  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
        Send to leader
        ->  XN Sort  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
              Sort Key: nytaxirides.derived_col1
              ->  XN HashAggregate  (cost=90005018.50..90005019.00 rows=200 width=12)
                    ->  XN Partition Loop  (cost=90000000.00..90004018.50 rows=200000 width=12)
                          ->  XN Seq Scan PartitionInfo of ant321.nytaxirides  (cost=0.00..17.50 rows=1 width=0)
                                Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                          ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45002000.50 rows=200000 width=12)
                                ->  S3 HashAggregate  (cost=45000000.00..45000000.50 rows=200000 width=4)
                                      ->  S3 Seq Scan ant321.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..30000000.00 rows=3000000000 width=4)

```

```
EXPLAIN SELECT type, COUNT(*) FROM ant321.NYTaxiRides WHERE year = 2016 AND month IN (1,2) GROUP BY 1 ORDER BY 1 ;

```

```
QUERY PLAN
XN Merge  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
  Merge Key: nytaxirides."type"
  ->  XN Network  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
        Send to leader
        ->  XN Sort  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
              Sort Key: nytaxirides."type"
              ->  XN HashAggregate  (cost=75000042.50..75000042.51 rows=1 width=30)
                    ->  XN Partition Loop  (cost=75000000.00..75000037.50 rows=1000 width=30)
                          ->  XN Seq Scan PartitionInfo of ant321.nytaxirides  (cost=0.00..17.50 rows=1 width=22)
                                Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                          ->  XN S3 Query Scan nytaxirides  (cost=37500000.00..37500010.00 rows=1000 width=8)
                                ->  S3 Aggregate  (cost=37500000.00..37500000.00 rows=1000 width=0)
                                      ->  S3 Seq Scan ant321.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..30000000.00 rows=3000000000 width=0)

```

## * Matt S.: Missing these 3 per slide 27
* Examine the explain plan differences based on the filters used in a query.
* Build a aggregate or roll-up table using the view to populate a Redshift DAS table. ETL options here.
* Crawl and explore the aggregate table using Amazon QuickSight

## Workshop - Scenario #4: Plan for the Future

* Allow for trailing 5 quarters reporting by adding the Q4 2015 data to Redshift DAS:
	* Anticipating the we’ll want to ”age-off” the oldest quarter on a 3 month basis, architect your DAS table to make 	this easy to maintain and query.
	* Adjust your Redshift Spectrum table to exclude the Q4 2015 data.
* Develop and execute a plan to move the Q4 2015 data to S3.
	* What are the discrete steps to be performed?
	* What extra-Redshift functionality must be leverage as of Monday, November 27, 2018?
	* Simulating the extra-Redshift steps with the existing Parquet data, age-off the Q4 2015 data from Redshift DAS 	and perform any needed steps to maintain a single version of the truth.


> Mark S.: Slide 35 and 36 look same. 
