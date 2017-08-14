---
title: 翻译：kinesis-streams-vs-firehose
date: 2017-08-14 16:51:36
tags: [AWS, kinesis, stream]
---

# 原文

https://www.sumologic.com/blog/devops/kinesis-streams-vs-firehose/

One question I get is “what is Amazon Kinesis, and what can it do for me?” I also get a lot of questions around Kinesis Streams vs. Firehose? Let’s go through the key concepts and show how to get started with logging using the Kinesis Connector.

什么是Amazon Kinesis？Kenisis Streams和Kenisis Firehose有什么区别？

In December 2013, Amazon Web Services released Kinesis, a managed, dynamically scalable service for the processing of streaming big data in real-time. Since that time, Amazon has been steadily expanding the regions in which Kinesis is available, and as of this writing, it is possible to integrate Amazon’s Kinesis producer and client libraries into a variety of custom applications to enable real-time processing of streaming data from a variety of sources.

Kinesis acts as a highly available conduit to stream messages between data producers and data consumers. Data producers can be almost any source of data: system or web log data, social network data, financial trading information, geospatial data, mobile app data, or telemetry from connected IoT devices. Data consumers will typically fall into the category of data processing and storage applications such as Apache Hadoop, Apache Storm, and Amazon Simple Storage Service (S3), and ElasticSearch.

# Key Concepts

It’s helpful to understand some key concepts when working with Kinesis Streams. The basic unit of scale when working with streams is a shard. A single shard is capable of ingesting up to 1MB or 1,000 PUTs per second of streaming data, and emitting data at a rate of 2MB per second.

Shards scale linearly, so adding shards to a stream will add 1MB per second of ingestion, and emit data at a rate of 2MB per second for every shard added. Ten shards will scale a stream to handle 10MB (10,000 PUTs) of ingress, and 20MB of data egress per second. You choose the number of shards when creating a stream, and it is not possible to change this via the AWS Console once you’ve created a stream.

It is possible to dynamically add or remove shards from a stream using the AWS Streams API. This is called resharding. Resharding cannot be done via the AWS Console, and is considered an advanced strategy when working with Kinesis. A solid understanding of the subject is required prior to attempting these operations.

Adding shards essentially splits shards in order to scale the stream, and removing shards merges them. Data is not discarded when adding (splitting) or removing (merging) shards. It is not possible to split a single shard into more than two, nor to merge more than two shards into a single shard at a time.

Adding and removing shards will increase or decrease the cost of your stream accordingly. Per the Amazon Kinesis Streams FAQ, there is a default limit of 10 shards per region. This limit can be increased by contacting Amazon Support and requesting a limit increase. There is no limit to the number of shards or streams in an account.

Records are units of data stored in a stream and are made up of a sequence number, partition key, and a data blob. Data blobs are the payload of data contained within a record. The maximum size of a data blob before Base64-encoding is 1MB, and is the upper limit of data that can be placed into a stream in a single record. Larger data blobs must be broken into smaller chunks before putting them into a Kinesis stream.

Partition keys are used to identify different shards in a stream, and allow a data producer to distribute data across shards.

Sequence numbers are unique identifiers for records inserted into a shard. They increase monotonically, and are specific to individual shards.

# Amazon Kinesis Offerings

Amazon Kinesis is currently broken into three separate service offerings.

## Kinesis Streams

Kinesis Streams is capable of capturing large amounts of data (terabytes per hour) from data producers, and streaming it into custom applications for data processing and analysis. Streaming data is replicated by Kinesis across three separate availability zones within AWS to ensure reliability and availability of your data.

Kinesis Streams is capable of scaling from a single megabyte up to terabytes per hour of streaming data. You must manually provision the appropriate number of shards for your stream to handle the volume of data you expect to process. Amazon helpfully provides a shard calculator when creating a stream to correctly determine this number. Once created, it is possible to dynamically scale up or down the number of shards to meet demand, but only with the AWS Streams API at this time.

It is possible to load data into Streams using a number of methods, including HTTPS, the Kinesis Producer Library, the Kinesis Client Library, and the Kinesis Agent.

By default, data is available in a stream for 24 hours, but can be made available for up to 168 hours (7 days) for an additional charge.

Monitoring is available through Amazon Cloudwatch.

## Kinesis Firehose

Kinesis Firehose is Amazon’s data-ingestion product offering for Kinesis. It is used to capture and load streaming data into other Amazon services such as S3 and Redshift. From there, you can load the streams into data processing and analysis tools like Elastic Map Reduce, and Amazon Elasticsearch Service. It is also possible to load the same data into S3 and Redshift at the same time using Firehose.

Firehose can scale to gigabytes of streaming data per second, and allows for batching, encrypting and compressing of data. It should be noted that Firehose will automatically scale to meet demand, which is in contrast to Kinesis Streams, for which you must manually provision enough capacity to meet anticipated needs.

As with Kinesis Streams, it is possible to load data into Firehose using a number of methods, including HTTPS, the Kinesis Producer Library, the Kinesis Client Library, and the Kinesis Agent. Currently, it is only possible to stream data via Firehose to S3 and Redshift, but once stored in one of these services, the data can be copied to other services for further processing and analysis.

Monitoring is available through Amazon Cloudwatch.

## Kinesis Analytics

Kinesis Analytics is Amazon’s forthcoming product offering that will allow running of standard SQL queries against data streams, and send that data to analytics tools for monitoring and alerting. This product has not yet been released, and Amazon has not published details of the service as of this date.

## Kinesis vs SQS

Amazon Kinesis is differentiated from Amazon’s Simple Queue Service (SQS) in that Kinesis is used to enable real-time processing of streaming big data. SQS, on the other hand, is used as a message queue to store messages transmitted between distributed application components.

Kinesis provides routing of records using a given key, ordering of records, the ability for multiple clients to read messages from the same stream concurrently, replay of messages up to as long as seven days in the past, and the ability for a client to consume records at a later time. Kinesis Streams will not dynamically scale in response to increased demand, so you must provision enough streams ahead of time to meet the anticipated demand of both your data producers and data consumers.

SQS provides for messaging semantics so that your application can track the successful completion of work items in a queue, and you can schedule a delay in messages of up to 15 minutes. Unlike Kinesis Streams, SQS will scale automatically to meet application demand. SQS has lower limits to the number of messages that can be read or written at one time compared to Kinesis, so applications using Kinesis can work with messages in larger batches than when using SQS.

# Getting Started with AWS Kinesis

Amazon has published an excellent tutorial on getting started with Kinesis in their blog post Building a Near Real-Time Discovery Platform with AWS. It is recommended that you give this a try first to see how Kinesis can integrate with other AWS services, especially S3, Lambda, Elasticsearch, and Kibana.

Once you’ve taken Kinesis for a test spin, you might consider integrating with an external service such as SumoLogic to analyze log files from your EC2 instances using their Amazon Kinesis Connector. Information about the Amazon Kinesis Connector can be found on the open-source at Sumo Logic page. The code has been published in the SumoLogic Github repository. You may also want to check out “Sumo Logic App for Amazon VPC Flow Logs using Kinesis” for additional insights.

# About the Authors

Steve Tidwell has been working in the tech industry for over two decades, and has done everything from end-user support to scaling a global data ingestion and analysis platform to handle data analysis for some of the largest streaming events on the Web. He is currently Lead Architect for a well-known tech news site, where he plots to take over the world with cloud based technologies from his corner of the office.

Michael Floyd is the Head of Developer Programs at Sumo Logic where manages developer relations, supports the developer community and is the editor of devops.sumologic.com.