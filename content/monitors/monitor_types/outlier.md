---
title: Outlier monitor
kind: documentation
aliases:
    - /guides/outliers
description: "Alert on members of a group behaving differently than the others"
further_reading:
- link: "monitors/notifications"
  tag: "Documentation"
  text: Configure your monitor notifications
- link: "monitors/downtimes"
  tag: "Documentation"
  text: Schedule a dowtime to mute a monitor
- link: "monitors/monitor_status"
  tag: "Documentation"
  text: Consult your monitor status
---

Outlier Detection is an algorithmic feature that allows you to detect when some members of a group are behaving strangely compared to the others. For example, you could detect that one web server in a pool is processing an unusual number of requests, and hence should be a target for replacement. Or, you could get an early warning that significantly more 500s are happening in one AWS Availability Zone (AZ) than the others, which might indicate an issue brewing in that AZ.

{{< img src="monitors/monitor_types/outliers/outliers-metric-alert.png" alt="outliers metric alert" responsive="true" popup="true" style="width:80%;">}}

## How to Use Outlier Detection on Your Data

The `outliers` query function, when applied to your query, returns the usual results but with outliers series marked.

You can use this function to display and alert on outliers in your data. To try it out, you’ll first need a metric for which a group of hosts (or availability zones, partitions, etc) should exhibit uniform behavior. For the function to work, be sure that there are at least 3 or more members in the group. Given that, here are two ways to use outlier detection on that group.

### Show Outliers in Dashboards or Screenboards

Here's a graph of Gunicorn requests by host with outlier detection enabled.

{{< img src="monitors/monitor_types/outliers/outliers-graph-dbscan-gunicorn.png" alt="outliers graph dbscan Gunicorn" responsive="true" popup="true" style="width:80%;">}}

You can see that one of the series is an outlier: it is handling significantly lower traffic than the others for the time window in question.

To set up an outlier detection graph for your data add a metric to the graph showing all series in the groups. Then apply the outlier detection algorithm by adding the `outliers` function on your data. After applying the function, any outlier series is colored with a bold, warm palette, while all other series are colored with a lightweight, greyscale color palette.

First create a new timeseries graph on your dashboard with your chosen metric.

{{< img src="monitors/monitor_types/outliers/outliers-dash-choose-metrics-updated.png" alt="outliers dash choose metrics updated" responsive="true" popup="true" style="width:80%;">}}

To enable outlier detection, click on the `+` icon on the right side of the metrics line. Choose **Algorithms** from the function categories, then pick one of the four outlier algorithms.

{{< img src="monitors/monitor_types/outliers/outliers-algorithm-selector.png" alt="outliers algorithm selector" responsive="true" popup="true" style="width:80%;">}}

This applies the outliers function to your graph, and you’ll see any outliers in the group highlighted in bold, warm colors.

{{< img src="monitors/monitor_types/outliers/outliers-algorithm-annotated-newer.png" alt="outliers algorithm annotated newer" responsive="true" popup="true" style="width:80%;">}}

There are several outlier detection algorithms you can choose. The default algorithm (DBSCAN) and parameter values should work for most scenarios. However, if you see too many or too few outliers identified, you can tune the algorithm or try an alternate algorithm. To learn more, see the "Outlier Algorithms and Parameters" section below.

### Alert on Outliers

You can also define a monitor to alert when an outlier is detected in an important group.

{{< img src="monitors/monitor_types/outliers/outliers-alert-snapshot.png" alt="outliers alert snapshot" responsive="true" popup="true" style="width:80%;">}}

For example, to alert when a Cassandra host is abnormally loaded compared to the rest of the group, you can [add a new outlier monitor][1] for the metric.

Navigate to the [New Monitor][2] page and click **Outlier**. Then fill out the **Define the metric** section just as you would for any other monitor.

{{< img src="monitors/monitor_types/outliers/outliers-new-monitor-define-metric.png" alt="outliers new monitor define metric" responsive="true" popup="true" style="width:80%;">}}

In the [alert conditions][3], select the grouping and timeframe. Then select an algorithm and parameter values to use for outlier detection.

{{< img src="monitors/monitor_types/outliers/outliers-newer-monitor-set-conditions.png" alt="outliers newer monitor set condition" responsive="true" popup="true" style="width:80%;">}}

