+++
title = 'Trouble with distributed systems'
date = 2022-05-22T13:57:56+05:30 
mathjax = true
+++

- *anything that can go wrong will go wrong* 
- This chapter is a thoroughly pessimistic and depressing overview of things that may go wrong in a distributed system
- working with distributed systems vs writing software for on a single computer
	- problems with networks
	- clocks and timing issues
- ## Faults and Partial Failures
	- An individual computer with good software is usually either fully functional or entirely broken, but not something in between
	- when the hardware is working correctly, the same operation always produces the same result (it is deterministic)
	- The design goal of computer is always correct computation. Why so?
		- if an internal fault occurs like (e.g., memory corruption or a loose connector), we prefer a computer to crash completely rather than returning a wrong result because wrong results are difficult and confusing to deal with
		- computers hide the fuzzy physical reality on which they are implemented and present an idealized system mode
	- writing software that runs on several computers, connected by a network, the situation is fundamentally different 
	- *Partial Failure:* In a distributed system, there may well be some parts of the system that are broken in some unpredictable way, even though other parts of the system are working fine 
	- The difficulty is that partial failures are *nondeterministic*
		- if you try to do anything involving multiple nodes and the network, it may sometimes work and sometimes unpredictably fail
	- This *nondeterminism* and possibility of *partial failures* is what makes distributed systems hard to work with 
- ### Cloud Computing and Supercomputing

<h3>HPC vs Cloud Computing</h3>

<table>
  <thead>
    <tr>
      <th>High-Performance Computing</th>
      <th>Cloud Computing</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Supercomputers with thousands of CPUs for scientific tasks</td>
      <td>Multi-tenant datacenters shared by numerous organizations</td>
    </tr>
    <tr>
      <td>Checkpointing to restart computations from last checkpoint</td>
      <td>Commodity computers connected via IP network</td>
    </tr>
    <tr>
      <td>Deals with partial failure by escalating into total failure</td>
      <td>Elastic/on-demand resource allocation and uninterrupted service for internet-related applications</td>
    </tr>
    <tr>
      <td>Typically built from specialized hardware, where each node is quite reliable</td>
      <td>Made using commodity hardware which provides equivalent performance but has a higher rate of failure</td>
    </tr>
    <tr>
      <td>Supercomputers often use specialized network topologies, such as multi-dimensional meshes and toruses, which yield better performance</td>
      <td>Datacenter networks are often based on IP and Ethernet, arranged in Clos topologies to provide high bisection bandwidth</td>
    </tr>
  </tbody>
</table>

- The bigger a system gets, the more likely it is that one of its components is broken. In a system with thousands of nodes, it is reasonable to assume that something is always broken
- If the system can tolerate failed nodes and still keep working as a whole, that is a very useful feature for operations and maintenance
- If we want to make distributed systems work, we must *accept the possibility of partial failure* and build fault-tolerance mechanisms into the software
- *build a reliable system from unreliable components* 
- ### Unreliable Networks
	- *shared-nothing systems:* a bunch of machines connected by a network. The network is the only way those machines can communicate
	- The internet and most internal networks in datacenters (often Ethernet) are *asynchronous packet networks*. In this kind of network, one node can send a message (a packet) to another node, but the network gives no guarantees
	- Things can that go wrong?
		- request may have been lost (someone unplugged a network cable)
		- request may be waiting in a queue and will be delivered later (network or recipient are overloaded)
		- remote node failure (crash or powered down)
		- garbage collection pauses 
		- request processed by remote node but response lost in network/response has been delayed (network or own machine overloaded)
	- The sender can’t even tell whether the packet was delivered: the only option is for the recipient to send a response message, which may in turn be lost or delayed
	- The usual way of handling this issue is a *timeout*: after some time you give up waiting and assume that the response is not going to arrive. But in this case also we don't know whether the remote host got the request or not. 
- ### Network Faults in Practice
	- systemic studies indicate network faults can be can be surprisingly common even in controlled environments like datacenters 
	- studies of Network Faults in datacenters
		- ~12 network faults per month in a mid-sized datacenter of which half disconnected a single machine, and half disconnected an entire rack
		- On studying the failure rates of of components like top-of-rack switches, aggregation switches, and load balancers It found that adding redundant networking gear doesn’t reduce faults as much as you might hope, since it doesn’t guard against human error
		-  faults can occur during a software upgrade for a switch could trigger a network topology reconfiguration, during which network packets could be delayed for more than a minute
	- Even if network faults are rare in your environment, the fact that faults can occur means that your *software needs to be able to handle them* 
	- Handling network faults doesn’t necessarily mean tolerating them
		- you do need to know how your software reacts to network problems and ensure that the system can recover from them
		- Chaos Monkey: deliberately trigger network problems and test the system’s response
