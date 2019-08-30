---
layout: post
title: Streaming state changes with Kafka Connect and KSQL
---

When I build streaming applications for a proof of concept or I want to try something new, I always try and use real-world data.

One source I've recently being using is Poloniex. It's a cryptocurrency exchange that exposes many different endpoints for consumption by either HTTP or WS.

An endpoint that caught my eye was named returnTicker, which has the following description...

> Retrieves summary information for each currency pair listed on the exchange

and returned the following data...

```json
...
{ BTC_BCN:
   { id: 7,
     last: '0.00000024',
     lowestAsk: '0.00000025',
     highestBid: '0.00000024',
     percentChange: '0.04347826',
     baseVolume: '58.19056621',
     quoteVolume: '245399098.35236773',
     isFrozen: '0',
     high24hr: '0.00000025',
     low24hr: '0.00000022' },
  USDC_BTC:
   { id: 224,
     last: '6437.65329245',
     lowestAsk: '6436.73575054',
     highestBid: '6425.68259132',
     percentChange: '0.00744080',
     baseVolume: '1193053.18913982',
     quoteVolume: '185.43611063',
     isFrozen: '0',
     high24hr: '6499.09114231',
     low24hr: '6370.00000000' },
...
```
Looking at the payload that it returned blossomed an idea. I could store this data in a KSQL Table and as each currency pair has an `id` this could be used as the key. I could then use the KSQL Table to enrich a stream in the future or use the REST interface the KSQL Server exposes to receive state changes.

So I did just that. 

So I'm running Kafka, Zookeeper, Kafka Connect, KSQL Server, KSQL CLI, MS SQL and a Schema Registry in Docker. I also have an application running which is polling the Poloniex endpoint (https://poloniex.com/public?command=returnTicker) every 5 seconds, converting the JSON into a model and persisting that to a database.

The below script creates the Connector.

```bash
#!/bin/bash

curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
    "name": "jdbc_source_mssql",
    "config": {
            "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
            "connection.url": "jdbc:sqlserver://db:1433;databaseName=dev",
            "connection.user": "SA",
            "connection.password": "Password10",
            "topic.prefix": "mssql-poloniex-",
            "table.whitelist" : "Ticker",
            "mode":"incrementing", "incrementing.column.name" : "Id",
            "transforms": "ValueToKey",
            "transforms.ValueToKey.type":"org.apache.kafka.connect.transforms.ValueToKey",
            "transforms.ValueToKey.fields":"CurrencyId"
            }
    }'
```

Its nothing fancy, using incrementing mode with the `Id` column and an SMT to specify the `CurrencyId` field as the key for the record.

With the application and Connector running we should see the ticker data in the Topic.

```bash
ksql> show topics;

 Kafka Topic            | Registered | Partitions | Partition Replicas | Consumers | ConsumerGroups 
----------------------------------------------------------------------------------------------------
 _schemas               | false      | 1          | 1                  | 0         | 0              
 docker-connect-configs | false      | 1          | 1                  | 0         | 0              
 docker-connect-offsets | false      | 25         | 1                  | 0         | 0              
 docker-connect-status  | false      | 5          | 1                  | 0         | 0              
 mssql-poloniex-Ticker  | false      | 101        | 1                  | 0         | 0              
----------------------------------------------------------------------------------------------------
ksql> print 'mssql-poloniex-Ticker';
Format:AVRO
8/30/19 12:12:59 PM UTC, 7, {"Id": 102, "Last": "0.00000006", "LowestAsk": "0.00000006", "HighestBid": "0.000000
05", "PercentChange": "0.20000000", "BaseVolume": "0.50319564", "QuoteVolume": "9212611.26993458", "IsFrozen": 0
, "High24Hr": "0.00000006", "Low24Hr": "0.00000005", "CurrencyId": "7"}
8/30/19 12:12:59 PM UTC, 14, {"Id": 103, "Last": "0.00000355", "LowestAsk": "0.00000356", "HighestBid": "0.00000
354", "PercentChange": "-0.02472527", "BaseVolume": "28.79985773", "QuoteVolume": "8308441.94234158", "IsFrozen"
: 0, "High24Hr": "0.00000369", "Low24Hr": "0.00000330", "CurrencyId": "14"}
8/30/19 12:12:59 PM UTC, 20, {"Id": 104, "Last": "0.00026168", "LowestAsk": "0.00026682", "HighestBid": "0.00026
302", "PercentChange": "-0.09116799", "BaseVolume": "0.71899165", "QuoteVolume": "2698.59625966", "IsFrozen": 0,
 "High24Hr": "0.00028793", "Low24Hr": "0.00025000", "CurrencyId": "20"}
```
Now we can create a KSQL Table for the data.

```
ksql> create table ticker_table with (kafka_topic='mssql-poloniex-Ticker', value_forma
t='avro', key='CurrencyId');
```
We specify the Table name, the Topic name which contains the records, the output format and the key we want to use for the Table. If we describe the Table we'll be able to verify that the Tabled is keyed by the `CurrencyId`.

