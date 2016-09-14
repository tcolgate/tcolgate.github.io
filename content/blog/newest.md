---
title: "Time Series Search in Prometheus"
date: "2016-09-14"
draft: true
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
* Normalize this downsampled data to sit between a range of -1 and 1
* Quantize the resulting data into a set of ranges (typically 8)
* Assign a letter to each quantized value.

The resulting string encodes the shape of the time series data. Any two time
series with the same final string encoding will have a similar basic shape.
The SAX and [ISAX2](http://www.cs.ucr.edu/~eamonn/iSAX_2.0.pdf) papers give
some example values for the quantization bands they found effective.

After some experementation I eventually implented, and successfully used this
technique using a tool called
[lookalike](https://github.com/tcolgate/lookalike), developed for Cacti. I
toyed with ideas for implementing a similar solution for OpenTSDB, and
InfluxDB, but neither's internals made it particularly easy (the latter used a
map/reduce model internally that proved tricky, the former is written in java)

## Prometheus Implementation

Prometheus is essentially an in-memory time series database with some support
for spilling out to disk. This has the advantage that all the data is easily
available when an internal function executes.

Whilst prometheus does not support string data for values, float64 does give us
plenty of room to pack in a PAA encoding. 8 3-bit samples were generally
sufficient for a reasonable match in lookalike. We can store 52 bit integers
exactly in a floa64, which is far morei space than the 24-bits we require.

## Conclusion

