# Introduction
Setup predictionIO machine learning for Recommendation. Useful machine learning to show subject(products, shop, etc) to users with the highest score based on prediction results. Can be used in e-Commerce site or else.

# Gist
- PredictionIO platform - Open source machine learning stack for building, evaluating and deploying engine with machine learning algorithm
- Event Server - Open source machine learning analytic layer
- Template Gallery - Template engine to derive from

# Requirements
- Scala 2.11.12
- Spark 2.1.3
- Apache PredictionIO v0.14.0
- Storage(Postgres, mysql, or elasticsearch)


# Flow
1. Our apps(mobile, website, etc) send data to Event Server(collect the data)
2. PredictionIO engine build predictive model(s) with one or more algorithm using the data from (1). Have to periodically run "pio train" to train and deploy model data into webservice. 
3. Query the result sets from deployed webservice to get the "recommendation" subject(product, shop, etc).

# Installation

## Apache PredictionIO
Download binary release of Apache PredictionIO:

```
wget https://downloads.apache.org/predictionio/0.14.0/apache-predictionio-0.14.0-bin.tar.gz
tar zxvf apache-predictionio-0.14.0-bin.tar.gz
```
After that installing dependencies, in this case is Spark.

## Spark
Create vendors folder to host our dependencies

```
cd PredictionIO-0.14.0
mkdir vendors
```

Get Spark version 2.1.3 since PredictionIO binary was build using this version.

```
wget https://archive.apache.org/dist/spark/spark-2.1.3/spark-2.1.3-bin-hadoop2.3.tgz
tar zxvfC spark-2.1.3-bin-hadoop2.3.tgz vendors
```

## Setup storage(postgres)
Download JDBC for postgres(connector) https://jdbc.postgresql.org/download/postgresql-42.2.18.jar and paste into `PredictionIO-0.14.0/lib` folder. Create database, eg: pio. Edit `conf/pio-env.sh` based on your system setup.

```
SPARK_HOME=$PIO_HOME/vendors/spark-2.1.3-bin-hadoop2.3
POSTGRES_JDBC_DRIVER=$PIO_HOME/lib/postgresql-42.2.18.jre7.jar
MYSQL_JDBC_DRIVER=$PIO_HOME/lib/mysql-connector-java-5.1.41.jar # download this connector if using MySQL

PIO_STORAGE_SOURCES_PGSQL_TYPE=jdbc
PIO_STORAGE_SOURCES_PGSQL_URL=jdbc:postgresql://localhost:54321/pio # change according to your postgres setup
PIO_STORAGE_SOURCES_PGSQL_USERNAME=postgres # change according to your postgres setup
PIO_STORAGE_SOURCES_PGSQL_PASSWORD= # change according to your postgres setup
PIO_STORAGE_SOURCES_PGSQL_PORT=54321 # change according to your postgres setup
```

Start services by running `bin/pio eventserver &`. To check status just run `bin/pio status`.

## Create engine
In this setup, i'm using Recommendation template to derive from. Clone this repo https://github.com/apache/predictionio-template-recommender.

```
git clone git@github.com:apache/predictionio-template-recommender.git MyRecommendation
```

After that, we must create Event Server instance. Make sure export env variable for "pio" command into your profile.

```
PATH=$PATH:/home/yourname/PredictionIO-0.14.0/bin;
```

Create Access key and ID

```
cd MyRecommendation
pio app new MyFirstApp

# output
...
[INFO] [App$] Initialized Event Store for this app ID: 1.
[INFO] [App$] Created new app:
[INFO] [App$]       Name: MyFirstApp
[INFO] [App$]         ID: 1
[INFO] [App$] Access Key: SXJi06vJCH643bLLmRddg4guAUP1ce3TQOPYj_FVIW3dIHDjWIpB70I4UWj9Wjlo
```

Make sure run PredictionIO server(if using mysql or postgres). Event server by default will be running on "localhost:7070"
```
pio eventserver &
```

Check status by running
```
pio status
```

If everything OK, should see following message
```
...
(sleeping 5 seconds for all messages to show up...)
Your system is all ready to go.
```

## Sending data into Event Server
Sending our data into Event Server to enable PredictionIO train the data model. Can use following PHP-SDK https://github.com/apache/predictionio-sdk-php. Or run following for example (Rate Event).

```
curl -i -X POST http://localhost:7070/events.json?accessKey=$ACCESS_KEY \
-H "Content-Type: application/json" \
-d '{
  "event" : "rate",
  "entityType" : "user",
  "entityId" : "1",
  "targetEntityType" : "shop",
  "targetEntityId" : "5",
  "properties" : {
    "rating" : 5
  }
  "eventTime" : "2020-10-31T09:39:45.618-08:00"
}'
```

To query from event server, run following:

```
curl -i -X GET "http://localhost:7070/events.json?accessKey=$ACCESS_KEY"
```

Since we require more data to train, this "dummy" data are very useful. Install pyhton SDK:

```
sudo pip install predictionio
```

Get dummy data
```
cd MyRecommendation
curl https://raw.githubusercontent.com/apache/spark/master/data/mllib/sample_movielens_data.txt --create-dirs -o data/sample_movielens_data.txt
python data/import_eventserver.py --access_key $ACCESS_KEY
```

Once success, should see following:

```
Importing data...
1501 events are imported.
```

By now, all the data are stored inside event store (postgres).

## Deploy our engine into web service
Now we can build, train, and deploy the engine

```
cd MyRecommendation
```

Edit `engine.json` file to match our app

```
{
  "id": "default_setting",
  "description": "Default settings",
  "engineFactory": "org.example.recommendation.RecommendationEngine",
  "datasource": {
    "params" : {
      "appName": "MyFirtApp",
      "name": "datasource-name",
    }
  },
  "algorithms": [
    {
      "name": "als",
      "params": {
        "rank": 10,
        "numIterations": 10,
        "lambda": 0.01,
        "seed": 3
      }
    }
  ]
}
```

Build the engine

```
pio build --verbose
```

This command should take few minutes for the first time; all subsequent builds should be less than a minute. Upon successful build, you should see a console message similar to the following:

```
[INFO] [Console$] Your engine is ready for training.
```

Finally train our data that we send into Event Server previously:
```
pio train

# or
pio train -- --driver-memory 8g --executor-memory 8g
```

When your engine is trained successfully, you should see following:

```
[INFO] [CoreWorkflow$] Training completed successfully.
```

Lastly, deploy webservice so that, we can query the Predictive result and show to users.

```
pio deploy
```

If the engine is deployed successfully, you should see following message:

```
[INFO] [HttpListener] Bound to /0.0.0.0:8000
[INFO] [MasterActor] Bind successful. Ready to serve.
```

By default, the deployed engine will be running at http://localhost:8000. Visit this page to see the status. To get the predictive result, run following:

```
curl -H "Content-Type: application/json" \
-d '{ "user": "4", "num": 6 }' http://localhost:8000/queries.json

# user -> user that we want to recommend
# num -> number of recommended items
```

If successfuly, should see the output similar like:

```
{
  "itemScores":[
    {"item":"22","score":4.072304374729956},
    {"item":"62","score":4.058482414005789},
    {"item":"75","score":4.046063009943821},
    {"item":"68","score":3.8153661512945325},
    {"item":"43","score":3.8153661512945325},
    {"item":"23","score":2.1233361512943423}
  ]
}
```

Note: To update the model periodically with new data, simply set up a cron job to call `pio train` and `pio deploy`. The engine will continue to serve prediction results during the re-train process. After the training is completed, pio deploy will automatically shutdown the existing engine server and bring up a new process on the same port. Done.

