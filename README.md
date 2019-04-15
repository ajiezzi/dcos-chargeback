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
	
#### Chargeback measurement



The DC/OS vCPU and memory costs are calculated using the following formulas.

The formula will be using the following parameters:

General parameters:

	* Time increment: minutes
	* Total minutes in a month: 43,800

Cluster parameters:

	* Total EC2 cost of the DC/OS agents
	* Total vCPU's of the DC/OS agents: m4.large (2 vCPu, 8 GB memory) x 10 instances = 20 vCPU's
	* Total memory of the DC/OS agents: m4.large (2 vCPu, 8 GB memory) x 10 instances = 80 GB memory
	* vCPU/memory weight: Weighted fraction that you can use to disproportionately divide the instance cost between vCPU and memory. For example, m4.large would be .25

Task parameters:

	* Marathon vCPU allocation
	* Marathon memory allocation
	* Total task duration in minutes

Formulas:

Task vCPU cost = (task vCPU allocation/total vCPUs) * (vCPU/memory weight) * (total EC2 cost) * (task run time/total mins)

Task memory cost = (task memory allocation/total memory) * (1 - vCPU/memory weight) * (total EC2 cost) * (task run time/total mins)

#### Example

Example parameters:

	Total minutes: 43,800
	EC2 Instance Type: m4.large (2 vCPU, 8 GB memory)
	Number of agents: 10
	Total vCPU's: 20
	Total memory: 80
	vCPU/memory weight: .25
	Total EC2 cost of agents: $1,000

Example Tasks:

	Task1 - 6 vCPU, 1 GB memory, 43,800 minutes
	Task2 - 10 vCpu, 10 GB memory, 43,800 minutes
	Task3 - 2 vCPU, 2 GB memory, 21,900 minutes
	Task4 - 2 vCPU, 2 GB memory, 43,800 minutes
	Task5 - 2 vCPU, 4 GB memory, 21,900 minutes

	Task1 vCPU cost = (6/20) * (.25) * (1000) * (43800/43800) = $75
	Task2 vCPU cost = (10/20) * (.25) * (1000) * (43800/43800) = $125
	Task3 vCPU cost = (2/20) * (.25) * (1000) * (21900/43800) = $12.5
	Task4 vCPU cost = (2/20) * (.25) * (1000) * (43800/43800) = $25
	Task5 vCPU cost = (2/20) * (.25) * (1000) * (21900/43800) = $12.5
     
