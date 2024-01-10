---
title: "Spark Stream Aggregation: A Beginner’s Journey with Practical Code Example"
description: ""
date: 2024-01-03T17:29:06.599Z
preview: ""
draft: false
tags: ["spark","spark streaming aggregation"]
categories: ["streaming","spark"]
banner: "https://cdn-images-1.medium.com/max/9706/0*EiqbOjU9ysuUXIKz"
---

## Spark Stream Aggregation: A Beginner’s Journey with Practical Code Example

Welcome to the fascinating world of Spark Stream Aggregation! This blog is tailored for beginners, featuring hands-on code examples from a real-world scenario of processing vehicle data. I suggest reading the blog first without the code and then reading the code with the blog.

## Setting up Spark Configuration, Imports & Parameters

First, let’s understand our setup. We configure the Spark environment to use RocksDB for state management, enhancing the efficiency of our stream processing:

    spark.conf.set("spark.sql.streaming.stateStore.providerClass", "com.databricks.sql.streaming.state.RocksDBStateStoreProvider")

    !pip install faker_vehicle
    !pip install faker

    from faker import Faker
    from faker_vehicle import VehicleProvider
    from pyspark.sql import functions as F
    from pyspark.sql.types import StringType
    import uuid
    from pyspark.sql.streaming import StreamingQuery
    import logging
    
    logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    logger = logging.getLogger(__name__)
    
    # define schema name and where should the table be stored
    schema_name = "test_streaming_joins"
    schema_storage_location = "/tmp/CHOOSE_A_PERMANENT_LOCATION/"
    target_table = f"{schema_name}.streaming_aggregation"
    checkpoint_location= f"{schema_storage_location}{target_table}/_checkpoint/",
    silver_target_table=  f"{schema_name}.silver_streaming"
    silver_checkpoint_locattion = f"{schema_storage_location}{silver_target_table}/_checkpoint/",
    column_to_watermark_on = "timestamp"
    how_late_can_the_data_be = "3 minutes"
    
    create_schema_sql = f"""
        CREATE SCHEMA IF NOT EXISTS {schema_name}
        COMMENT 'This is {schema_name} schema'
        LOCATION '{schema_storage_location}';
    """
    logger.info(f"Executing SQL: {create_schema_sql}")
    spark.sql(create_schema_sql)

