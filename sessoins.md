# Expedia
[Styx](https://github.com/Hotelsdotcom/styx)

Web-resource is more harder every year,
system bigger and need to recreate system.

Evolution:
- akamai frontend
- akamai > nginx frontends
- styx: each node on jvm: non-blocking IO

**Styx** scheme:
contains end-point router and a lot of plugins before it.

Arch: Akamai switching data on node.
Inner edge:
- styx routing
- auth/bot managment
- proxies millions RPM
- routing logic by headers and requests

Routing:
- multi-region
- 2 phases:
  - end-point resolution (like a path)
  - application resolution (http://www.clara-rules.org/)

Routing might use threshold breaker value.
Bot treatments:
- recaptcha
- tarpit
- blackhole
- DOM mangling

Bot classification works in bot score and cassandra database data.
If it's bot, cassandra call recaptcha.

Consul using like an orchestrator engine and managing cloud-gate-proxy.
Cloud-gate-proxy is styx. lobot is mechanism, that support persistence
data for user, like a sharing if.

Bot detection:
- voight-kampff
- auto detect: tensorflow

Monitoring:
- spark streaming (short term)
- QWS firehouse to S3 / athena for queries (long term)

Tracing: uses haystack.

    What we learned:
    - don't build monolith edge layer
    - don't block
    - remain IO bound, avoid rendering
    - conffiles don't scale
    - be resilient


# Quantifying scalability
> What's happening under high load?
  Cook&Rasmussen describe boundaries.
> How can we define and reason about capacity?
  Look at utilization increasing, nut it's not linear.

Ideal scaling is linear, we've added node and get equal
throughtput like a one node. It's not cheaped.
Math:

    x(N) = lambda * N/1

But our cluster isn't perfect.
We need to use parallel queues for tasks (e.g. scatter-gather).
So our math:

    x(N) = lambda * N / (1 + omega * (N-1))
    where omega is fraction, that can't be done in parallel

Sometimes our workers have depencies on each other,
it's add delay, do more work:
N workers = N(N-1) pairs, and now we got:

    x(N) = lambda * N / (1 + omega * (N-1) + kappa * N * (N-1)),
    kappa - penalty coef (the slowest factor)

How does degradation works:
[calc](https://www.desmos.com/calculator/3cycsgdl0b)

So scalability is:

    x(N) = lambda * N / (1 + omega * (N-1) + kappa * N * (N-1))

Load is concurrency - number of reqs in progress.
Easy to measure: sum(latency)/interval
- MySQL: show status like 'Threads_running'
- Apache: active worker count

The USL can reveal the workload failure boundary approaching.
This framework is for making systems look really bad.

Concurrency vs Lattency is good metric, and is quite useful.


# Machine learning in OPS
> You don't have data without information,
  but you cannot have information without data.

4 algorythms types (predicative analytics):
- information based
- similarity based
- probability based
- error based


# DDoS wargames
NS1 have DNS server whole world, anycast IP space.
Single attacker hits single POP.

Distributed DDoS: need granular visibility and mitigation tools.
DNS attacks:
- random label, asda32.foo.com
- reflection amd amplification (DNS and NTP), cpsc.gov
- flood/volumetruc. fill the pipes
- host enumeration, e.g. mail1.foo.com, mail2.foo.com
- ???

Dealing with attacks:
- visibility (packet inspect, alert)
- mitigation (filtering and rate limits)
- automation (failover POPs and learning traffic flow)

Attacker:
- prepare plan
- brings up attack infrastructure
- tries to throw defenders for a loop
- mutate attack over time
Defenders:
- exercise visibility tools
- exercise mitigation tools

Tools NS1 use:
- visibility
  - pktvisor, packetbeat, ntopng
  - ELK, Grafana
- attack infrastructure:
  - terraform (look examples in pic), cloud providers
  - custom controller scripts
- traffic generation
  - flamethrower: `./flame -v 3 -g randomlabel localhost 100 3 3`
  - hping3
  - pcap replay

During drills found:
- documentation tools syntax is wrong
- couldn't see some of attack traffic on dashboard
- attack caused an increase in our cache usage
- filed 6 bug tickets
- two different cheat sheets describing mitigation commands
- two mitigation command failed to work properly, throwing error messages

Tips for success:
- use *real data*
- attacks have to be realistic
- record session, *keep a diary*
- attacker should put the real time into planning,
  don't save it for last minute
- consistency of the drills is important
- keep it fun

Future:
- more tools, e.g. [drool](https://www.dns-oarc.net/tools/drool)
- more distribution, spoofing source ips
- introduce artificail constraints
- surprise unplanned attacks

https://github.com/ns1


# Make loadbalancing great again
https://traefik.io/
https://emilevauge.github.io/velocityLondon2017/

Microservices architecture is a new type
of services like a dynamic, configuration
change eberytime.
Reverse proxy make possible to manage traffic
into microservices, different types - web, api, dns, etc

As example reverse proxy might be is Nginx.
But we need upadte nginx confs with update of
our apps — it's not too easy (need sync).

Traefic has API from microserfices, when
microserfice might change treafik config.

Features:
- single binary
- backends: docker, swarm, consul, mesos, etcd, kubernetes...
- hot realoading
- load balancing: WRR, DRR
  (dynamical RR - more error on server less weight)
- circut breakers (unprocessed request'll be removed
  and the next req will be dropped)
- websockets
- HTTP/2

SSL intergrated with let's encrypt.
What's new:
- **session affinity**
- cluster mode
- swarm mode
- basic auth
- **healthcheck**
- prometheus metrics
- eureka, rancher
- `traefik bug` command
- dashboard filter
- basic auth frontend
- dynamodb
- GRPC
- custom headers, HSTS, error pages
- proxy protocol
- statd, datadog


# Continuous performance engineering (CPE)
Rapid delivery of change is critical to business survival.
Immediacy is the expectation for perfomance, but increaing
rate of change means more risk of breaking performance:
- no time for traditional testing
- problems cat remain dormant until next peak
- high cost to resolution if issues in prod

Do things early:
- local development:
  - design it right
  - code it right
  - perf test before build
- CI/CD server:
  - load test in build
  - run a big regression load test

CPE:
- look for the right stuff
- risk based approach
- shift left: "do things early"
- collaborate: do things together,
  even they proffessionals in different things
- automate: get tests in the pipeline
- analyse: automatic sign-off
- manege what's left

How to automate perf analysis: 
- identify metric using a Service Metric Framework
- compare to thresholds
- compare to baselines
- match expected patterns
- rules engine to give pass/fail

What have big value:
- constantly education and replicate to other team
- always write documentation
- give people correct tools

Cost of bug is growing from concept (> developing > testing >) to producion.


# Scaling up your monitoring
Metrics > Dashboard > Alerts

How to fix a lot of alerts and graphs?
[Refocus](https://github.com/salesforce/refocus) tools presented as graph with blocks.
We may use tree view or block, we've filters.

[Argus](https://github.com/salesforce/argus) is a time-series monitoring and alerting platform.:
- alerts
- dashboards
- events
- data
- manage via REST

[Pyplyn](https://github.com/salesforce/Pyplyn) is tool that extracts data from various sources,
transforms and gives it meaning, and sends it to other systems for consumption:
- process metrics from multi sources
- JSON data presentation
- scale better than ingress scripts


# Online performance analysis
[Strymon](http://strymon.systems.ethz.ch/) is a system for predictive datacenter
analytics and management via queryable online simulations:
- queries
- complex analytics
- simulations

Strymon is build on timely:
- arbitary cyclic dataflows
- logical timestamps
- async execution
- log latency
https://github.com/frankmcsherry/timely-dataflow

Hard to troubleshoot:
- dynamic workloads
- many tasks, activities, depencies
- bottleneck csuses are usually not isolated

Here and next talking about parallel execution in programm.
(Critical participation)[http://strymon.systems.ethz.ch/critical_path.html]:
estimation in the critical path of programm activity
CP = C(a) * aw / N(te-ts),
  C(a) - centrality: the number if paths this activity appears on
  aw - activity duration: edge weight
  N - total number paths between times t.end and t.start

Snailtrail CP-based summary:
- activity type, here we're talking
  about serializations and other
  params between workers
- straggler, what worker is bottleneck?
- opertator summary, make more parallelysm for
  most heavy operations
- communication channels


# Failover early
> By failing to prepare, you are preparing to fail.

- risk is the probability of something adverse to occur
- risk mitigation reduce the likehood or the consequeances if its occurrence
  - risk avoidance
  - risk limitataion
  - risk transference
  - risk acceptance
- mitigation cost <= cost of failure

Risk mitigation in DevOps:
- already monitor "adversities" on real time
- automate error handling
- call if constantly availability instead of risk mitigation

CDNs and failover:
- failover means serving an alternate response
  when an error condition is met

Request errors:
- CDNs captire errors that do not show up in your logs:
  - conn timeouts
  - network problems
- transform error to client friendly messages

It our origin location is dropped, then we're switching
our traffic into cloud storage at CDN point.
Also good idea — detect bot on your CDN location.

Possible errors:
- origin infrastruct error: CDN might not catch
  infrastructure, database, overflows, etc.

Best prectices:
- monitoring locally
- validate transactions at the back end
- keep checkout process state at the client
  and let it decide when to retry
- retry 3 times reqs in others endpoints if previous has failed
- set timeouts in each retry to reduce client w8 time
- control the release at the container lvl
- centralize your release at the pipeline lvl
  and use visualizations to digest it
- implementing a catch all strategy for all errors:
  create customize error pages, need ID on page

Failover *good* use cases:
+ network related (connectivity errors)
+ nullpotent reqs (GET)
+ improved efficiency iver origin:
  reduced latency, better capability

*Not good* use cases:
- infrastructure errors
- independent deplyment pipelines
- when request is non-idempotent (POST, PUT, DELETE)
- when cost or operational burden is higher than consequences


# Sketching data structures
Algoryth efficiency:
  error probability (more higher in probabilistyc algo)
  vs implementation complexity.

What are sketches:
- probabilistic algorithms
- summarize stream of data
- streaming data queries

How does algorythms work:
- calculate sream of the data
- make uniform distribution
- prepare data structure
- estimator process and got gauss +/- epsilon
Estimator may works with:
- bit-pattern (e.g. longest run of 0s)
- presence (is the bit set?)
- order statistics (smallest value seen so far)

**Bloom filter** is a space-efficient probabilistic data structure,
that is used to test whether an element is a member of a set.
+ denote read stories

Hash convert to *bitmap* and test it for presence.
But collisions are possible, error rates:

    (1 - e^(-k*n/m))^k

+ no false negatives
+ small memory footprint
- small false positive rate
- catn't retrieve or delete items

Extentions:
- counting
- count-min sketch

**Hyper Log Log**  s the observation that the cardinality of a multiset
of uniformly distributed random numbers can be estimated by calculating
the maximum number of leading zeros in the binary representation of each number in the set.
+ denote read stories
+ count unique words used

Algo used bit data patterns.
Error rates:

    log2 log2 Nmax + O(1)
    it's about 2% of errors

+ uniform distributions
+ log! log! space!
+ commutativity

**t-digest** construction algorithm uses a variant of 1-dimensional
k-means clustering to produce a data structure that is related to the Q-digest.
This t-digest data structure can be used to estimate quantiles or compute other rank statistics.
+ denote read stories
+ count unique words used
+ estimate percentiles

Algo is CDFs.
Error rate is non constant.

Pipelines in production
- veneur
- algebird
- google sawzall

Evaluating sketches:
- performance
- error rate
- distortion
- tuning

Brief list of others sketchers:
- skip lists
- frequency: count-min sketch, heavy hitters
- membership: bloom filters, cuckoo hashing
- cardinality: hyperloglog
- geometric data: coresets, locality-sensitive hashing

Error is tradeoff in algorithms.
Approximations are often good enough and a hell of a lot cheaper.