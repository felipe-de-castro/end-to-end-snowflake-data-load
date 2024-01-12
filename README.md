## Continuos Data Load - AWS S3 to Snowflake (Snowpipe) ❄️

This project demonstrates how to set up a Continuous Data Load using Snowpipe in Snowflake, utilizing the serverless solution for effortless file transfer from AWS S3 to Snowflake tables. 

By focusing on CSV and JSON file formats, we ensure a versatile approach, eliminating the need for external resources and optimizing cost-effectiveness. The potential of Snowflake and AWS S3 for streamlined and efficient data integration.

If you are on an account admin or have a higher access level (because you are trying this on a newly created personal Snowflake), otherwise will be needed permission to the current role and user.

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

#### Querying the staged JSON for later on create a copy command for the pipe.

Json used as format that allows for semi-structured data, because it does not require a schema. 

The format for selecting data includes all of the following:

*   the format for select: `$1:<key>`  such as `$1:department` etc.
*   for parsing nest data: `$1:<key>:<subkey>` such as `$1:job:salary.`
*   for converting the object to a specific type will be uses `::` . Example: `$1:id::int` (key id as a INT) .

```plaintext
-- cleaning/organizing the json file --
select 
     $1:id::int as id
    ,$1:first_name::string as first_name
    ,$1:last_name::string as last_name
    ,$1:email::string as email
    ,$1:location::string as location
    ,$1:department::string as department
from @project_snowflake.public.aws_stage_json;
```

#### Creating table

Creating table to load the data from csv and json into it.

```plaintext
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

#### Creating snowpipe structure with copy command.

*   First, for good practice, create a schema to store the pipes.

```plaintext
-- create schema for the pipe (can use the same schema) --
   create or replace schema project_snowflake.pipe;
```

*   The pipe used the `COPY INTO` to load data automatically whenever a new notification for file ingestion gets received .

\*\* The function of the `COPY INTO` command is used to loads data from staged files to an existing table (files must already be staged).

*   The `AUTO_INGEST=TRUE` setting is crucial for reading event notifications sent from an S3 bucket to an SQS queue when new data is loaded.
*   Create for CSV and JSON files. Note that for the **JSON** file was used the query of the staged to build as a copy command.

```plaintext
-- create pipe from the CSV copy command --
   create or replace pipe project_snowflake.pipe.employee_csv
   auto_ingest = true
   as 
   copy into project_snowflake.public.employees -- table created
   from @project_snowflake.public.aws_stage_csv;
```

```plaintext
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

#### Setting the event notification for the S3 bucket

```plaintext
-- describe the pipe and use the NOTIFICATION CHANNEL -- 
   desc pipe project_snowflake.pipe.employee_csv;
   desc pipe project_snowflake.pipe.employee_json;
```

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/953a226bd0fd2983726ca2348fe7fdb65d37d294592049b9.png)

*   In **AWS S3**: got to **Properties** of the current **bucket**. 
    *   Scroll to the **EVENT NOTIFICATION** for setting up:
        *   **Prefix**: ‘employee/’ 
        *   **Object creation**: ‘All objects create event’
        *   **Destination**: ‘SQS’
        *   **SQS queue**: paste de queue ARN;

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/4714b12e534c70357de5233bc26cc44a20964fb2a8e086ab.png)

#### Restarting the pipe to load the data

The following will refresh the pipe (i.e. copying the specified staged data files to the Snowpipe ingest queue for loading into the target table).

```plaintext
-- restart the pipe -- 
   alter pipe project_snowflake.pipe.employee_csv REFRESH;
   alter pipe project_snowflake.pipe.employee_json REFRESH;
```

#### Pipe status

 Retrieves a JSON representation of the current status of a pipe such as:

*   executionState,
*   lastIngestedFilePath
*    pendingFileCount etc

```plaintext
 select SYSTEM$PIPE_STATUS('project_snowflake.pipe.employee_json');
```

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/0d385678e63e058dc1a360ea8893767a46c23d193bbe3f44.png)

#### Final query of the table

*   Each file has a list of 100 employees.

```plaintext
 select * from project_snowflake.public.employees;
```

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/721a397c894c3cc328545c686731f968555697464ce7c966.png)

####  Conclusion:

When dealing with larger file sizes, particularly in the range of 100MB to 200MB, the loading process tends to take an unexpectedly extended amount of time.

 In contrast, employing the copy command allows for a faster loading time for the same file. Snowpipe proves to be more suitable for scenarios involving very small files with low frequency, where the latency of file ingestion is not a critical factor.

Fell free to try it yourself. The datafiles are in the github folder in this project.

#### Reference:

[Snowflake Official Documentation](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-auto-s3#step-3-configure-security)
