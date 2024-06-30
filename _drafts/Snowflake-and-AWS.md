---
layout: post
title: "Snowflake and AWS"
subtitle: "a match made in the cloud"
background: '/img/posts/aws-lift-and-shift/post-header.png'
description: ""
---

## Introduction
In the ever-evolving landscape of data management, Snowflake has emerged as a revolutionary cloud-based data warehousing
solution. Snowflake is a fully-managed service that provides a single platform for data warehousing, data lakes, data 
engineering, data science, and data application development. It is designed to support a wide range of data workloads, 
from traditional batch data processing to real-time analytics and machine learning.
In this post, I will talk about how to get started with Snowflake and how to hook it up into an AWS account so we can 
start importing data from AWS.  The post assumes you have an existing AWS account and some AWS knowledge.  If you don't 
have an account you can create a free one [here](https://aws.amazon.com/free/?gclid=CjwKCAjw1emzBhB8EiwAHwZZxYs-W2OIk-0jkipyv5iChHdAkg8E4ehqrY94sI_Jd2CSpgEINxtOXxoCJoYQAvD_BwE&trk=7541ebd3-552d-4f98-9357-b542436aa66c&sc_channel=ps&ef_id=CjwKCAjw1emzBhB8EiwAHwZZxYs-W2OIk-0jkipyv5iChHdAkg8E4ehqrY94sI_Jd2CSpgEINxtOXxoCJoYQAvD_BwE:G:s&s_kwcid=AL!4422!3!651751058790!e!!g!!aws%20account!19852662149!145019243897&all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all). 
These are the steps that we will follow:

1. Create a Snowflake account
2. Connect Snowflake to AWS
3. Create an Snowflake Integration
4. Create basic Snowflake Infrastructure
   1. Database
   2. Schema
   3. Stage
   4. Table
   5. SnowPipe
5. Run Terraform
6. Upload a test file
7. Query Stage Table

## Creating a Snowflake account
Snowflake provides the ability to create a free account.  The account lasts 30 days or 400 dollars (at the time of 
writing) in credits.  You can choose any of their account types Standard Edition, Enterprise Edition, or Business 
Critical Edition.  The Standard Edition is the most basic account type with full access to all Snowflake standard 
features.  The Enterprise Edition has additional features designed for the needs of large enterprises.  The Business
Critical Edition offers higher levels of protection tailored towards customers with sensitive data such as protected 
health information (PHI).  You can read more about these in more detail on the [Snowflake Website](https://docs.snowflake.com/en/user-guide/intro-editions).

For our purposes, we will use the Standard Edition as the free credits will last longer and we don't need any special 
features to achieve the purposes of this post. In order to create a Snowflake account, go to their website [here](https://signup.snowflake.com/?utm_source=google&utm_medium=paidsearch&utm_campaign=na-us-en-brand-trial-exact&utm_content=go-eta-evg-ss-free-trial&utm_term=c-g-snowflake%20trial-e&_bt=579123129595&_bk=snowflake%20trial&_bm=e&_bn=g&_bg=136172947348&gclsrc=aw.ds&gad_source=1&gclid=CjwKCAjw1emzBhB8EiwAHwZZxYDOg0yMhHHNDUU0LXTgv-Nmps3PO1-N_nZXgxjnLv5oifkQ-vfahRoCVq4QAvD_BwE).
There you can fill out basic information and select your account type.  The process is easy and straight forward.  After 
you follow the prompts and enter your information, including choosing the type of account, which we have previously
discussed, you will get an email to confirm your email and you will be able to login to your account. This [video](https://www.youtube.com/watch?v=AYVwv3K0MIw)
shows the process step by step.

## Connect to AWS
In order to connect to AWS, you will need to create a Role to give Snowflake access.  The role should have the
corresponding policy to give the appropriate access.  The Snowflake integration will need a trusted relationship with 
AWS to be able to connect.  In AWS, this is done by creating a third party trust relationship declared in the  role. 
For such relationship, we will need the ARN and external id from the Snowflake integration.

The policy that will be attached to this role, needs to provide access to S3.  We will give it access to list, put and 
delete files.  

## Snowflake Integration
In Snowflake, we need to create an Integration which specifies the ARN of the role that we have created in AWS and the 
S3 location that such integration will read from.  When this is created, you can then use the ARN and external id that 
it provides and put those values in the AWS role.

As you can see there is a bit of chicken and egg situation as the Role needs information from the Integration and the 
integration needs information from the Role.  We will deal with this in Terraform by creating a temporary role, adding 
such role's ARN to the integration.  Creating the integration and collecting the data and use such data to create the
role that we will actually use.  Then we will update the integration with the ARN from this role.  The full terraform 
code is in Github here and below you can see the snipped to create this part.

```terraform
# Step 1: Create a Temporary IAM Role
resource "aws_iam_role" "temporary_snowflake_role" {
  name = "temporary_snowflake_role"

  assume_role_policy = <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "s3.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }
  EOF
}

# Step 2: Create the Snowflake Storage Integration
resource "snowflake_storage_integration" "aws_s3_integration" {
  name                      = "AWS_S3_INTEGRATION"
  storage_provider          = "S3"
  storage_aws_role_arn      = aws_iam_role.temporary_snowflake_role.arn
  enabled                   = true
  storage_allowed_locations = ["s3://${var.bucket_name}/"]

  lifecycle {
    ignore_changes = [
      storage_aws_role_arn
    ]
  }
}

# Step 3: Create the Actual IAM Role with the Actual ARN
resource "aws_iam_role" "snowflake_role" {
  name = "snowflake_role"

  assume_role_policy = <<EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "${snowflake_storage_integration.aws_s3_integration.storage_aws_iam_user_arn}"
        },
        "Action": "sts:AssumeRole",
        "Condition": {
          "StringEquals": {
            "sts:ExternalId": "${snowflake_storage_integration.aws_s3_integration.storage_aws_external_id}"
          }
        }
      }
    ]
  }
  EOF

  depends_on = [snowflake_storage_integration.aws_s3_integration]
}

# Step 4: Create the Policy to give snowflake access
resource "aws_iam_policy" "snowflake_s3_policy" {
  name        = "snowflake_s3_policy"
  description = "Policy to allow Snowflake access to S3 bucket"
  policy      = <<EOF
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": [
                  "s3:PutObject",
                  "s3:GetObject",
                  "s3:GetObjectVersion",
                  "s3:DeleteObject",
                  "s3:DeleteObjectVersion"
              ],
              "Resource": "arn:aws:s3:::${var.bucket_name}/*"
          },
          {
              "Effect": "Allow",
              "Action": [
                  "s3:ListBucket",
                  "s3:GetBucketLocation"
              ],
              "Resource": "arn:aws:s3:::${var.bucket_name}",
              "Condition": {
                  "StringLike": {
                      "s3:prefix": [
                          "*"
                      ]
                  }
              }
          }
      ]
  }
  EOF
}

# Step 5: Attach the policy to the role
resource "aws_iam_role_policy_attachment" "snowflake_role_policy_attachment" {
  role       = aws_iam_role.snowflake_role.name
  policy_arn = aws_iam_policy.snowflake_s3_policy.arn
}

# Step 6: Update the Snowflake Storage Integration to Use the Actual IAM Role
resource "null_resource" "update_snowflake_integration" {
  provisioner "local-exec" {
    environment = {
      SNOWSQL_PWD = var.snowflake_password
    }
    command = <<EOT
    snowsql -a '${var.snowflake_account}.${var.snowflake_region}' -u '${var.snowflake_username}' -r '${var.snowflake_role}' -q "alter storage integration ${snowflake_storage_integration.aws_s3_integration.name} set STORAGE_AWS_ROLE_ARN='${aws_iam_role.snowflake_role.arn}'"
    EOT
  }

  depends_on = [
    aws_iam_role.snowflake_role,
    aws_iam_role_policy_attachment.snowflake_role_policy_attachment
  ]
}

# Step 7: Enable notifications on S3 so that the integration works as the files arrive
resource "aws_s3_bucket_notification" "s3_notification" {
  bucket = var.bucket_name
  queue {
    queue_arn     = snowflake_pipe.snowpipe.notification_channel
    events        = ["s3:ObjectCreated:*"]
  }

}

# Step 8: Detach Policies from Temporary IAM Role
resource "null_resource" "detach_policies_from_temp_role" {
  provisioner "local-exec" {
    command = <<EOT
    attached_policies=$(aws iam list-attached-role-policies --role-name ${aws_iam_role.temporary_snowflake_role.name} --query "AttachedPolicies[].PolicyArn" --output text)
    for policy_arn in $attached_policies; do
      aws iam detach-role-policy --role-name ${aws_iam_role.temporary_snowflake_role.name} --policy-arn $policy_arn
    done
    EOT
  }
}

# Step 9: Delete the Temporary IAM Role
resource "null_resource" "cleanup_temporary_role" {
  provisioner "local-exec" {
    command = "aws iam delete-role --role-name ${aws_iam_role.temporary_snowflake_role.name}"
  }

  depends_on = [null_resource.detach_policies_from_temp_role]
}

```

## Basic Snowflake Infrastructure
Now that we are connected, we will need a place to store the data that we will be pumping. All data in Snowflake is kept
in databases.  Each database can have one or multiple schemas.  Schemas are logical groups for all your "database objects"
 such as Tables, views, procedures, etc.  The schemas play an important part in security as many of the features can be 
applied at schema level.

The database and schema are straight forward to create and the main thing to choose is the naming (which we all know is 
the harder part to anything involved in Computer Science).  We also need an external stage, which is another database 
object stored in a schema.  This object stores the URL to files in cloud storage, the settings used to access the cloud 
storage account, and convenience settings such as the options that describe the format of staged files. The external 
stage can get credentials in a number of ways, in this case, we will leverage the storage integration that we had created
which contains the role and policy. Then we need a table to store the data that we will be importing from the files. 
This table will be a simple table, with two fields.  One will be for the raw data and one that will capture the ingested 
timestamp.  Here is the terraform to create the infrastructure discussed so far:

```terraform
resource "snowflake_database" "CRAZY_DB" {
  name                        = var.database_name
  comment                     = "test comment"
  data_retention_time_in_days = 0
}

resource "snowflake_schema" "CRAZY_SCHEMA" {
  database = snowflake_database.CRAZY_DB.name
  name     = var.schema_name
  comment  = "A schema."

  is_transient        = true # For our purposes, we don't want to retain the data.
  is_managed          = false
  data_retention_days = 0
}

resource "snowflake_stage" "CRAZY_STAGE" {
  name        = var.stage_name
  url         = "s3://${var.bucket_name}/"
  database    = snowflake_database.CRAZY_DB.name
  schema      = snowflake_schema.CRAZY_SCHEMA.name
  storage_integration = snowflake_storage_integration.aws_s3_integration.name
  file_format = "FORMAT_NAME = ${snowflake_database.CRAZY_DB.name}.${snowflake_schema.CRAZY_SCHEMA.name}.${snowflake_file_format.ndjson_gz_file_format.name}"
  depends_on = [snowflake_file_format.ndjson_gz_file_format]
}

resource "snowflake_table" "CRAZY_TABLE" {
  database        = snowflake_database.CRAZY_DB.name
  schema          = snowflake_schema.CRAZY_SCHEMA.name
  name            = var.table_name
  change_tracking = true
#   cluster_by      = var.cluster_by    <-- Talk about clustering the table, but leave it out of this example

  column {
    name = "DATA"
    type = "VARIANT"
  }
  column {
    name = "INGESTED_TIMESTAMP"
    type = "TIMESTAMP_NTZ(9)"
    default {
      expression = "CURRENT_TIMESTAMP()"
    }
  }
}
```

Now that we have our database, schema and table where to put the data, we need to actually import the data.  With Snowflake,
there are multiple ways to import the data.  Snowflake provides a variety of ways to import data form running queries 
using the [COPY INTO <table>](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table), to manually loading via
the web interface.  For our purposes, I am going to use an automated Snowpipe using cloud messaging, which is a way to 
automatically detect when files are placed in the source.  We will use configure the COPY INTO command query to read 
data from the files and add it to our table. Semi-structured data types, such as JSON and Avro, are supported by Snowpipe.
It is important to remember to enable SQS notification in the S3 bucket so that Snowpipe can import the files as they come.
Here is the terraform for the snowpipe and the file format:

```terraform
resource "snowflake_pipe" "snowpipe" {
  depends_on = [snowflake_table.CRAZY_TABLE,snowflake_stage.CRAZY_STAGE, null_resource.update_snowflake_integration]
  copy_statement = "COPY INTO ${snowflake_table.CRAZY_TABLE.database}.${snowflake_table.CRAZY_TABLE.schema}.${snowflake_table.CRAZY_TABLE.name} (DATA) FROM (SELECT $1 FROM @${snowflake_stage.CRAZY_STAGE.database}.${snowflake_stage.CRAZY_STAGE.schema}.${snowflake_stage.CRAZY_STAGE.name})"
  database       = snowflake_database.CRAZY_DB.name
  name           = var.snowpipe_name
  schema         = snowflake_schema.CRAZY_SCHEMA.name
  auto_ingest = true
}

resource "snowflake_file_format" "ndjson_gz_file_format" {
  name              = "LOAD_NDJSON"
  database          = var.database_name
  schema            = var.schema_name
  compression       = "AUTO"

  binary_format     = "HEX"
  date_format       = "AUTO"
  time_format       = "AUTO"
  timestamp_format  = "AUTO"

  format_type       = "JSON"

  depends_on = [snowflake_database.CRAZY_DB, snowflake_schema.CRAZY_SCHEMA]
}
```

## Running Terraform
As you know, I am a firm believer on automating everything so as shown above I went with Terraform to create the infrastructure
for this post instead of doing it manually from the console.  The Snowflake wiki in medium shows how to this same process
in the console if you are interested in doing it that way.  You can find such article [here](https://snowflakewiki.medium.com/connecting-snowflake-to-aws-ef7b6de1d6aa).
The full terraform code for this post can be found [here](TODO: Insert Github address here).  I am assuming that you have
Terraform installed in you computer as well as AWS CLI.  Before you can run this terraform script, you will need to 
install SnowSQL.  SnowSQL is a CLI client provided by Snowflake.  I used to update the integration once we get the real 
role information.  You can use this [link](https://docs.snowflake.com/en/user-guide/snowsql-install-config) to install 
SnowSQL if you don't have it already.  Once that is installed, you can proceed to initialize the Terraform working 
directory with the command "terraform init".  The results should look like this:

![terraform init](/img/posts/snowflake-and-aws/terraform-init.png)

Then you should proceed to create the infrastructure by using the "terraform apply" command.  That will create all the
infrastructure you need to start uploading data.  Once this command is successful, you cna login to Snowflake and see
the data, schema and table are all created.  You can also login to AWS and see your policies and S3 bucket SQS notification.

## Uploading a test file
At this point, if we upload a file in the S3 bucket the data should be ingested by the Snowpipe and loaded into our table.
This is a sample of the file contents:

![Sample File](/img/posts/snowflake-and-aws/sample-file.png)

It is simple NDJson file simulating events with eventId, eventType, eventTime and eventPayload.  We are going to upload
this the compress version of this file into S3.

This is how the S3 looks like:

![S3 File](/img/posts/snowflake-and-aws/s3-file-view.png)

And this is the table:

![Table View](/img/posts/snowflake-and-aws/table-view.png)

As you can see the data was loaded on the table.  In this case, we loaded the full json document into a VARIANT field. 
We can run some queries directly in the JSON data from Snowsight.  Here are some sample queries:

![Table View](/img/posts/snowflake-and-aws/count-query.png)

![Table View](/img/posts/snowflake-and-aws/event-type-query.png)

![Table View](/img/posts/snowflake-and-aws/event-fields-query.png)

## Conclusion and Next Steps
The post have taken you from 0 to loading data into a Snowflake account levering AWS to load the data and terraform to 
create the infrastructure.  In the ELT cycle, we have done extract and load.  I am going to leave the transform for 
another post where we can create an stream that keeps track of the new data as it gets ingested by the snowpipe.  Then 
we will use a task to periodically import data that we have not yet consumed based on the changes on the stream.  At this
point, in the task, we can write code to transform the data into any tables we want.

I hope you enjoy this post and have learned something.  Keep reading, keep learning!
