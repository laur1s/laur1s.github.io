---
title: "RDS Multi AZ vs Read Replica"
date: 2021-11-01T14:53:45+02:00
draft: no
---

Hi, in this post I'm going to go back to a fundamental AWS service - RDS. Specifically, I'll focus on two ways to ensure resilience of your RDS database: RDS Multi AZ and RDS Read replica.

Looking into AWS RDS console, it looks like a very simple service: you can choose which database engine you want to use, pick some additional parameters, and launch the database. However behind the single interface lies a complexity: each database engine type has slightly different concepts and functionality that you have to be aware of. For simplicity sake, this post is going to focus on RDS MySQL.
{{< figure src="/images/rds/1.jpg" title="RDS interface for creating a database" >}}

When looking to improve resilience of your database deployment there are two options for RDS:  Multi-AZ deployment and read replicas. This post is going to discuss the differences between these two RDS services and when to chose one or the other.

## RDS Multi-AZ deployment

When you create a Multi-AZ RDS deployment it launches a secondary standby instance in a different Availability Zone. That helps by to provide data redundancy, eliminate I/O freezes, and minimize latency spikes during system backups.
Therefore, Multi-AZ Deployment can help by increasing availability of your database, in case one AZ goes down. If Multi-AZ is enabled it also uses the standby instance for doing system backups.

To change the database to multi AZ simply click `Modify` and then check the radio button to create a standby instance:
{{< figure src="/images/rds/2.jpg" >}}
{{< figure src="/images/rds/3.jpg" >}}

Enabling Multi-AZ deployment takes around 30 minutes and can also cause some downtime to the database. Therefore, if it’s a production database make sure do do that during the maintenance window.

Multi-AZ replicates your data synchronously so you can be sure that data is saved in both AZ’s. In case one AZ goes down Amazon RDS performs an automatic failover to the standby instance.

## RDS Read replicas

As a name of the service suggests,  RDS read Replica can be used to launch a read only copy of your database. The really cool feature is that read replicas can be launched in other AWS regions than the main database. So if your compliance model requires to backup data in different AWS regions you can achieve it with read replica. Data between primary and read replica are copied asynchronously.  Read replica can be used to provide applications read only copy of your data as described in the diagram bellow. If primary database fails read replica can be promoted to become a primary database. It’s important to know that different then Multi-AZ, the promotion is not automatic so you’d have to manually do this step.
{{< figure src="/images/rds/4.jpg" title="diagram for RDS read replica" >}}

To create a read replica in AWS RDS console select your database instance and click on `Create read replica`. Then,  type replica instance identifier, select region where you want to launch the replica instance as well as select the appropriate security and subnet groups.

{{< figure src="/images/rds/5.jpg" title="Creating RDS read replica" >}}

After replica, creation is complete we can see that there are now to databases `wordpress` that has a primary role and `replica` with a replica role:
{{< figure src="/images/rds/6.jpg">}}

From the RDS logs we can observe that what it actually did in the background was restoring  the instance from a primary instance snapshot and starting an  asynchronous replication between these two instances.

Now, let's simulate the failure of a primary instance and try promoting the replica instance to be a primary.

We ca click on `actions -> promote` to achieve it. However, the warning is displayed to make sure to stop any transactions to the primary instance and wait for the Read Replica lag to be zero. To observe the replica lag, AWS provides a Cloudwatch metric for a reader instance:

{{< figure src="/images/rds/7.jpg" title="Observing replica lag for a reader instance" >}}

The Replica lag is zero so we can continue to promote the replica instance.
After the modification is complete we can see that AWS didn’t change the previous primary node to be read replica. Instead promoting read replica actually completely disables  replication between these two instances and they are now two independent databases:

{{< figure src="/images/rds/8.jpg" title="RDS console after promoting the read replica" >}}
It’s important to note that it’s not possible to demote the instance back to read replica. If I wanted another read replica I’d have to select one of the two databases that I have and create a new read replica.

## Conclusion

We can see that bot  Multi-AZ and Read replica  can be used to ensure resilience for RDS.
Because Multi-AZ provides automatic failover and synchronous replication it’s best to use Multi-AZ to ensure availability of your database and Read replicas to offset reads from a primary database as well as have a disaster recovery setup. Everything is also summarized in the table bellow:

|    | Multi-AZ | Read-replica    |
| ----------- | ----------- | ------------- |
| Replication      | Synchronous       | Asynchronous  |
| Can be used between AWS regions   | No       | Yes   |
| Can be used to make database highly available  | Yes       | No   |
| Provides automatic failover  | Yes       | No   |
| Can improve database performance by providing endpoint for database reads  | No       | Yes   |



If you have any questions or feedback feel free to leave a comment or contact me.
