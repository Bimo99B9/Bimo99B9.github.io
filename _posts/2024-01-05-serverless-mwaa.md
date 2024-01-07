---
layout: single
title: Serverless Apache Airflow with AWS MWAA
excerpt: "Optimizing ETL processes using a serverless architecture with Apache Airflow on AWS MWAA, achieving significant cost savings and efficiency."
date: 2024-01-05
classes: wide
header:
  teaser: /assets/images/data-eng-serverless-mwaa/img7.png
  teaser_home_page: true
  icon: /assets/images/data-eng-serverless-mwaa/aws-icon.png
categories:
  - data engineering
  - cloud computing
tags:  
  - AWS
  - MWAA
  - Apache Airflow
  - serverless
  - ETL
  - data processing
  - automation

---

At Vumi, we update daily market data of all kinds and from multiple providers, such as prices, events (dividends and splits), asset allocation (by sectors, countries, categories...), market changes (delisted stocks, ticker changes), and many other things. This involves millions of pieces of data that we ingest, process, transform, and load into our databases every day.

However, most of these ETL processes only need to be run once a day. So we set up an architecture that would allow us to have an Apache Airflow available only the hours we needed it. How did we do that?

![](/assets/images/data-eng-serverless-mwaa/img1.png)


Vumi is deployed on AWS, and this provider offers MWAA (Managed Workflows for Apache Airflow) as a solution for deploying an Airflow environment. It uses an S3 bucket to store DAGs, requirements, and other necessary files. In addition, it allows environments to be monitored with CloudWatch, which is very useful when the environment is not active.

![](/assets/images/data-eng-serverless-mwaa/img2.png)

However, AWS does not offer a Serverless implementation of the MWAA environment, but when the environment is deleted, all components, including the database, are completely removed, losing metadata and other configuration information such as roles and permissions, as well as logs of DAG executions. This makes it very complicated to create and destroy an environment every day. At the same time, having a continuously active environment is inefficient, as it consumes resources even when no tasks are being processed.

To solve this, we use two processes that create and import, and export and delete environment details before and after DAG scheduling. These two processes are orchestrated with Step Functions and scheduled with EventBridge. To save the data, a DAG pauses execution and exports in S3 all information from the active DAGs to a CSV file.

The next day, a similar process retrieves the configuration of the exported environment and uses it to recreate it. When the creation is finished, another DAG restores the active DAGs.

![](/assets/images/data-eng-serverless-mwaa/img1.jpg)

The concrete implementation has two additional stacks. One of them polls the environment to verify its creation or deletion, and to know when to run the metadata export and import DAGs. The other uses EventBridge and SNS to send notifications when part of the process fails.

![](/assets/images/data-eng-serverless-mwaa/img6.png)

The complete export and re-import flow is as follows:

1) EventBridge Scheduler starts the execution of the pause Step Function at the time scheduled by a Cron expression.
2) GetEnvironment from the AWS SDK retrieves the details of the active environment.
3) The environment-backup.json object is saved to the backup S3. This file contains the environment configuration, such as Airflow version, configuration options, the S3 path of the environment DAGs, number of workers...

   ![](/assets/images/data-eng-serverless-mwaa/img7.png)

4) A token is created for the export DAG.
5) The StoreMetadata lambda starts the export DAG in the still active environment. This DAG pauses the running DAGs, and exports all Apache Airflow data and variables. If it fails, it reactivates them.

   ![](/assets/images/data-eng-serverless-mwaa/img9.png)

6) Environment metadata is exported to S3 in multiple CSVs.

   ![](/assets/images/data-eng-serverless-mwaa/img11.png)

7) The export DAG notifies the Step Function of the success.
8) The environment is removed via the AWS SDK.
9) The polling step function periodically checks the status of the environment until successful deletion is confirmed.

To recreate the environment the next day, the process is similar, although in this case an additional lambda is used to create the environment with the information from the previously exported environment-backup.json.

![](/assets/images/data-eng-serverless-mwaa/img12.png)

The last lambda invokes the re-import DAG when the environment finishes creating. This DAG reads the exported CSV files from S3 and re-imports it into the Airflow database, thus activating the DAGs and resuming the execution of the environment.

![](/assets/images/data-eng-serverless-mwaa/img13.png)

This solution saves up to 87.5% of the cost of the environment compared to running it continuously without implementing the automation explained above.

In the long term, it will be necessary to extend the time we need the cluster active, especially when non-daily ETL processes are required. At the moment, these continuous processes are few and light, and we do not use MWAA for this.

If this sounds like a useful solution, AWS provides an implementation in its [GitHub repository](https://github.com/aws-samples/amazon-mwaa-examples/tree/main/usecases/start-stop-mwaa-environment#building-and-deploying-the-project), although we had to make changes to resolve some issues it had. For example, the CSV export and read CSV had inconsistencies that periodically caused DAGs to be disabled. Also, backups grow after many cycles, and it is necessary to periodically clean the environment's databases.