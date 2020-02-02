# Kafka Connect: How it let us down?

About a year ago me and [@minutis](https://github.com/minutis) had a chance to try out [Kafka Connect](https://docs.confluent.io/3.0.0/connect/). We used it as the backbone of one of our [ETL](https://www.webopedia.com/TERM/E/ETL.html) processes but eventually, we chose a different approach. In this post, I'll try to remember what problems we met, and why Kafka Connect didn't fit our needs.

For those of you, who do not know what Kafka Connect is, it is a framework for connecting Apache Kafka to external systems such as databases, search indexes and file systems.
Kafka Connect allows both: write data from external source system to Kafka topic and export data from Kafka topic to external system.

## Main Kafka Connect concepts

I'm not going in-depth to each and every Kafka Connect component, there is plenty of information online on how Kafka Connect is designed and how it works, however, I'll try to describe them, in short, to give you an idea on how Kafka Connect works so that you have more context on what I'm going to write further in this post.

So Main Kafka Connect components are:
- Connector - unit of work and also a logical implementation of integrations with external systems. There are two types of Connectors: Source Connectors, responsible for reading from external systems to Kafka and Sink Connectors responsible for writing data from Kafka to external systems. Confluent platform, the main contributor of Kafka Connect, has quite detailed [docummentation](https://docs.confluent.io/current/connect/devguide.html#) on how to implement your own Source and Sink connectors.

- Task - a unit of work. When you are configuring Connector for an external system, you can define the maximum number of tasks. This number defines how many processes in parallel should read from your external system (or write to it). So the work done by Connector is parallelized by the number of tasks.

- Worker - component, responsible for task execution. Kafka Connect can work in two modes: standalone and distributed. In standalone mode, you have one Worker process responsible for executing your connector task, configured on properties file. In distributed mode, you can start many worker processes and distribute them all across your Kafka cluster. Also, in distributed mode, all connector configuration is done by using [Kafka Connect Rest API](https://docs.confluent.io/current/connect/references/restapi.html).

- Transform - transformation applied to Kafka's message after Connector ingests data, but before data is written to Kafka topic. There are many [Transforms implementations](https://docs.confluent.io/current/connect/transforms/index.html). It is also very easy to implement custom transforms.

## Why it failed us?

The first time I found out about Kafka Connect, I was excited. It looked like really nice and thought through solution for managing different ETL pipelines. It has Rest API for adding/removing connectors, starting stopping and scaling tasks and monitoring task statuses. Extensibility looked really promising too, you can easily add your own connector and transform implementations, without forking Kafka source code and scale it through as many worker processes as you need.
Without further investigations, we decided to try it out and what we experienced wasn't as nice, as we hoped :).

### Too early to use in production

At the time we were experimenting with Kafka Connect it wasn't stable enough. It had some bugs, the quality of open-sourced connectors was quite poor, and there were a few architecture flaws which were a deal-breaker for us.

#### Bugs
In our use case, we wanted to write data to [HDFS](https://www.ibm.com/analytics/hadoop/hdfs). For that we decided to use open-sourced [kafka-connect-hdfs](https://github.com/confluentinc/kafka-connect-hdfs) connector implementation. At the time we used it, it was pretty much unusable:
- We had corrupted files after Kafka rebalance [#268(open)](https://github.com/confluentinc/kafka-connect-hdfs/issues/268)
- We had limited datatypes available [#49(open)](https://github.com/confluentinc/kafka-connect-hdfs/issues/49)
- We were not able to partition data by multiple fields (we had implemented our own solution for this one) [#commons-53(fixed)](https://github.com/confluentinc/kafka-connect-storage-common/issues/53)
- We had tasks failing to resume after a pause [#53(fixed)](https://github.com/confluentinc/kafka-connect-hdfs/issues/53)

After the experience with open source connectors we saw, that they are not only buggy but also lack the features we need. We decided to use our own connector implementations and didn't stop believing in Kafka Connect. Although we encountered some Kafka bugs, they were fixed fast enough ([KAFKA-6252](https://issues.apache.org/jira/browse/KAFKA-6252)).

Some minor Kafka Connect bugs left unfixed though. One of them, worth mentioning is [KAFKA-4107](https://issues.apache.org/jira/browse/KAFKA-4107). In the process of testing, we had some cases when we needed to delete and recreate some of the connectors. Kafka Connect provides REST API endpoint for Connector deletion, however when you delete connector through this API, old task offset remain undeleted, so you can not create a connector with the same name. We found a workaround for this problem: we've added connector versioning (appended version numbers on connector name), to avoid conflicts with offsets from deleted tasks.

#### Rebalance all the time
This was the Kafka Connect design flaw. Kafka Connect rebalanced *all* of the tasks on its cluster every time you changed task set (add or delete a task or a connector, etc.). That meant all running tasks had to be stopped and re-started. The time needed to rebalance all your tasks grows significantly each time you add a new connector and becomes unacceptable when it comes to ~100 tasks. 
This was the biggest roadblock for us since we had a dynamic environment where the task set was changing rapidly, so rebalancing was happening too.
Well, today this **is not a problem anymore**. With Kafka 2.3.0 which came not so long ago, this flaw was fixed. You can read more on that [here](https://cwiki.apache.org/confluence/display/KAFKA/KIP-415%3A+Incremental+Cooperative+Rebalancing+in+Kafka+Connect).

## Conclusion
We droped the idea of using Kafka Connect (some time in 2018). We droped it, because it wasn't production ready yet, and it didn't fully covered our use cases. Today, many of the problems we met are fixed (some of them are not, but you can find workarounds). I'm still kind of skeptikal about Kafka Connect, however trying and experimenting with it was really fun. I'd say you should consider Kafka Connect only if you are willing to invest time in implementing your own Connectors.
