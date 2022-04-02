# Data Engineering Zoomcamp Final Project 
This repo contains the work carried out as part of the final project for the DE zoomcamp. This course is conducted by Alexey Grigorev. You can take the course at any time and it is free. Links to course mentioned below:
* https://github.com/DataTalksClub/data-engineering-zoomcamp
* https://www.youtube.com/c/DataTalksClub/playlists

## 1. About the Project
In response to the economic devastation of the COVID-19 pandemic, the United States Congress passed the Coronavirus Aid, Relief, and Economic Security (CARES) act. 

"The Paycheck Protection Program established by the CARES Act, is implemented by the Small Business Administration with support from the Department of the Treasury.  This program provides small businesses with funds to pay up to 8 weeks of payroll costs including benefits. Funds can also be used to pay interest on mortgages, rent, and utilities.

The Paycheck Protection Program prioritizes millions of Americans employed by small businesses by authorizing up to $659 billion toward job retention and certain other expenses." -> [Department of the Treasury](https://home.treasury.gov/policy-issues/coronavirus/assistance-for-small-businesses/paycheck-protection-program)

You can read more about the bill [here](https://www.congress.gov/bill/116th-congress/house-bill/748)

This project aims to create a pipeline using a mix of technologies to get the data from the SBA website, make the appropriate transformations, upload to Google Cloud Service, and finally present the data in a dashboard using Google Data Studio

**Data set**: https://data.sba.gov/dataset/ppp-foia

**NAICS Codes**: https://www.census.gov/naics/?48967

 
## 2. Process

I will be using GCP cloud storage as the data lake and Airflow in order to get the data into GCP. 

The data workflow will consist of the following steps:
1) Set up GCP and create our project 
2) Use terraform to create our infrastructure in GCP 
3) Using Airflow through Docker, run our pipeline by:
4) Downloading the csv files from the SBA website 
5) Use Pyspark in order to make necessary transformations to our CSV files and save as Parquet
6) Upload the parquet files into Google Cloud Storage 
7) Upload our files in Google Cloud Storage into a table in BigQuery 
8) Take our Bigquery data and create a dashboard in Google data studio 

## 3. Setting up GCP 
Before working with our data, we first need to set up GCP:
* Create an account with Google email 
* Set up the project once you are on the google cloud console
* setup service account and authentication for this project and download auth-keys
    * go to IAM & Admin -> service accounts -> create service account
    * go to manage keys -> create new json key
    * save to key folder in project 
* download SDK for local setup 
    * https://cloud.google.com/sdk
* set up environment variable to point to your downloaded auth-keys 

    <code> export GOOGLE_APPLICATION_CREDENTIALS="<path/to/your/service-account-authkeys>.json"</code>

    Refresh token/session, and verify authentication

    <code>gcloud auth application-default login</code>

* create access to 
    * storage admin
    * storage object admin 
    * BigQuery admin
* Under APIs and Services, enable the following:  
    * Identity and Access Management (IAM) API
    * IAM service account credentials API

* In the keys folder, replace the text file with your json file containing your key 
 
## 4. Terraform 

https://www.terraform.io/downloads

**Creating GCP Infrastructure with Terraform**

Within the terraform folder there are 3 files:
* .terraform-version: just has the version number of terraform 
* main.tf:
    * terraform 
        * this section of the code declares the terraform version, the backend (whether local, gcs, or s3), and the required providers which specifies public libraries where we will be getting functions from (kind of like python libraries and pip)
    * provider
        * adds a set of predefined resource types and data sources that terraform can manage such as google cloud storage bucket, data lake bucket, bigquery dataset, etc 
        * where the variable for credentials would go
    * resource 
        * Specify and configure the resources 

    * Do not change anything in this file. All changes should be made in the variables file 

* variables.tf: 
    * This file is where you specify the instances in the main file that use "var"
    * locals: similar to constants 
    * vairables: 
        * generally passed during runtime
        * can have default values 
    * change all information that have comments to what would be applicable for you 

   
**Execution Steps**

