+++
title = 'Reliable, Scalable, and Maintainable Applications'
date = 2022-05-22T13:57:56+05:30 
mathjax = true
+++

1. Standard building blocks of data intensive applications
		databases
		caches
		search indexes
		stream processing
		batch processing
	The above sounds painfully obvious ? 
		 because data systems are successful abstractions 
	API abstracts away the data system 
2.  Focus on 3 concerns in software systems
	1. ***Reliability*** 
		-  The systems should continue to work *correctly* even in face of adversity
		-  Tolerate user mistakes
		-  Performance good enough under expected load and volume 
		-  Prevents unauthorized access and abuse
		- ***"Continuing to work correctly even when things go wrong"*** 
			- Fault tolerant/ Resilient
			- Things that can go wrong are called fault
			-  System that can anticipate fault and deal with it are called fault tolerant
			-  Difference between fault and failure
				- Fault a single component deviating from spec
				- Failure when the system as a whole fails providing the desired service
			- Multiple faults lead to failure of the service 
			- Chaos Monkey (Netflix)
				- Intentionally Inducing faults to check fault tolerance machinery in action
		- Types of Faults
			- Hardware Faults
			- Software Faults
				- Allowing processes to crash and restart 
				- Careful thinking about assumptions and interactions
				- Measuring, monitoring and analyzing
			- Human Errors
				- How to make systems reliable in spite of unreliable humans 
					- Allow quick and easy recovery from human errors
					- Quick rollback of configuration changes
					- Roll out code slowly 
					- clear monitoring and performance metrics
	2. ***Scalability***
		- If a system grows in a particular way what are are ways of dealing with the growth?
		- How can we add computing resources to handle the additional growth
			- Load Parameters (Best choice depends on the architecture)
				- Request Per Seconds
				- Ratio of reads to write to database
				- No. of simultaneously active users in a chatroom
				- The hit rate on a cache
			- Describing load -> Describing Performance
				- Increase load parameter and keep system resources unchanged how is the performance of the system affected?
				- When you increase load parameter, how much do you need to increase the resources to keep the performance unchanged ?
			- Batch Processing System
				- We care about throughput, the number of jobs we can process per second or the total time it takes to run a job
			- Online Systems
				- We care about response time
					- Response time: This is what client sees
					- Latency: the duration that a request is waiting to be handled during which it is latent, awaiting service
			- ***Response Percentiles*** 
				- **Median**: If median response time is 200 ms then half your requests return in less than 200 ms, and half your requests take longer than that
					- Metric to understand how long typically users have to wait
					- 95th, 99th, and 99.9th percentiles
						- if the 95th percentile response time is 1.5 seconds, that means 95 out of 100 requests take less than 1.5 seconds, and 5 out of 100 requests take 1.5 seconds or more
				- ***Tail Latencies***
					- Directly affect user experience 
						- Amazon response time requirements for internal services in terms of the 99.9th percentile even though it only affects 1 in 1,000 requests
						- customers with the slowest requests are often those who have the most data on their accounts because they have made many purchase that is, they’re the most valuable customers
						- Optimizing the 99.99th percentile (the slowest 1 in 10,000 requests) was deemed too expensive and to not yield enough benefit for Amazon’s purposes
				- ***Head of Line Blocking*** 
					- Only takes a small number of slow requests to hold up the processing of subsequent requests Even if those subsequent requests are fast to process on the server, the client will see a slow overall response time due to the time waiting for the prior request to complete, thats why necessary to measure response on client side
				- ***Tail Latency Amplification***
					- If end user requires multiple backend calls to serve a request a single slow call can make the entire end user request slow
		- ***Approaches For Dealing With Load***
			- Easy: distributing stateless services across multiple machines
			- Hard: taking stateful data systems from a single node to a distributed setup
			- In an early-stage startup or an unproven product it’s usually more important to be able to iterate quickly on product features than it is to scale to some hypothetical future load
			- Architecture is built around certain assumptions (load parameters) and if they are wrong the scaling efforts are wasted. 
	3. ***Maintainability***
		- Operability
			- Make it easy for operations teams to keep the system running smoothly.
			- **Good operations can often work around the limitations of bad (or incomplete) software, but good software cannot run reliably with bad operations**
	      - Simplicity (Managing complexity)
		      - Make it easy for new engineers to understand the system
		      - when the system is harder for developers to understand and reason about
			      - hidden assumptions
			      - unintended consequences
			      - unexpected interactions
			 - Making a system simpler does not necessarily mean reducing its functionality; it can also mean removing accidental complexity
				 - Accidental Complexity 
					 - complexity not inherent in the problem that the software solves (as seen by the users) but arises only from the implementation.
				 - Removing Complexity
					 - Good abstraction
						 - hide a great deal of implementation detail behind a clean,simple-to-understand façade
						 - finding good abstractions is very hard
						 - In distributed systems many good algorithms but much less clear how to package them into algorithms that help in managing the Complexity of the system
					- reusable components	 
		- Evolvability
			- Make it easy for engineers to make changes to the system in the future
			- agility on a data system level: evolvability
