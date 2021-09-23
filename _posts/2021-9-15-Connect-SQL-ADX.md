---
layout: post
title: Retrieve data from a SQL database in Azure Data Explorer
comments: false
---

Azure Data Explorer is a great and powerful platform for when you are working with timeseries data such as sensor measurements and logs.  Yet it is not designed to store relational data or act as an OLTP database.
However, there can be situations where you want to combine relational data with the timeseries data that is stored in Azure Data Explorer.  This post briefly explains how this can be done.

Imagine a farm where fruit is grown in a controlled environment.  The farmer can change all kinds of settings in that environment -think about temperature, light, watering, ...  
The environment is equipped with numerous sensors that measure the temperature, humidity, size of the plants, etc...
After each harvest, the farmer sows new plants and a new cycle starts, perhaps with different settings, based on the experience of the last harvest,  and all in a controlled environment.

Suppose the information on when a new cycle starts and stops is kept in an Azure SQL database, while all sensor-readings are ingested in an Azure Data Explorer database.  Now, what if you want to create a query in Azure Data Explorer which combines both the sensor-readings that are stored in Azure Data Explorer, and the cycle information that is stored in Azure SQL ?  There are two ways to do that, and I'll describe both ways here.

## Use an external table

An **external table** is an entity in Data Explorer that references data stored outside of the Azure Data Explorer database.  A SQL Server table is one of the supported data stores for such an external table.

Creating an external table that references a table in a SQL Server database is fairly easy:

```kql
.create external table CyclusInformation (id:int, cyclus_number:int, startdate:datetime, enddate:datetime) 
kind=sql
table=Cyclus
(
   h@'Server=tcp:xxxxx.database.windows.net,1433;Initial Catalog={databasename};Persist Security Info=False;User ID={username};Password={password};'
)
with
(
   createifnotexists = false,
   primarykey = id,
   firetriggers=id
)
```

The above statement creates an `external table` which is called `CyclusInformation` and it refers to the table `Cyclus` that exists in the SQL Server database defined by the connection-string.

Once the `external table` is defined, you can query it via KQL just like you would do with any regular Data Explorer table:

```kql
CyclusInformation
| project cyclus_number, startdate
| take 10
```

The drawback of using external tables in KQL, is that filtering and ordering is done by Azure Data Explorer and not by SQL Server.  If one in SQL Server, the performance would be better.

## Use an inline SQL query

Another way to integrate data from a SQL Server database in Azure Data Explorer is to make use of the `sql_request` plugin.  This plugin allows you to write a native SQL query in Azure Data Explorer.  The query is executed by SQL Server, so performance wise, this is a better solution.  
However, the drawback here is that the `sql_request` plugin only returns a single row.

Given our scenario, suppose you want to retrieve measurements from Data Explorer that were valid in the cyclus with cyclus-number 4, you could use the following KQL query:

```kql
let cyclusinformation =
  materialize (
      evaluate sql_request('Server=tcp:xxxxx.database.windows.net,1433;Initial Catalog={databasename};Persist Security Info=False;User ID={usename};Password={password}',
                           'SELECT startdate, enddate FROM Cyclus where cyclus_number = 4')
  );
let s = cyclusinformation | project startdate;
let e = cyclusinformation | project enddate;
metrics
| where timestamp >= toscalar(s) and (isnull(toscalar(e)) or timestamp <= toscalar(e))
| where tag == 'temperature'
```

## What about secrets

As can be seen in the above code snippets, both the `external table` and the `sql_request` approach require a connectionstring to the SQL Server database.  We don't want to have any passwords and usernames in our code.  It would be ideal if we could avoid having a username and password visible in the connectionstring.  This can be done by making use of a managed identity to access the SQL Server database.  A few simple steps need to be taken to make this possible:

- Make sure that the Azure Data Explorer cluster has an assigned identity
- Enable Azure AD authentication on the Azure SQL Server.  This is done by defining an Azure AD Admin on the SQL Server
- Create a user in the Azure SQL database that represents the service that accesses the SQL database.  In this case, this is the Azure Data Explorer cluster:

  ```sql
  CREATE USER {adx-cluster-name} FROM EXTERNAL PROVIDER
  ALTER ROLE db_datareader ADD MEMBER {adx-cluster-name}
  ```

- change the connectionstring to `Server=tcp:xxxxxx.database.windows.net,1433;Initial Catalog={databasename};Persist Security Info=False;User ID={adx-cluster-name};Authentication="Active Directory Integrated";`

## Conclusion

We know that Azure Data Exporer is a great tool for working with timeseries data.  In this blog post I've shown that it is also possible to combine the data that is present in Data Explorer with external data.