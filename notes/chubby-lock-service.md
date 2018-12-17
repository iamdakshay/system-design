# The Chubby Lock Service
The Chubby Lock Service is a distributed course-grained locking service. Chubby also provides low-volume storage to be used as a repository for distributed systems' configuration changes. Chubby's most popular use has been as a name service.
## Overview
Chubby's goal is to allow clients to synchronize their activities and agree to basic information about their systems. Chubby uses the Paxos distributed consensus protocol to solve for asynchronous consensus. Each Chubby instance (cell) is dedicated to a single data center and typically serves about 10,000 machines.

These notes do not go into the details of Paxos distributed consensus. Refer to the notes on [Raft Distributed Consensus](https://github.com/jguamie/system-design/blob/master/notes/raft-distributed-consensus.md) (Raft is very similar to Paxos on the surface).
## Design
<img src="https://github.com/jguamie/system-design/blob/master/images/chubby-system.png" align="middle" width="50%">

Chubby wasn't designed to be a library that only provides Paxos distributed consensus, but instead as library that accesses a centralized lock service. This makes it easier for developers to maintain their existing program structures. Minimal effort is required to elect a master and write to an existing file server.

The storage feature is important as services need to advertise Chubby's results with others e.g., after a primary is elected or when data is repartitioned. Not having a separate service for sharing the results reduces the number of servers that clients depend on. Another benefit is that the consistency features of the protocol become shared.

Each Chubby cell typically has 5 replicas. This is to ensure high availability. A service advertising its primary through a Chubby file can have thousands of clients that need to observe this file. 93% of RPCs are KeepAlives between a Chubby client and a Chubby cell.

Chubby uses event notification to notify clients of any changes to Chubby files. For example, clients and replicas of a service will need to be notified if the master/primary changes. Most common events include file contents modified, child node added/removed/modified, and Chubby master failed over.
## Coarse-Grained Locks versus Fine-Grained Locks
Chubby utilizes coarse-grained locks (hours or days) over fine-grained locks (seconds or less). Coarse-grained locks are less load-intensive on the lock service. Lock-acquisition rates are rare in comparison to a client's transaction rates. Clients are not significantly delayed by the temporary unavailability of a lock server.

With fine-grained locks, even a brief unavailability of a lock server would cause many clients to stall. If client require fine-grained locks, it is straightforward for clients to implement their own into their applications.
## Locks and Sequencers
For writes, only one client can hold the lock in exclusive (writer) mode. For reads, any number of clients can hold the lock in shared (reader) mode. 

Chubby uses advisory (cooperative) locks over mandatory locks--the lock doesn't prevent other clients from accessing a file. With advisory locks, clients are expected to check for a lock prior to taking action on a file. Google has standards to ensure developers perform error checking by writing `assert(lock X is held)`. This minimizes the benefit provided by mandatory locks.

Mandatory locks prevent users from accessing a locked file for debugging or administrative purposes. If a file must be accessed, an entire application would need to be shutdown or rebooted to break the mandatory lock. Mandatory locks require extensive modifications and overhead to Chubby to support them in a meaningful way.

With distributed systems, receiving messages out of order is a problem that needs to be addressed--Chubby uses sequence numbers to do so. A client can request for a sequencer on files it holds a lock on. A sequencer is a byte-string that contains:
* Name of the lock
* Lock mode (exclusive or shared)
* Lock generation number

The client passes this sequencer to the appropriate file servers. The file servers are expected to validate the sequencer and protect the client's operations in the given lock mode.

For file servers that do not support sequencers, Chubby provides a lock-delay period--typically one minute. The lock-delay protects from regular problems caused by message delays and restarts.

## Example: Chubby Name Service
Chubby's success as a name service is due to its use of consistent client caching (distributed consensus) over time-based caching. Developers appreciate not having to manage a cache timeout such as DNS's time-to-live (TTL) value.

Caching within the standard internet naming system, DNS, is based on time. Each DNS entry has a TTL.
* Long TTL values lead to long client fail-over times.
* Short TTL values will overload DNS servers.

Chubby's caching uses explicit invalidations--a constant rate of session KeepAlive requests can maintain a large number of cache entries indefinitely.

A single Chubby name service cell is able to handle 90,000 clients communicating directly with it.
## Examples
* **Google File System**. Chubby is used to appoint a GFS master server. It is also used to store metadata. Chubby is the root of its distributed data structures.
* **Bigtable**. Chubby is used to elect a master, allow the master to discover the servers it controls, and allow clients to find the master. It is also used to store metadata. Chubby is the root of its distributed data structures.
* **Access Control Lists (ACLs).** Chubby is used as a repository for ACL files.
# References
1. Burrows, Mike, "[The Chubby lock service for loosely-coupled distributed systems](https://ai.google/research/pubs/pub27897)," 7th USENIX Symposium on Operating Systems Design and Implementation (OSDI), {USENIX} (2006)