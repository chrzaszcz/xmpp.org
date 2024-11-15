---
title: MongooseIM 6.3 - Monitor with Prometheus, scale up with CockroachDB
date: 2024-11-15
author: Paweł Chrząszcz (Erlang Solutions)
categories: ["Miscellaneous"]
---

[MongooseIM](https://www.erlang-solutions.com/technologies/mongooseim) is a scalable, efficient, high-performance instant messaging server using the proven, open, and extensible XMPP protocol. With each new version, we introduce new features and improvements. For example, version 6.2.0 introduced our new CETS in-memory storage, making setup and autoscaling in cloud environments easier than before (see the [blog post](https://www.erlang-solutions.com/blog/mongoose-im-6-2/) for details). The latest release 6.3.0 is no exception. The main highlight is the complete instrumentation rework, allowing seamless integration with modern monitoring solutions like Prometheus.

Additionally, we have added CockroachDB to the list of supported databases, so you can now let this highly scalable database grow with your applications while avoiding being locked into your cloud provider.

## Observability and instrumentation

In software engineering, **observability** is the ability to gather data from a running system in order to figure out what is going inside: is it working as expected? Does it have any issues? How much load is it handling, and could it do more? There are many ways to improve the observability of a system, and one of the most important is **instrumentation**. Just like adding extra measuring equipment to a physical system, this means adding extra code to the software. It allows the system administrator to observe the internal state of the system. This comes with a price. There is more work for the developers, increased complexity, and potential performance degradation caused by the collection and processing of additional data.

However, the benefits usually outweigh the costs, and the ability to inspect the system is often a critical requirement. It is also worth noting that the metrics and events gathered by instrumentation can be used for further automation, e.g. for autoscaling or sending alarms to the administrator.

### Instrumentation in MongooseIM

Even before our latest release of MongooseIM, there have been multiple means to observe its behaviour:

**Metrics** provide numerical values of measured system properties. The values change over time, and the metric can present current value, sum from a sliding window, or a statistic (histogram) of values from a given time period. Prior to version 6.3, MongooseIM used to store such metrics with the help of the [exometer](https://github.com/Feuerlabs/exometer) library. To view the metrics, one had to configure an Exometer exporter, which would periodically send the metrics to an external service using the Graphite protocol. Because of the protocol, the metrics would be exported to [Graphite](https://graphiteapp.org) or [InfluxDB version 1](https://docs.influxdata.com/influxdb/v1/supported_protocols/graphite/). One could also query a limited subset of metrics using our [GraphQL API](https://esl.github.io/MongooseDocs/latest/graphql-api/admin-graphql-doc.html#definition-MetricAdminQuery) (or the legacy REST API) or with the command line interface. Alternatively, metrics could be retrieved from the Erlang shell of a running MongooseIM node.

**Logs** are another type of instrumentation present in the code. They inform about events occurring in the system and since version 4, they are events with extensible map-like structure and can be formatted e.g. as plain text or JSON. Subsequently, they can be shown in the console or stored in files. You can also set up a log management system like the [Elastic (ELK) Stack](https://www.elastic.co/elastic-stack) or Splunk – see the [documentation](https://esl.github.io/MongooseDocs/latest/operation-and-maintenance/Logging/) for more details.

Prior to version 6.3.0, the instrumented code needed to separately call the log and metric API. Updating a metric and logging an event required two distinct function calls. Moreover, if there were multiple metrics (e.g. execution time and total number of calls), there would be multiple function calls required. The main issue of this solution was however the hardcoding of Exometer as the metric library and the limitation of the Graphite protocol used to push the metrics to external services.

### Instrumentation rework in MongooseIM 6.3

The lack of support for the modern and widespread Prometheus protocol was one of the main reasons for the complete rework of instrumentation in version 6.3, which is summarised in the following diagram:

{{< figure src="/images/blog/mongooseim_instrumentation.png" >}}

The most noticeable difference is that in the instrumented code, there is just one event emitted. Such an event is identified by its **name** and a key-value map of **labels** and contains **measurements** (with optional metadata) organised in a key-value map. Each event has to be **registered** before its instances are emitted with particular measurements. The point of this preliminary step is not only to ensure that all events are handled but also to provide additional information about the event, e.g. the measurement keys that will be used to update metrics. Emitted events are then handled by configurable **handlers**. Currently, there are three such handlers. Exometer and Logger work similarly as before, but there is a new Prometheus handler as well, which stores the metrics internally in a format compatible with [Prometheus](https://prometheus.io/docs/introduction/overview/) and exposes them over an HTTP API. This means that any external service can now scrape the metrics using the Prometheus protocol. The primary case would be to use Prometheus for metrics collection, and a graphical tool like Grafana for display. If you however prefer InfluxDB version 2, you can easily configure a [scraper](https://docs.influxdata.com/influxdb/v2/write-data/developer-tools/scrape-prometheus-metrics/), which would periodically put new data into InfluxDB.

Apart from supporting the Prometheus protocol, additional benefits of the new solution include easier configuration, extensibility, and the ability to add more handlers in the future. You can also have multiple handlers enabled simultaneously, allowing you to gradually change your metric backend from Exometer to Prometheus. Conversely, you can also disable all instrumentation, which was not possible prior to version 6.3. Although it might make little sense at first glance, because it can render the system a black box, it can be useful to gain extra performance in some cases, e.g. if the external metrics like CPU usage are enough, in case of an isolated embedded system, or if resources are very limited.

There are more options possible, and you can find them in the [documentation](https://esl.github.io/MongooseDocs/latest/configuration/instrumentation/). You can also find more details and examples of instrumentation in the detailed [blog post](https://www.erlang-solutions.com/blog/mongooseim-6-3-prometheus-cockroachdb-and-more/).

## CockroachDB – a database that scales with MongooseIM

MongooseIM works best when paired with a relational database like PostgreSQL or MySQL, enabling easy cluster node discovery with [CETS](https://esl.github.io/MongooseDocs/latest/tutorials/CETS-configure/) and persistent storage for users' accounts, archived messages and other kinds of data. Although such databases are not horizontally scalable out of the box, you can use managed solutions like [Amazon Aurora](https://aws.amazon.com/rds/aurora/), [AlloyDB](https://cloud.google.com/products/alloydb) or [Azure Cosmos DB for PostgreSQL](https://learn.microsoft.com/en-us/azure/cosmos-db/postgresql/). The downsides are the possible vendor lock-in and the fact that you cannot host and manage the DB yourself. With version 6.3 however, the possibilities are extended to [CockroachDB](https://www.cockroachlabs.com/). This PostgreSQL-compatible distributed database can be used either as a provider-independent cloud-based solution or as an internally hosted cluster. You can instantly set it up in your local environment and take advantage of the horizontal scalability of both MongooseIM and CockroachDB. If you want to learn how to deploy both MongooseIM and CockroachDB in Kubernetes, see the [documentation](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-with-kubernetes) for CockroachDB and the [Helm chart](https://artifacthub.io/packages/helm/mongoose/mongooseim) for MongooseIM, together with our recent [blog post](https://www.erlang-solutions.com/blog/instant-scalability-with-mongooseim-and-cets/) about setting up an auto-scalable cluster. If you are interested in having an auto-scalable solution deployed for you, please consider our [MongooseIM Autoscaler](https://www.erlang-solutions.com/landings/mongooseim-autoscaler/).

## What's next?

You can read more about MongooseIM 6.3 in the detailed [blog post](https://www.erlang-solutions.com/blog/mongooseim-6-3-prometheus-cockroachdb-and-more/). We also recommend visiting our [product page](https://www.erlang-solutions.com/technologies/mongooseim/) to see the possible options of support and the services we offer. You can also try the server out at [trymongoose.im](https://trymongoose.im/).

[Read about Erlang Solutions as sponsor of the XSF](/sponsors/erlang-solutions/).

{{< figure src="/images/logos/erlang-solutions.png" class="p-2 sponsor-logo" >}}