To ensure that your alert is properly calibrated, you can set the time window at the top of the screen and use the reverse (<<) button to look back in time for when outliers would have be found and alerted. This is also a good way to tune the parameters to the specific outliers algorithm you’re using.

{{< img src="monitors/monitor_types/outliers/outliers-new-monitor-graph-calibrate.png" alt="outliers new monitor graph calibrate" responsive="true" popup="true" style="width:80%;">}}

## Reference: Outlier Algorithms and Parameters

There are two different types of outlier detection algorithms you can use on your data: DBSCAN/ScaledDBSCAN and MAD/ScaledMAD. We recommend starting with the default algorithm, DBSCAN. If you have trouble detecting the right outliers, you can adjust the parameters to DBSCAN or try the alternate algorithm, MAD. If you have metrics on a larger scale that look to be closely clustered but the DBSCAN/MAD algorithms are identifying some as outliers, try the scaled algorithms. Explanation of each algorithm and its parameters follows.

### DBSCAN/ScaledDBSCAN

A natural way to group together hosts that are behaving similarly is to use a clustering algorithm. We use [DBSCAN][4], a popular density-based clustering algorithm, for this purpose. DBSCAN works by greedily agglomerating points that are close to each other. Clusters with few points in them are considered outliers.

Traditionally, DBSCAN takes: 1) a parameter 𝜀 that specifies a distance threshold under which two points are considered to be close; and 2) the minimum number of points that have to be within a point’s 𝜀-radius before that point can start agglomerating. The image below shows an example of DBSCAN in action on points in the plane. There are two clusters. The large points had enough close neighbors to agglomerate those points, while the small colored points did no agglomerating themselves but are within the 𝜀-radius of a large point. The points in black are the outliers.

{{< img src="monitors/monitor_types/outliers/outliers-dbscan-2d.png" alt="outlier dbscan 2d" responsive="true" popup="true" style="width:60%;">}}

#### Parameters

We use a simplified form of DBSCAN to detect outliers on time series. We consider each host to be a point in d-dimensions, where d is the number of elements in the time series. Any point can agglomerate, and any point that is not in the largest cluster is considered an outlier.

We set the initial distance threshold as follows. We create a new median time series by taking the median of the values from the existing time series at every time point. Then we calculate the Euclidean distance between each host and the median series. The threshold is the median of those distances, multiplied by a normalizing constant.

The only parameter we take is `tolerance`, the constant by which the initial threshold is multiplied to yield DBSCAN’s distance parameter 𝜀. Here is DBSCAN with a tolerance of 3.0 in action on a pool of Cassandra workers:

{{< img src="monitors/monitor_types/outliers/outliers-dbscan-cassandra.png" alt="outliers dbscan cassandra" responsive="true" popup="true" style="width:80%;">}}

You should set the tolerance parameter depending on how similarly you expect your group of hosts to behave—larger values allow for more tolerance in how much a host can deviate from its peers.

#### ScaledDBSCAN

The distance threshold of the DBSCAN algorithm is independent of the overall scale of the metrics. Consider a group of constant time series with values {1000, 1001, 1002, 1005, 1015}. The median series is a constant series at 1002. DBSCAN with a tolerance of 3.0 identifies the series at 1015 to be an outlier, even though it may be almost indistinguishable from the other series visually on the graph when the origin of the y-axis is at 0.

The ScaledDBSCAN algorithm scales the distance threshold according to the relative magnitudes of the median series and the hosts’ distances to the median series. In most situations, it behaves the same as regular DBSCAN does. However, when the median series is large compared to the distances to the median series, the distance threshold becomes proportional to the size of the median series. As a result, assessing whether two time series are close depends on the scale of the median series.

Here is a comparison of DBSCAN and ScaledDBSCAN with tolerances of 3 on field data size in a group of Elasticsearch nodes:

{{< img src="monitors/monitor_types/outliers/outliers-scaled-dbscan-es.png" alt="outliers scaled dbscan es" responsive="true" popup="true" style="width:80%;">}}

### MAD/ScaledMAD

The  [Median Absolute Deviation][5] is a robust measure of variability, and can be viewed as the robust analog for standard deviation. Robust statistics describe data in such a way that they are not unduly influenced by outliers.

For a given set of data D = {d<sub>1</sub>, ..., d<sub>n</sub>}, the deviations are the difference between each d<sub>i</sub> and median(D). The MAD is then the median of the absolute values of all the deviations. For example if D = {1, 2, 3, 4, 5, 6, 100}, then the median is 4, the deviations are {-3, -2, -1, 0, 1, 2, 96}, and the MAD is 2. (Note that the standard deviation by contrast is 33.8.)