- ### Detecting Faults
	- systems need to automatically detect faults 
		- A load balancer needs to stop sending requests to a dead node
		- In a single leader database if the leader fails then one of the followers needs to be promoted to leader
	- How can we tell if a node is dead ?
		- if the machine is reachable but the process crashed then os will refuse connection on that port 
		- if the process crashed but the the os is running then a script can notify other nodes about the crash so that another node can takeover quickly
		- Detecting link failure at hardware level needs access to management interface of network switches. If we are connecting via internet or in a shared datacenter that option is gone 
		- If a router is sure that the IP address you’re trying to connect to is unreachable, it may reply to you with an ICMP Destination Unreachable packet
	- Even if the TCP acknowledges that the packet was delivered there is no guarantee that the application would have handled it before crashing. To know whether an request succeeded the application needs to send a positive response
	- if something has gone wrong we can retry(TCP retries or application level retries) a few times, wait for a timeout to elapse and declare the node dead
	- ### Timeouts and Unbounded Delays
		- If a timeout is the only sure way of detecting a fault, then how long should the time‐ out be? 
		- long time out: a long wait until a node is declared dead and during this time, users may have to wait or see error messages
		- short time out: detects faults faster, but carries a higher risk of incorrectly declaring a node dead when in fact it has only suffered a temporary slowdown
		- what is a reasonable time out period to use?
			- suppose in a system every packet is delivered within $d$ time or it is lost
			- we can assume that a non failed node handles the request in $r$ time
			- a reasonable delay to expect should be $2d + r$ under these guarantees
		- Most systems don't operate under such guarantees: *asynchronous networks* have unbounded delays(there is no upper time limit on the time it may take for a packet to arrive)
	- ### Network congestion and queueing
		- queueing at different places
			- network switch
				- *Network congestion:* when multiple nodes attempt to send packets to the same destination, causing a delay in packet transmission as they are queued up within the network switch.
			- operating system 
				- if all CPU cores are currently busy, the incoming request from the network is queued by the operating system until the application is ready to handle it
			-  VM manager
				- In virtualized environments, a running operating system is often paused for tens of milliseconds while another virtual machine uses a CPU core. During this time, the VM cannot consume any data from the network, so the incoming data is queued (buffered) by the virtual machine monitor
			- Sender
				- TCP performs flow control (also known as congestion avoidance or backpressure), in which a node limits its own rate of sending in order to avoid overloading a network link or the receiving node
		- In public clouds and multi-tenant datacenters, resources like network links, switches, and even machine components are shared among many customers. This sharing can lead to variable network delays, especially when batch workloads, such as MapReduce
		- In such scenarios to find the time out period we can measure the distribution of network round-trip times over an extended period, and over many machines, to determine the expected variability of delays
		- rather than using configured constant timeouts, systems can continually measure response times and their variability (jitter), and automatically adjust time‐ outs according to the observed response time distribution. (checkout TCP retransmission timeout it works in the same way)
	- ## Synchronous Versus Asynchronous Networks
		- can we deliver packets with a fixed maximum delay and no packets drop making the systems much simpler?
		- To answer this question, it’s interesting to compare datacenter networks to the traditional fixed-line telephone network(non-cellular, non-VoIP)
			- we hardly see any delayed audio frames and dropped calls in telephone network. A phone call requires a constantly low end-to-end latency and enough bandwidth to transfer the audio samples of your voice. It would be nice to have such similar reliability and predictability in computer networks
			- In a call over a telephone network a *circuit* is established a fixed, guaranteed amount of bandwidth is allocated for the call, along the entire route between the two callers
			- When a call is established, it is allocated 16 bits of space within each frame (in each direction). Thus, for the duration of the call, each side is guaranteed to be able to send exactly 16 bits of audio data every 250 microseconds
			- This kind of network is *synchronous*: even as data passes through several routers, it does not suffer from queueing, because the 16 bits of space for the call have already been reserved in the next hop of the network
			- because there is no queueing, the maximum end-to-end latency of the network is fixed. We call this a *bounded delay*
	- ### Can we not simply make network delays predictable?
		- A circuit network has fixed reserved bandwidth which no one else can use when the circuit is established, whereas the packets of a TCP connection opportunistically use whatever network bandwidth is available. 
		- If datacenter networks and the internet were circuit-switched networks, it would be possible to establish a guaranteed maximum round-trip time when a circuit was set up
		- Ethernet and IP are packet-switched protocols, which suffer from queueing and thus unbounded delays in the network. No concept of circuit
		- Why do datacenter networks and the internet use packet switching? 
			- The answer is that they are optimized for bursty traffic
			- A circuit is good for an audio or video call, which needs to transfer a fairly constant number of bits per second for the duration of the call
			- if you want to transfer a file over a circuit network, you would have to guess a bandwidth allocation.
			- If you guess too low, the transfer is unnecessarily slow
			- If you guess too high, the circuit cannot be set up (because the net‐ work cannot allow a circuit to be created if its bandwidth allocation cannot be guaranteed)
			- using circuits for bursty data transfers wastes network capacity and makes transfers unnecessarily slow.
			- TCP dynamically adapts the rate of data transfer to the available network capacity.
	- ## Unreliable Clocks
		- durations vs point in time
		- *durations:* request timed out, 99th percentile response time, qps a service handles
		- *point-in-time:* when the remainder is sent, when does this cache entry expire
		- ### Monotonic Versus Time-of-Day Clocks
			- #### Time-of-day clocks
				- What you intuitively expect a clock to be also known as wall clock
				- Time-of-day clocks are usually synchronized with NTP
				- time-of-day clocks also have various oddities
					- if the local clock is too far ahead of the NTP server, it may be forcibly reset and appear to jump back to a previous point in time
					- These jumps, as well as the fact that they often ignore leap seconds, make time-of-day clocks unsuitable for measuring elapsed time
			- #### Monotonic clocks
				- measuring a duration (time interval), such as a timeout or a service’s response time
				- check the value of monotonic time at two points and the difference between them tells how much time has elapsed
				- NTP may adjust the frequency at which the monotonic clock moves forward (this is known as slewing the clock) if it detects that the computer’s local quartz is moving faster or slower than the NTP server
				- By default, NTP allows the clock rate to be spee‐ ded up or slowed down by up to 0.05%, but NTP cannot cause the monotonic clock to jump forward or backward.
			- In a distributed system, using a monotonic clock for measuring elapsed time (e.g., timeouts) is usually fine, because it doesn’t assume any synchronization between different nodes’ clocks
		- ### Clock Synchronization and Accuracy
			- Monotonic clocks don’t need synchronization, but time-of-day clocks need to be set according to an NTP server or other external time source in order to be useful.
			- Our methods for getting a clock to tell the correct time isn't nearly as reliable or accurate
				- The quartz clock in a computer is not very accurate: it drifts. Google assumes a clock drift of 200 ppm (parts per million) for its servers which is equivalent to 6 ms drift for a clock that is resynchronized with a server every 30 seconds, or 17 seconds drift for a clock that is resynchronized once a day
				- If a computer’s clock differs too much from an NTP server, it may refuse to synchronize, or the local clock will be forcibly reset
				- If a node is accidentally firewalled off from NTP servers, the misconfiguration may go unnoticed for some time.
				- NTP synchronization *can only be as good as the network delay*, so there is a limit to its accuracy when you’re on a congested network with variable packet delays.
				- Some NTP servers are wrong or misconfigured, reporting time that is off by hours. NTP clients are quite robust, because they query several servers and ignore outliers
				- Leap seconds result in a minute that is 59 seconds or 61 seconds long, which messes up timing assumptions in systems that are not designed with leap seconds in mind (many major crashes due to leap seconds)
				- In virtual machines, the hardware clock is virtualized, When a CPU core is shared between virtual machines, each VM is paused for tens of milli‐ seconds while another VM is running From an application’s point of view, this pause manifests itself as the clock suddenly jumping forward
			- possible to achieve very good clock accuracy
				- MiFID II draft European regulation for financial institutions requires all high-frequency trading funds to synchronize their clocks to within 100 microseconds of UTC
				- Such accuracy can be achieved using GPS receivers, the Precision Time Protocol (PTP), and careful deployment and monitoring
		- ### Relying on Synchronized Clocks
			- robust software needs to be prepared to deal with incorrect clocks.
			- incorrect clocks easily go unnoticed.
			- if its quartz clock is defective or its NTP client is misconfigured, most things will seem to work fine, even though its clock gradually *drifts further and further away from reality*
			- if you use software that requires synchronized clocks, it is essential that you also carefully monitor the clock offsets between all the machines.
		- #### Timestamps for ordering events 
			- if two clients write to a distributed database, who got there first? Which write is the more recent one?
			- Conflicts may happen when time stamps fail to order even correctly. When a write has happened one after the other but their time stamp do not reflect that, the node will incorrectly drop the older value and accept the recent value
			- This conflict resolution strategy is called last write wins (LWW), and it is widely used in both multi-leader replication and leaderless databases such as Cassandra 
				- Database writes can mysteriously disappear: a node with a lagging clock is unable to overwrite values previously written by a node with a fast clock
				- LWW cannot distinguish between writes that occurred sequentially in quick succession i.e. concurrent writes
				- So-called logical clocks which are based on incrementing counters rather than an oscillating quartz crystal, are a safer alternative for ordering events 
		- #### Clock readings have a confidence interval
			- Even if we are able get a very fine grained measurement, that doesn't mean value is accurate to that precision. (drift, network latency,network congestion can make the clock readings variable)
			- it doesn’t make sense to think of a clock reading as a point in time—it is more like a range of times, within a confidence interval: for example, a system may be 95% confident that the time now is between 10.3 and 10.5 seconds past the minute
			- The uncertainty bound can be calculated based on your time source(GPS receiver, atomic clock, uncertainty is based on the expected quartz drift since your last sync with the server, plus the NTP server’s uncertainty, plus the network round-trip time to the server).Unfortunately, most systems don’t expose this uncertainty: for example, when you call clock_gettime(), the return value doesn’t tell you the expected error of the timestamp
			- An interesting exception is Google’s TrueTime API in Spanner, which explicitly reports the confidence interval on the local clock. When you ask it for the current time, you get back two values: `[earliest, latest]`, which are the earliest possible and the latest possible timestamp
		- #### Synchronized clocks for global snapshots
			- The most common implementation of snapshot isolation requires a monotonically increasing transaction ID
			- However, when a database is distributed across many machines, potentially in multiple datacenters, a global, monotonically increasing transaction ID (across all partitions) is difficult to generate
			- Can we use the timestamps from synchronized time-of-day clocks as transaction IDs? If we could get the synchronization good enough, they would have the right properties
			- Google spanner snapshot isolation implementation:
				- if you have two confidence intervals, each consisting of an earliest and latest possible timestamp $(A = [A_{earliest}, A_{latest}] \ and \ B = [B_{earliest}, B_{latest}])$, and those two intervals do not overlap $(i.e., A_{earliest} < A_{latest} < B_{earliest} < B_{latest})$, then B definitely happened after A—there can be no doubt. Only if the intervals overlap are we unsure in which order A and B happened
				- In order to ensure that transaction timestamps reflect causality, Spanner deliberately waits for the length of the confidence interval before committing a read-write transaction
				- By doing so, it ensures that any transaction that may read the data is at a sufficiently later time, so their confidence intervals do not overlap.
				- In order to keep the wait time as short as possible, Spanner needs to keep the clock uncertainty as small as possible
	- ### Process Pauses
		- a garbage collector (GC) that occasionally needs to stop all running threads. These “stop-the-world” GC pauses have sometimes been known to last for several minutes
		- In virtualized environments, a virtual machine can be suspended (pausing the execution of all processes and saving the contents of memory to disk) and resumed
		- On end-user devices such as laptops, execution may also be suspended and resumed arbitrarily
		- When the operating system context-switches to another thread, or when the hypervisor switches to a different virtual machine (when running in a virtual machine), the currently running thread can be paused at any arbitrary point in the code.In the case of a virtual machine, the CPU time spent in other virtual machines is known as *steal time* 
		- If the application performs synchronous disk access, a thread may be paused waiting for a slow disk I/O operation to complete
		- If the operating system is configured to allow swapping to disk (paging), a simple memory access may result in a page fault that requires a page from disk to be loaded into memory. The thread is paused while this slow I/O operation takes place.
		- All of these occurrences can preempt the running thread at any point and resume it at some later time, without the thread even noticing.
		- A node in a distributed system must assume that its execution can be paused for a significant length of time at any point, even in the middle of a function
		- Limiting the impact of garbage collection
			- An emerging idea is to treat GC pauses like brief planned outages of a node, and to let other nodes handle requests from clients while one node is collecting its garbage
			- If the runtime can warn the application that a node soon requires a GC pause, the application can stop sending new requests to that node, wait for it to finish process‐ ing outstanding requests, and then perform the GC while no requests are in progress
			- A variant of this idea is to use the garbage collector only for short-lived objects (which are fast to collect) and to restart processes periodically, before they accumulate enough long-lived objects to require a full GC of long-lived objects
			- One node can be restarted at a time, and traffic can be shifted away from the node before the planned restart, like in a rolling upgrade
	- ## Knowledge, Truth, and Lies
		- The consequences of these issues(unreliable network with variable delays, and the systems may suffer from partial failures, unreliable clocks, and processing pauses) in distributed systems are profoundly disorienting if you’re not used to distributed systems
		 - A node can only find out what state another node is in (what data it has stored, whether it is correctly functioning, etc.) by exchanging messages with it
		 - In a distributed system, we can state the assumptions we are making about the behaviour (the system model) and design the actual system in such a way that it meets those assumptions
		- Algorithms can be proved to function correctly within a certain system model.
		- This means that reliable behaviour is achievable, even if the underlying system model provides very few guarantees.
	- ### The Truth Is Defined by the Majority
		- network with an *asymmetric fault*: a node is able to receive all messages sent to it, but any outgoing messages from that node are dropped or delayed
			- Even though the node is working, other node cannot hear its response and will declare it dead after some time
			- In a slightly better scenario, if the semi disconnected node detects that none of the messages it is sending is getting any response it may and realize that there may be a fault in the network. Nevertheless the node may be declared dead
			- In a third scenario there may be long GC pauses that stops sending and receiving request and response. The other nodes wait and retry and finally declare the node dead. 
		- The moral of these stories is that a node cannot necessarily trust its own judgment of a situation
		- A distributed system cannot exclusively rely on a single node, because a node may fail at any time, potentially leaving the system stuck and unable to recover
		- many distributed algorithms rely on a *quorum*, that is, voting among the nodes
		- If a quorum of nodes declares another node dead, then it must be considered dead, even *if that node still very much feels alive* Xd
		- #### The leader and the lock
			- Frequently, a system requires there to be *only one of some thing*?
				- Only one node is allowed to be the leader for a database partition, to avoid split brain
				- Only one transaction or client is allowed to hold the lock for a particular resource or object, to prevent concurrently writing to it and corrupting it
				- Only one user is allowed to register a particular username, because a username must uniquely identify a user
			- if a node believes that it is the chosen one(even though majority of the nodes have declared it dead) and continues to behave like it, it could cause problems 
			- data corruption bug due to an incorrect implementation of locking
				- you want to ensure that a file in a storage service can only be accessed by one client at a time because multiple access can cause the data to be corrupted 
				- we implement this by requiring a client to obtain a lease from a lock service before accessing the file
				- if the client holding the lease is paused for too long, its lease expires. Another client can obtain a lease for the same file, and start writing to the file
				- When the paused client comes back, it believes that it still has valid lease and proceeds to write and corrupts the file
			- #### Fencing tokens
				- We need to ensure that node is not under the false belief that it is the chosen one. A simple technique that achieves this goal is called fencing
				- every time the lock server grants a lock or lease, it also returns a fencing token, which is a number that increases every time a lock is granted
				- We can then require that every time a client sends a write request to the storage service, it must include its current fencing token.
				- When a dead client comes back online and sends its write request to the client along with a fencing token, its write will be rejected if a write with a higher token value has already been processed
	- ### Byzantine Faults
		- Fencing tokens can detect and block a node that is inadvertently acting in error, however if a node acts maliciously and wants to subvert system guarantees it could easily do it by sending fake messages 
		- The discussion in the book is limited to the assumption that nodes are unreliable but honest 
		- Distributed systems problems become much harder if there is a risk that nodes may “lie” (send arbitrary faulty or corrupted responses)
		- if a node may claim to have received a particular message when in fact it didn’t. Such behaviour is known as a *Byzantine fault*, and the problem of reaching consensus in this untrusting environment is known as the *Byzantine Generals Problem*
		- A system is *Byzantine fault-tolerant* if it continues to operate correctly even if some of the *nodes are malfunctioning and not obeying the protocol*, or if *malicious attackers* are interfering with the network.
		- Situations where Byzantine fault may arise
			- In aerospace environments, the data in a computer’s memory or CPU register could become corrupted by radiation leading it to respond to other nodes in arbitrarily unpredictable ways
			- In a system with multiple participating organizations, some participants may attempt to cheat or defraud others. In such circumstances, it is not safe for a node to simply trust another node’s messages. e.g. Bitcoin
		- The discussion here is limited to non byzantine fault tolerant systems 
		- A bug in the software could be regarded as a Byzantine fault, but if you deploy the same software to all nodes, then a Byzantine fault-tolerant algorithm cannot save you
		- To use this approach against bugs, you would have to have four independent implementations of the same software and hope that a bug only appears in one of the four implementations
	- ### System Model and Reality
		- Algorithms need to be written in a way that does not depend too heavily on the details of the hardware and software configuration on which they are run
		- This in turn requires that we somehow formalize the kinds of faults that we expect to happen in a system
		- We do this by defining a *system model*, which is an abstraction that describes what *things an algorithm may assume* 
			- System models wrt *timing* assumptions
				- *Synchronous model:* The synchronous model assumes bounded network delay, bounded process pauses,and bounded clock error. Not a realistic model for most practical systems
				- *Partially synchronous model:* Partial synchrony means that a system behaves like a synchronous system most of the time, but it sometimes exceeds the bounds for network delay, process pauses, and clock drift. This is a realistic model of many systems: most of the time, networks and processes are quite well behaved
				- *Asynchronous model:* an algorithm is not allowed to make any timing assumptions—in fact, it does not even have a clock (so it cannot use timeouts)
			- system model wrt node failure assumptions 
				- *Crash-stop faults:* Node can stop in only one way i.e. crashing and never come back
				- *Crash-recovery faults:* We assume that nodes may crash at any moment, and perhaps start responding again after some unknown time. In this failure model nodes are assumed to have stable storage. 
				- *Byzantine (arbitrary) faults:* Nodes may do absolutely anything
			- For modeling real systems, the partially synchronous model with crash-recovery faults is generally the most useful model. 
	- ### Correctness of an algorithm
		- what is means for an algorithm to be correct?
		- we can write down the properties we want of a distributed algorithm to define what it means to be correct
		- #### Safety and liveness
			- safety 
				- Safety is often informally defined as *nothing bad happens*
				- If a safety property is violated, we can point at a particular point in time at which it was broken
				- After a safety property has been violated, the violation cannot be undone the damage is already done
				- For distributed algorithms, it is common to require that safety properties always hold, in all possible situations of a system model
			- liveness 
				- it is defined as *something good eventually happens*
				- liveness properties often include the word “eventually” in their definition
				- it may not hold at some point in time, but there is always hope that it may be satisfied in the future
				- with liveness properties we are allowed to make caveats: for example, we could say that a request needs to receive a response only if a majority of nodes have not crashed
		- #### Mapping system models to the real world
			- When implementing distributed systems messy reality of real world come out. It becomes clear that system model is a simplified abstraction of reality
			- Quorum algorithms rely on node remembering the data, that it claims to have stored. If a node may suffer from amnesia and forget previously stored data, that breaks the quorum condition
			- what happens when assumption gets broken -> correctness of algorithm gets broken 
			- Perhaps a new system model? assumption being, storage nodes mostly survive crashes, but sometimes may be lost, This makes reasoning harder
			- The theoretical description of an algorithm can declare certain things correct under certain assumptions, but in real world those assumptions can break and needs to be handled.
			- This doesn't make theoretical models useless, quite the opposite, They are incredibly helpful for distilling down the complexity of real systems to a manageable set of faults that we can reason about
			- We can prove algorithms correct by showing that their properties always hold in some system model.
			- *Proving an algorithm correct does not mean its implementation on a real system will necessarily always behave correctly*
			- But it’s a very good first step, because the theoretical analysis can uncover problems in an algorithm that might remain hidden for a long time in a real system, and that only come to bite you when your assumptions (e.g., about timing) are defeated due to unusual circumstances