This was project done for the purpose of exploring OpenNebula use cases as part of "Virtuelization and Cloud Computing" subject at UKIM - FCSE Skopje

## Software tried and/or used: ##
  * NFS with DRBD
  * GlusterFS
  * MariaDB
  * Pen load balancer
  * Apache2.4 with PHP-FPM
  * UCARP
  
## Initial idea ##
The point of this setup is to find a way of updating all the hosts in the setup, and rebooting them one-by-one, without downtime. All of the nodes needed to be synchronized to prevent split-brain scenarios or data corruption, and need to be aware of each others (non-)existence.

The initial design of the cluster was the following:
  * 1 x Load Balancer for the incoming connections (1 UCARP IP, always master)
  * 2 x Apache based web hosts with PHP-FPM on them, connecting to the DataBase (DB) and File Storage(FS) servers (sharing the UCARP IP when the load balancer was down)
  * 2 x File Storage servers (sharing 1 UCARP IP)
  * 2 x DataBase servers (sharing 1 UCARP IP)

The setup has three sub problems which are going to be discussed:
  * Failover - detection and recovery
  * Syncing file storage nodes
  * Syncing database nodes

## Failover - detection and recovery ##
CARP is IP based failover solution implemented in BSD systems, later implemented in Linux systems as UCARP. It works by providing a floating IP address to a single node, this node then handles all the incoming connections. This failed to work due to OpenNebulas internal networking. The floating IP can't be assigned from inside the VMs, it has to be done by OpenNebula. Potential solution to this would be to create Virtual Router kind of VM where this floating IP would be handled by Keepalived (very similar to UCARP, same principles apply) on OpenNebula level. I couldn't do this at the moment.

The idea of using UCARP would have allowed to have one Load Balancer, which when shut down transfers its role to one of the Web servers, going from load balanced system with 2 parallel systems, to Active-Passive configuration - backup system with hot standby. Since I can't use UCARP I simply left it out of the equation making the load balancer single point of failure.

## Syncing file storage nodes ##
The first try was using NFS and DRBD together. It will work with UCARP and would provide active-passive setup with hot standby backup. It worked, but since UCARP wasn't available for failover the idea was abandoned. GlusterFS was used instead.

GlusterFS is clustered file storage system which can work in active-active configuration. The clients themselves do the failure detection and they automatically transfer to the next available node. The node information can be shared via-network and can be dynamically updated. GlusterFS in this configuration was used with replica nodes (mirror nodes). The performance was somewhat worse than with NFS.

## Syncing database nodes ##
MariaDB comes bundled with Galera cluster. It allows multi-master configuration for MySQL/MariaDB. The same configuration can be achieved inside of MariaDB, but it's prone to split-brain, so it's advised to be done in master-slave configuration where the slaves are read-only. Galera uses out-of-band communication to achieve this, and is setup with additional configuration files rather than inside of MySQL/MariaDB.

In this setup there are shortcomings of using only two nodes. If one of the nodes is shutdown unexpectedly, the other node goes into failsafe where it doesn't accept queries to prevent split-brain. If the node is shutdown gracefully then the rest of the cluster (one node) continues to operate without issues. To resolve this third node needs to be implemented which will be Arbitrator node, or forcefully disable the split-brain failsafe which didn't work during test phase.

Wordpress by design doesn't allow usage of clustered databases. There are two solutions for this, one of them is Wordpress module which allows that (HyperDB), the other one is using standard TCP load balancer.
* HyperDB worked, but not out of the box. Takes some time to detect failed node and timeouts are sometimes possible. Requires modifying Wordpress to make it work.
* Pen load balancer was used instead. It listens on localhost on the common port for MySQL/MariaDB and then redirects the TCP stream to one of the nodes available. This makes it possible for any software using databases to work with clustered databases. This balancer utilizes temporary blacklisting to prevent common timeouts on failed nodes.

## Final design of the cluster ##
  * 1 x Load Balancer for the incoming connections (single point of failure)
  * 2 x Apache based web hosts with PHP-FPM on them, connecting to the DataBase (DB) and File Storage(FS) servers
  * 2 x File Storage servers with GlusterFS
  * 2 x DataBase servers with MariaDB and Galera cluster.

The single point of failure can be mitigated with Keepalived in OpenNebula, and the Galera cluster failsafe can be mitigated with third, Arbitrator node. Except for the load balancer, any single node can be down at any time, one of each group of hosts at most (3 total at most).
