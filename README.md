Project Documentation for Earthquake Data Processing Pipeline


Project Overview

This project is designed to fetch earthquake data from the USGS API, process it, and load it into Google Cloud Storage (GCS) and BigQuery. The pipeline performs data transformations and type enforcement, storing intermediate results in Parquet format and finally writing processed data to BigQuery for analysis.


Components and Flow

1) Using Dataflow:




2) Using Dataproc:





Google cloud services used to orchestrate this ETL (Extract, Transform, Load) process

1. Apache Beam, with the Dataflow runner
2. Dataproc 




Data Sources and API Details

Data Source: USGS Earthquake API
API Endpoint: https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_month.geojson
Data Format: JSON, in GeoJSON structure
Fields of Interest:
  features.properties.mag_: Earthquake magnitude
  features.properties.place_: Location description
  features.geometry.coordinates_: Longitude, latitude, depth


 Pipeline Architecture

This pipeline is divided into three layers following the Bronze, Silver, and Gold layer model:

  Bronze Layer: Data is fetched from the USGS API and stored as raw JSON files in GCS.
  Silver Layer: Raw data is transformed, flattened, and stored in Parquet format.
  Gold Layer: Transformed data is uploaded to BigQuery for analysis.


Building pipeline with Dataflow 

1. Configuration and Initialization

Dependencies: Key Python libraries include apache_beam, requests, and pyarrow. Google Cloud-specific options are used to define credentials and pipeline options.
Environment Setup: Google Cloud credentials are set up via an environment variable, and configurations such as project ID, bucket name, temporary paths, and job options for Dataflow are specified.


2. Stage 1: Fetch and Store Raw Data

Function: _fetch_data_from_api(api_url)_
  Fetches earthquake data from the specified USGS API URL, handling potential connection errors.
  Input: API URL as a string
  Output: JSON data if successful; None otherwise.


DoFn: _ProcessEarthquakeData_
   Converts the fetched JSON data into JSON strings, preparing it for writing to GCS as raw data.
   Output: JSON strings ready for writing to GCS.


Pipeline Step: Writing Raw Data to GCS
  GCS Path: Configured gcs path that leads to bronze layer to store raw data
  File Format: json


Optimization Techniques
  Batch API Requests: Optimized for minimal data requests and parallel execution.



4. Stage 2: Transform and Process Data

DoFn: _FlattenGeoJsonData_
  Flattens GeoJSON earthquake data into a structured dictionary format, extracting essential fields for further processing.
  Output: Flattened JSON object with fields like id, magnitude, place, longitude, latitude, and depth etc.


DoFn: _ProcessData_
  Adds an _area_ field to the data by parsing and extracting the specific area from the _place_ field.
  Helper Function: _extract_area(place)_
     Uses regex to extract location information from the _place_ field.
     Output: Extracted area, if found.


DoFn: _AddIngestionDate_
   Adds new _ingestion_date_ field for daily ingestion and transformation


DoFn: _EnforceDataTypes_
  Enforces data types to ensure data aligns with the schema requirements.
  Output: Dictionary with fields cast to types compatible with the Parquet schema and BigQuery table requirements.


Pipeline Step: Writing Transformed Data to Parquet
  Path: Configured gcs path that leads to silver layer to store transformed data
  File Format: .parquet
  Schema: Defined using PyArrow to specify data types, ensuring compatibility with BigQuery.


Optimization Techniques
  Flatten Data: Optimized for minimal shuffling and efficient parsing.
  Transform Data: Pre-processes fields for schema compliance and uses early data type enforcement.
  Write to Parquet: Parquet files ensure efficient data storage and faster read/write times.


5. Stage 3: Load Data to BigQuery

Pipeline Step: Loading Parquet Data into BigQuery
  Source: Reads from Parquet files in GCS.
  Destination: Writes to BigQuery, specifying a schema to ensure data compatibility.
  Table Schema: Configured with BigQuery-compatible data types.
  Table Path: {project_id}.{dataset_id}.{table_id}
  Write Mode: WRITE_TRUNCATE to overwrite existing data, CREATE_IF_NEEDED to create       the table if it does not exist.


Optimization Techniques
  BigQuery Schema Matching:
    Strict schema with compatible data types has been defined, ensuring that all fields match BigQuery’s expectations. This prevents schema auto-detection issues and avoids conversion overhead during data load.
  Batch Write Mode: 
