# What Is Time-series Data?

What is this "time-series data" that we keep talking about, and how and why is
it different from other data?

Some think of "time-series data" as a sequence of data points, measuring the same
thing over time, stored in time order. That's true, but it just scratches the surface.

Others may think of a series of numeric values, each paired with a timestamp,
defined by a name and a set of labeled dimensions (or "tags"). This appears
often in monitoring applications like collecting server metrics, which often
view time-series data as having a very specific form:

```bash
Name:    CPU

Tags:    Host=MyServer, Region=West

Data:
2020-01-01 01:02:00    70
2020-01-01 01:03:00    71
2020-01-01 01:04:00    72
2020-01-01 01:05:01    68
```

This is certainly one way to model time-series data (and one that TimescaleDB
supports), but not a definition of the data itself.

For example, even in many monitoring applications, different metrics
are often collected together (e.g., CPU load, used memory, network
statistics, disk usage and iops). So, it does not always make sense to think
of each metric separately.  Consider this alternative "wider" data model that
maintains the correlation between metrics collected at the same time.

```bash
Metrics: cpu_load, used_mem, net_recv, net_xmit, disk_usage, disk_iops_read, disk_iops_write

Tags:    Host=MyServer, Region=West

Data:
2020-01-01 01:02:00    6.6   40.0    18405    6176   15150    38    82
2020-01-01 01:03:00    9.7   41.4    26116    8160   15171    42    95
2020-01-01 01:04:00   10.2   46.7    51976   15238   15194    68   161
2020-01-01 01:05:01   27.8   55.2   120598   36163   15229   104   390
```

Further, in a number of applications, different sources of data can be
generating different sets of metrics.  For example, the above server metrics
might be collected from services, but different applications may be
generating different metrics. Moving from IT monitoring to a more
industrial IoT application, monitoring an manufacturing line might
involve four different types of sensors.  And so while your application
might be collecting data from 100s of sensors per type, each type of sensor
provides different metrics.

And not all time-series data involves regular metrics, but can also
events.  For example, your application might be collecting product usage
events, whether gaming ("bought a digital good"), e-commerce ("added
something to shopping cart"), or SaaS application ("created a new object").

```bash
2020-01-01 01:02:00   user_11   200   "view object"    1616
2020-01-01 01:02:00   user_23   200   "purchase"        753
2020-01-01 01:02:07   user_86   200   "add to cart"   12391
2020-01-01 01:02:21   user_11   403   "view object"    6916
```

Other domains combine many types of time-series data, including
regular, irregular, and event data.  Take financial trading
applications, for example.
One set of time-series data is the irregular feed of "tick"
data, each tick measuring the smallest possible price fluctuation for any
particular asset (and thus includes a timestamp, asset name, and price).
A second set of data involves trades, which represents events when
a particular user buys or sells a particular asset (at the price specified
by the most recent tick data).
And candlestick data gives a broader market perspective, typically rolling
up prices and trades to some time interval (say 5 minutes) and showing
the open, close, high, and low prices during that interval.

In short, these types of data all belong in a **broad** category,
whether temperature readings from a sensor, the price of a
stock, the status of a machine, or even the number of logins to an app.

[//]: # (Comment: Can be call out the following sentence?
[//]: # Some special design thing?

**Time-series data is data that collectively represents
how a system, process, or behavior changes over time.**

## Characteristics of Time-series Data [](characteristics)

If you look closely at how time-series data is produced and ingested,
there are important characteristics that time-series databases like
TimescaleDB typically leverage:

- **Time-centric**: Time is a primary axis of the data, and data records
always have a timestamp.

- **Append-mostly**: New data that arrives is almost always recorded as a
new entry. That is, the workload is heavily append-mostly (INSERTs),
and much more rarely involves in-place modifications (UPDATEs or
DELETEs), with the exception of bulk deletes to old intervals of data.

- **Recent-mostly**: New data is typically about recent time intervals
(i.e., it's often inserted in "loose time order"), and we only more
rarely make updates or backfill missing data about old intervals.

The frequency or regularity of data is less important though; it can be
collected every microsecond or hour.  It can also be collected at regular or
irregular intervals (e.g., when some *event* happens, as opposed to at
pre-defined times).

But haven't databases long had time fields?  A key difference between
time-series data (and the databases that support them), compared to more
traditional transactional workloads powering "business" applications,
is that **changes to the data are inserts, not overwrites**.

## Time-series Data Is Everywhere [](is-everywhere)

Time-series data is everywhere, but there are environments where it
is especially being created in torrents.

- **Monitoring computer systems**: VM, server, container metrics
(CPU, free memory, net/disk IOPs),
service and application metrics (request rates, request latency).

- **Financial trading systems**: Classic securities, newer cryptocurrencies,
payments, transaction events.

- **Internet of Things**: Data from sensors on industrial machines
and equipment,
wearable devices, vehicles, physical containers, pallets,
consumer devices for smart homes, etc.

- **Eventing applications**: User/customer interaction data like clickstreams,
pageviews, logins, signups, etc.

- **Business intelligence**: Tracking key metrics and the overall health
of the business.

- **Environmental monitoring**: Temperature, humidity, pressure, pH,
pollen count, air flow, carbon monoxide (CO), nitrogen dioxide (NO2),
particulate matter (PM10).

- (and more)

**Next:** [What is TimescaleDB?][architecture]

[architecture]: /introduction/architecture
