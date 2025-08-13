---
title: "Understanding metrics collection, filtering, and enrichment with Diginsight"
author: "Dario Airoldi"
date: "2025-08-12"
categories: [news, code, copilot, development]
image: "images/000.00 diginsight.span_duration.png"
draft: false
---

# Introduction

**Diginsight** is a .NET observability framework that provides **automatic instrumentation for applications using OpenTelemetry**. 

One of its key capabilities is the automatic collection of performance and operational metrics **without requiring manual instrumentation**.

Diginsight automatically produces metrics such as:

- **"diginsight.span_duration"**: latency of a span
- **"diginsight.query_cost"**: RU cost of a CosmosDB query  
- **"diginsight.request_size"**: request size of an HTTP request
- **"diginsight.response_size"**: response size of an HTTP request

In this article we'll learn that:

- Diginsight collects metrics using **OpenTelemetry** and **.NET Activity** classes.
- **Metrics are collected** during the activity lifecycle.
- Metrics are **filtered** and **enriched** before being sent to the OpenTelemetry collector, according to the application configuration.

## Table of Contents

1. [Why Automatic Metrics Collection Matters](#why-automatic-metrics-collection-matters)
2. [Understanding .NET Activities and Metrics](#understanding-net-activities-and-metrics)
3. [Metric Collection](#metric-collection)
4. [Metric Filtering and Tag Enrichment](#metric-filtering-and-tag-enrichment)
   - [MetricRecordingNameBasedFilter service](#metricrecordingnamebasedfilter-service)
   - [MetricRecordingTagsEnricher service](#metricrecordingtagsenricher-service)
5. [Startup Sequence Configuration](#startup-sequence-configuration)
6. [Summary](#summary)
7. [References](#references)

## Why Automatic Metrics Collection Matters

Manual metrics instrumentation can be:

- **Time-consuming**: Requires developers to add instrumentation code throughout the application.
- **Error-prone**: Easy to miss critical operations or add inconsistent measurements.
- **Maintenance overhead**: Metrics code needs to be updated as business logic changes

Diginsight solves these challenges by detecting and measuring key operations automatically, providing immediate observability insights with minimal configuration.

## Understanding .NET Activities and Metrics

Diginsight's metrics collection follows a four-step process within each metric recorder component:

1. **Collection**: Metrics are automatically captured during .NET Activity lifecycles
2. **Filtering**: Configurable filters determine which activities should generate metrics
3. **Enrichment**: Additional contextual tags are added based on configuration
4. **Export**: Processed metrics are sent to OpenTelemetry collectors

This integrated approach ensures that you get meaningful metrics **without noise** or **performance impact**.

**.NET Activities** represent units of work in your application (HTTP requests, database queries, method calls). Diginsight hooks into the Activity lifecycle events:

- **ActivityStarted**: Sets up context for metric collection
- **ActivityStopped**: Records duration and other measurements
- **Activity Tags**: Provide context that becomes metric labels

This automatic hooking means metrics are captured by OpenTelemetry, for any instrumented operation, without additional code.

## Metric Collection 

Metrics are collected across the application flow during the .NET Activities' lifetime.

The image below shows the **`SpanDurationMetricRecorder`** that records the **`diginsight.span_duration`** metric, at the end of an activity lifecycle.

![alt text](<images/002.01 SpanDurationMetricRecorder recording span_duration metric.png>)

The following code shows how the `SpanDurationMetricRecorder` handles the end of an activity lifecycle. 
Notice how it:

1. Checks if the metric should be recorded (filtering)
2. Extracts basic tags like span name and status
3. Adds enrichment tags from configuration
4. Records the final metric with all tags

```csharp
void IActivityListenerLogic.ActivityStopped(Activity activity)
{
    string activityName = activity.OperationName;

    try
    {
        Type? callerType = activity.GetCallerType();
        IDiginsightActivitiesMetricOptions activitiesOptions = activitiesOptionsMonitor.Get(callerType);
        if (!(metricFilter?.ShouldRecord(activity) ?? activitiesOptions.RecordSpanDurations))
            return;

        //Tag traceId = new("trace_id", activity.TraceId.ToString());
        Tag nameTag = new("span_name", activityName);
        Tag statusTag = new("status", activity.Status.ToString());
        Tag[] tags = metricEnricher is not null ? [nameTag, statusTag, .. metricEnricher.ExtractTags(activity)] : [nameTag, statusTag];

        Metric.Record(activity.Duration.TotalMilliseconds, tags);
    }
    catch (Exception exception)
    {
        logger.LogWarning(exception, "Unhandled exception while recording span duration metric of activity {ActivityName}", activityName);
    }
}
```

The metric is only recorded **if the activity is not filtered out** by `metricFilter?.ShouldRecord(activity)`.

Also, **the metric is enriched** with a set of tags such as `span_name` and `status`, and possibly additional tags extracted by `metricEnricher.ExtractTags(activity)`.

Similar logic happens when recording the `query_cost`, `request_size`, and `response_size` metrics.

## Metric Filtering and Tag Enrichment

Filtering and enrichment are carried out by means of two services:

- **`MetricRecordingNameBasedFilter`**: with **metricFilter?.ShouldRecord(activity)**, it decides whether a specific activity should emit the **span_duration** metric

- **`MetricRecordingTagsEnricher`**: with **metricEnricher.ExtractTags(activity)**, it enriches the activity with tags according to the application configuration.

### MetricRecordingNameBasedFilter service

`MetricRecordingNameBasedFilter` filters activities based on their names, normally from a section such as **`Diginsight:Activities:SpanMeasuredActivityNames`**.

`SpanMeasuredActivityNames` can be empty, meaning that all activities are recorded, or it can contain a list of activity names to be recorded.

```json
"Diginsight": {
  "Activities": {
    "SpanMeasuredActivityNames": {
    },
    "MetricSpecificSpanMeasuredActivityNames": [
      {
        "MetricName": "diginsight.span_duration",
        "ActivityNames": {
        }
      },
      {
        "MetricName": "diginsight.query_cost",
        "ActivityNames": {
          "CosmosDbExtensions.GetItemLinqQueryableObservable": true
        }
      }
    ],
  }
}
```

`MetricSpecificSpanMeasuredActivityNames` allows you to specify activities for specific metrics, such as `diginsight.span_duration` and `diginsight.query_cost`.

The image below shows the **`MetricRecordingNameBasedFilter`** implementation, which receives enabled activities by means of a `MetricRecordingNameBasedFilterOptions` structure that is initialized in the startup sequence.

![alt text](<images/002.02 SpanDMetricRecordingNameBasedFilter base implementation.png>)

### MetricRecordingTagsEnricher service

`MetricRecordingTagsEnricher` adds tags to the generated metrics, normally from a section such as **`Diginsight:Activities:MetricTags`**.

```json
"Diginsight": {
  "Activities": {
    "MetricTags": [
      "category_name",
      "plant_id",
      "plant_type",
      "plant_name",
      "plant_company",
      "device_type",
      "user_company"
    ],
    "MetricSpecificTags": [
      {
        "MetricName": "diginsight.span_duration",
        "MetricTags": [
        ]
      },
      {
        "MetricName": "diginsight.query_cost",
        "MetricTags": [ "database", "application_name" ]
      }
    ]
  }
}
```

`MetricSpecificTags` allows you to specify additional tags for specific metrics, such as `diginsight.span_duration` and `diginsight.query_cost`.

The image below shows the **`MetricRecordingTagsEnricher`** implementation, which receives the configured tags by means of a `MetricRecordingTagsEnricherOptions` structure that is initialized in the startup sequence.

![alt text](<images/002.02a MetricRecordingTagsEnricher base implementation.png>)

Adding tags to a metric allows to **filter and group metrics** in the OpenTelemetry collector, for example by `plant_name`, `plant_id`, `category_name`, etc.
> TODO: show query splitting cost by plant_name

For the example case of `diginsight.query_cost`, the tag `database` is added to the metric values to allow splitting the **query-generated cost** for each database.

> TODO: show query splitting cost by database or by application_name

## Startup Sequence Configuration

The `MetricRecordingNameBasedFilter` and `MetricRecordingTagsEnricher` services are configured in the startup sequence, as shown in the code snippet below.

In particular, `SpanMeasuredActivityNames`, `MetricSpecificSpanMeasuredActivityNames`, `MetricTags`, and `MetricSpecificTags` are read from the configuration.

Then, for any of the supported metrics, a **named configuration** is created (for example, `diginsight.span_duration`, `diginsight.query_cost`, etc.), and a **named singleton** is registered with the associated configuration.

```csharp
if (openTelemetryOptions.EnableMetrics)
{
    var diginsightConfig = configuration.GetSection(ConfigurationPath.Combine(diginsightConfKey, "Activities"));

    var defaultMetricActivities = diginsightConfig.GetSection("SpanMeasuredActivityNames").Get<IDictionary<string, bool>>() ?? new Dictionary<string, bool>();
    var metricSpecificActivities = diginsightConfig.GetSection("MetricSpecificSpanMeasuredActivityNames").Get<MetricRecordingNameBasedFilterOptions[]>() ?? Array.Empty<MetricRecordingNameBasedFilterOptions>();
    logger.LogDebug("Found {Count} metric-specific activity configurations", metricSpecificActivities.Length);

    var defaultMetricTags = diginsightConfig.GetSection("MetricTags").Get<string[]>() ?? Array.Empty<string>();
    logger.LogDebug("Default MetricTags: {Tags}", string.Join(", ", defaultMetricTags));
    var metricSpecificTags = diginsightConfig.GetSection("MetricSpecificTags").Get<MetricRecordingEnricherOptions[]>() ?? Array.Empty<MetricRecordingEnricherOptions>();
    logger.LogDebug("Found {Count} metric-specific tag configurations", metricSpecificTags.Length);

    var metricNames = new[] { "diginsight.span_duration", "diginsight.query_cost", "diginsight.request_size", "diginsight.response_size" };
    foreach (var metricName in metricNames)
    {
        // named configuration including metric specific activities
        services.Configure<MetricRecordingNameBasedFilterOptions>(metricName, options =>
        {
            options.MetricName = metricName;

            var activitiesToUse = new Dictionary<string, bool>(defaultMetricActivities);
            var metricConfig = metricSpecificActivities?.FirstOrDefault(m => m.MetricName == options.MetricName);
            if (metricConfig != null) { activitiesToUse.AddRange(metricConfig.ActivityNames); }
            options.ActivityNames = activitiesToUse;
        });
        // named configuration including metric specific tags
        services.Configure<MetricRecordingEnricherOptions>(metricName, options =>
        {
            options.MetricName = metricName;

            var tagsToUse = new List<string>(defaultMetricTags);
            var metricConfig = metricSpecificTags?.FirstOrDefault(m => m.MetricName == options.MetricName);
            if (metricConfig != null) { tagsToUse.AddRange(metricConfig.MetricTags); }
            options.MetricTags = tagsToUse;
        });

        // named filter with associated configuration
        services.AddNamedSingleton<IMetricRecordingFilter, MetricRecordingNameBasedFilter>(
            metricName, (sp, key) =>
            {
                var optsions = sp.GetRequiredService<IOptionsMonitor<MetricRecordingNameBasedFilterOptions>>().Get((string)key!);
                var filter = new MetricRecordingNameBasedFilter(optsions);
                return filter;
            }
        );
        // named enricher with associated configuration
        services.AddNamedSingleton<IMetricRecordingEnricher, MetricRecordingTagsEnricher>(metricName, (sp, key) =>
        {
            var optsions = sp.GetRequiredService<IOptionsMonitor<MetricRecordingEnricherOptions>>().Get((string)key!);
            var filter = new MetricRecordingTagsEnricher(optsions);
            return filter;
        });
    }
```

> The code above is taken from `ObservabilityExtensions.AddObservability()` method into `Diginsight.Components.Configuration` assembly and it is used in all Diginsight Samples, available into the [Diginsight.Samples](https://github.com/diginsight/samples) repository.

After the configuration and named services registration, the Recorder class just needs to ensure it retrieves the named service according to the metric name it is recording.

In the snippet below, we can see the `SpanDurationMetricRecorder` constructor, which retrieves the named services for `IMetricRecordingFilter` and `IMetricRecordingEnricher` by means of `serviceProvider.GetNamedService<IMetricRecordingFilter>(metricName)`.

```csharp
public SpanDurationMetricRecorder(
    IServiceProvider serviceProvider,
    ILogger<SpanDurationMetricRecorder> logger,
    IClassAwareOptionsMonitor<DiginsightActivitiesOptions> activitiesOptionsMonitor,
    IMeterFactory meterFactory
)
{
    this.logger = logger;
    this.activitiesOptionsMonitor = activitiesOptionsMonitor;
    this.meterFactory = meterFactory;

    IDiginsightActivitiesMetricOptions activitiesOptions = activitiesOptionsMonitor.CurrentValue;
    var metricName = activitiesOptions.MetricName;

    // Get metric (delayed) with lazy initialization
    this.lazyMetric = new Lazy<Histogram<double>>(() => {
        IDiginsightActivitiesMetricOptions options = activitiesOptionsMonitor.CurrentValue;
        return meterFactory.Create(options.MeterName)
                            .CreateHistogram<double>(options.MetricName, options.MetricUnit ?? "ms", options.MetricDescription);
    });

    var metricFilter = serviceProvider.GetNamedService<IMetricRecordingFilter>(metricName);
    this.metricFilter = metricFilter ?? serviceProvider.GetRequiredService<IMetricRecordingFilter>();

    var metricEnricher = serviceProvider.GetNamedService<IMetricRecordingEnricher>(metricName);
    this.metricEnricher = metricEnricher ?? serviceProvider.GetRequiredService<IMetricRecordingEnricher>();

```

## Summary

Diginsight transforms metrics collection from a manual, error-prone process into an automatic, configurable system that provides immediate value with minimal effort.

**Key Benefits:**

- **Zero-code metrics**: Get essential performance metrics without instrumentation code
- **Production-ready**: Built-in error handling and performance optimizations
- **Business-aware**: Rich tagging enables filtering by business dimensions
- **Flexible**: Fine-grained control over what gets measured and how

**Best Practices:**

1. Start with default settings to get immediate value
2. Gradually add business-specific tags as observability needs mature
3. Use metric-specific configurations to optimize for different operation types
4. Leverage filtering to focus on critical business operations

This approach enables teams to achieve comprehensive observability with minimal development overhead while maintaining the flexibility to evolve their monitoring strategy as applications scale.

## References

**OpenTelemetry Documentation**

- [OpenTelemetry Metrics Specification](https://opentelemetry.io/docs/specs/otel/metrics/) - Official specification for metrics collection and export
- [OpenTelemetry .NET Getting Started](https://opentelemetry.io/docs/languages/net/getting-started/) - Introduction to OpenTelemetry for .NET applications
- [OpenTelemetry .NET Metrics](https://opentelemetry.io/docs/languages/net/instrumentation/#metrics) - Detailed guide on implementing metrics in .NET

**.NET Documentation**

- [.NET Activity Class](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.activity) - Microsoft documentation for the Activity class used in distributed tracing
- [.NET Metrics](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/metrics) - Official guide to metrics collection in .NET applications
- [Dependency Injection in .NET](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection) - Understanding DI patterns used in the configuration system

**Observability and Monitoring**

- [Prometheus Metrics Types](https://prometheus.io/docs/concepts/metric_types/) - Understanding different types of metrics (counters, gauges, histograms)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/) - Creating visualizations for collected metrics
- [Application Performance Monitoring Best Practices](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) - Microsoft's guide to application monitoring

**Configuration and Patterns**

- [Options Pattern in .NET](https://docs.microsoft.com/en-us/dotnet/core/extensions/options) - Understanding the configuration pattern used for metric filtering and enrichment
- [Named Services in Dependency Injection](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection-guidelines) - Advanced DI patterns for service registration

**Diginsight Related Articles**

- [Application Observability Concepts](https://diginsight.github.io/telemetry/src/docs/01.%20Concepts/00.01%20-%20Observability%20Concepts.html) - Discusses the principles of observability in applications and the roles of .NET, OpenTelemetry, and Diginsight.

