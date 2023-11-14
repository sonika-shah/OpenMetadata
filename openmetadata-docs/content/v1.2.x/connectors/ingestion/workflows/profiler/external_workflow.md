---
title: External Profiler Workflow
slug: /connectors/ingestion/workflows/profiler/external-workflow
---

# External Profiler Workflow

{% note %}

Note that this requires OpenMetadata 1.2.1 or higher.

{% note %}

Consider a use case where you have a large database source with multiple databases and schemas which are maintained by 
different teams within your organization. You have created multiple database services within OpenMetadata depending on 
your use case by applying various filters on this large source. Now, instead of running a profiler pipeline for each 
service, you want to run a **single workflow profiler for the entire source**, irrespective of the OpenMetadata service which
an asset would belong to. This document will guide you on how to achieve this.

{% note %}

Note that running a single profiler workflow is only supported if you run the workflow **externally**, not from OpenMetadata.

{% /note %}

{% partial file="/v1.2/connectors/external-ingestion-deployment.md" /%}


## 1. Define the YAML Config

You will need to prepare a yaml file for the data profiler depending on the database source. 
You can get details of how to define a yaml file for data profiler for each connector [here](https://docs.open-metadata.org/v1.2.x/connectors/database).

For example, consider if the data source was snowflake, then the yaml file would have looked like as follows.


```snowflake_external_profiler.yaml
source:
  type: snowflake
  serviceConnection:
    config:
      type: Snowflake
      username: my_username
      password: my_password
      account: snow-account-name
      warehouse: COMPUTE_WH
  sourceConfig:
    config:
      type: Profiler
      generateSampleData: true
      # schemaFilterPattern:
      #   includes:
      #   # - .*mydatabase.*
      #   - .*default.*
      # tableFilterPattern:
      #   includes:
      #   # - ^cloudfront_logs11$
      #   - ^map_table$
      #   # - .*om_glue_test.*
processor:
  type: "orm-profiler"
  config: {}
    # tableConfig:
    # - fullyQualifiedName: local_snowflake.mydatabase.mydschema.mytable
    #   sampleDataCount: 50
    # schemaConfig:
    # - fullyQualifiedName: demo_snowflake.new_database.new_dschema
    #   sampleDataCount: 50
    #   profileSample: 1
    #   profileSampleType: ROWS
    #   sampleDataStorageConfig:
    #     bucketName: awsdatalake-testing
    #     prefix: data/sales/demo1
    #     overwriteData: false
    #     storageConfig:
    #       awsRegion: us-east-2
    # databaseConfig:
    # - fullyQualifiedName: snowflake_prod.prod_db
    #   sampleDataCount: 50
    #   profileSample: 1
    #   profileSampleType: ROWS
    #   sampleDataStorageConfig:
    #     bucketName: awsdatalake-testing
    #     prefix: data/sales/demo1
    #     overwriteData: false
    #     storageConfig:
    #       awsRegion: us-east-2
sink:
  type: metadata-rest
  config: {}
workflowConfig:
  loggerLevel: DEBUG
  openMetadataServerConfig:
    hostPort: http://localhost:8585/api
    authProvider: openmetadata
    securityConfig:
      jwtToken: "your-jwt-token"

```

{% note %}

Note that we do **NOT pass the Service Name** in this yaml file, unlike your typical profiler workflow

{% /note %}


## 2. Run the Workflow

### Run the Workflow with the CLI

One option to running the workflow externally is by leveraging the `metadata` CLI.

After saving the YAML config, we will run the command:

```
metadata profile -c <path-to-yaml>
```

### Run the Workflow from Python using the SDK

If you'd rather have a Python script taking care of the execution, you can use:

```python
from metadata.workflow.profiler import ProfilerWorkflow
from metadata.workflow.workflow_output_handler import print_status

# Specify your YAML configuration
CONFIG = """
source:
  ...
workflowConfig:
  openMetadataServerConfig:
    hostPort: 'http://localhost:8585/api'
    authProvider: openmetadata
    securityConfig:
      jwtToken: ...
"""

def run():
    workflow_config = yaml.safe_load(CONFIG)
    workflow = ProfilerWorkflow.create(workflow_config)
    workflow.execute()
    workflow.raise_from_status()
    print_status(workflow)
    workflow.stop()


if __name__ == "__main__":
  run()
```