# Silos
3 ways of devops:

- flow
- feedback
- continious learning

Operations as mutual service.

# Shell Ninja
[Source](https://github.com/ro14nd-talks/shell-ninja)  
[Bash testing](https://github.com/sstephenson/bats)  

## Why?

 - shell is eveywhere.
 - it's least common denominator
 - fast turnaround

## History

* sh by Ken Thomson
* year: 1971
* CLI
* IO redirection
* no variable, no pipelines

## Bournee Shell by Stephen Bourne

* 1979
* inludes if, for
* variables, environments

## C-sh

* 1979
* aliases
* history
* ~
* job command

## Bash by Brian Fox

* 1989

```
#!/bin/bash

# fail on every and undefined vars
set -eu
        
# fail in a single failed command in a pipeline
set -o pipefail
        
# save global script args
args=("$@")

hasflag() {
    # - is default value in ${1:-}
    local flag=${1}
    for arg in "${ARGS[@}"; do
        # make check safe with using "
        if [ "$arg" == "$flag" ]; then
            echo "true"
            return
        fi
    done
    echo "false"
}

# here we call true or false command,
#   also possible to use return 1 or return 0
if $(hasflag --devops); then
    echo "Hello, Berlin!"
fi

loadmodules() {
    cmd=${ARGS[0]}
    for module in $(basedir)/modules/*; do
        if [ "$(basedir)/modules/$cmd" = "$module" ]; then
            source $(basedir)/modules/${module}
            # call function run() in module
            eval "${module}::run"
        fi
    done
}

debug() {
    if $(hasflag --verbose -v); then
        export PS4='+ ${BASH_SOURCE[0]} ${LINE_NO}'
        set -x
    fi
}

# replaca strong
check_error() {
    echo "${msg//ERROR}"
    # logic here
}
```

Bash testing:

```
@test "addition using dc" {
  result="$(echo 2 2+p | dc)"
  [ "$result" -eq 4 ]
}

@test "No image driven" {
    runldeploy $(PATH)/canary -- deployment=bla"

    echo $output
    [ $status -eq 1 ]
    asseert_regexp "No new image"
    asseert_regexp "--image"
    asseert_regexp "canary"
}
```

# kubectl hacking
[source](https://github.com/loodse/kubectl-hacking)

[bash-completion](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)  
[powerline go](https://github.com/justjanne/powerline-go)  

# Open source pentesting and security analysis tools
[Search vulnerabilities](https://www.cvedetails.com/)  
[Exploit DB](https://www.exploit-db.com/)  
[Vulnerable demo application](https://github.com/cschneider4711/Marathon)  

Testing at first SQL injection:

* add ' into username or search field as example
* request get param add "OR 1=1"

Cross-site scripting:

* search zzzz`<u>zzzz</u>`zzz instead zzzzzzzzzz
* try to add <script>alert(1);</script>
* use `<script>` tags in create user page (`<title>` tag missing escaping, and add `"><script>alert(1)</script>` to close tag earlier)

Check image location — here `URL.php?photo=defailt.png`, next step — check modified URL: `URL.php?photo=../../../../../../etc/password`.  

Tools, that might help us:

* __OWASP zed attack proxy__ ([ZAP](https://www.zaproxy.org/)) and add proxy into browser — for analyze requests, look into alert, there possible to search different errors. ZAP also support active scan (in background this add SQL injection).  

Next step: using ZAP API in jenkins task, like a `curl -s URL/JSON/ascan...`. 

# DevSecOps
[checkmarx products](https://www.checkmarx.com/products/)

Main thread about how it's important to check your softvare for vulnerabilities:

- static analyze code
- penTesting (dynamic testing); here testing framework: telerik, ranonex, tricents

So full cycle:

- DEV:
  - design
  - code
  - check-in
  - build
  - test/QA
- OPS:
  - deploy
  - operate
  - monitor
    - run security checks, validate product state

Agile process:

- discovery
  - discovery
  - setup
  - user managment
- intergrate
  - project onboarding
  - SDLC intergration
  - reporting intergration
  - tracking intergration
  - training
- operate
  - hosting operation
  - results review
  - appSec helpdesk
- review
  - program mgt
  - KPI review

Operate and review are appSec levels.

# Hystrix vs Istio
[Circuit breaker](https://microservices.io/patterns/reliability/circuit-breaker.html)  
[Slides](https://www.exoscale.com/syslog/istio-vs-hystrix-circuit-breaker/)  

Options of circuit strategy:

- number of failed calls
- elapsed time strategy
  - fixed
  - doubling
  - somethong else
- number of successful calls

E-commerce will be the next as example.

Logging:

- fire-and-forget
  - async calls

Recommendations:

- sync req/response
- optional
- fallback options
  - display no recommendations
  - static recomendation set

Pricing:

- sync req/responce
- required
  - but better sell at a slightly ourdated price!
- fallback options

Payment:

- sync req/responce
- required
- fallback:
  - accept potentially bad payments

Strategies:

- black box
  - implementations
    - proxies
    - service meshes
  - fits
    - fail fast
- white box
  - libraries
    - hystrix
    - resilience 4J

Istio is about service mesh.  
  
Resilience 4J is a lighweight fault tolerance library (Netflix hystrix), but designed for Java8;  
features:

- circuit breaker
- rate limiting
- cache

# Seven security sins
[OWASP_Top_10-2017-ru.pdf](https://www.owasp.org/images/9/96/OWASP_Top_10-2017-ru.pdf)  

Threat modeling tools:

- microsoft threat modeling tool (TMT)
- diagram oriented tools
- etc.

Do it:

1. Whiteboard hacking (read «OWASP testing guide», [checklist](https://www.owasp.org/index.php/Testing_Checklist))
2. Security architecture (we need good crypto, leak DB is stupid; look at list of aws s3 leaks):
    - don't use CBC anymore
    - don't rely on java.util.Random for security relevant code use java.util.SecureRandom instead
    - use vaults
    - with microservices backend is not longer «fully trusted»: use HTTPS with PKI for backend request
    - CIS Benchmarks of your products ([link](https://www.cisecurity.org/cis-benchmarks/))
3. Security Automation — ZAP (or [FindBugs](http://findbugs.sourceforge.net/) plugin)
    - DAST (dynamic app sec testing)
    - SAST (static ..)
    - interactive IAST — only commecrial good ones
    - use git «secrets --scan-history»
4. Offensive penetration tests
5. Organozational changes
     - security role in every team
     - process responsibility
     - security information hub
     - security code review
6. Continious training:
     - knowing your enemy
     - gamification (capture the flag)
7. Breach preparation:
     - assume breach
     - logging & monitoring (use SIEM tools, Garbage-in-garbage-out, login failures, etc.)
     - incident simulation drills

# Self service
[link](https://www.rundeck.com/self-service)

# Prometheus
Timeline in a nutshell:
- initial commit 2012-11-24
- public announcement 2015-01-26
- join kubernetes and 2nd project hosted by the CNCF (cloud native computing foundation) 2016-05-09
- server v1.0 2016-07-18
- server v2.0 2017-11-08
- graduated within CNCF (2nd project and Kubernetes) 2018-08-09

# Workshops

## Deploy miscroservices like a Ninja with Istio service mesh
[Istio presentation](http://www.devopstrain.pro/istio/)  
[Register](bit.do/istio1), auth code: 5CPL  

Tiny running fix: microk8s.enable istio and answer «N».  

    $ kubectl describe pod -l=app=httpbin
    $ kubectl get pod httpbin-5c649bb49c-bfnvh -o yaml  

## CI/CD
[author](https://www.manuelpais.net/)  
[katacoda link](https://www.katacoda.com/manuelpais)  
[laboratory](https://katacoda.com/manuelpais/courses/treating-your-pipeline-as-a-product)  
[cd-pipeline](https://github.com/SkeltonThatcher/releasability-book/tree/master/examples/cd-pipeline)  
[pipeline-as-code](https://github.com/manupaisable/pipeline-as-code)

CI/CD evolution anti-patterns:
- CD retrofitted into CI server
  - CI = "using a CI server"
  - little thought given to pipeline design
  - tools in the chain get no love
- Familiarity first
  - Not using "best tool for the job"
  - Stretching tools beyong their nature
- "Not invented here" syndrome
  - hard to evolve home-grown tools
  - tooling team(s) fear changes
  - missing out on emergent good practices
- scaling first
  - not clear what "scaling" means
  - too much / too early focud in standartization

Over time issues arise...
- scalability
- intergration
- evolvability
- performance
- resilience
- bad coding/scripting practives

Sometimes delivart becomes bottleneck.  

Delivery (CI/CD) systems:
- CI tool
- CD tools (orhestration)
- plugins / third party tools
- pipeline definitions
- source repos
- artifact repos
- CI/CD infrastructure

Practives for resiliency:
- CI/CD immutable infrastructure
- blue-green deployments of updates
- pipeline configuration-as-code
- application infrastructure-as-code
- aggregated logging
- monitoring & alerting
- auto-scaling

Fast(er) feedback loops:
- build + unit tests under 5 minutes
- full pipeline run under 1 hour
- alert in deviations (above AND below)
- wait times highlighted

Practised for fast feedback:
- parallelize whenever possible
- dunamyc test environment / scheduling (fail quickly)
- measure activity times/deviations
- focud on flow efficiency (remove longest bottleneck first)

> «If your architecture doesn't fudamentally support Continious Delivery, then you're going nowhere.» 

Use small work items (batch size) for faster feedback loops.

Old view of the wirld: 

    |-------------------time between failures-------------------|  
                  |---------time to repair---------|  

New view of the wirld:

    |---------time between failures---------|  
              |--time to repair--|  

Pipeline design:
- intergration tests
- functional tests
- performance tests
- complience acceptance
- change approval board
- PROD

Tools:
- [jenkins](https://jenkins.io/) — builder
- [artifactory](https://jfrog.com/artifactory/) — storage container
- [SonarQube](https://www.sonarqube.org/) — static analyze code
- [cAdvisor](https://github.com/google/cadvisor) — resource monitoring
- [kibana](https://www.elastic.co/products/kibana) — UI log analyzer
