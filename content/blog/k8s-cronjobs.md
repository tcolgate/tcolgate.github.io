---
title: "Prometheus: K8s Cronjob alerts (Qubit Eng Blog)"

draft: false
date: "2018-03-04"

description: ""
categories:
    - "Prometheus"
    - "Kubernetes"
    - "monitoring"
    - "alerting"
---
*Note:* This was originally posted on the [Qubit Engineering
Blog](https://medium.com/@tristan_96324/prometheus-k8s-cronjob-alerts-94bee7b90511)
[Coveo](https://www.coveo.com/en). It was written, and illustrated, by me, but
would have been unreadable but for input and editing by my fantastic teammates.

Kubernetes [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#cron-jobsL) and [Cronjobs]( https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) are powerful tools that present some
interesting challenges to monitoring. We’ll demonstrate rules using
[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) and Prometheus to monitor and alert on failed jobs.
We’ll discuss how the rules were formulated, and how to leverage Prometheus
label matching techniques for advanced alerting.  I have attempted to
demonstrate the workflow for developing such alerts.  The final result, for the
bored or impatient, can be found
[here](https://github.com/tcolgate/prometheus-rules/blob/master/kubernetes/cron.yaml).

Alerting of traditional Unix cronjobs was “simple”, if the job failed
you’d get an email. Most job scheduling systems that have followed have
provided the same experience, Kubernetes does not. One [excellent](https://www.robustperception.io/monitoring-batch-jobs-in-python/)
approach to alerting jobs is to use the Prometheus push gateway, allowing
us to push richer metrics than simple success/failure. This approach has
it’s downsides; we have to update the code for our jobs, we also have to
explicitly configure a push gateway location and update it if it changes
(a burden alleviated by the pull based metrics for long lived workloads).
Tools such as [Sentry](https://sentry.io/welcome/) can also be used, but will also require changes to
simple jobs.With some effort we can achieve an easy middle ground.

## K8s Jobs and Cronjobs

In Kubernetes `cronjobs` are built on `jobs` which are themselves built
on pods. Jobs are powerful things allowing us to implement several
different workflows, the combination of options can be overwhelming
compared to a traditional Unix cron job.

The variety of options makes it difficult to establish one simple rule for
alerting failed jobs. Things are easier if we restrict ourselves to a
simpler subset of possible options. We will focus on the following style
of job:

* Run periodically on a schedule
* Non-concurrent

Concurrent jobs permit a second instance of a task to start if a previous
task for a job is still running, this is the behaviour of a classic cron
task. I have focused on non-concurrent tasks. I suggest that
non-concurrent tasks have the desired behaviour for the majority of
typical systems administration tasks (database backups, batch reindexes
and such like), people often overlook the possibility of job runs
overlapping.

The relationship between cronjobs and jobs makes the task ahead difficult.
To make our life easier we will put one requirement on the jobs we create,
they will have to include a label that associates them with the original
cronjob. Below we present an example of our ideal cronjob:

```yaml
   apiVersion: batch/v1beta1
   kind: CronJob
   metadata:
    name: our-task
   spec:
    schedule: "*/5 * * * *"
    successfulJobsHistoryLimit: 3
    concurrencyPolicy: Forbid
    jobTemplate:
      metadata:
        labels:
          cronjob: our-task # <-- match created jobs with the cronjob
      spec:
        backoffLimit: 3
        template:
          metadata:
            labels:
              cronjob: our-task
          spec:
            containers:
            - name: our-task
              command:
              - /user/bin/false
              image: alpine
            restartPolicy: Never
```

## Building our alert

We are also going to need some metrics to get us started. K8s does not
provide us any by default, but fortunately we can leverage
[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) (you will need at least version 1.2). Once
installed we will get the following metrics for the above job:

```promql
   kube_cronjob_labels{cronjob="our-task", namespace="default"} 1
   kube_job_created{job="our-task-1520165700", namespace="default"} 1.520165707e+09
   kube_job_failed{condition="false", job="our-task-1520165700", namespace="default"} 0
   kube_job_failed{condition="true", job="our-task-1520165700", namespace="default"} 1
   kube_job_labels{job="our-task-1520165700", label_cronjob="our-task", namespace="default"} 1
```

This shows the primary set of metrics we will be using to construct our
alert. What is not shown above is the status of the cronjob. The big
challenge with K8s cronjob alerting is that cronjobs themselves do not
possess any status information, beyond the last time the cronjob created a
job. The status information only exists on the job that the cronjob
creates.

In order to determine if our cronjob is failing, our first order of
business is to find which jobs we should be looking at. A K8s cronjob
creates new job objects and keeps a number of them around to help us debug
the runs of our jobs. We have to be determine which job corresponds to the
last run of our cronjob. If we have added the cronjob label to the jobs as
above, we can find the last run time of the jobs for a given cronjob as
follows:

```yaml
         max(
           kube_job_status_start_time
           * ON(exported_job) GROUP_RIGHT()
           kube_job_labels{label_cronjob!=""}
         ) BY (exported_job, label_cronjob)
```

This query demonstrates an important technique when working with
kube-state-metrics. For each API object it exported data on, it exports a
time series including all the labels for that object. These time series
have a value of 1. As such we can join the set of labels for an object
onto the metrics about that object by multiplication.

Depending on how your Prometheus instance is configured, the value of the
job label on your metrics will likely be “kube-state-metrics”.
kube-state-metrics adds a `job` label itself with the name of the job
object. Prometheus resolves this collision of label names by including the
raw metric’s label as an `exported_job label`.

Since we are querying the start time of jobs, and there should only every
be one job with a given name, you may wonder why we need the
max aggregation. Manually plugging the query into Prometheus may convince
you that it is unnecessary. Consider though that you may have multiple
instances of kube-state-metrics running for redundancy. Using max ensures
our query is valid even if we have multiple instances of
kube-state-metrics running. Issues of duplicate metrics are common when
constructing production recording rules and alerts.

We can find the start time of the most recent job for a given cronjob by
finding the maximum of all job start times as follows:

```yaml
         max(
           kube_job_status_start_time                                    
           * ON(exported_job) GROUP_RIGHT()
           kube_job_labels{label_cronjob!=""}
         ) BY (label_cronjob),
```

The only difference between this and the previous query is in the labels
used for the aggregation. Now that we have the start time of each job, and
the start time of the most recent job, we can do a simple equality match
to find the start time of the most recent job for a given cronjob. We will
create a metric for this

```yaml
  - record: job_cronjob:kube_job_status_start_time:max                    
    expr: |
      label_replace(
        label_replace(
          max(
            kube_job_status_start_time
            * ON(exported_job) GROUP_RIGHT()
            kube_job_labels{label_cronjob!=""}
          ) BY (exported_job, label_cronjob)

          == ON(label_cronjob) GROUP_LEFT()

          max(
            kube_job_status_start_time
            * ON(exported_job) GROUP_RIGHT()
            kube_job_labels{label_cronjob!=""}
          ) BY (label_cronjob),
          "job", "$1", "exported_job", "(.+)"),
        "cronjob", "$1", "label_cronjob", "(.+)")
```

In addition to the equality test, we have also taken the opportunity to
adjust the labels to be a little more aesthetically pleasing. Now that we
have the most recently started job for a given cronjob, it is a simple
step to find which, if any, have failed attempts:

```yaml
   - record: job_cronjob:kube_job_status_failed:sum
     expr: |
        clamp_max(
          job_cronjob:kube_job_status_start_time:max,
        1)

        * ON(job) GROUP_LEFT()
   
        label_replace(
          label_replace(
            (kube_job_status_failed != 0),
            "job", "$1", "exported_job", "(.+)"),
          "cronjob", "$1", "label_cronjob", "(.+)")
```

The initial `clamp_max` clause is used to transform our start times metric
into a set of time series we can use purely for the purpose of label
matching to filter another set of metrics. Multiplication by 1 (or
addition of 0), is a useful means of filter and merging time series and it
is well worth taking the time to understand the technique. We adjust the
labels on the raw `kube_job_status_failed` to match our start time metric so
ensure the labels have the same meaning as those on our
`job_cronjob:kube_job_status_start_time:max metric`. The label matching on
the multiplication will then perform our filtering. We now have a metric
containing the set of most recently failed jobs, labelled by their parent
cronjob. Constructing an alert from here is trivial:

```yaml
 - alert: CronJobStatusFailed
    expr: |
      job_cronjob:kube_job_status_failed:sum
      * ON(cronjob) GROUP_RIGHT()
      kube_cronjob_labels
      > 0
    for: 1m
    annotations:
      description: '{{ $labels.cronjob }} last run has failed {{ $value }}
 times.'
```

We use the `kube_cronjob_labels` here to merge in labels from the original
cronjob. By labelling cronjobs with Slack channel, email address, or
OpsGenie team information, alerts can be routed to the specific team that
owns the job.

## Conclusion

Constructing Prometheus recording rules and alerts can be tricky. The
example presented above, and in the Apdex alerting article, are fairly
typical of the techniques needed for more advanced alerts. Hopefully these
two articles have given you some insight and tips for your own alerting
needs.