Once the files are established, you can run these commands within the folder (except for destroy)
* <code>terraform init</code>: Initialize and Install
* <code>terraform plan</code>: Match changes against the previous state
* <code>terraform apply</code>: Apply changes to cloud 
* <code>terraform destroy</code>: Remove your stack from the cloud 

## 5. Airflow 

We will be using Apache Airflow through docker. 

In docker, make sure that everything is up to date and that you have the correct amount of ram allocated (5gb min, ideally 8)

### Setup 
Create an airflow folder and within the folder, import the official image and setup form the latest airflow version. This will download the the docker-compose.yaml file in the folder 

<code>curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.2.4/docker-compose.yaml'</code>

Next we need to set the aiflow user and create the dags, logs, and plugins folders 

<code>mkdir -p ./dags ./logs ./plugins</code>
<code>echo -e "AIRFLOW_UID=$(id -u)" > .env</code>

* The dags directory is where you will be storing the airflow pipelines 
* The logs directory stores the log information between the scheduler and the workers 
* The plugins folder stores any custom function that are used within your dags 

In order to ensure that airflow will work with GCP, we will create a custom Dockerfile and take the base image from our docker-compose.yaml file. The necessary google requirements will then be installed through the Dockerfile. (https://airflow.apache.org/docs/docker-stack/recipes.html) We then connect our docker-compose file to our Dockerfile by replacing the image with build. 

Our Dockerfile essentially downloads the google cli tools and saves into our path, sets our user, and then imports the python packages from a saved requirements file 

While we are editing the docker-compose.yaml file, also change the AIRFLOW__CORE__LOAD_EXAMPLES to false or else it will load predefined dag examples. 

## 6. Aiflow DAGS

 The  DAG that is outlined below is not the most efficient. Since there are 13 large csv files to be downloaded from the SBA website, within our DAG file I created a list with the links. Then I set the start date to be 13 days prior to the current date of the run. Once Airflow runs, it counts the difference in days between the run date and the start date and chooses the corresponding link in the list. The approach is pretty roundabout, but I have created it this way for a few reasons. One, in order to resemble how more realistic batch data processing would work. Two, it allowed more immediate feedback when working on the project so I could move on to other sections of the DAG without waiting for all 13 files to be downloaded at once. And 3, it was an excuse to learn how to have different parts of the DAG communicate with each other. 
 
The DAG we are using is broken down into 6 different steps:
1) wget_task: This task counts the difference in days between the run date and the current date. This number will correspond with one of the 13 links in the next task
2) choose_link: This task takes in the value generated by the previous task and returns the corresponding link to the next task 
3) download_dataset_task: This task takes the link given by the previous task and proceeds to download the CSV file 
4) format_to_parquet_task: This task uses pyspark in order to read the downloaded csv file, make the necessary transformations, and save the file as multiple parquet files. We will go into more detail in the next section
5) local_to_gcs_task: This task takes the saved parquet files and then uplaods them to our data lake in google cloud storage 
6) bigquery_external_table_task: This task takes the data in google cloud storage and uploads them to our data warehouse in BigQuery 

## 7. PySpark 
 
The format_to_parquet_task in our Dag contains our PySpark Code. This function first creates a SparkSession. Then it applies a schema which I have created on the saved csv to create a pyspark dataframe. It then does the same thing to our predownloaded csv file that contains our NAICS codes such that they have no issues when merging. The dates are also transformed so that pyspark can more easily read them as datetime types. The zip codes in the data are not uniform in the way that they are read in, so a transformation is done so that google data studio can more easily read them in as postal code types. Finally, it drops some unecessary columns and saves down in a folder of parquet files
 
## 8. Google Data Studio Dashboard
 
The dashboard is broken up into two pages. The first being an overview with a breakdown based on different categories and the second as a map with a table with information based on zip code. Sorry that it isnt that pretty. 
 
Link to the dashboard [here](https://datastudio.google.com/reporting/382aebdd-ae5d-4683-ab4c-2e81357e7295)
 


