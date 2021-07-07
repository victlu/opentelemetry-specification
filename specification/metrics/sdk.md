# Metrics SDK

**Status**: [Experimental](../document-status.md)

**Owner:**

* [Reiley Yang](https://github.com/reyang)

**Domain Experts:**

* [Bogdan Drutu](https://github.com/bogdandrutu)
* [Josh Suereth](https://github.com/jsuereth)
* [Joshua MacDonald](https://github.com/jmacd)

Note: this specification is subject to major changes. To avoid thrusting
language client maintainers, we don't recommend OpenTelemetry clients to start
the implementation unless explicitly communicated via
[OTEP](https://github.com/open-telemetry/oteps#opentelemetry-enhancement-proposal-otep).

<details>
<summary>
Table of Contents
</summary>

* [MeterProvider](#meterprovider)
* [MeasurementProcessor](#measurementprocessor)
* [MetricProcessor](#metricprocessor)
* [MetricExporter](#metricexporter)
  * [Push Metric Exporter](#push-metric-exporter)
  * [Pull Metric Exporter](#pull-metric-exporter)

</details>

## MeterProvider

TODO:

* Allow multiple pipelines (processors, exporters).
* Configure "Views".
* Configure timing (related to [issue
  1432](https://github.com/open-telemetry/opentelemetry-specification/issues/1432)).

## MeasurementProcessor

`MeasurementProcessor` is an interface which allows hooks when a
[Measurement](./api.md#measurement) is recorded by an
[Instrument](./api.md#instrument).

`MeasurementProcessor` MUST have access to:

* The `Measurement`
* The `Instrument`, which is used to report the `Measurement`
* The `Resource`, which is associated with the `MeterProvider`

In addition to things listed above, if the `Measurement` is reported by a
synchronous `Instrument` (e.g. [Counter](./api.md#counter)),
`MeasurementProcessor` MUST have access to:

* [Baggage](../baggage/api.md)
* [Context](../context/context.md)
* The [Span](../trace/api.md#span) which is associated with the `Measurement`

Depending on the programming language and runtime model, these can be provided
explicitly (e.g. as input arguments) or implicitly (e.g. [implicit
Context](../context/context.md#optional-global-operations) and the [currently
active span](../trace/api.md#context-interaction)).

```text
+------------------+
| MeterProvider    |                 +----------------------+            +-----------------+
|   Meter A        | Measurements... |                      | Metrics... |                 |
|     Instrument X |-----------------> MeasurementProcessor +------------> In-memory state |
|     Instrument Y +                 |                      |            |                 |
|   Meter B        |                 +----------------------+            +-----------------+
|     Instrument Z |
|     ...          |                 +----------------------+            +-----------------+
|     ...          | Measurements... |                      | Metrics... |                 |
|     ...          |-----------------> MeasurementProcessor +------------> In-memory state |
|     ...          |                 |                      |            |                 |
|     ...          |                 +----------------------+            +-----------------+
+------------------+
```

### Aggregator

An `Aggregator` is a type of `MeasurementProcessor` and computes "aggregate"
data from [Measurements](./api.md#measurement) and its `In-Memory State` into
[Pre-Aggregated Metric](./datamodel.md#timeseries-model).

Diagram:

```text

                 +---------------------+
                 | Aggregator          |
                 |                     |
 [Measurements]------> "Aggregate"     |
                 |         |           |
                 | +-------V---------+ |
                 | |                 | |
                 | | In-Memory State | |
                 | |                 | |
                 | +-------+---------+ |
                 |         |           |
                 |         V           |
                 |      "Collect" +------[Pre-Aggregated Metrics]-->
                 |                     |
                 +---------------------+
```

An `Aggregator` MUST provide an interface to "aggregate" [Measurement](./api.md#measurement)
data into its `In-Memory State`.

An `Aggregator` MUST provide an interface to "collect" [Pre-Aggregated Metric](./datamodel.md#timeseries-model)
data from its `In-Memory State`.

An `Aggregator` MUST have read/write access to memory storage (`In-Memory
State`) where it can store/retreive/manage its own internal state. SDK MUST
provide consideration and control for memory availability and usage.

An `Aggregator` MUST have access to or given the `View` configuration so it can
be properly configure. e.g. For Monoticity and/or Temporality.

SDK MUST provide aggregators to support the default aggregator configuration
per instrument kind. e.g. A "Sum" aggregator to compute the sum "aggregate" for
counter instruments and a "Histogram" aggregator for histogram instruments.

### Last Value Aggregator

The Last Value Aggregator supports the [Gauge Metric Point](./datamodel.md#gauge)
and is default aggregator for the following instruments.

* [Asynchronous Gauge](./api.md#asynchronous-gauge) instrument.

Last Value Aggregator stores the following in memory:

* Last Value (latest Measurement given.)

### Sum Aggregator

The Sum Aggregator supports the [Sum Metric Point](./datamodel.md#sums)
and is default aggregator for the following instruments.

* [Counter](./api.md#counter) instruments.
* [UpDown Counter](./api.md#updown-counter) instruments.
* [Asynchronous Counter](./api.md#asynchronous-counter) instruments.
* [Asynchronous UpDown Counter](./api.md#asynchronous-updown-counter) instruments.

The Sum Aggregator MUST be configurable to support different Monoticity and/or
Temporality.

Sum Aggregator stores the following in memory:

* Time window (e.g. start time, end time)
* Sum (sum of Measurements per Monoticity and Temporality configuration.)

### Histogram Aggregator

The Histogram Aggregator supports the [Histogram Metric Point](./datamodel.md#histogram)
and is default aggregator for the following instruments.

* [Histogram](./api.md#histogram) instruments.

The Histogram Aggregator MUST be configurable to support different Temporality.

Histogram Aggregator stores the following in memory:

* Time window (e.g. start time, end time)
* Count
* Sum
* Bucket Counts (per configured boundaries)

### An example of SDK implementation

SDK expand combination of attribute keys/values of each measurement
and direct each distinct combination to a different instance of an aggregator.

```text
+------------------+  +-------------------------------------------------------------+
| MeterProvider    |  | SDK                                                         |
|   Meter A        |  |                     +----------------------+                |
|     Instrument X |-----[Measurements]---->| MeasurementProcessor |                |
|     Instrument Y |  |                     +-----------+----------+                |
|     ...          |  |                                 |                           |
|     ...          |  |                           [Measurements]                    |
+------------------+  |                                 |                           |
                      |        SDK expand Measurements per View configuration       |
                      |        (e.g. by attribute key/value pairs)                  |
                      |                                 |                           |
                      |                                 |"Aggregate"                |
                      | Measurement #1:                 V                           |
                      |                      +---------------------+                |
                      |      B=Y ----------->| Aggregator #1 (B=Y) |---->+          |
                      |                      +---------------------+     |          |
                      |      A=X -------+                                |          |
                      |                 |    +---------------------+     |          |
                      | Measurment #2:  +--->| Aggregator #2 (A=X) |---->|          |
                      |                 |    +---------------------+     |          |
                      |      A=X -------+                                |          |
                      |                      +---------------------+     |          |
                      |      B=Z ----------->| Aggregator #3 (B=Z) |---->|          |
                      |                      +---------------------+     |          |
                      |                                                  |"Collect" |
                      |                                                  |          |
                      |                                              [Metrics]      |
                      |                                                  |          |
                      |                                        +---------V-------+  |
                      |                                        | MetricProcessor |  |
                      |                                        +---------+-------+  |
                      |                                                  |          |
                      |                                              [Metrics]      |
                      |                                                  |          |
                      |                                        +---------V-------+  |
                      |                                        | MetricExporter  |=====>
                      |                                        +-----------------+  |
                      +-------------------------------------------------------------+
```

## MetricProcessor

`MetricProcessor` is an interface which allows hooks for [pre-aggregated metrics
data](./datamodel.md#timeseries-model).

Built-in metric processors are responsible for batching and conversion of
metrics data to exportable representation and passing batches to exporters.

The following diagram shows `MetricProcessor`'s relationship to other components
in the SDK:

```text
+-----------------+            +-----------------+            +-----------------------+
|                 | Metrics... |                 | Metrics... |                       |
> In-memory state | -----------> MetricProcessor | -----------> MetricExporter (push) |--> Another process
|                 |            |                 |            |                       |
+-----------------+            +-----------------+            +-----------------------+

+-----------------+            +-----------------+            +-----------------------+
|                 | Metrics... |                 | Metrics... |                       |
> In-memory state |------------> MetricProcessor |------------> MetricExporter (pull) |--> Another process (scraper)
|                 |            |                 |            |                       |
+-----------------+            +-----------------+            +-----------------------+
```

## MetricExporter

`MetricExporter` defines the interface that protocol-specific exporters must
implement so that they can be plugged into OpenTelemetry SDK and support sending
of telemetry data.

The goal of the interface is to minimize burden of implementation for
protocol-dependent telemetry exporters. The protocol exporter is expected to be
primarily a simple telemetry data encoder and transmitter.

Metric Exporter has access to the [pre-aggregated metrics
data](./datamodel.md#timeseries-model).

There could be multiple [Push Metric Exporters](#push-metric-exporter) or [Pull
Metric Exporters](#pull-metric-exporter) or even a mixture of both configured on
a given `MeterProvider`. Different exporters can run at different schedule, for
example:

* Exporter A is a push exporter which sends data every 1 minute.
* Exporter B is a push exporter which sends data every 5 seconds.
* Exporter C is a pull exporter which reacts to a scraper over HTTP.
* Exporter D is a pull exporter which reacts to another scraper over a named
  pipe.

### Push Metric Exporter

Push Metric Exporter sends the data on its own schedule. Here are some examples:

* Sends the data based on a user configured schedule, e.g. every 1 minute.
* Sends the data when there is a severe error.

### Pull Metric Exporter

Pull Metric Exporter reacts to the metrics scrapers and reports the data
passively. This pattern has been widely adopted by
[Prometheus](https://prometheus.io/).