#### Parameters

In our case, the data set is the set of all points in every time series. We take the MAD of all the points then multiply it by a normalizing constant and our first parameter, `tolerance`. The constant normalizes MAD so that it is comparable to the standard deviation of the normal distribution. The tolerance parameter then specifies how many “deviations” a point has to be away from the median for it to be considered an outlier.

Now to mark a time series as an outlier, we use the second parameter, `pct`. If more than pct% of a particular series’ points are considered outliers, then the whole series is marked to be an outlier. Here is MAD with a tolerance of 3 and pct of 20 in action when comparing the average system load by availability zone:

{{< img src="monitors/monitor_types/outliers/outliers-mad-az.png" alt="outliers mad az" responsive="true" popup="true" style="width:80%;">}}

The tolerance parameter should be tuned depending on the expected variability of the data. For example, if the data is generally within a small range of values, then this should be small. On the other hand, if points can vary greatly, then you want a higher scale so these variabilities do not trigger a false positive.

#### ScaledMAD

Like for DBSCAN, the MAD algorithm is designed to be independent of the overall magnitude of the metrics. If D = {1000, 1001, 1002, 1005, 1010}, the median is 1002, and the MAD is 2. Even though the point at 1010 seems close to the median in terms of their relative scales, it is still an outlier point for a tolerance of 3.

The ScaledMAD algorithm, like ScaledDBSCAN, considers the relative scales of the MAD and the median. In most cases, it behaves the same as the MAD algorithm does. However, when the MAD of the data set becomes small compared to the median, the measure of deviation becomes proportional to the median. Therefore, determining whether a point is an outlier depends on the overall scale of the metrics.

Here is an example of MAD and ScaledMAD algorithms for comparing the usable memory in Cassandra hosts. Both have tolerances of 3 and pct of 20:

{{< img src="monitors/monitor_types/outliers/outliers-scaled-mad-cassandra.png" alt="outliers scaled mad Cassandra" responsive="true" popup="true" style="width:80%;">}}

### DBSCAN vs. MAD

So which algorithm should you use? For most outliers, any algorithm performs well at the default settings. However, there are subtle cases where one algorithm is more appropriate than the other.

In the following image, we see a group of hosts flushing their buffers together while one host is flushing its buffer slightly later. DBSCAN picks this up as an outlier whereas MAD does not. This is a case where we would prefer to use MAD, as we don’t care about when the buffers get flushed.  

The synchronization of the group is just an artifact of the hosts being restarted at the same time. On the other hand, if instead of flushed buffers, the metrics below represented a scheduled job that actually should be synchronized across hosts, DBSCAN would be the right choice.

{{< img src="monitors/monitor_types/outliers/outliers-flushing.png" alt="outliers flushing" responsive="true" popup="true" style="width:80%;">}}

#### Scaled vs. Regular Algorithms

In most situations, the scaled algorithms behaves the same as their regular counterparts. However, if DBSCAN/MAD algorithms are identifying outliers within a closely clustered group of metrics, and you would like the outlier detection algorithm to scale with the overall magnitude of the metrics, try the scaled algorithms.

### Setting up alerts

When setting up an outlier alert, an important parameter is the size of the time window. If the window size is too large, by the time an outlier is detected, the bad behavior might have been going on for longer than one would like. If the window size is too short, the alerts are not as resilient to unimportant, one-off spikes.

Both algorithms are set up to identify outliers that differ from the majority of metrics that are behaving similarly. If your hosts exhibit “banding” behavior as shown below (perhaps because each band represents a different shard), we recommend tagging each band with an identifier, and setting up outlier detection alerts on each band separately.

{{< img src="monitors/monitor_types/outliers/outliers-banding.png" alt="outliers banding" responsive="true" popup="true" style="width:80%;">}}

## Further Reading 
{{< partial name="whats-next/whats-next.html" >}}

[1]: https://app.datadoghq.com/monitors#create/outlier
[2]: https://app.datadoghq.com/monitors#/create
[3]: /monitors/monitor_types/#define-the-conditions
[4]: https://en.wikipedia.org/wiki/DBSCAN
[5]: https://en.wikipedia.org/wiki/Median_absolute_deviation
