# DC/OS Chargeback

## Background

Provide a mechanism to track cluster utilization for tenants and be able to charge them back for their usage of the cluster.

## Chargeback calculation

The DC/OS chargeback calculation will start by summarizing all of the AWS EC2 agents in the DC/OS cluster. This will allow us to summarize the total vCPU and memory available for the month. To track task duration, we will use minutes as the time increment. Each task will be rounded up to the nearest minute. A weighted fraction will also be used to disproportionately divide the instance cost between vCPU and memory. For example, an m4.large (2 vCPU, 8GB memory) would be .25.

The formula parameters will be broken down into two groups, cluster and task.

1. Cluster parameters:

   * **Total EC2 cost of the DC/OS agents:** $1,000
   * **Total vCPU's of the DC/OS agents:** m4.large (2 vCPu, 8 GB memory) x 10 instances = 20 vCPU's
   * **Total memory of the DC/OS agents:** m4.large (2 vCPu, 8 GB memory) x 10 instances = 80 GB memory
   * **vCPU/memory weight:** m4.large (2 vCPu, 8 GB memory) = .25

2. Task parameters:

   * **Marathon vCPU allocation**
   * **Marathon memory allocation**
   * **Total task duration in minutes**

vCPU and Memory Formulas:

**Task vCPU cost** = (task vCPU allocation/total vCPUs) * (vCPU/memory weight) * (total EC2 cost) * (task run time/total mins)

**Task memory cost** = (task memory allocation/total memory) * (1 - vCPU/memory weight) * (total EC2 cost) * (task run time/total mins)

### Example calcuation

**Total minutes:** 43,800 <br>
**EC2 Instance Type:** m4.large (2 vCPU, 8 GB memory) <br>
**Number of agents:** 10 <br>
**Total vCPU's:** 20 <br>
**Total memory:** 80 <br>
**vCPU/memory weight:** .25 <br>
**Total EC2 cost:** $1,000 <br>

DC/OS task actuals:

| Task Name | vCPU | memory | task duration |
| --------- |:----:|:------:|:-------------:|
| task1     | 6    | 1 GB   | 43800 minutes |
| task2     | 10   | 10 GB  | 43800 minutes |
| task3     | 2    | 2 GB   | 21900 minutes |
| task4     | 2    | 2 GB   | 43800 minutes |
| task5     | 2    | 4 GB   | 21900 minutes |

Task vCPU calculations:
	
**task1 vCPU cost** = (6/20) * (.25) * (1000) * (43800/43800) = **$75** <br>
**task2 vCPU cost** = (10/20) * (.25) * (1000) * (43800/43800) = **$125** <br>
**task3 vCPU cost** = (2/20) * (.25) * (1000) * (21900/43800) = **$12.50** <br>
**task4 vCPU cost** = (2/20) * (.25) * (1000) * (43800/43800) = **$25** <br>
**task5 vCPU cost** = (2/20) * (.25) * (1000) * (21900/43800) = **$12.50** <br>
	
## Implementation details

Leverage the DC/OS and/or Mesos APIs on a regular basis to monitor current utilization (things like CPU allocation and/or utilization). 

Three main components to the implementation:

1. Gathering the data

    * Two high-level data sources (could use both)
        * Mesos API - potentially harder to consume, may be able to identify actual utilization vs. just allocation.  Need to look at the API to see what data it generates.
        * Marathon API - only indicates allocation, and only for Marathon tasks, but perhaps this is the most relevant in the short-term.
    * Three high-level patterns:
        * Do we wanna poll the API on some interval?  Need to determine the interval for this.  How do we handle ephemerality?
        * Do we want to subscribe to one of the event APIs?  We'll have info on stop/start, but what happens if we miss something / how do we handle long-running jobs?
        * Some combination of the two - detect start/stops, but also poll on a regular basis. 

2. Storing the data

    * Store to Amazon RDS.  Multiple options (I think any of these would be valid):
        * MySQL
        * MariaDB
        * PostgreSQL

3. Consume and present the data.

    * Simple python script should be able to accomplish the consumption and parsing. Can probably leverage alot of the Marathon autoscaler patterns used for:
        * authentication
        * query
        * parsing