Used the FILE_LOADS write method, which is optimized for loading larger data volumes into BigQuery by staging files in GCS before bulk loading. This method is faster and more reliable for high-volume data.

  
  
6. Deployment and Environment Setup
Prerequisites: 
   Google Cloud SDK: To interact with Google Cloud services.
   Apache Beam and Python: Install Beam SDK and required Python packages.
   BigQuery and GCS Permissions: Ensure that service accounts have appropriate roles.

Deployment Steps: 
  Set environment variables (e.g., GOOGLE_APPLICATION_CREDENTIALS).
  Deploy Beam pipeline to Dataflow using PipelineOptions.
  Set up Airflow for periodic runs.

 Pipeline Options

General Configuration

  Project ID: earthquake-data-usgs
  Region: us-central1
  Runner: DataflowRunner

Google Cloud Storage Paths

  Bronze Layer: Raw data in JSON format
  Silver Layer: Processed data in Parquet format, ready for further analysis or BigQuery ingestion.

BigQuery Configuration
  Dataset: earthquake_db
  Tables: historical_data_dataflowRunner, daily_data_dataflowRunner
  Write Method: FILE_LOADS, optimized for larger data volumes.


Conclusion

This data pipeline enables efficient processing of real-time earthquake data, with automated ingestion, transformation, and loading into GCS and BigQuery. The pipeline design promotes modularity, making it extensible for future use cases like enhanced data validation or additional transformations. The Dataflow runner ensures scalability, supporting high-throughput processing in the cloud.












Building pipeline with Dataproc/pyspark 


 1. Source Code Structure:

   _main.py_: Contains the main pipeline logic.
   _utils.py_: Defines helper functions for API calls, data transformations, and GCS upload functionality.

 2. Environment Setup:

Libraries:
  _requests_: For handling HTTP requests to fetch data from USGS API.
   _google.cloud_: For interacting with Google Cloud Storage and BigQuery.
   _pyspark_: For distributed data processing.
  _os_: For setting up environment variables.


Optimization:

Environment Configuration: Used environment variables for sensitive data like API keys or paths to Google Cloud credentials, keeping the setup secure and manageable.

Dependency Management: Only imported necessary libraries and consider using virtual environments for dependency isolation.


3.Class Definition: Utils

The Utils class contains utility functions for fetching data from the API, uploading files to GCS, and transforming data.

 _fetch_api_request(self, url)_: Connects to the USGS API, verifies the connection status, and returns earthquake data in JSON format.

 _upload_to_gcs_bronze(self, file_name, data, bucket_name)_: Writes the fetched data to a JSON file and uploads it to GCS under the Bronze layer.

 _read_data_from_gcs(self, bucket_name, source_blob_name)_: Reads JSON data directly from GCS and loads it into a dictionary.

 _flatten_data(self, spark, geojson_data)_: Parses and flattens JSON data into a Spark DataFrame with a defined schema.

 Optimization: 

      Schema Enforcement: Defined schema before creating the DataFrame to prevent Spark from inferring data types, which can be costly.
Optimize Data Transformation: Used built-in Spark functions wherever possible instead of Python loops, as Spark’s functions are more efficient.

 _upload_to_gcs_silver(self, df, bucket_name, destination_blob_name)_: Saves the transformed DataFrame as a Parquet file in GCS.
   
Optimization:
    
Efficient File Format: Used Parquet over JSON to leverage its columnar storage, which saves storage space and speeds up data loading in BigQuery.


4. Main Pipeline Execution

The main execution script orchestrates the entire pipeline, from data fetching to loading in BigQuery.

5. Execution Flow

1. Data Fetching: Data is pulled from the USGS API.
2. Bronze Layer: Raw data is saved as JSON in GCS.
3. Data Transformation:

  JSON data is parsed and flattened.
  Converted into a Spark DataFrame.
  Stored as Parquet in GCS (Silver Layer).

4. Gold Layer: The Parquet data is loaded into BigQuery with an additional ingestion_date field.


Scheduling and Automation

The pipeline is scheduled to run periodically using Apache Airflow.
 DAG Definition: An Airflow DAG defines the sequence of tasks to start the Dataflow pipeline.
  Frequency: Set schedule intervals (e.g., @daily or @weekly) in the DAG to control pipeline runs.
  Airflow DAG scripts:
     earthquake_dataflow_dag.py
     earthquake_dataproc_dag.py




This documentation now provides not only a comprehensive overview of the pipeline but also incorporates specific optimizations that improve performance, reduce costs, and make the workflow scalable for larger datasets.
