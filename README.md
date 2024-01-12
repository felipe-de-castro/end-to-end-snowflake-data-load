## Continuos Data Load - AWS S3 to Snowflake (Snowpipe) ❄️

Snowflake offers a serverless solution that seamlessly loads files from Object Storage, such as S3, directly into Snowflake tables. This process eliminates the need for external resources, and Snowflake only incurs charges based on compute usage during the data loading phase.

**Snowpipe** stands out as one of the simplest and most widely embraced solutions, demanding minimal effort and earning its classification as a zero-code solution. 

**Snowpipe** efficiently loads data from files as soon as they become available in the designated stage, automatically executing a copy statement to seamlessly integrate the data into Snowflake.

```plaintext
 create or replace database project_snowflake;
```

```plaintext
   create or replace storage integration s3_int
   type =  external_stage
   storage_provider = 'S3'
   enabled = true 
   storage_aws_role_arn = 'arn:aws:iam::395521588262:role/snowflake-acess-role'  -- created in aws iam>role>"aws account"> 
   storage_allowed_locations = ('s3://snowflakess3bucketproject/csv/','s3://snowflakess3bucketproject/json/');
```

```plaintext
    desc integration s3_int;
```

```plaintext
//File Format
-- create a file format csv --
   create or replace file format project_snowflake.public.aws_file_format_csv
   type = csv
   field_delimiter = ','
   skip_header = 1
   null_if = ('Null', 'null')
   empty_field_as_null = true;
   
-- create a file format json --
   create or replace file format project_snowflake.public.aws_file_format_json
   type = json;   
```

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

```plaintext
-- list file in the stage --     
   list @project_snowflake.public.aws_stage_csv;
   list @project_snowflake.public.aws_stage_json;
```

```plaintext

-- can query the file in stage -- 
   -- csv --
   select $1,$2,$3,$4,$5,$6 from @project_snowflake.public.aws_stage_csv;
   -- json --
   select $1 from @project_snowflake.public.aws_stage_json;
   
```

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