```
ksql> describe extended ticker_table;

Name                 : TICKER_TABLE
Type                 : TABLE
Key field            : CURRENCYID
Key format           : STRING
Timestamp field      : Not set - using <ROWTIME>
Value format         : AVRO
Kafka topic          : mssql-poloniex-Ticker (partitions: 101, replication: 1)

 Field         | Type                      
-------------------------------------------
 ROWTIME       | BIGINT           (system) 
 ROWKEY        | VARCHAR(STRING)  (system) 
 ID            | BIGINT                    
 LAST          | VARCHAR(STRING)           
 LOWESTASK     | VARCHAR(STRING)           
 HIGHESTBID    | VARCHAR(STRING)           
 PERCENTCHANGE | VARCHAR(STRING)           
 BASEVOLUME    | VARCHAR(STRING)           
 QUOTEVOLUME   | VARCHAR(STRING)           
 ISFROZEN      | BIGINT                    
 HIGH24HR      | VARCHAR(STRING)           
 LOW24HR       | VARCHAR(STRING)           
 CURRENCYID    | VARCHAR(STRING)           
-------------------------------------------

Local runtime statistics
------------------------


(Statistics of the local KSQL server interaction with the Kafka topic mssql-poloniex-T
icker)
```

Then we can run an KSQL command to verify that the Table is being populated with data. The first column from the left is the `timestamp` and the second column in is the key which is the `CurrencyId`. 

```
ksql> select * from ticker_table;
1567168110559 | 7 | 203 | 0.00000006 | 0.00000006 | 0.00000005 | 0.20000000 | 0.49651928 | 9084554.02673273 | 0 | 0.00000006 | 0.00000005 | 7
1567168110560 | 14 | 204 | 0.00000354 | 0.00000354 | 0.00000353 | -0.02747252 | 28.92635877 | 8344153.09344625 | 0 | 0.00000369 | 0.00000330 | 14
1567168110560 | 20 | 205 | 0.00026168 | 0.00026682 | 0.00026330 | -0.09116799 | 0.71899165 | 2698.59625966 | 0 | 0.00028793 | 0.00025000 | 20
```

So if we think about the data inside the table it would look something like this...

| ROWTIME | ROWKEY | ID | LAST | LOWESTASK | HIGHESTBID | PERCENTCHANGE | BASEVOLUME | QUOTEVOLUME | FROZEN | HIGH24HR | LOW24HR | CURRENCYID
| ------------- |:-:| :--:| :---------:| :---------:| :---------:| :---------:| :---------:| :---------:| :---------:| :---------:| :--------:| :--------:|
| 1567168110559 | 7 | 203 | 0.00000006 | 0.00000006 | 0.00000005 | 0.20000000 | 0.49651928 | 9084554.02673273 | 0 | 0.00000006 | 0.00000005 | 7
| 1567168110560 | 14 | 204 | 0.00000354 | 0.00000354 | 0.00000353 | -0.02747252 | 28.92635877 | 8344153.09344625 | 0 | 0.00000369 | 0.00000330 | 14
| 1567168110560 | 20 | 205 | 0.00026168 | 0.00026682 | 0.00026330 | -0.09116799 | 0.71899165 | 2698.59625966 | 0 | 0.00028793 | 0.00025000 | 20

When the endpoint is scraped again in 5 seconds time, the table will have its state updated with the new data. After the state has been updated it represents the Ticker data value at that current moment in time. 

The last step is to send a HTTP request to the KSQL Server and create a query. Here we are selecting the `CurrencyId`, `HighestBid` and `LowestAsk` from the Ticker Table. So each time the state is refreshed with new data from Poloniex this will be streamed back.

```
⇒  curl -X "POST" "http://localhost:8088/query" \
     -H "Content-Type: application/vnd.ksql.v1+json" \
     -d $'{
  "ksql": "SELECT currencyId, highestbid, lowestask FROM ticker_table;"
}'
```

We should then see the data returned.

```json
{"row":{"columns":["231","0.03118013","0.03199900"]},"errorMessage":null,"finalMessage":null,"terminal":false}
{"row":{"columns":["242","0.06246167","0.06305349"]},"errorMessage":null,"finalMessage":null,"terminal":false}
{"row":{"columns":["210","0.00001830","0.00001833"]},"errorMessage":null,"finalMessage":null,"terminal":false}
{"row":{"columns":["253","0.00023012","0.00023222"]},"errorMessage":null,"finalMessage":null,"terminal":false}
{"row":{"columns":["216","0.00244131","0.00246000"]},"errorMessage":null,"finalMessage":null,"terminal":false}
{"row":{"columns":["203","3.23667520","3.26070201"]},"errorMessage":null,"finalMessage":null,"terminal":false}
```

Pretty cool!

Quick recap.....we have an application which is scraping an endpoint exposed by Poloniex every 5 seconds and persisting this data into a MS SQL Database. We are then using Kafka Connect to stream the new data into Kafka then utilise KSQL to build a KSQL Table which keeps a materialized view of the Ticker data. We can then use a query to stream changes of the state over HTTP or alternatively, use the state to enrich a KSQL Stream.

DK
