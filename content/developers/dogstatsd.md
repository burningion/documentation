---
title: DogStatsD
kind: documentation
description: This page explains what DogStatsD is, how it works, and what data it accepts.
aliases:
  - /guides/dogstatsd/
  - /guides/DogStatsD/
further_reading:
- link: "developers/metrics"
  tag: "Documentation"
  text: Learn more about Metrics
- link: "developers/libraries"
  tag: "Documentation"
  text: Official and Community-contributed API and DogStatsD client libraries
- link: "https://github.com/DataDog/dd-agent/blob/master/dogstatsd.py"
  tag: "Github"
  text: DogStatsD source code
---

The easiest way to get your custom application metrics into Datadog is to send them to DogStatsD, a metrics aggregation service bundled with the Datadog Agent. DogStatsD implements the [StatsD][5] protocol and adds a few Datadog-specific extensions:

* Histogram metric type
* Service checks and Events
* Tagging

**Note**: DogStatsD does NOT implement the following from StatsD:

* Gauge deltas (see [this issue][1])
* Timers as a native metric type (though it [does support them via histograms](#timers))

**Note**: Any StatsD client works just fine, but using the Datadog DogStatsD client gives you a few extra features.

## How It Works

DogStatsD accepts [custom metrics][6], events, and service checks over UDP and periodically aggregates and forwards them to Datadog.
Because it uses UDP, your application can send metrics to DogStatsD and resume its work without waiting for a response. If DogStatsD ever becomes unavailable, your application won't skip a beat.

{{< img src="developers/dogstatsd/dogstatsd.png" alt="dogstatsd"  responsive="true" popup="true">}}

As it receives data, DogStatsD aggregates multiple data points for each unique metric into a single data point over a period of time called the flush interval. Let's walk through an example to see how this works.

Suppose you want to know how many times your Python application is calling a particular database query. Your application can tell DogStatsD to increment a counter each time the query is called:

```python

def query_my_database():
    dog.increment('database.query.count')
    # Run the query ...
```

If this function executes one hundred times during a flush interval (ten
seconds, by default), it sends DogStatsD one hundred UDP packets that say
"increment the counter 'database.query.count'". DogStatsD aggregates these
points into a single metric value—100, in this case—and send it to Datadog
where it is stored and available for graphing alongside the rest of your metrics.

## Setup

First, edit your `datadog.yaml` file to uncomment the following lines:
```
use_dogstatsd: yes

...

dogstatsd_port: 8125
```

Then [restart your Agent][7].

Once done, your application can reliably reach the [DogStatsD client library][2] for your application language and you'll be ready to start hacking. You _can_ use any generic StatsD client to send metrics to DogStatsD, but you won't be able to use any of the Datadog-specific features mentioned above.

By default, DogStatsD listens on UDP port **8125**. If you need to change this, configure the `dogstatsd_port` option in the main [Agent configuration file][3]:

    # Make sure your client is sending to the same port.
    dogstatsd_port: 8125

[Restart DogStatsD][7] to effect the change.

## Data Types

While StatsD only accepts metrics, DogStatsD accepts all three major data types Datadog supports: metrics, events, and service checks. This section shows typical use cases for each type.

Each example is in Python using [datadogpy][8], but each data type shown is supported similarly in [other DogStatsD client libraries][2].

### Metrics

The first four metrics types —gauges, counters, timers, and sets— are familiar to StatsD users. The last one—histograms—is specific to DogStatsD.

#### Gauges

Gauges track the ebb and flow of a particular metric value over time, like the number of active users on a website:

```python

from datadog import statsd

statsd.gauge('mywebsite.users.active', get_active_users())
```

#### Counters

Counters track how many times something happens _per second_, like page views:

```python

from datadog import statsd

def render_page():
  statsd.increment('mywebsite.page_views') # add 1
  # Render the page...
```

With this one line of code we can start graphing the data:

{{< img src="developers/dogstatsd/graph-guides-metrics-page-views.png" alt="graph guides metrics page views" responsive="true" popup="true">}}

DogStatsD normalizes counters over the flush interval to report
per-second units. In the graph above, the marker is reporting
35.33 web page views per second at ~15:24. In contrast, if one person visited
the webpage each second, the graph would be a flat line at y = 1.

To increment or measure values over time rather than per second, use a gauge.

#### Sets

Sets count the number of unique elements in a group. To track the number of unique visitors to your site, use a set:

```python

def login(self, user_id):
    statsd.set('users.uniques', user_id)
    # Now log the user in ...
```

#### Timers

Timers measure the amount of time a section of code takes to execute, like the time it takes to render a web page. In Python, you can create timers with a decorator:

```python

from datadog import statsd

@statsd.timed('mywebsite.page_render.time')
def render_page():
  # Render the page...
```

or with a context manager:

```python

from datadog import statsd

def render_page():
  # First some stuff we don't want to time
  boilerplate_setup()

  # Now start the timer
  with statsd.timed('mywebsite.page_render.time'):
    # Render the page...
```

In either case, as DogStatsD receives the timer data, it calculates the statistical distribution of render times and sends the following metrics to Datadog:

- `mywebsite.page_render.time.count` - the number of times the render time was sampled
- `mywebsite.page_render.time.avg` - the average render time
- `mywebsite.page_render.time.median` - the median render time
- `mywebsite.page_render.time.max` - the maximum render time
- `mywebsite.page_render.time.95percentile` - the 95th percentile render time

Under the hood, DogStatsD actually treats timers as histograms; Whether you send timer data using the methods above, or send it as a histogram (see below), you'll be sending the same data to Datadog.

#### Histograms

Histograms calculate the statistical distribution of any kind of value. Though it would be less convenient, you could measure the render times in the previous example using a histogram metric:

```python

from datadog import statsd

...
start_time = time.time()
page = render_page()
duration = time.time() - start_time
statsd.histogram('mywebsite.page_render.time', duration)

def render_page():
  # Render the page...
```

This produces the same five metrics shown in the Timers section above: count, avg, median, max, and 95percentile.

But histograms aren't just for measuring times. You can track distributions for anything, like the size of files users upload to your site:

```python

from datadog import statsd

def handle_file(file, file_size):
  # Handle the file...

  statsd.histogram('mywebsite.user_uploads.file_size', file_size)
  return
```

Since histograms are an extension to StatsD, use a [DogStatsD client library][2].

#### Metric option: Sample Rates

Since the overhead of sending UDP packets can be too great for some performance
intensive code paths, DogStatsD clients support sampling,
i.e. only sending metrics a percentage of the time. The following code sends
a histogram metric only about half of the time:

```python

dog.histogram('my.histogram', 1, sample_rate=0.5)
```

Before sending the metric to Datadog, DogStatsD uses the `sample_rate` to
correct the metric value, i.e. to estimate what it would have been without sampling.

**Sample rates only work with counter, histogram, and timer metrics.**

### Events

DogStatsD can emit events to your Datadog event stream. For example, you may want to see errors and exceptions in Datadog:

```python

from datadog import statsd

def render_page():
  try:
    # Render the page...
    # ..
  except RenderError as err:
    statsd.event('Page render error!', err.message, alert_type='error')
```

### Service Checks

Finally, DogStatsD can send service checks to Datadog. Use checks to track the status of services your application depends on:

```python

from datadog import statsd

conn = get_redis_conn()
if not conn:
  statsd.service_check('mywebsite.can_connect_redis', statsd.CRITICAL)
else:
  statsd.service_check('mywebsite.can_connect_redis', statsd.OK)
  # Do your redis thing...
```

## Tagging

You can add tags to any metric, event, or service check you send to DogStatsD. For example, you could compare the performance of two algorithms by tagging a timer metric with the algorithm version:

```python

@statsd.timed('algorithm.run_time', tags=['algorithm:one'])
def algorithm_one():
    # Do fancy things here ...

@statsd.timed('algorithm.run_time', tags=['algorithm:two'])
def algorithm_two():
    # Do fancy things (maybe faster?) here ...
```

Since tagging is an extension to StatsD, use a [DogStatsD client library][2].

## Datagram Format

This section specifies the raw datagram format for each data type DogStatsD accepts. You don't need to know this if
you're using any of the DogStatsD client libraries, but if you want to send data to DogStatsD without the libraries
or you're writing your own library, here's how to format the data.

### Metrics

`metric.name:value|type|@sample_rate|#tag1:value,tag2`

- `metric.name` — a string with no colons, bars, or @ characters. See the [metric naming policy][4].
- `value` — an integer or float.
- `type` — `c` for counter, `g` for gauge, `ms` for timer, `h` for histogram, `s` for set.
- `sample rate` (optional) — a float between 0 and 1, inclusive. Only works with counter, histogram, and timer metrics. Default is 1 (i.e. sample 100% of the time).
- `tags` (optional) — a comma separated list of tags. Use colons for key/value tags, i.e. `env:prod`. The key `device` is reserved; Datadog drops a user-added tag like `device:foobar`.

Here are some example datagrams:

    # Increment the page.views counter
    page.views:1|c

    # Record the fuel tank is half-empty
    fuel.level:0.5|g

    # Sample the song length histogram half of the time
    song.length:240|h|@0.5

    # Track a unique visitor to the site
    users.uniques:1234|s

    # Increment the active users counter, tag by country of origin
    users.online:1|c|#country:china

    # Track active China users and use a sample rate
    users.online:1|c|@0.5|#country:china

### Events

`_e{title.length,text.length}:title|text|d:timestamp|h:hostname|p:priority|t:alert_type|#tag1,tag2`

- `_e` - The datagram must begin with `_e`
- `title` — Event title.
- `text` — Event text. Insert line breaks with an escaped slash (`\\n`)
- `|d:timestamp` (optional) — Add a timestamp to the event. Default is the current Unix epoch timestamp.
- `|h:hostname` (optional) - Add a hostname to the event. No default.
- `|k:aggregation_key` (optional) — Add an aggregation key to group the event with others that have the same key. No default.
- `|p:priority` (optional) — Set to 'normal' or 'low'. Default 'normal'.
- `|s:source_type_name` (optional) - Add a source type to the event. No default.
- `|t:alert_type` (optional) — Set to 'error', 'warning', 'info' or 'success'. Default 'info'.
- `|#tag1:value1,tag2,tag3:value3...` (optional)— ***The colon in tags is part of the tag list string and has no parsing purpose like for the other parameters.*** No default.

Here are some example datagrams:

    # Send an exception
    _e{21,36}:An exception occurred|Cannot parse CSV file from 10.0.0.17|t:warning|#err_type:bad_file

    # Send an event with a newline in the text
    _e{21,42}:An exception occurred|Cannot parse JSON request:\\n{"foo: "bar"}|p:low|#err_type:bad_request

### Service Checks

`_sc|name|status|d:timestamp|h:hostname|#tag1:value1,tag2,tag3:value3,...|m:service_check_message`

- `_sc` — the datagram must begin with `_sc`
- `name` — Service check name.
- `status` — Integer corresponding to the check status (OK = 0, WARNING = 1, CRITICAL = 2, UNKNOWN = 3).
- `d:timestamp` (optional) — Add a timestamp to the check. Default is the current Unix epoch timestamp.
- `h:hostname` (optional) — Add a hostname to the event. No default.
- `#tag1:value1,tag2,tag3:value3,...` (optional) — ***The colon in tags is part of the tag list string and has no parsing purpose like for the other parameters.***No default.
- `m:service_check_message` (optional) — Add a message describing the current state of the service check. *This field MUST be positioned last among the metadata fields.* No default.

Here's an example datagram:

    # Send a CRITICAL status for a remote connection
    _sc|Redis connection|2|#redis_instance:10.0.0.16:6379|m:Redis connection timed out after 10s

## Send metrics and events using DogStatsD and the shell

For Linux and other Unix-like OS, we use Bash.
For Windows we need Powershell and [powershell-statsd][9], a simple Powershell function that takes care of the network bits for us.

The idea behind DogStatsD is simple: create a message that contains information about your metric/event, and send it to a collector over UDP on port 8125. [Read more about the message format](#datagram-format).

### Sending metrics

The format for sending metrics is `metric.name:value|type|@sample_rate|#tag1:value,tag2,` so let's go ahead and send datapoints for a gauge metric called custom_metric with the shell tag. We use a locally installed Agent as a collector, so the destination IP address is 127.0.0.1.

On Linux:

```
vagrant@vagrant-ubuntu-14-04:~$ echo -n "custom_metric:60|g|#shell" >/dev/udp/localhost/8125
```

or

```
vagrant@vagrant-ubuntu-14-04:~$ echo -n "custom_metric:60|g|#shell" | nc -4u -w0 127.0.0.1 8125
```

On Windows:
```
PS C:\vagrant> .\send-statsd.ps1 "custom_metric:123|g|#shell"
PS C:\vagrant>
```

On any platform with Python (on Windows, the Agent's embedded Python interpreter can be used, which is located at `C:\Program Files\Datadog\Datadog Agent\embedded\python.exe`):

```python
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM) # UDP
sock.sendto("custom_metric:60|g|#shell", ("localhost", 8125))
```

### Sending events

The format for sending events is:
```
_e{title.length,text.length}:title|text|d:date_happened|h:hostname|p:priority|t:alert_type|#tag1,tag2.
```
Here we need to calculate the size of the event's title and body.

On Linux:
```
vagrant@vagrant-ubuntu-14-04:~$ title="Event from the shell"
vagrant@vagrant-ubuntu-14-04:~$ text="This was sent from Bash!"
vagrant@vagrant-ubuntu-14-04:~$ echo "_e{${#title},${#text}}:$title|$text|#shell,bash"  >/dev/udp/localhost/8125
```
On Windows:

```
PS C:\vagrant> $title = "Event from the shell"
PS C:\vagrant> $text = "This was sent from Powershell!"
PS C:\vagrant> .\send-statsd.ps1 "_e{$($title.length),$($text.Length)}:$title|$text|#shell,powershell"
```

## Further Reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: https://github.com/DataDog/dd-agent/pull/2104
[2]: /libraries/
[3]: https://github.com/DataDog/dd-agent/blob/master/datadog.conf.example
[4]: /developers/metrics/#metric-names 
[5]: https://github.com/etsy/statsd
[6]: /getting_started/custom_metrics/
[7]: /agent/faq/agent-commands
[8]: http://datadogpy.readthedocs.io/en/latest/
[9]: https://github.com/joehack3r/powershell-statsd/blob/master/send-statsd.ps1
