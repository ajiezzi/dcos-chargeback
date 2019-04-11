## DC/OS Chargeback

#### Background

We need to be able to track cluster utilization for tenants and be able to charge them for their usage of the cluster.

#### Discussion

Looking at using the DC/OS and/or Mesos APIs on a regular basis to monitor current utilization (things like CPU allocation and/or utilization).  Need to figure out where to store it, and how to generate reports.  Should be doable but need more research on best way to handle this, cause there are a lot of different ways this could be achieved.

Looks like there are three main components to this:

* Figuring out how to actually gather the data.

    * Two high-level data sources (could use both)
        * Mesos API - potentially harder to consume, may be able to identify actual utilization vs. just allocation.  Need to look at the API to see what data it generates.
        * Marathon API - only indicates allocation, and only for Marathon tasks, but perhaps this is the most relevant in the short-term.
    * Three high-level patterns:
        * Do we wanna poll the API on some interval?  Need to determine the interval for this.  How do we handle ephemerality?
        * Do we want to subscribe to one of the event APIs?  We'll have info on stop/start, but what happens if we miss something / how do we handle long-running jobs?
        * Some combination of the two - detect start/stops, but also poll on a regular basis. 

* Figuring out where to store the data (and how to store it):

    * Store to Amazon RDS.  Multiple options (I think any of these would be valid):
        * MySQL
        * MariaDB
        * PostgreSQL
    * Table structure - not critical, but thinking these columns (potentially broke across a couple tables):
        * taskid
        * some tenant label (from mesos, marathon, or elsewhere)
        * start time
        * end time
        * cpu allocation
        * mem allocation
        * gpu allocation
        * total cpu cycles? potentially cpu start counter, cpu end counter, something related here.  TBD.
        * total memory usage (probably not feasible)

* Figuring out how to consume and/or parse the data and generate reports in a user-consumable format. This can/will come later.

	* Simple python script should be able to accomplish the consumption and parsing. Can probably leverage alot of the Marathon autoscaler patterns used for:
        * authentication
        * query
     
