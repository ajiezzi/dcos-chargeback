## DC/OS Chargeback

#### Background

We need to be able to track cluster utilization for tenants and be able to charge them for their usage of the cluster.

#### Implementation

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

The DC/OS chargeback calculation will start by summarizing all of the AWS EC2 agents in the DC/OS cluster. This will allow us to summarize the total vCPU and memory available for the month. To track task duration, we will use minutes as the time increment. Each task will be rounded up to the nearest minute. 

The formula parameters will be broken down into two groups, cluster and task.

1. Cluster parameters:

<p>**Total EC2 cost of the DC/OS agents:** $1,000</p>
<p>**Total vCPU's of the DC/OS agents:** m4.large (2 vCPu, 8 GB memory) x 10 instances = 20 vCPU's</p>
**Total memory of the DC/OS agents:** m4.large (2 vCPu, 8 GB memory) x 10 instances = 80 GB memory
**vCPU/memory weight:** Weighted fraction that you can use to disproportionately divide the instance cost between vCPU and memory. For example, m4.large would be .25

2. Task parameters:

**Marathon vCPU allocation**
**Marathon memory allocation**
**Total task duration in minutes**

Formulas:

**Task vCPU cost** = (task vCPU allocation/total vCPUs) * (vCPU/memory weight) * (total EC2 cost) * (task run time/total mins)

**Task memory cost** = (task memory allocation/total memory) * (1 - vCPU/memory weight) * (total EC2 cost) * (task run time/total mins)

##### Example calcuation

Total minutes: 43,800
EC2 Instance Type: m4.large (2 vCPU, 8 GB memory)
Number of agents: 10
Total vCPU's: 20
Total memory: 80
vCPU/memory weight: .25
Total EC2 cost: $1,000

DC/OS tasks:

| Task Name | vCPU | memory | task duration |
| --------- |:----:|:------:|:-------------:|
| task1     | 6    | 1 GB   | 43800 minutes |
| task2     | 10   | 10 GB  | 43800 minutes |
| task3     | 2    | 2 GB   | 21900 minutes |
| task4     | 2    | 2 GB   | 43800 minutes |
| task5     | 2    | 4 GB   | 21900 minutes |


Task1 vCPU cost = (6/20) * (.25) * (1000) * (43800/43800) = $75
Task2 vCPU cost = (10/20) * (.25) * (1000) * (43800/43800) = $125
Task3 vCPU cost = (2/20) * (.25) * (1000) * (21900/43800) = $12.5
Task4 vCPU cost = (2/20) * (.25) * (1000) * (43800/43800) = $25
Task5 vCPU cost = (2/20) * (.25) * (1000) * (21900/43800) = $12.5
