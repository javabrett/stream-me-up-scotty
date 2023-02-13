<div align="center" padding=25px>
    <img src="images/confluent.png" width=50% height=50%>
</div>

# <div align="center">Service for Business - Confluent Cloud Workshop</div>
## <div align="center">Lab Guide</div>

## **Agenda**
1. [Log Into Confluent Cloud](#step-1)
1. [Create an Environment and Cluster](#step-2)
1. [Setup ksqlDB](#step-4)
1. [Create an API Key Pair](#step-6)
2. [Getting Started with Apache Kafka and Java](#step-7)
3. [Create Streams and Tables using ksqlDB](#step-9)
4. [Stream Processing with ksqlDB](#step-10)
5. [Connect BigQuery sink to Confluent Cloud](#step-11)
6. [Clean Up Resources](#step-11)
7. [Confluent Resources and Further Testing](#confluent-resources-and-further-testing)

***

## **Prerequisites**

1. Confluent Cloud Account
    * Sign-up for a Confluent Cloud account [here](https://www.confluent.io/confluent-cloud/tryfree/).

    > **Note:** You will create resources during this workshop that will incur costs. New sign-ups receive $400 to spend within Confluent Cloud during their first 60 days. This will cover the cost of resources created during the workshop.

1. Ports `443` and `9092` need to be open to the public internet for outbound traffic. To check, try accessing the following from your web browser:
    * http://portquiz.net:443
    * http://portquiz.net:9092

***

## **Objective**

In this workshop you will learn how Confluent Cloud can enable you to quickly and easily stand up a Confluent Cloud cluster, run some simple Java clients to produce and consume data, run a simple KStreams stream-processing application, and run a ksqlDB stream-processing application.

***

## <a name="step-1"></a>Step 1: Log Into Confluent Cloud

1. First, access Confluent Cloud sign-in by navigating [here](https://confluent.cloud).
1. When provided with the *username* and *password* prompts, fill in your credentials.
    > **Note:** If you're logging in for the first time you will see a wizard that will walk you through the some tutorials. Minimize this as you will walk through these steps in this guide.

***

## <a name="step-2"></a>Step 2: Create an Environment and Cluster

An environment contains Confluent clusters and its deployed components such as Connect, ksqlDB, and Schema Registry. You have the ability to create different environments based on your company's requirements. Confluent has seen companies use environments to separate Development/Testing, Pre-Production, and Production clusters.

1. Click **+ Add environment**.
    > **Note:** There is a *default* environment ready in your account upon account creation. You can use this *default* environment for the purpose of this workshop if you do not wish to create an additional environment.

    * Specify a meaningful `name` for your environment and then click **Create**.
        > **Note:** It will take a few minutes to assign the resources to make this new environment available for use.
    * When prompted, select the free **Essentials** package as the Stream Governance package for your new Environment.  Click **Begin Configuration**, then select **AWS** as your cloud provider, and select **Sydney (ap-southeast-2)** as the Region for Stream Governance, then click **Enable**.

1. Now that you have an environment, let's create a cluster. Select **Create cluster on my own**.
    > **Note**: Confluent Cloud clusters are available in 3 types: **Basic**, **Standard**, and **Dedicated**. Basic is intended for development use cases so you should use that for this lab. Basic clusters only support single zone availability. Standard and Dedicated clusters are intended for production use and support Multi-zone deployments. If you’re interested in learning more about the different types of clusters and their associated features and limits, refer to this [documentation](https://docs.confluent.io/current/cloud/clusters/cluster-types.html).

    * Choose the **Basic** cluster type.

    * Click **Begin Configuration**.

    * Choose **AWS** as your Cloud Provider and your preferred Region (**Sydney (ap-southeast-2)**).

    * When prompted for payment info, select **Skip payment** in the bottom-right corner.  The $400 credit provided with new accounts is more than sufficient for this workshop.

    * Specify a meaningful **Cluster Name** and then review the associated *Configuration & Cost*, *Usage Limits*, and *Uptime SLA* before clicking **Launch Cluster**.

***

## <a name="step-4"></a>Step 4: Setup ksqlDB

1. On the navigation menu, select **ksqlDB** and click **Create cluster myself**.

1. Select **Global Access** and then **Continue**.

1. Name your ksqlDB application and set the streaming units to 1.

1. Click **Launch Application**!
> **Note:** A streaming unit, also known as a Confluent Streaming Unit (CSU), is the unit of pricing for Confluent Cloud ksqlDB. A CSU is an abstract unit that represents the linearity of performance.

***

## <a name="step-6"></a>Step 6: Create an API Key Pair

1. Select **API keys** on the navigation menu.

1. If this is your first API key within your cluster, click **Create key**. If you have set up API keys in your cluster in the past and already have an existing API key, click **+ Add key**.

1. Select **Global Access**, then click Next.

1. Save your API key and secret - you will need these during the workshop.

1. After creating and saving the API key, you will see this API key in the Confluent Cloud UI in the **API keys** tab. If you don’t see the API key populate right away, refresh the browser.

***

## <a name="step-7"></a>Step 7: Getting Started with Apache Kafka and Java

Complete the exercises outlined in [Getting Started with Apache Kafka and Java](https://developer.confluent.io/get-started/java).

***

## <a name="step-9"></a>Step 9: Create Streams and Tables using ksqlDB

Visit https://developer.confluent.io/tutorials/location-based-alerting/confluent.html and read the ksqlDB use-case.  Then visit your ksqlDB cluster editor, and run the following query:

```
SET 'auto.offset.reset' = 'earliest';

-- Creates a table of merchant data including the calculated geohash
CREATE TABLE merchant_locations (
  id INT PRIMARY KEY,
  description VARCHAR,
  latitude DECIMAL(10,7),
  longitude DECIMAL(10,7),
  geohash VARCHAR
) WITH (
  KAFKA_TOPIC = 'merchant-locations',
  VALUE_FORMAT = 'JSON',
  PARTITIONS = 6
);

-- Creates a table to lookup merchants based on a
--    substring (precision) of the geohash
CREATE TABLE merchants_by_geohash
WITH (
  KAFKA_TOPIC = 'merchant-geohash',
  FORMAT = 'JSON',
  PARTITIONS = 6
) AS
SELECT
  SUBSTRING(geohash, 1, 6) AS geohash,
  COLLECT_LIST(id) as id_list
FROM merchant_locations
GROUP BY SUBSTRING(geohash, 1, 6)
EMIT CHANGES;

-- Creates a stream of user location data including the calculated geohash
CREATE STREAM user_locations (
  id INT,
  latitude DECIMAL(10,7),
  longitude DECIMAL(10,7),
  geohash VARCHAR
) WITH (
  KAFKA_TOPIC = 'user-locations',
  VALUE_FORMAT = 'JSON',
  PARTITIONS = 6
);

-- Creates a stream of alerts when a user's geohash based location roughly
--    intersects a collection of merchants locations from the
--    merchants_by_geohash table.
CREATE STREAM alerts_raw
WITH (
  KAFKA_TOPIC = 'alerts-raw',
  VALUE_FORMAT = 'JSON',
  PARTITIONS = 6
) AS
SELECT
  user_locations.id as user_id,
  user_locations.latitude AS user_latitude,
  user_locations.longitude AS user_longitude,
  SUBSTRING(user_locations.geohash, 1, 6) AS user_geohash,
  EXPLODE(merchants_by_geohash.id_list) AS merchant_id
FROM user_locations
LEFT JOIN merchants_by_geohash ON SUBSTRING(user_locations.geohash, 1, 6) =
  merchants_by_geohash.geohash
PARTITION BY null
EMIT CHANGES;

-- Creates a stream of promotion alerts to send a user when their location
--    intersects with a merchant within a specified distance (0.2 KM)
CREATE STREAM promo_alerts
WITH (
  KAFKA_TOPIC = 'promo-alerts',
  VALUE_FORMAT = 'JSON',
  PARTITIONS = 6
) AS
SELECT
  alerts_raw.user_id,
  alerts_raw.user_geohash,
  merchant_locations.description AS merchant_description,
  CAST(
    GEO_DISTANCE(alerts_raw.user_latitude, alerts_raw.user_longitude,
                 merchant_locations.latitude, merchant_locations.longitude,
        'KM') * 1000 AS INT) as distance_meters,
  STRUCT(lat := CAST(alerts_raw.user_latitude AS DOUBLE), lon := CAST(alerts_raw.user_longitude AS DOUBLE)) AS geopoint
FROM alerts_raw
LEFT JOIN merchant_locations on alerts_raw.merchant_id = merchant_locations.id
WHERE GEO_DISTANCE(
        alerts_raw.user_latitude, alerts_raw.user_longitude,
        merchant_locations.latitude, merchant_locations.longitude, 'KM') < 0.2
PARTITION BY null
EMIT CHANGES;
```

Then generate sample data and read-back the results:

```
-- For the purposes of this recipe when testing by inserting records manually,
--  a short pause between these insert groups is required. This allows
--  the merchant location data to be processed by the merchants_by_geohash
--  table before the user location data is joined in the alerts_raw stream.
INSERT INTO MERCHANT_LOCATIONS (id, latitude, longitude, description, geohash) VALUES (1, 14.5486606, 121.0477211, '7-Eleven RCBC Center', 'wdw4f88206fx');
INSERT INTO MERCHANT_LOCATIONS (id, latitude, longitude, description, geohash) VALUES (2, 14.5473328, 121.0516176, 'Jordan Manila', 'wdw4f87075kt');
INSERT INTO MERCHANT_LOCATIONS (id, latitude, longitude, description, geohash) VALUES (3, 14.5529666, 121.0516716, 'Lawson Eco Tower', 'wdw4f971hmsv');
```

Wait 10 seconds, then:

```
INSERT INTO USER_LOCATIONS (id, latitude, longitude, geohash) VALUES (1, 14.5472791, 121.0475401, 'wdw4f820h17g');
INSERT INTO USER_LOCATIONS (id, latitude, longitude, geohash) VALUES (2, 14.5486952, 121.0521851, 'wdw4f8e82376');
INSERT INTO USER_LOCATIONS (id, latitude, longitude, geohash) VALUES (2, 14.5517401, 121.0518652, 'wdw4f9560buw');
INSERT INTO USER_LOCATIONS (id, latitude, longitude, geohash) VALUES (2, 14.5500341, 121.0555802, 'wdw4f8vbp6yv');
```

Finally, read the results:

Set property `auto.offset.reset` to `Earliest` in the query form.  Then run:

```
SELECT * FROM promo_alerts EMIT CHANGES LIMIT 3;
```

***

## <a name="step-10"></a>Step 10: Stream Processing with ksqlDB

1. Create a **PRODUCT_TXN_PER_HOUR** table based on the **INVENTORY** table and **TRANSACTIONS** stream.  Make sure to first set 'auto.offset.reset' = 'earliest' before running the query.

```SQL
CREATE TABLE PRODUCT_TXN_PER_HOUR WITH (FORMAT='AVRO') AS
SELECT T.FULLDOCUMENT->PROD_ID,
       COUNT(*) AS TXN_PER_HOUR,
       MAX(I.TXN_HOUR) AS EXPECTED_TXN_PER_HOUR,
       (CAST(MAX(I.AVAILABLE) AS DOUBLE)/ CAST(MAX(I.CAPACITY) AS DOUBLE))*100 AS STOCK_LEVEL, I.NAME AS PRODUCT_NAME
FROM  TRANSACTIONS T
      LEFT JOIN INVENTORY I
      ON T.FULLDOCUMENT->PROD_ID = I.PRODUCT_ID
WINDOW HOPPING (SIZE 1 HOUR, ADVANCE BY 5 MINUTES)
GROUP BY T.FULLDOCUMENT->PROD_ID,
         I.NAME;
```

2. Create a stream on the underlying topic backing the **PRODUCT_TXN_PER_HOUR** table that you just created
    * Determine the name of the backing topic by navigating to the **Topics** tab on the left hand side menu under **Cluster**.  You should see a topic that begins with **pksqlc-**… and ends with **PRODUCT_TXN_PER_HOUR**. Click on this topic and copy down this topic name as it will be required for the following query
    * Create the stream based on the backing topic for PRODUCT_TXN_PER_HOUR table

```SQL
CREATE STREAM PRODUCT_TXN_PER_HOUR_STREAM WITH (KAFKA_TOPIC='pksqlc-...PRODUCT_TXN_PER_HOUR', FORMAT='AVRO');
```

3. Now you want to perform a query to see which products you should create promotions for based on the following criteria
    * High inventory level (>80% of capacity)
    * Low transactions (< expected transactions/hour)

```SQL
CREATE STREAM ABC_PROMOTIONS AS
SELECT  ROWKEY,
        TIMESTAMPTOSTRING(ROWTIME,'yyyy-MM-dd HH:mm:ss','Europe/London') AS TS,
        AS_VALUE(ROWKEY -> PROD_ID) AS PROD_ID ,
        ROWKEY -> PRODUCT_NAME AS PRODUCT_NAME,
        STOCK_LEVEL ,
        TXN_PER_HOUR ,
        EXPECTED_TXN_PER_HOUR
   FROM PRODUCT_TXN_PER_HOUR_STREAM
WHERE TXN_PER_HOUR < EXPECTED_TXN_PER_HOUR
  AND  STOCK_LEVEL > 80
  ;
```

4. Query the results.  Make sure to set ‘auto.offset.reset=earliest’

```SQL
SELECT * FROM ABC_PROMOTIONS EMIT CHANGES;
```

***

## <a name="step-11"></a>Step 11: Connect BigQuery sink to Confluent Cloud

The next step is to sink data from Confluent Cloud into BigQuery using the [fully-managed BigQuery Sink connector](https://docs.confluent.io/cloud/current/connectors/cc-gcp-bigquery-sink.html). The connector will send real time data on promotions into BigQuery.

1. First, you will create the connector that will automatically create a BigQuery table and populate that table with the data from the promotions topic within Confluent Cloud. From the Confluent Cloud UI, click on the Connectors tab on the navigation menu and select **+Add connector**. Search and click on the BigQuery Sink icon.

2. Enter the following configuration details. The remaining fields can be left blank.

<div align="center">

| Setting            | Value                        |
|------------------------|-----------------------------------------|
| `Topics`      | pksqlc-...ABC_PROMOTIONS |
| `Name`              | BigQuerySinkConnector                 |
| `Input message format`           | Avro            |
| `Kafka API Key`    | From step 6           |
| `Kafka API Secret` | From step 6              |
| `GCP credentials file`    | [JSON file link](https://drive.google.com/drive/folders/1EOYZkyWvpnCydBCv2hv8DMH8FlZKuXKV)            |
| `Project ID`    | Will be provided during workshop          |
| `Dataset`    | Will be provided during workshop          |
| `Auto create tables`    | True          |
| `Tasks`    | 1             |

</div>

3. Click on **Next**.

4. Before launching the connector, you will be brought to the summary page.  Once you have reviewed the configs and everything looks good, select **Launch**.

5. This should return you to the main Connectors landing page. Wait for your newly created connector to change status from **Provisioning** to **Running**.

6. Shortly after, the workshop instructor will switch over to the BigQuery page within Google Console to show that a table matching the topic name you used when creating the BigQuery connector in Confluent Cloud has been created within the **workshop** dataset.  Clicking the table name should open a BigQuery editor for it:



***

## <a name="step-12"></a>Step 12: Clean Up Resources

Deleting the resources you created during this workshop will prevent you from incurring additional charges.

1. The first item to delete is the ksqlDB application. Select the Delete button under Actions and enter the Application Name to confirm the deletion.

2. Delete the BigQuery sink connector by navigating to **Connectors** in the navigation panel, clicking your connector name, then clicking the trash can icon in the upper right and entering the connector name to confirm the deletion.

3. Delete the Cluster by going to the **Settings** tab and then selecting **Delete cluster**

4. Delete the Environment by expanding right hand menu and going to **Environments** tab and then clicking on **Delete** for the associated Environment you would like to delete

***

## <a name="confluent-resources-and-further-testing"></a>Confluent Resources and Further Testing

Here are some links to check out if you are interested in further testing:

* Confluent Cloud [Basics](https://docs.confluent.io/cloud/current/client-apps/cloud-basics.html)

* [Quickstart](https://docs.confluent.io/cloud/current/get-started/index.html) with Confluent Cloud

* Confluent Cloud ksqlDB [Quickstart](https://docs.confluent.io/cloud/current/get-started/ksql.html)

* Confluent Cloud [Demos/Examples](https://docs.confluent.io/platform/current/tutorials/examples/ccloud/docs/ccloud-demos-overview.html)

* ksqlDB [Tutorials](https://kafka-tutorials.confluent.io/)

* Full repository of Connectors within [Confluent Hub](https://www.confluent.io/hub/)
