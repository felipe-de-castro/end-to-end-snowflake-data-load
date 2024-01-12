## Continuos Data Load - AWS S3 to Snowflake (Snowpipe) ❄️

This project demonstrates how to set up a Continuous Data Load using Snowpipe in Snowflake, utilizing the serverless solution for effortless file transfer from AWS S3 to Snowflake tables. 

By focusing on CSV and JSON file formats, we ensure a versatile approach, eliminating the need for external resources and optimizing cost-effectiveness. The potential of Snowflake and AWS S3 for streamlined and efficient data integration.

### Snowpipe

**Snowpipe** stands out as one of the simplest and most widely embraced solutions, demanding minimal effort and earning its classification as a zero-code solution. 

It efficiently loads data from files into Snowflake as soon as the data or files become available in the designated stage. This process involves executing a copy statement to transfer the data into Snowflake. The Snowflake platform is informed about the presence of new staged files through Event Notification (**AWS SQS**) from the S3 bucket and by invoking Snowpipe REST endpoints.

Continuous data load in Snowflake using Snowpipe require a few steps that will be shown below:

#### Create a database to work on:

```plaintext
create or replace database project_snowflake;
```

#### Create Storage Integration and Access Permissions

*   In **AWS S3**, create a bucket and _subfolders_ to load files (csv and json) and copy the bucket link¹ from each to set in the `storage_allowed_locations` ;
*   Now, in **AWS IAM** (Identity and Access Management ), go to **Role** and **“Create role”:**
    *   **Select trusted entity: AWS ACCOUNT**
    *   **An AWS account: This accout (XXXXXX)**
    *   **Option**: Require external ID and set to a dummy number (00000)
    *   **Permissions policies:** AmazonS3FullAccess.
*   After creating the Role, copy the ARN from its summary and paste in the `storage_aws_role_arn` and **run the query**.

_Bucket links:_

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/5017b8cd36d0accaf55b5a5e22d465395a02d3e4ca7c66f4.png)

```plaintext
create or replace storage integration s3_int
type =  external_stage
storage_provider = 'S3'
enabled = true 
storage_aws_role_arn = 'arn:aws:iam::395521588262:role/snowflake-acess-role'  -- created in aws iam>role>"aws account"> 
storage_allowed_locations = ('s3://snowflakess3bucketproject/csv/','s3://snowflakess3bucketproject/json/');
```

Using _**DESC integration \<name\_integration>**_ will describe the properties of an integration to continue setting the role integration.

*   In AWS IAM, got to the created **Role** > **Trust relationships**\> **Edit trust policy** and set the desired values in the trust policy, as the print below.

```plaintext
desc integration s3_int;
```

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/ca95b56e8ec96e08cac6dc1a2eb944be2a0b4508c5410591.png)

#### Create File Format for CSV and Json

 File format is a set of instructions that defines how data is organized and formatted within files stored in a cloud-based object storage service.  When you load data into Snowflake from external files, the file format provides the necessary information for Snowflake to interpret and process the data correctly.

In the case, will be created for **CSV** and **JSON** files.

```plaintext
-- file format for csv --
   create or replace file format project_snowflake.public.aws_file_format_csv
   type = csv
   field_delimiter = ','
   skip_header = 1
   null_if = ('Null', 'null')
   empty_field_as_null = true;
   
-- file format for json --
   create or replace file format project_snowflake.public.aws_file_format_json
   type = json;   
```

#### Creating Stage

A Snowflake stage specifies where the data files are stored(i.e “staged”) so that the data in the files can be loaded into a table (can be staged extenal or internal). 

Snowflake suggests a best practice of establishing an **external stage** that points to the bucket. It is advisable to utilize this external stage for data loading operations instead of directly interfacing with the bucket.

```plaintext
// Preparing Stage - csv and json   
-- create a stage for csv --
   create or replace stage project_snowflake.public.aws_stage_csv
   url = 's3://snowflakess3bucketproject/csv/'
   storage_integration = s3_int
   file_format = project_snowflake.public.aws_file_format_csv;

-- create a stage for json --
   create or replace stage project_snowflake.public.aws_stage_json
   url = 's3://snowflakess3bucketproject/json/'
   storage_integration = s3_int
   file_format = project_snowflake.public.aws_file_format_json;   
   
```

Validate if the stage is correct by listing the files on the stage;

```plaintext
list @project_snowflake.public.aws_stage_csv;
list @project_snowflake.public.aws_stage_json;
```

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/1d5c06bdef1dba4294c82f25a89a8f1e45c3902426f8e4bc.png)

#### **Querying Data in Staged Files**

Snowflake supports querying data files in its internal or external stages (Amazon S3, Google Cloud Storage, or Microsoft Azure) using standard SQL. This is beneficial for inspecting file contents before or after data loading operations.

Note that for CSV  the alias `$1`  results the first column from the staged file and for the JSON returns all the key-value pairs.  
 

```plaintext
-- querying staged csv --
select $1,$2,$3,$4,$5,$6 from @project_snowflake.public.aws_stage_csv;
```

```plaintext
-- querying staged json --
select $1 from @project_snowflake.public.aws_stage_json;   
```

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/8f28085a2c30b7aaa252a3a885232f5deb5da7555e7e465e.png)

#### Creating table 

```plaintext
//Create table - CSV & JSON
-- create table to receive the data from csv and json -- 
   create or replace table project_snowflake.public.employees (
   id int,
   first_name string,
   last_name string,
   email string,
   location string,
   department string
   );
```

```plaintext
-- cleaning the json file --
select 
$1:id::int as id
,$1:first_name::string as first_name
,$1:last_name::string as last_name
,$1:email::string as email
,$1:location::string as location
,$1:department::string as department
from @project_snowflake.public.aws_stage_json;
```

```plaintext

      
// PIPE STRUCTURE
-- create schema for the pipe (can use the same schema) --
   create or replace schema project_snowflake.pipe;
-- create pipe from the CSV copy command --
   create or replace pipe project_snowflake.pipe.employee_csv
   auto_ingest = true
   as 
   copy into project_snowflake.public.employees -- table created
   from @project_snowflake.public.aws_stage_csv;
-- create pipe from the JSON copy command --
   create or replace pipe project_snowflake.pipe.employee_json
   auto_ingest = true
   as 
   copy into project_snowflake.public.employees -- table created
   from (select 
            $1:id::int as id
           ,$1:first_name::string as first_name
           ,$1:last_name::string as last_name
           ,$1:email::string as email
           ,$1:location::string as location
           ,$1:department::string as department
           from @project_snowflake.public.aws_stage_json);
```

```plaintext
-- describe the pipe and use the NOTIFICATION CHANNEL | bucket > properties>event notification   
   desc pipe project_snowflake.pipe.employee_csv;
   desc pipe project_snowflake.pipe.employee_json;
```

```plaintext
-- restart the pipe -- 
   alter pipe project_snowflake.pipe.employee_csv REFRESH;
   alter pipe project_snowflake.pipe.employee_json REFRESH;
```