![Photo by [Kalen Emsley](https://unsplash.com/@kalenemsley?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral)](https://cdn-images-1.medium.com/max/9706/0*EiqbOjU9ysuUXIKz)

## Simulating Data Streams

Using the Faker library, we simulate a vehicle data stream. This approach helps us create a realistic data processing environment that is crucial for our learning:

    # Using Faker to define functions for fake data generation
    fake = Faker()
    fake.add_provider(VehicleProvider)
    
    event_id = F.udf(lambda: str(uuid.uuid4()), StringType())
    
    vehicle_year_make_model = F.udf(fake.vehicle_year_make_model)
    vehicle_year_make_model_cat = F.udf(fake.vehicle_year_make_model_cat)
    vehicle_make_model = F.udf(fake.vehicle_make_model)
    vehicle_make = F.udf(fake.vehicle_make)
    vehicle_model = F.udf(fake.vehicle_model)
    vehicle_year = F.udf(fake.vehicle_year)
    vehicle_category = F.udf(fake.vehicle_category)
    vehicle_object = F.udf(fake.vehicle_object)
    
    latitude = F.udf(fake.latitude)
    longitude = F.udf(fake.longitude)
    location_on_land = F.udf(fake.location_on_land)
    local_latlng = F.udf(fake.local_latlng)
    zipcode = F.udf(fake.zipcode)
    
    # COMMAND ----------
    
    # MAGIC %md
    # MAGIC ### Generate Streaming source data at your desired rate
    
    # COMMAND ----------
    
    # Generate streaming data
    def generated_vehicle_and_geo_df(rowsPerSecond: int =100, numPartitions: int = 10):
        logger.info("Generating vehicle and geo data frame...")
        return (
            spark.readStream.format("rate")
            .option("numPartitions", numPartitions)
            .option("rowsPerSecond", rowsPerSecond)
            .load()
            .withColumn("event_id", event_id())
            .withColumn("vehicle_year_make_model", vehicle_year_make_model())
            .withColumn("vehicle_year_make_model_cat", vehicle_year_make_model_cat())
            .withColumn("vehicle_make_model", vehicle_make_model())
            .withColumn("vehicle_make", vehicle_make())
            .withColumn("vehicle_year", vehicle_year())
            .withColumn("vehicle_category", vehicle_category())
            .withColumn("vehicle_object", vehicle_object())
            .withColumn("latitude", latitude())
            .withColumn("longitude", longitude())
            .withColumn("location_on_land", location_on_land())
            .withColumn("local_latlng", local_latlng())
            .withColumn("zipcode", zipcode())
        )
    
    # You can uncomment the below display command to check if the code in this cell works
    display(generated_vehicle_and_geo_df())

## Aggregation Modes in Spark Streaming

Stream Aggregation in Spark lets us process continuous data streams. Our code demonstrates how to generate and aggregate streaming data. Spark Streaming provides three primary modes for data output during stream processing:

### Complete Mode

* Functionality: Outputs the entire result table after every update.

* Use Cases: Ideal for situations where the complete dataset is essential for each update, such as dashboards reflecting total counts.

### Append Mode

* Functionality: Only new rows added since the last update are output.

* Use Cases: Suitable for non-aggregate queries where each data point is independent, like tracking unique events.

### Update Mode

* Functionality: Outputs only the rows that have changed since the last update.

* Use Cases: Perfect for aggregate queries where you need insights into how aggregated metrics evolve over time.

Each mode caters to different requirements, offering flexibility in streaming applications.

## Applying Aggregation in Practice

Our code showcases these modes in action, applying them to our simulated vehicle data stream:

    def streaming_aggregation(rows_per_second: int = 100, 
                              num_partitions: int = 10,
                              how_late_can_the_data_be :str = "30 minutes",
                              window_duration: str = "1 minutes",
                              checkpoint_location: str = checkpoint_location,
                              output_table_name: str = target_table) -> StreamingQuery:
        """
        Aggregate streaming data and write to a Delta table.
        
        Parameters:
        - rows_per_second (int): Number of rows per second for generated data.
        - num_partitions (int): Number of partitions for the generated data.
        - window_duration (str): Window duration for aggregation.
        - checkpoint_location (str): Path for checkpointing.
        - output_table_name (str): Name of the output Delta table.
    
        Returns:
        - StreamingQuery: Spark StreamingQuery object.
        """
        
        logger.info("Starting streaming aggregation...")
    
        raw_stream = generated_vehicle_and_geo_df(rows_per_second, num_partitions)
    
        aggregated_data = (
            raw_stream
            .withWatermark(column_to_watermark_on, how_late_can_the_data_be)
            .groupBy(
                F.window(column_to_watermark_on, window_duration),
                "zipcode"
            )
            .agg(
                F.min("vehicle_year").alias("oldest_vehicle_year")
            )
        )
    
        query = (
            aggregated_data
            .writeStream
            .queryName(f"write_stream_to_delta_table: {output_table_name}")
            .format("delta")
            .option("checkpointLocation", checkpoint_location)
            .outputMode("append")
            .toTable(output_table_name)
              # This actually starts the streaming job
        )
        
        logger.info(f"Streaming query started with ID: {query.id}")
        logger.info(f"Current status of the query: {query.status}")
        logger.info(f"Recent progress updates: {query.recentProgress}")
    
        # If you want to programmatically stop the query after some condition or time, you can do so. 
        # For the sake of this example, I am NOT stopping it. But if you need to, you can invoke:
        # query.stop()
        # logger.info("Streaming query stopped.")
    
        return query
    
    
    
    streaming_aggregation(
       checkpoint_location = checkpoint_location,
       output_table_name = target_table,
       how_late_can_the_data_be = "5 minutes"
    )

### What is most critical to understand is the Watermarking column.

* Understand Watermarking: Watermarking in Spark Streaming specifies how long the system should wait for late-arriving data. It helps manage state by removing or ignoring old data that’s too late to be considered relevant.

* Identify Event Time Column: Watermarking is typically applied to the column that represents the event time — the time at which the event actually occurred, as opposed to the time the system processed it. Choose a column that reliably indicates the event time.

* Consistency and Accuracy: Ensure the chosen column has consistent and accurate timestamps. Only consistent or accurate timestamps can lead to the correct processing of late data. For example, the column should generally be monotonically increasing and not have future-dated timestamps.

* Data Granularity: Consider the granularity of the timestamp. For example, if your timestamps are recorded to the nearest second, minute, or hour, ensure this granularity level aligns with your processing and latency requirements.

* Business Logic: Align your choice with the business logic of your application. The column should be relevant to how you want to handle late data in your specific use case.

* Watermarking Strategy: Decide on your watermarking strategy (e.g., how much delay to allow) based on the nature of your data and application requirements. This will also influence the choice of the column.

* Testing and Monitoring: After implementing watermarking, continuously monitor the system’s behaviour to ensure it handles late data as expected. Adjust the watermarking settings and the chosen column if necessary.

* Code Implementation: In Spark Structured Streaming, you can define a watermark on a DataFrame using the withWatermark method. For example:

    df.withWatermark("timestampColumn", "1 hour")

## Best Practices and Considerations

* Choose the right column to watermark on

* Choosing the Right Mode: Understand your data and requirements to select the most appropriate mode.

* State Management: Leverage RocksDB for efficient state management, especially in stateful aggregations. We had already set it up on top of the notebook.

* Understand different data windowing methods.

* Scalability: Design your aggregation logic with scalability in mind, especially for applications expected to grow over time.

* Notion about your backfill strategy. Depending on the source, you might have to order your backfill data when you stream it. Kafka and kinesis are ordered by design. However, file-based sources need to be ordered before they can be streamed.

## Download the code

[https://github.com/jiteshsoni/material_for_public_consumption/blob/main/notebooks/spark_stream_aggregations.py](https://github.com/jiteshsoni/material_for_public_consumption/blob/main/notebooks/spark_stream_aggregations.py)

## Conclusion

This blog provides a beginner-friendly introduction to Spark Stream Aggregation, supported by practical code examples. Dive in and explore the world of real-time data processing with Spark!

## Thank You for Reading!

I hope you found this article helpful and informative. If you enjoyed this post, please consider giving it a clap 👏 and sharing it with your network. Your support is greatly appreciated!

— [**CanadianDataGuy](https://canadiandataguy.com/)**
