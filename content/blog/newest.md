---
title: "Time Series Search in Prometheus"
date: "2016-09-14"
draft: false
description: "This post describes an experiment in extending Prometheus to support time searching matching"
categories:
    - "prometheus"
    - "fun"
    - "timeseries"
---

# Introduction

*Note:*: This post describes an extentsion to [prometheus](https://prometheus.io) that is unlikely to ever make it upstream.

I should say up-front that I make no claim to any real expertise in this field.
I have gleaned what I could from papers written by those far more qualified
than I, and applied it with a degree of success that could equally be
attributable to blind luck. That being said, I had fun doing it, and my
experience may be intersting, or enlightening, to others.

## Time Series Indexing

Traditionally, when we talk about an index for a timeseries database, we are
talking about an index of the metadata. In the case of prometheus, such an
index would help us locate all the series with certain labels, allowing us to
quickly query, for example, all the _http\_requests\_total_ timeseries that
match a given _handler_ or response _code_. Whilst that is a worthy topic to
explore, this article is not about that.

Most of the timeseries databases we are used to dealing with will not attempt
to index the actual data. It is rare to want to search based on values. This is
especially true when data is stored using floating point, or is derived from
fast changing counters. Searching for an exact float64 value is of limited use.
Even searching specific numbers within a known range is not that useful. Traditional
indexing methods are just not of much interest for the time series data itself.

## Finding signals in the noise

Whilst it's rare to want to find a specific value in our data, it's not
uncommon to with to find correlations between data series. "Who is doing
that?!", is a not uncommon refrain from the desperate systems or network
administrator. Correlation does not imply causation, but it can get the witch hunt
off to a good start.

I often found myself in this situation when working as a network administrator
on a medium sized office and datacenter network in a previous role. The total
estate consisted of several dozen high density switches, totalling a few thousand
ports. It was not uncmmon for a single user monopolise the, relatively small,
central office Internet pipe. Finding which one of the many 1Gb/s ports was
originating the 40Mb/s spike was tedious, and usually involved scrolling
through pages of cacti graphs to locate the floor and port reponsible.

It seemed that it should be possible to find graphs that contained features
with a passing resemblance to the problematic spike. My manual search was
really the search to answer the question of "Which graph has a spike on it that
looks a bit like the one on the Internet link?". But how to get a computer to
do this for us?

## Feature space reduction

Unsuprisingly, many greater minds than mine had expelled much effort to solve
this problem (though, none that I know of, were hunting for rouge bittorrent
users).

The answer lies in a set of techniques called Dimensionality Reduction. The key
is to convert our time series data from it's initial state, we'll call temporal
space, to something called a Feature Space. Feature Spaces represent features
of the data over time, rather than the speific data samples themselves. Once
converted, we effectively reduce the resolution of the data in the feature
space, allowing us to compare the features, rather than the raw data. Such,
comparisons can be much "fuzzier".

The best known example of a feature-space conversion is the Fourier Tranform
and its variants (DCTs, and wavelet transoforms for example). Rather than
represent the time series data as a series of values of the time domain, these
tell us which frequencies are present in the data. These frequencies are the
features.

The broad idea is to convert our time series data into some other form that
more directly represents the features, then compare those features for the
similarities we are interested in.

As tantalising as the frequency domain conversion are, it turns out that some
of their properties  make them less ideal for correlation of spikey network
traffic. In particular, a small shift of a spike of taffic in the temporal
space (appearing very slightly earlier or later in one graph from another), can
result in very different frequency domain data. Small square spikes generally
don't encode well to frequency domain. Such spikes are common in network
monitoring data.

The [SAX](http://www.cs.ucr.edu/~eamonn/SAX.pdf) project came up with an
alternative approach which is easy to understand, and very easy to implememt.

## Peace-wise Aggregate Approximation

The process is straight forward:

* Downsample the data to a convenient number of samples. Typically 6 to 8 are
  enogh
* Normalize this downsampled data (usually [z-Normalization](http://jmotif.github.io/sax-vsm_site/morea/algorithm/znorm.html)
* Quantize the resulting data into a set of ranges (typically 8)
* Assign a letter to each quantized value.

The resulting string encodes the shape of the time series data. Any two time
series with the same final string encoding will have a similar basic shape.
The SAX and [ISAX2](http://www.cs.ucr.edu/~eamonn/iSAX_2.0.pdf) papers give
some example values for the quantization bands they found effective.

After some experementation I eventually implented, and successfully used this
technique using a tool called
[lookalike](https://github.com/tcolgate/lookalike), developed for Cacti.

I toyed with ideas for implementing a similar solution for OpenTSDB, and
InfluxDB, but neither's internals made it particularly easy (the latter used a
map/reduce model internally that proved tricky, the former is written in java)

## Prometheus Implementation

Whilst prometheus does not support string data for values, float64 does give us
plenty of room to pack in a PAA encoding. 8 3-bit samples were generally
sufficient for a reasonable match in lookalike. We can store 52 bit integers
exactly in a floa64, which is far more space than the 24-bits we require.

Prometheus is essentially an in-memory time series database with some support
for spilling out to disk. This has the advantage that all the data is easily
available when an internal function executes.

Proetheus has no internal support for downsampling, so this needed to be done
within the function (Lookalike used the inherent downsampling of an RRD file).
For prometheus I opted to implement
[Least-Triangle-Three-Bucket](http://skemman.is/stream/get/1946/15343/37285/3/SS_MSthesis.pdf).
This was largely chosen because it seemed interesting, and there is a somewhat
"visual" component to what PAA is trying to do. In practice, the technique is
expensive and simpler methods should be tried for comparison.

The process is adjusted as follows:

* LTTB Downsample the data to 8 samples.
* z-Normalize this downsampled data.
* Quantize the resulting data.
* Take each 3 bit sample and pack into a sinlge integer
* Convert that integer to a float64

## Usage

The paa() function takes a range vector and returns an instant vector. The actual value
returned is not of much use directly.

Using the function is rather clumsy. By convention, prometheus functions that
alter the meaning of the data they process remove __name__, this makes
determining what has matched a bit tricky (label_replace tricks can be used to
store the original name on a label, I've left such magic out of the examples
for the sake of clarity).

To calculate a single paa for the all time series for last 15 minutes:

``` paa({__name__=~".+"}[15m]) ```

If we want to find all time series that correlate with our network traffic on eth0:

```
paa({__name__=~".+"}[15m]) ==
paa(task:node_network_transmit_bytes:rate1m{device="etho"}[15m])
```

We can establish a recording rules that will track the PAA for a 15m window
of etwork traffic:

```
task:node_network_receive_bytes:paa15m_rate1m =
  paa(task:node_network_receieve_bytes:rate1m[15m])
task:node_network_transmit_bytes:paa15m_rate1m =
  paa(task:node_network_transmit_bytes:rate1m[15m])
```

If we want to find all network interfaces trasmitting traffic that correlates
strong with traffic received at myhost eth1):

```
task:node_network_transmit_bytes:paa15m_rate1m ==
task:node_network_receive_bytes:paa15m_rate1m{instane="myhost",device="eth1"}
```

Since this is using pre-recorded values, the match does not need do do any calculation and
should therefore be very fast indeeed.

## Performance

Prometheus has excellent support internally for testing and benchmarking of its
functions. Adding a couple of benchmarks shows paa() is well within the bounds of what is reasonable
for a prometheus function (though it is possible that the data generated by the benchmark suite
is being very kind here).

```
$ go test -v -bench=. -run=XXX 2> /dev/null                                            paa ]
BenchmarkHoltWinters4Week5Min-4             3000            369268 ns/op          986205 B/op        105 allocs/op
BenchmarkHoltWinters1Week5Min-4            10000            102932 ns/op          207921 B/op         89 allocs/op
BenchmarkHoltWinters1Day1Min-4             20000             76443 ns/op          146480 B/op         88 allocs/op
BenchmarkChanges1Day1Min-4                 30000             54977 ns/op          109584 B/op         83 allocs/op
BenchmarkPAA1Day1Min-4                     20000             61826 ns/op          109768 B/op         88 allocs/op
BenchmarkPAA15Min5Sec-4                   100000             17012 ns/op           14794 B/op         84 allocs/op
PASS
ok      github.com/prometheus/prometheus/promql 8.941s

```

## Problems

The code does not grecefully handle NaN or Inf values in time series. Lookalike
catered for this by adding an additional letter to the PAA representation
that was used if a time series had no values in a given period. The
downsampling function would need adjusting to take this into account, and the
integer representation would need an extra bit per sample to allow such a
representation.

In any sufficiently large environment, simple correlation will be fairly
common.

The PAA itself is relatively hard work to calculate. In a large environment,
calculating a PAA in a reocording rule for a large number of time series will
probably require considerable extra CPU, compared to typical rate or aggregate.
However, since the usage and nature of the result is quite separate from data
you might be graphing, it would not be unreasonable to have a separate instance
purely for the purpose of indexing and searching.

Prometheus uses a double-delta encoding scheme for it's internal and on-disk
data storage. PAA value will not change "smoothely", and are likely to stress
the storage more than a regular time series.

## Conclusion

PAA is a simple and effective technique for time series indexing and searching.
I'm satisified that my naive implementation performs well enough that it could
be used for streaming indexing at scale.

