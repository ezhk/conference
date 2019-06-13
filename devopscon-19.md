# Deploy miscroservices like a Ninja with Istio service mesh
[Istio presentation](http://www.devopstrain.pro/istio/)  
[Register](bit.do/istio1), auth code: 5CPL  

Tiny running fix: microk8s.enable istio and answer «N».  

    $ kubectl describe pod -l=app=httpbin
    $ kubectl get pod httpbin-5c649bb49c-bfnvh -o yaml  

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
