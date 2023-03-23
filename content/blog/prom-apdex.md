---
title: "Prometheus: Apdex alerting (Qubit Eng Blog)"

draft: false
date: "2018-03-03"

description: ""
categories:
    - "Prometheus"
    - "timeseries"
    - "apdex"
    - "monitoring"
---

*Note:* This was originally posted on the [Qubit Engineering
Blog](https://medium.com/@tristan_96324/prometheus-apdex-alerting-d17a065e39d0)),
and CopyRight is, at the time of writing, owned by
[Coveo](https://www.coveo.com/en). It was written, and illustrated, by me, but
would have been unreadable but for input and editing by my fantastic teammates.

I’m going to take a break from our series on Production Kubernetes to
write up a couple of smaller articles on Prometheus. This article will
give an overview of Apdex, and go on to describe a practical approach to
implementing Apdex alerts in Prometheus.

## Why Apdex?

Apdex provides a single number that attempts to quantify a user’s
experience of requests to a web service. We first decide what we consider
to be an acceptable response time from our service, in Apdex terminology
this is called the T value. We then classifies requests as follows:

* All error responses are intolerable.
* All responses qucker than T are satisfactory.
* All responses slower than T, but quicker than 4T are tolerable.
* All responses slower than 4T are intolerable.

The Apdex score for a service, over a given time, is a ratio of
intolerable vs satisfactory and tolerable responses.

 apdex = (satisfactory + (tolerable / 2)) / total

 High error rates and slow response times will result in a lower Apdex
 score. By encapsulating errors and high latency in a single metric Apdex
 attempts to quantify our users experience of our service as a single
 number. Since a low Apdex score implies either high rates of errors or
 undesirable response latencies it is a reliable metric to alarm on. While
 a low Apdex score reliably indicates issues, depending on where it is
 measured from, a high score may not guarantee all is well.

## Application metrics

Since Apdex represents user experience we would ideally measure it
externally from the application. Load balancer metrics are a good choice,
([Traefik](https://traefik.io/)provides all the required metrics). Apdex can
also be measured directly from the application as long as we keep in mind that
we are not guaranteed to be seeing all errors. Refused connection are an
example of an error that is probably not measurable from inside the
application. On the other hand, application side measurement allows us more
control over the metrics we produce.

You can find a simple example Go application with suitably instrumented
handlers, along with Prometheus configuration files,
[here](https://github.com/tcolgate/prometheus-rules/tree/master/apd). The
instrumentation provide the following metrics:

* `http_apdex_target_seconds` is a gauge metric providing our target request
  duration.
* `http_request_duration_seconds` is a histogram of response times with
  handler name and response code labels.

The Prometheus documentation rightly cautions against adding too many
labels to metrics. Including the response code and handler as a labels is
questionable. Request codes are arbitrary (though usually between 200 and
599) and including them can result in an explosion of the number of time
series your application is reporting. In addition to this, it has been
suggested that differentiating errors with labels can lead poor
dashboarding and alerting by allowing people to only focus on successful
calls. For the purpose of Apdex calculation we need to differentiate
errors from successes. In our example we use the response code, other
approaches are possible. In practice your author has not seen problems
including code and handler as labels on production services.

Since we will probably want to our histogram for more than just Apdex
calculations, we include some additional buckets. The only requirement is
that we must have buckets at exactly T and 4T, other buckets are included
here to demonstrate that they do not impact the overall calculation.

## Calculating Apdex

The approach given below uses some relatively advanced Prometheus query
techniques which I hope you will find interesting and useful in future. We
will walk through each rule, giving the motivation and a brief discussion
of how it works. First off let us look at our raw material, what metrics
is our application producing (I have filtered out series that we do not
need for our calculations).

```PromQL
   http_request_apdex_target_seconds 0.1
   http_request_duration_seconds_bucket{code="200",handler="/healthz",le="0.1"} 10
   http_request_duration_seconds_bucket{code="200",handler="/healthz",le="0.4"} 10
   http_request_duration_seconds_bucket{code="200",handler="/healthz",le="+Inf"} 10
   http_request_duration_seconds_bucket{code="200",handler="/q",le="0.1"} 775
   http_request_duration_seconds_bucket{code="200",handler="/q",le="0.4"} 1605
   http_request_duration_seconds_bucket{code="500",handler="/q",le="0.1"} 390
   http_request_duration_seconds_bucket{code="500",handler="/q",le="0.4"} 390
   http_request_duration_seconds_count{code="500",handler="/q"} 390
```

Since each instance of the application may expose a different value for
any metric, we must establish what the current preferred T value is. We
shall use the current maximum for a given job.

```yaml
   - record: job:http_request_apdex_target_seconds:max
     expr: max(http_request_apdex_target_seconds) BY (job)
```

{{< figure src="/img/prom-apdex/prom-apdex.fig1.png" >}}

Our new metric is convenient for adding to graphs to indicate the expected
latency, but we must adjust it to be useful for alerting. Prometheus histograms
use a label (`le`) to label each bucket. We are going to want to be able to
select out buckets based on our `T` value. We could have simply produced some
metrics with the required labels, but as alternative, we can also use the
following rules to produce the required series on the Prometheus server side

```yaml
   - record: job_le:http_request_apdex_target_seconds:max
     expr: |
       clamp_max(
          count_values(
            "le",
            job:http_request_apdex_target_seconds:max
          ) BY (job),
          1)

   - record: job_le:http_request_apdex_target_seconds:4_times_max
     expr: |
       clamp_max(
         count_values(
           "le",
           job:http_request_apdex_target_seconds:max * 4
         ) BY (job),
         1)
```

These rules transform our selected `job:http_apdex_target_seconds:max` in to
two new metrics. These represent `T` and `4T`. Rather than simply having a
metric with the specific values, `count_values` transforms the value of a
time series into a label, `le` in our case, with the value of the label being
a string representation of the values. Since we know we only have 1
occurrence of this metric for a given instance of our application, we know
the count we always be 1. This results in the following metrics

```PromQL
   job_le:http_request_apdex_target_seconds:max{le="0.1"} = 1
   job_le:http_request_apdex_target_seconds:4_times_max{le="0.4"} = 1
```

The `le` label can now be used for label matching in calculations involving
our histogram. We can finally calculate our Apdex score. We will adjust
the formula slightly

```
   apdex = (
           (
             satisfactoryRate +
             (satisfactor + tolerableRate)
           ) / 2
          ) / totalRate
```

We use rates rather than total requests. We could have used `increase()`  in
place of `rate()`, however Prometheus estimates increases by multiply rates
by the requested duration, the end result is equivalent. Also, since our
histogram buckets count all samples with duration less than or equal to
the specific bucket, the `4T` bucket will also include sample from the
`T` bucket. We could subtract, but we can also just move the division.

For low traffic services it is often desirable to ignore our own health
checks. Health checks are useful for establishing if a service is capable
of serving traffic, but rarely catch unexpected errors. They often only
trivially test back end services, so are unlikely to timeout in the way
that bad user requests can. If we do not ignore our health checks they may
artificially raise our Apdex score hiding real problems in a fog of fake
successes. We will ignore requests to our health check handler.

Translating this to PromQL we get

```yaml
   - record: job:http_request_apdex:max
     expr: |
      (
        sum(
          rate(        
            http_request_duration_seconds_bucket{
              handler!="/healthz",
              code!~"5.."}[1m]
          )
          * ON(job, le) GROUP_LEFT()
          job_le:http_request_apdex_target_seconds:max
        ) BY (job)
       
        +
       
        sum(
          rate(
            http_request_duration_seconds_bucket{
              handler!="/healthz",
              code!~"5.."}[1m]
          )
          * ON(job, le) GROUP_LEFT()
          job_le:http_request_apdex_target_seconds:4_times_max
        ) BY (job)
       
      ) / 2
       
      /
       
      sum(
        rate(
          http_request_duration_seconds_count{
            handler!="/healthz"}[1m]
        )
      ) BY (job)
```

This query can be divided up into simpler sections. First we calculate the
rate of satisfactory requests. As per the rules given in the first
section, this is made up of all successful (non-5xx) requests that have
been sampled by our `le=”T”` bucket. We then calculate our tolerable
requests from the `4T` bucket. Finally we divide the sum of those by 2, and
then by the total number of requests to the service. The bucket selection
is achieved through label matching on our `le` label. The `GROUP_LEFT` clauses
are used to retain any additional instrumentation labels that were present
on the original time series.

Finally we can create an alert.

```yaml
   - alert: HTTPApdexViolation
     expr: job:http_request_apdex:max < 0.8
     for: 5m
     annotations:
       description: |
         {{ $labels.job }} has ApDex has dropped below 0.8 {{printf "%.2f" $value}}
```

We could allow the alert threshold, here set to 0.8, to be specified by
the application by exposing it as a metrics, as we did for the T value. To
do so though would encourage developers to lower the alert threshold in
response to an application which is not meeting its Apdex target. In my
opinion it is more honest to adjust the target response time rather than
the alert threshold. This encourages discussions around what is really
achievable, rather than simply twiddling a “magic alert threshold”.

## Conclusion

Apdex is a useful metric for monitoring, and especially for alerting. It
is a concise representation of user experience. Prometheus provides us all
the tools we need to calculate Apdex. The same technique presented here
can be used to estimate similarly “Business-centric” metrics, such as
error-budgets and service availability.
