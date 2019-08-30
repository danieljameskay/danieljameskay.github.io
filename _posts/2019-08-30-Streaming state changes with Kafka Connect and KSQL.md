---
layout: post
title: Streaming stage changed with Kafka Connect and KSQL
---

When ever I'm building streaming solutions in my own time, I always try and make them as "real-word" as possible with real data.

Twitter is a good source for real data...people tweeting about their day, pets food etc.

One source I've recently being using is Poloniex. Its a cryptocurrency exchange that exposes many different endpoints for consumption by either HTTP or WS.

An endpoint that caught my eye was named return24hrVolume, which has the following description...

> Returns the 24-hour volume for all markets as well as totals for primary currencies. Primary currencies include BTC, ETH, USDT, USDC and show the total amount of those tokens that have traded within the last 24 hours.

and returned the following data...

```json
{"BTC_BCN":{"id":7,"last":"0.00000005","lowestAsk":"0.00000006","highestBid":"0.00000005","percentChange":"-0.16666666","baseVolume":"0.29430197","quoteVolume":"5514462.31371204","isFrozen":"0","high24hr":"0.00000006","low24hr"
:"0.00000005"},"BTC_BTS":{"id":14,"last":"0.00000350","lowestAsk":"0.00000351","highestBid":"0.00000347","percentChange":"-0.01960784","baseVolume":"27.56079306","quoteVolume":"7941010.48504995","isFrozen":"0","high24hr":"0.000
00369","low24hr":"0.00000330"},"BTC_CLAM":{"id":20,"last":"0.00025837","lowestAsk":"0.00025842","highestBid":"0.00025787","percentChange":"-0.12020294","baseVolume":"0.72377756","quoteVolume":"2681.37211323","isFrozen":"0","hig
h24hr":"0.00030100","low24hr":"0.00025000"}
```
Looking at the payload that is returned I had an idea. You could store this data in a KSQL Table, each currency pair has an `id` which could be used as the key.  We could then use this KSQL Table to enrich a stream or use the REST interface the KSQL Server exposes to recieve state changes.

So I did just that. So I'm running Kafka, Zookeeper, Kafka Connect, KSQL Server, KSQL CLI, MS SQL and a Schema Registry in Docker.

I also have an application running which is polling the Poloniex endpoint (https://poloniex.com/public?command=returnTicker) every 20 seconds, converting the JSON in a to model and persisting that to a database.

I've got a script which creates the Connector.

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
            "mode":"incrementing", 
            "incrementing.column.name" : "Id",
            "transforms": "ValueToKey",
            "transforms.ValueToKey.type":"org.apache.kafka.connect.transforms.ValueToKey",
            "transforms.ValueToKey.fields":"CurrencyId"
            }
    }'
```

Its nothing fancy, using incrementing mode with the Id column and an SMT to specify the currency id field as the key for the record.

With the application running and the connector running we should see data in the Topic.

