# Expedia
[Styx](https://github.com/Hotelsdotcom/styx)<br />

Web-resource is more harder every year,<br />
system bigger and need to recreate system.<br />

Evolution:
- akamai frontend
- akamai > nginx frontends
- styx: each node on jvm: non-blocking IO

**Styx** scheme:<br />
contains end-point router and a lot of plugins before it.<br />

Arch: Akamai switching data on node.<br />
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

Routing might use threshold breaker value.<br />
Bot treatments:
- recaptcha
- tarpit
- blackhole
- DOM mangling

Bot classification works in bot score and cassandra database data.<br />
If it's bot, cassandra call recaptcha.<br />

Consul using like an orchestrator engine and managing cloud-gate-proxy.<br />
Cloud-gate-proxy is styx. lobot is mechanism, that support persistence<br />
data for user, like a sharing if.<br />

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

Ideal scaling is linear, we've added node and get equal<br />
throughtput like a one node. It's not cheaped.<br />
Math:

    x(N) = lambda * N/1

But our cluster isn't perfect.<br />
We need to use parallel queues for tasks (e.g. scatter-gather).<br />
So our math:

    x(N) = lambda * N / (1 + omega * (N-1))
    where omega is fraction, that can't be done in parallel

Sometimes our workers have depencies on each other,<br />
it's add delay, do more work:<br />
N workers = N(N-1) pairs, and now we got:

    x(N) = lambda * N / (1 + omega * (N-1) + kappa * N * (N-1)),
    kappa - penalty coef (the slowest factor)

How does degradation works:<br />
[calc](https://www.desmos.com/calculator/3cycsgdl0b)

So scalability is:

    x(N) = lambda * N / (1 + omega * (N-1) + kappa * N * (N-1))

Load is concurrency - number of reqs in progress.<br />
Easy to measure: sum(latency)/interval
- MySQL: show status like 'Threads_running'
- Apache: active worker count

The USL can reveal the workload failure boundary approaching.<br />
This framework is for making systems look really bad.<br />

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
NS1 have DNS server whole world, anycast IP space.<br />
Single attacker hits single POP.<br />

Distributed DDoS: need granular visibility and mitigation tools.<br />
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
https://traefik.io/<br />
https://emilevauge.github.io/velocityLondon2017/<br />

Microservices architecture is a new type<br />
of services like a dynamic, configuration<br />
change eberytime.<br />
Reverse proxy make possible to manage traffic<br />
into microservices, different types - web, api, dns, etc<br />

As example reverse proxy might be is Nginx.<br />
But we need upadte nginx confs with update of<br />
our apps — it's not too easy (need sync).<br />

Traefic has API from microserfices, when<br />
microserfice might change treafik config.<br />

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

SSL intergrated with let's encrypt.<br />
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
Rapid delivery of change is critical to business survival.<br />
Immediacy is the expectation for perfomance, but increaing<br />
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
Metrics > Dashboard > Alerts<br />

How to fix a lot of alerts and graphs?<br />
[Refocus](https://github.com/salesforce/refocus) tools presented as graph with blocks.<br />
We may use tree view or block, we've filters.<br />

[Argus](https://github.com/salesforce/argus) is a time-series monitoring and alerting platform.:
- alerts
- dashboards
- events
- data
- manage via REST

[Pyplyn](https://github.com/salesforce/Pyplyn) is tool that extracts data from various sources,<br />
transforms and gives it meaning, and sends it to other systems for consumption:
- process metrics from multi sources
- JSON data presentation
- scale better than ingress scripts


# Online performance analysis
[Strymon](http://strymon.systems.ethz.ch/) is a system for predictive datacenter<br />
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

Here and next talking about parallel execution in programm.<br />
(Critical participation)[http://strymon.systems.ethz.ch/critical_path.html]:<br />
estimation in the critical path of programm activity<br />
**CP = C(a) * aw / N(te-ts)**

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

It our origin location is dropped, then we're switching<br />
our traffic into cloud storage at CDN point.<br />
Also good idea — detect bot on your CDN location.<br />

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
Algoryth efficiency:<br />
  error probability (more higher in probabilistyc algo)<br />
  vs implementation complexity.<br />

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

**Bloom filter** is a space-efficient probabilistic data structure,<br />
that is used to test whether an element is a member of a set.<br />
+ denote read stories<br />

Hash convert to *bitmap* and test it for presence.<br />
But collisions are possible, error rates:

    (1 - e^(-k*n/m))^k

+ no false negatives<br />
+ small memory footprint<br />
- small false positive rate<br />
- catn't retrieve or delete items<br />

Extentions:
- counting
- count-min sketch

**Hyper Log Log**  s the observation that the cardinality of a multiset<br />
of uniformly distributed random numbers can be estimated by calculating<br />
the maximum number of leading zeros in the binary representation of each number in the set.<br />
+ denote read stories<br />
+ count unique words used<br />

Algo used bit data patterns.<br />
Error rates:

    log2 log2 Nmax + O(1)
    it's about 2% of errors

+ uniform distributions<br />
+ log! log! space!<br />
+ commutativity<br />

**t-digest** construction algorithm uses a variant of 1-dimensional<br />
k-means clustering to produce a data structure that is related to the Q-digest.<br />
This t-digest data structure can be used to estimate quantiles or compute other rank statistics.
+ denote read stories
+ count unique words used
+ estimate percentiles

Algo is CDFs.<br />
Error rate is non constant.<br />

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

Error is tradeoff in algorithms.<br />
Approximations are often good enough and a hell of a lot cheaper.