# Drilling into data with Drill

### Prerequisites:

* Java 7 or newer
* Download and unzip [Apache Drill](http://drill.apache.org)
* [MongoDB 3.x](https://www.mongodb.org/downloads)
* Unzip the data.zip (from this repository)

## Drilling!

#### Start Apache Drill 

In this lab you will use Apache Drill in a stand alone mode.

Let's call the directory where you have unzipped Apache Drill `$DRILL_HOME`

```
cd $DRILL_HOME

# Linux/OSX
bin/drill-embedded

# Windows
bin/sqlline -u jdbc:drill:zk=local

```

You can now execute a simple query:

```
SELECT * FROM cp.`employee.json` LIMIT 20;

+--------------+------------------+-------------+------------+--------------+---------------------+-----------+----------------+-------------+------------------------+----------+----------------+------------------+-----------------+---------+--------------------+
| employee_id  |    full_name     | first_name  | last_name  | position_id  |   position_title    | store_id  | department_id  | birth_date  |       hire_date        |  salary  | supervisor_id  | education_level  | marital_status  | gender  |  management_role   |
+--------------+------------------+-------------+------------+--------------+---------------------+-----------+----------------+-------------+------------------------+----------+----------------+------------------+-----------------+---------+--------------------+
| 1            | Sheri Nowmer     | Sheri       | Nowmer     | 1            | President           | 0         | 1              | 1961-08-26  | 1994-12-01 00:00:00.0  | 80000.0  | 0              | Graduate Degree  | S               | F       | Senior Management  |
| 2            | Derrick Whelply  | Derrick     | Whelply    | 2            | VP Country Manager  | 0         | 1              | 1915-07-03  | 1994-12-01 00:00:00.0  | 40000.0  | 1              | Graduate Degree  | M               | M       | Senior Management  |
| 4            | Michael Spence   | Michael     | Spence     | 2            | VP Country Manager  | 0         | 1              | 1969-06-20  | 1998-01-01 00:00:00.0  | 40000.0  | 1              | Graduate Degree  | S               | M       | Senior Management  |
...
...
```

You can select the output format of the shell using:

```
!set outputFormat csv

SELECT * FROM cp.`employee.json` LIMIT 20;


!set outputFormat vertical

SELECT * FROM cp.`employee.json` LIMIT 20;


!set outputFormat table

SELECT * FROM cp.`employee.json` LIMIT 20;


```

You can find more information abou the shell [here](https://drill.apache.org/docs/configuring-the-drill-shell/).



#### Apache Drill Web UI


You can also use the Drill Web Interface to run queries. Go to:

* [http://localhost:8047](http://localhost:8047)
 
 
Go in the Query tab and run another query. (```SELECT * FROM cp.`employee.json` LIMIT 20```)


### Configure "Storages"

Drill allows you to query many datasources: simple file system, distributed file system like HDFS or MapR-FS, cloud storage Amazon S3/Azure but also databases (NoSQL & RDBMS).

Let's create a new workspace to the directory where you have saved the dataset.

1. In the Web UI, go in the **Storage** tab
2. Click in the **Update** button for the **dfs** plugin. The dfs plugin is the "file system/distributed file system one
3. In the list of workspaces, add one entry to point to the directory where you "datasets" have been saved:

```
    "reviews": {
      "location": "/Users/tgrall/data/reviews",
      "writable": true,
      "defaultInputFormat": null
    },
    "data": {
      "location": "/Users/tgrall/data/",
      "writable": true,
      "defaultInputFormat": null
    }
```
4. Click the **Update** button


You can now use the review workspace into your query for example:

```
SELECT count(1) AS businesses
  FROM dfs.reviews.`business.json`
```

or

```
SELECT count(1)
  FROM dfs.data.`airport/*.csv`
```


### Discover Drill

Apache Drill is an ANSI SQL compliant engine, allowing you to write any type of queries: simple queries, join, aggregation functions, windowing function, ...

See below some examples:

* Find reviews of Leslie, and print the business name, text review and rating 

```
SELECT b.name `business`,  r.review_text, r.rating
FROM dfs.reviews.`review.json` r,
     dfs.reviews.`business.json` b,
     dfs.reviews.`user.json` u
WHERE b.business_id = r.business_id
AND r.user_id = u.user_id
AND u.name = 'Leslie'
ORDER BY b.name
```

* The business data contains a rating, print the number of businesses for each rating value.

```
SELECT rating, count(1) `nb`
FROM dfs.reviews.`business.json` 
GROUP BY rating
ORDER BY rating
```

* Print only the rating that have more than 30000 businesses

```

SELECT rating, count(1) `nb`
FROM dfs.reviews.`business.json` 
GROUP BY rating
HAVING count(1) > 30000
ORDER BY rating
```

### Query complex Data

Now you will learn how to query and use complex data structures of your JSON documents.

JSON document are rich structures, for example in the business documents you have:

* location: is a sub document
* categories: a list of values (simple string)

##### Sub Documents

To print the `zip` and `state` information from the business location you have to use the *dot* notation, see below

```
SELECT b.name, b.location.zip `zip`, b.location.state `state`
FROM dfs.reviews.`business.json` b
LIMIT 10
``` 
Note: when using the *dot* notation, like `b.location.state`, Drill considers the first element as the table name, this is why you should add an alias.

##### Arrays/Lists

When you have to work with lists, for example categories in business, you must **flatten** the data, to get one row for each category.

```
SELECT name, flatten(categories) category
FROM dfs.reviews.`business.json` 
LIMIT 10
``` 


**Questions**

* Q1- How many businesses in `MO` state ?

* Q2- Count the number of business by state, sorted by the state that has the most businesses ?

* Q3- Top 5 categories order starting from the most common one _(tip: you have to use the `flatten` function)_

**Solutions**

> * Solution 1
> 
>```
SELECT count(1) 
FROM dfs.reviews.`business.json` b
WHERE b.location.state = 'MO'
```


> * Solution 2

>```
SELECT b.location.state `state`, count(1) `total`
FROM dfs.reviews.`business.json`  b
GROUP BY b.location.state
ORDER BY `total` DESC
```

>* Solution 3

> ```
SELECT category, count(1) `nb_business`
FROM (
  SELECT name, flatten(categories) category
  FROM dfs.reviews.`business.json` 
)
GROUP BY category
ORDER BY `nb_business`
LIMIT 5
```

### Use multiple datasources

In this lab you will use MongoDB to store data and execute SQL queries that join file system and MongoDB data.

Let's call `$MONGO_HOME` the directory where Mongo is installed.

#### Start Mongo & Insert Data

1- Start MongoDB, from a terminal

```
cd $MONGO_HOME

mkdir data_to_delete

./bin/mongod --dbpath ./data_to_delete

```

2- Import user data into MongoDB

In a new terminal window:

```
./bin/mongoimport  -d drill -c users ~/data/reviews/user.json
```

User data are inserted into the `drill` database and `users` collection.


#### Enable MongoDB Storage Plugin

1. In the Web UI, go in the **Storage** tab
2. Click in the **Update** button for the **mongo** plugin.


### Query MongoDB from Drill

You have now the workspace `mongo` that you can use in your queries.

For example to count the number of users:

```
select count(1) from mongo.drill.users
```

### Join multiple datasources


Q1: Count the number of review per user, show the 10 most active users.

Answer 1:

```
SELECT r.user_id, u.name, count(1) `nb_reviews`
FROM dfs.reviews.`review.json` r,
     mongo.drill.users u
WHERE u.user_id = r.user_id     
GROUP BY r.user_id, u.name
ORDER BY `nb_reviews` DESC
LIMIT 10 
```


### Use view and materialized view

You can use views to protect sensitive data, for data aggregation, and to hide data complexity from users. 

1- Create a view that exposes business id, name and categories

```
CREATE OR REPLACE VIEW dfs.tmp.`business_category` as
  SELECT business_id, name, flatten(categories) category
  FROM dfs.reviews.`business.json` 

```

2- Use the view to find all bars (category equals 'BAR')

```
SELECT *
FROM dfs.tmp.`business_category`
WHERE category = 'Bar'
LIMIT 10
```

View does not store any data, it is storing the query and executes the query each time you call the view.

#### Materialized View (Table)

Apache Drill allows you to create 'table' from a query. In this case the result of the query will be saved in _new files_. By default the new table will be saved as Parquet files (optimized compressed, columnar oriented format), but it is also possible to save the data as JSON or CSV file.

For example, reviews is a large dataset, and we want to improve query performance. Using a `create table as` command and Parquet format query execution will be faster.

1- Create Table, as parquet file

```
CREATE TABLE dfs.tmp.`review_table`
 AS SELECT * FROM dfs.reviews.`review.json`
```

2- Use the Table

```
SELECT user_id, count(1) `nb`
FROM dfs.tmp.`review_table`
GROUP BY user_id
ORDER BY count(1) DESC
LIMIT 10
```

#### Views/Table Considerations

Views and Tables help people to reuse complex queries. When you create a view or a table you also define a "schema" ( the projection ), this is useful when you are working with JDBC/ODBC tools that will consume this data.

Creating a view is often the first step you have to do to expose Drill to 3rd party tools that need schemas.

This is also how you can secure data access controling who can access a specific objects.


#### Listing objects

You can look at the views/and table using the following commands in sqlline/terminal:

```
show schemas;

use dfs.tmp;

show tables;

```
You can also use the command:

```
!tables
```



### Custom Function (UDF)

Apache Drill has a rich set of function; however you may need for your project very specific business logic. For this you can develop your own function.

For example, you many need to mask some data when returning them to the user. Drill does not have this function, but you can create it, as documented [here](https://drill.apache.org/docs/tutorial-develop-a-simple-function/).

For this lab you do not have to create it, since it is packaged in the student kit.

#### Deploy the function

Copy the JARs files located in your zip file (`/udf/*.jar`) into:

1. `$DRILL_HOME/jars/3rdparty/` 
2. Restart Drill (type `!quit` in the Drill shell/sqlline to stop it)

#### Use the function in your queries

The function is now deployed and you can use in your queries, for example:

```
SELECT MASK(first_name, '*', 3) first_name , 
	       MASK(last_name, '#', 10) last_name
FROM cp.`employee.json` LIMIT 2;
```

You can now use this into views to hide data, for example:

```
CREATE OR REPLACE VIEW dfs.tmp.`masked_emp` AS
	SELECT MASK(first_name, '*', 3) first_name , 
	       MASK(last_name, '#', 10) last_name,
	       salary
   FROM cp.`employee.json`
```


And use it:

```
SELECT *
FROM dfs.tmp.`masked_emp`
WHERE first = 'Sheri'
```


#### Identify Your Data Breach with Apache Drill

In this article and sample code you will see how you can use custom function to:

* [identify data breach](https://www.mapr.com/blog/identify-your-data-breach-apache-drill).


### Work with CSV files

CSV is a very common format used in Big Data project. Apache Drill allows you to query CSV directly.

In the data folder you have a `/data/airport/passenger.csv` file.

1- Query CSV File

```
SELECT *
FROM dfs.data.`airport/*.csv`
```

2- Extract 'columns' from CSV

CSV lines are viewed as simple string, so you have to extract each columns and cast the value to the proper type.

```
SELECT
columns[0] as `DATE`,
columns[1] as `AIRLINE`,
CAST(columns[11] AS DOUBLE) as `PASSENGER_COUNT`
FROM dfs.data.`/airport/*.csv`
WHERE CAST(columns[11] AS DOUBLE) < 5

```

3- Create a view or table 

When you are working with CSV creating view or table will ease a lot the use of the dataset. For example:

```
CREATE TABLE dfs.tmp.`airport_data` AS
SELECT
CAST(SUBSTR(columns[0],1,4) AS INT)  `YEAR`,
CAST(SUBSTR(columns[0],5,2) AS INT) `MONTH`,
columns[1] as `AIRLINE`,
columns[2] as `IATA_CODE`,
columns[3] as `AIRLINE_2`,
columns[4] as `IATA_CODE_2`,
columns[5] as `GEO_SUMMARY`,
columns[6] as `GEO_REGION`,
columns[7] as `ACTIVITY_CODE`,
columns[8] as `PRICE_CODE`,
columns[9] as `TERMINAL`,
columns[10] as `BOARDING_AREA`,
CAST(columns[11] AS DOUBLE) as `PASSENGER_COUNT`
FROM dfs.data.`airport/passenger.csv`
```

and use this table:

```
SELECT *
FROM dfs.tmp.`airport_data`
```

### Working with Files

#### Looking in File System

Apache Drill provide many interesting tools to discover data stored on the file system and query the data.

```
use dfs.data;

show files;

show files in logs;

```

#### Querying File System


Drill will automatically navigate in sub directories to find files; and it will use the extension/format to parse the files.

It is very common to use a directory structure to partition the data. For example in the context of a log structure, you can organize the data per year/month.

```
logs
├── 2014
│   ├── 1
│   ├── 2
│   ├── 3
│   └── 4
└── 2015
    └── 1
```

You can query the data as follow:


Total number of lines in all directories

```
SELECT count(1)
FROM dfs.data.`logs/*`
```

Total number of lines for 2014

```
SELECT count(1)
FROM dfs.data.`logs/*`
WHERE dir0 = 2014
```


Count the number of lines per year

```
SELECT dir0, count(1)
FROM dfs.data.`logs/*`
GROUP BY dir0
```

If you want to get the number of lines per year and only for the first quarter?

```
SELECT dir0, count(1)
FROM dfs.data.`logs/*`
WHERE dir1 in (1,2,3)
GROUP BY dir0
```




## Conclusion

In this hands-on you have learned the basics of Apache Drill using simple files (JSON, CSV, Parquet) and Database (MongoDB) on a local installation.

Apache Drill can be deployed in a distributed environment, and allow you to query large and distributed data set for example in HDFS, MapR-FS, Hive, HBase, MapR-DB.

Drill is an extensible platform, developer can [create custom functions](https://drill.apache.org/docs/develop-custom-functions/), add new format, and new storage plugins. It is a very good opportunity [to contribute to Drill](https://drill.apache.org/docs/apache-drill-contribution-ideas/).

If you want to learn more about Apache Drill you can register for the free online training available at:

* [http://learn.mapr.com](http://learn.mapr.com)







