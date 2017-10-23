# Docs
youtube, keywords "CNCF" and "kubernetes"
- kubernetes.io
- cncf.io

gcloud — app, that works w/ google.cloud (it's google cloud SDM)
kubelet — app, that running containers (like a docker container)

    $ kubectl config use-context minikube
    $ kubectl get service
    $ kubectl get nodes
    $ kubectl get pods
    $ kubectl config view

kubeadm work w/ DicitalOcean:
  https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
  - network pods: https://kubernetes.io/docs/concepts/cluster-administration/addons/
    as example Weave Scope have used

# Intro
Look at nodes:
$ minikube dashboard

Create redis:
> web-interface, image=redis
$ kubectl get pods
    NAME                     READY     STATUS    RESTARTS   AGE
    redis-2913962463-g7531   1/1       Running   0          6m
$ kubectl exec -ti redis-2913962463-g7531 -- redis-cli
    127.0.0.1:6379> set foo bar
        OK
    127.0.0.1:6379> get foo
        "bar"

$ minikube ssh

    $ docker ps // containers redis
    $ docker ps|grep redis
        f8d1d7f7b60f        redis@sha256:07e7b6cb753f8d06a894e22af30f94e04844461ab6cb002c688841873e5e5116                                                  "docker-entrypoint.sh"   2 minutes ago       Up 2 minutes                            k8s_redis_redis-2913962463-g7531_default_67646fec-b322-11e7-b288-080027c7beb2_0
        e09284209a83        gcr.io/google_containers/pause-amd64:3.0                                                                                       "/pause"                 3 minutes ago       Up 3 minutes                            k8s_POD_redis-2913962463-g7531_default_67646fec-b322-11e7-b288-080027c7beb2_0

**Pause container: one reason - grab IP address, and redis container share its IP w/ pause-app.**

> pod.yaml:
    apiVersion: v1
    kind: Pod
    metadata:
      name: foobar
    spec:
      containers:
      - name: foo
        image: redis

$ kubectl create -f pod.yaml
    pod "foobar" created
$ kubectl get pods
    NAME                     READY     STATUS    RESTARTS   AGE
    foobar                   1/1       Running   0          5m
    redis-2913962463-g7531   1/1       Running   0          12m
$ kubectl exec -ti foobar -- redis-cli

# Docker ENV
$ minikube docker-env
    export DOCKER_TLS_VERIFY="1"
    export DOCKER_HOST="tcp://192.168.99.100:2376"
    export DOCKER_CERT_PATH="/Users/ezhichek/.minikube/certs"
    export DOCKER_API_VERSION="1.23"
    // Run this command to configure your shell:
    // eval $(minikube docker-env)

$ minikube ssh
    $ docker run -d --name bar busybox sleep 3600
    $ docker exec -ti bar cat /etc/hosts
        127.0.0.1   localhost
        ::1 localhost ip6-localhost ip6-loopback
        fe00::0 ip6-localnet
        ff00::0 ip6-mcastprefix
        ff02::1 ip6-allnodes
        ff02::2 ip6-allrouters
        172.17.0.6  39a13ab7f2fb
    $ docker run -d --net=container:bar --name baz busybox sleep 3600 # share netspace, look at --net
    $ docker exec -ti baz cat /etc/hosts
        127.0.0.1   localhost
        ::1 localhost ip6-localhost ip6-loopback
        fe00::0 ip6-localnet
        ff00::0 ip6-mcastprefix
        ff02::1 ip6-allnodes
        ff02::2 ip6-allrouters
        172.17.0.6  39a13ab7f2fb
    $ docker exec -ti bar cat /etc/hosts
        127.0.0.1   localhost
        ::1 localhost ip6-localhost ip6-loopback
        fe00::0 ip6-localnet
        ff00::0 ip6-mcastprefix
        ff02::1 ip6-allnodes
        ff02::2 ip6-allrouters
        172.17.0.6  39a13ab7f2fb
  Same files!

# Create replicas
$ kubectl scale deployment redis --replicas 5
    deployment "redis" scaled
$ kubectl get pods
    NAME                     READY     STATUS              RESTARTS   AGE
    foobar                   1/1       Running             0          14m
    redis-2913962463-5pvbw   1/1       Running             0          5m
    redis-2913962463-7djqp   1/1       Running             0          5m
    redis-2913962463-g7531   1/1       Running             0          22m
    redis-2913962463-kfh1r   0/1       ContainerCreating   0          5m
    redis-2913962463-q96bp   0/1       ContainerCreating   0          5m

# Delete pods
$ kubectl delete pods redis-2913962463-tkmvj
    pod "redis-2913962463-tkmvj" deleted
$ kubectl get pods
    NAME                     READY     STATUS              RESTARTS   AGE
    foobar                   1/1       Running             0          16m
    redis-2913962463-5pvbw   1/1       Running             0          7m
    redis-2913962463-7djqp   1/1       Running             0          7m
    redis-2913962463-g7531   1/1       Running             0          23m
    redis-2913962463-kfh1r   1/1       Running             0          7m
    redis-2913962463-p6rp3   0/1       ContainerCreating   0          5m
    redis-2913962463-tkmvj   0/1       Terminating         0          5m
But containers are 5, because desired state is 5.

$ kubectl delete pods foobar
    pod "foobar" deleted
$ kubectl get pods
    NAME                     READY     STATUS        RESTARTS   AGE
    foobar                   0/1       Terminating   0          17m
    redis-2913962463-5pvbw   1/1       Running       0          8m
    redis-2913962463-7djqp   1/1       Running       0          8m
    redis-2913962463-g7531   1/1       Running       0          24m
    redis-2913962463-kfh1r   1/1       Running       0          8m
    redis-2913962463-p6rp3   1/1       Running       0          6m
$ kubectl get pods
    NAME                     READY     STATUS    RESTARTS   AGE
    redis-2913962463-5pvbw   1/1       Running   0          8m
    redis-2913962463-7djqp   1/1       Running   0          8m
    redis-2913962463-g7531   1/1       Running   0          24m
    redis-2913962463-kfh1r   1/1       Running   0          8m
    redis-2913962463-p6rp3   1/1       Running   0          6m
foobas have deleted!

Throw web-interface we can show that replica set = 5,
  and here we see based name (redis-2913962463).

> rs.yaml:
    apiVersion: extensions/v1beta1
    kind: ReplicaSet
    metadata:
      name: foo
    spec:
      replicas: 2
      template:
        metadata:
          name: foobar
        spec:
          containers:
          - name: barbar
            image: redis

We've got error:
$ kubectl create -f rs.yaml
    The ReplicaSet "foo" is invalid:
    * spec.selector: Required value
    * spec.template.metadata.labels: Invalid value: map[string]string(nil): `selector` does not match template `labels`

Fix it, labels identify resources.
> rs.yaml:
    apiVersion: extensions/v1beta1
    kind: ReplicaSet
    metadata:
      name: foo
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: velocity
      template:
        metadata:
          name: foobar
          labels:
            app: velocity
        spec:
          containers:
          - name: barbar
            image: redis

$ kubectl create -f rs.yaml
    replicaset "foo" created
$ kubectl get replicaset
    NAME               DESIRED   CURRENT   READY     AGE
    foo                2         2         2         5m
    redis-2913962463   5         5         5         34m
$ kubectl get pods|grep foo
    foo-3h659                1/1       Running   0          6m
    foo-r7lgs                1/1       Running   0          6m

## Labels
Get labels:
$ kubectl get pods -Lapp
    NAME                     READY     STATUS    RESTARTS   AGE       APP
    foo-3h659                1/1       Running   0          6m        velocity
    foo-r7lgs                1/1       Running   0          6m        velocity
    redis-2913962463-5pvbw   1/1       Running   0          18m       redis
    redis-2913962463-7djqp   1/1       Running   0          18m       redis
    redis-2913962463-g7531   1/1       Running   0          35m       redis
    redis-2913962463-kfh1r   1/1       Running   0          18m       redis
    redis-2913962463-p6rp3   1/1       Running   0          16m       redis
$ kubectl get pods -l app=velocity
    NAME        READY     STATUS    RESTARTS   AGE
    foo-3h659   1/1       Running   0          7m
    foo-r7lgs   1/1       Running   0          7m

Relabel:
$ kubectl label pods foo-r7lgs app=oscon --overwrite
    pod "foo-r7lgs" labeled
$ kubectl get pods -Lapp
    NAME                     READY     STATUS    RESTARTS   AGE       APP
    foo-3h659                1/1       Running   0          8m        velocity
    foo-c54wr                1/1       Running   0          5m        velocity
    foo-r7lgs                1/1       Running   0          8m        oscon
    redis-2913962463-5pvbw   1/1       Running   0          20m       redis
    redis-2913962463-7djqp   1/1       Running   0          20m       redis
    redis-2913962463-g7531   1/1       Running   0          37m       redis
    redis-2913962463-kfh1r   1/1       Running   0          20m       redis
    redis-2913962463-p6rp3   1/1       Running   0          19m       redis

## Recreate RS
Check, that recreated:
$ kubectl delete pods foo-3h659
    pod "foo-3h659" deleted
$ kubectl get pods
    NAME                     READY     STATUS              RESTARTS   AGE
    foo-22bmf                0/1       ContainerCreating   0          5m
    foo-c54wr                1/1       Running             0          10m
    foo-r7lgs                1/1       Running             0          13m
    redis-2913962463-5pvbw   1/1       Running             0          25m
    redis-2913962463-7djqp   1/1       Running             0          25m
    redis-2913962463-g7531   1/1       Running             0          41m
    redis-2913962463-kfh1r   1/1       Running             0          25m
    redis-2913962463-p6rp3   1/1       Running             0          23m

Delete RS:
$ kubectl get rs
    NAME               DESIRED   CURRENT   READY     AGE
    foo                2         2         2         14m
    redis-2913962463   5         5         5         42m
$ kubectl delete rs foo
    replicaset "foo" deleted
$ kubectl get rs
    NAME               DESIRED   CURRENT   READY     AGE
    redis-2913962463   5         5         5         42m
$ kubectl get pods
    NAME                     READY     STATUS        RESTARTS   AGE
    foo-c54wr                0/1       Terminating   0          11m
    foo-r7lgs                1/1       Running       0          14m
    redis-2913962463-5pvbw   1/1       Running       0          26m
    redis-2913962463-7djqp   1/1       Running       0          26m
    redis-2913962463-g7531   1/1       Running       0          43m
    redis-2913962463-kfh1r   1/1       Running       0          26m
    redis-2913962463-p6rp3   1/1       Running       0          24m

$ kubectl get pods -Lapp
    NAME                     READY     STATUS    RESTARTS   AGE       APP
    foo-r7lgs                1/1       Running   0          16m       oscon
    redis-2913962463-5pvbw   1/1       Running   0          28m       redis
    redis-2913962463-7djqp   1/1       Running   0          28m       redis
    redis-2913962463-g7531   1/1       Running   0          44m       redis
    redis-2913962463-kfh1r   1/1       Running   0          28m       redis
    redis-2913962463-p6rp3   1/1       Running   0          26m       redis
foo-r7lgs stay, because has other label.

Try again:
$ kubectl create -f rs.yaml
    replicaset "foo" created
$ kubectl get rs
    NAME               DESIRED   CURRENT   READY     AGE
    foo                2         2         1         5m
    redis-2913962463   5         5         5         46m
$ kubectl get pods
    NAME                     READY     STATUS    RESTARTS   AGE
    foo-gxvpl                1/1       Running   0          5m
    foo-l0stb                1/1       Running   0          5m
    foo-r7lgs                1/1       Running   0          17m
    redis-2913962463-5pvbw   1/1       Running   0          29m
    redis-2913962463-7djqp   1/1       Running   0          29m
    redis-2913962463-g7531   1/1       Running   0          46m
    redis-2913962463-kfh1r   1/1       Running   0          29m
    redis-2913962463-p6rp3   1/1       Running   0          27m
$ kubectl get pods -Lapp
    NAME                     READY     STATUS    RESTARTS   AGE       APP
    foo-gxvpl                1/1       Running   0          5m        velocity
    foo-l0stb                1/1       Running   0          5m        velocity
    foo-r7lgs                1/1       Running   0          17m       oscon
    redis-2913962463-5pvbw   1/1       Running   0          29m       redis
    redis-2913962463-7djqp   1/1       Running   0          29m       redis
    redis-2913962463-g7531   1/1       Running   0          46m       redis
    redis-2913962463-kfh1r   1/1       Running   0          29m       redis
    redis-2913962463-p6rp3   1/1       Running   0          28m       redis
$ kubectl delete rs foo
    replicaset "foo" deleted
$ kubectl get pods -Lapp
    NAME                     READY     STATUS        RESTARTS   AGE       APP
    foo-gxvpl                0/1       Terminating   0          5m        velocity
    foo-l0stb                0/1       Terminating   0          5m        velocity
    foo-r7lgs                1/1       Running       0          18m       oscon
    redis-2913962463-5pvbw   1/1       Running       0          29m       redis
    redis-2913962463-7djqp   1/1       Running       0          29m       redis
    redis-2913962463-g7531   1/1       Running       0          46m       redis
    redis-2913962463-kfh1r   1/1       Running       0          29m       redis
    redis-2913962463-p6rp3   1/1       Running       0          28m       redis

**All version of API — different group/object, a.g.**
$ kubectl api-versions|grep auth
    authentication.k8s.io/v1
    authentication.k8s.io/v1beta1
v1 and v1beta1 are never compatible.


# Proxy
$ kubectl proxy
    Starting to serve on 127.0.0.1:8001
$ curl 127.0.0.1:8001 | head
    {
    "paths": [
      "/api",
      "/api/v1",
      "/apis",
      "/apis/",
      "/apis/admissionregistration.k8s.io",
      "/apis/admissionregistration.k8s.io/v1alpha1",
      "/apis/apiextensions.k8s.io",
      "/apis/apiextensions.k8s.io/v1beta1",]}

$ curl 127.0.0.1:8001/api/v1 # list of API objects
Like a web-interface:
    {
      "name": "configmaps",
      "singularName": "",
      "namespaced": true,
      "kind": "ConfigMap",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "shortNames": [
        "cm"
      ]
    },

Replicaset extension:
$ curl 127.0.0.1:8001/apis/extensions/v1beta

Detailed info:
$ kubectl -v=9  get pods
    <skip>
    I1017 12:00:16.821301   42707 round_trippers.go:417] curl -k -v -XGET  -H "Accept: application/json" -H "User-Agent: kubectl/v1.8.1 (darwin/amd64) kubernetes/f38e43b" https://192.168.99.100:8443/api/v1/namespaces/default/pods
    <skip>
^^^ It's internal call, throught API:
$ curl 127.0.0.1:8001/api/v1/namespaces/default/pods

Or for pod:
$ kubectl get pods
    NAME                     READY     STATUS    RESTARTS   AGE
    foo-r7lgs                1/1       Running   0          30m
    redis-2913962463-5pvbw   1/1       Running   0          42m
    redis-2913962463-7djqp   1/1       Running   0          42m
    redis-2913962463-g7531   1/1       Running   0          59m
    redis-2913962463-kfh1r   1/1       Running   0          42m
    redis-2913962463-p6rp3   1/1       Running   0          41m
$ curl 127.0.0.1:8001/api/v1/namespaces/default/pods/foo-r7lgs

Delete pods:
$ kubectl -v=9 delete pods redis-2913962463-jz98t:
    <skip>
    I1017 12:04:41.006223   42894 round_trippers.go:417] curl -k -v -XDELETE  -H "Accept: application/json, */*" -H "User-Agent: kubectl/v1.8.1 (darwin/amd64) kubernetes/f38e43b" https://192.168.99.100:8443/api/v1/namespaces/default/pods/redis-2913962463-jz98t
    <skip>


# Namespaces
$ kubectl get namespaces
    NAME          STATUS    AGE
    default       Active    3d
    kube-public   Active    3d
    kube-system   Active    3d
$ kubectl get pods --all-namespaces
    NAMESPACE     NAME                          READY     STATUS    RESTARTS   AGE
    default       foo-r7lgs                     1/1       Running   0          34m
    default       redis-2913962463-5pvbw        1/1       Running   0          46m
    default       redis-2913962463-7djqp        1/1       Running   0          46m
    default       redis-2913962463-kfh1r        1/1       Running   0          46m
    default       redis-2913962463-p6rp3        1/1       Running   0          45m
    default       redis-2913962463-zgkdw        1/1       Running   0          7m
    kube-system   kube-addon-manager-minikube   1/1       Running   2          3d
    kube-system   kube-dns-1326421443-4r4fj     3/3       Running   6          3d
    kube-system   kubernetes-dashboard-h0032    1/1       Running   2          3d
kubernetes-dashboard as example.

Namespace is a scope for the names/resources.
$ kubectl create -f pod.yaml
    pod "foobar" created
$ kubectl create -f pod.yaml
    Error from server (AlreadyExists): error when creating "pod.yaml": pods "foobar" already exists
Use other namespace:
$ kubectl create -f pod.yaml  -n kube-system
    pod "foobar" created

Add namespace into file:
> pod.yaml:
    apiVersion: v1
    kind: Pod
    metadata:
      name: foobar
      namespace: foobar
    spec:
      containers:
      - name: foo
        image: redis

$ kubectl create ns foobar
    namespace "foobar" created
$ kubectl create -f pod.yaml
    pod "foobar" created
$ kubectl get pod -n foobar
    NAME      READY     STATUS    RESTARTS   AGE
    foobar    1/1       Running   0          6m

Create ns w/ file:
> ns.yaml:
    apiVersion: v1
    kind: Namespace
    metadata:
      name: test
$ kubectl create -f ns.yaml
    namespace "test" created
$ kubectl get ns
    NAME          STATUS    AGE
    default       Active    3d
    foobar        Active    7m
    kube-public   Active    3d
    kube-system   Active    3d
    test          Active    6m

Create quota:
$ kubectl create quota foobar --hard=pods=1 -n foobar
    resourcequota "foobar" created
$ kubectl get resourcequotas -n foobar
    NAME      AGE
    foobar    6m

Manifest resource quota:
$ kubectl get resourcequotas -o yaml -n foobar
    apiVersion: v1
    items:
    - apiVersion: v1
    kind: ResourceQuota
    metadata:
      creationTimestamp: 2017-10-17T11:10:15Z
      name: foobar
      namespace: foobar
      resourceVersion: "13363"
      selfLink: /api/v1/namespaces/foobar/resourcequotas/foobar
      uid: c0a54084-b32b-11e7-b288-080027c7beb2
    spec:
      hard:
        pods: "1"
    status:
      hard:
        pods: "1"
      used:
        pods: "1"
    kind: List
    metadata:
    resourceVersion: ""
    selfLink: ""

Create pod in quota namespace:
> modify pod.yaml:
    apiVersion: v1
    kind: Pod
    metadata:
    name: foo
    namespace: foobar
    spec:
    containers:
    - name: foo
      image: redis
$ kubectl create -f pod.yaml
    Error from server (Forbidden): error when creating "pod.yaml": pods "foo" is forbidden: exceeded quota: foobar, requested: pods=1, used: pods=1, limited: pods=1

Remove namespace in file:
$ kubectl create -f pod.yaml
    pod "foo" created

We can use network policy between different namespaces.
Detailed about namespaces: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/

From PDF:
    Namespaces are intended to isolate multiple groups/teams and give them access to a set of resources. Each namespace can have quotas. In future releases, we will have RBAC policies in namespaces


Try this https://github.com/kubernetes/examples/blob/master/guestbook/all-in-one/guestbook-all-in-one.yaml
$ kubectl get service
    NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
    frontend       ClusterIP   10.0.0.189   <none>        80/TCP     5m
    kubernetes     ClusterIP   10.0.0.1     <none>        443/TCP    3d
    redis-master   ClusterIP   10.0.0.37    <none>        6379/TCP   5m
    redis-slave    ClusterIP   10.0.0.20    <none>        6379/TCP   5m
$ kubectl exec -ti redis-master-106238132-1ljs3 -- redis-cli info | grep connected_slaves
    connected_slaves:2

$ kubectl get svc
    NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
    frontend       ClusterIP   10.0.0.189   <none>        80/TCP     8m
    kubernetes     ClusterIP   10.0.0.1     <none>        443/TCP    3d
    redis-master   ClusterIP   10.0.0.37    <none>        6379/TCP   8m
    redis-slave    ClusterIP   10.0.0.20    <none>        6379/TCP   8m
So try in browser:
  http://localhost:8001/api/v1/proxy/namespaces/default/services/frontend/

# Monitoring
$ minikube addons enable heapster
$ kubectl get pod --all-namespaces
    kube-system   influxdb-grafana-82wtx         0/2       ContainerCreating   0          10m # metrics are here
$ kubectl top pod -n default
    NAME                           CPU(cores)   MEMORY(bytes)
    foo-r7lgs                      0m           1Mi
    frontend-1768566195-tz39b      0m           11Mi
    foo                            1m           1Mi
    redis-2913962463-5pvbw         1m           1Mi
    redis-slave-3837281623-dpqrs   1m           2Mi
    foobar                         1m           1Mi
    redis-2913962463-7djqp         0m           1Mi
    redis-master-106238132-1ljs3   1m           2Mi
    redis-2913962463-p6rp3         1m           1Mi
    redis-2913962463-kfh1r         1m           2Mi
    frontend-1768566195-bxx3h      0m           7Mi
    redis-slave-3837281623-z3xxj   1m           3Mi
    redis-2913962463-zgkdw         0m           1Mi
    frontend-1768566195-55xm9      0m           7Mi

# Services
$ kubectl get svc
    NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
    frontend       ClusterIP   10.0.0.74    <none>        80/TCP     1m # external IP
    kubernetes     ClusterIP   10.0.0.1     <none>        443/TCP    1m
    redis-master   ClusterIP   10.0.0.128   <none>        6379/TCP   1m
    redis-slave    ClusterIP   10.0.0.7     <none>        6379/TCP   1m
$ kubectl get endpoints
    NAME           ENDPOINTS                         AGE
    frontend       172.17.0.10:80,172.17.0.8:80      2m
    kubernetes     10.0.2.15:8443                    2m
    redis-master   172.17.0.6:6379                   2m
    redis-slave    172.17.0.7:6379,172.17.0.9:6379   2m

$ kubectl scale deployments frontend --replicas=1
    deployment "frontend" scaled
Abstractions stay at one place:
$ kubectl get svc
    NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
    frontend       ClusterIP   10.0.0.74    <none>        80/TCP     3m
    kubernetes     ClusterIP   10.0.0.1     <none>        443/TCP    3m
    redis-master   ClusterIP   10.0.0.128   <none>        6379/TCP   3m
    redis-slave    ClusterIP   10.0.0.7     <none>        6379/TCP   3m

But now only one endpoint:
$ kubectl get endpoints
    NAME           ENDPOINTS                         AGE
    frontend       172.17.0.8:80                     3m
    kubernetes     10.0.2.15:8443                    3m
    redis-master   172.17.0.6:6379                   3m
    redis-slave    172.17.0.7:6379,172.17.0.9:6379   3m

Creating service:
> pod.yaml:
    apiVersion: v1
    kind: Pod
    metadata:
    name: foo
    label:
      app: game
    spec:
    containers:
    - name: foo
      image: runseb/2048
> service.yaml:
    apiVersion: v1
    kind: Service
    metadata:
    name: game
    spec:
    selector:
      app: game
    ports:
    - protocol: TCP
      port: 80

$ kubectl create -f pod.yaml
    pod "foo" created
$ kubectl create -f service.yaml
    service "game" created

$ kubectl get pods
    NAME                           READY     STATUS    RESTARTS   AGE
    foo                            1/1       Running   0          53s
$ kubectl get svc
    NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
    game           ClusterIP   10.0.0.115   <none>        80/TCP     53s

Look at it: localhost:8001/api/v1/proxy/namespaces/default/services/game/
$ kubectl get svc game -o yaml | grep -i link
  selfLink: /api/v1/namespaces/default/services/game

How can we do this w/o proxy:
$ kubectl edit svc game
    type: ClusterIP > type: NodePort
$ kubectl get svc
    NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
    game           NodePort    10.0.0.115   <none>        80:30044/TCP   19m
$ kubectl get endpoints
    NAME           ENDPOINTS                         AGE
    game           172.17.0.10:80                    23m
$ minikube service game

// Define ports in endpoints:
// $ kubectl label pods frontend-1768566195-pn4fx app=game --overwrite
"Accessing Services" in docs for detail.

# DNS
$ kubectl run busybox --image=busybox --command sleep 3600
    deployment "busybox" created
$ kubectl exec -ti busybox-2125412808-dlzl6 -- nslookup frontend
    Server:    10.0.0.10
    Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      frontend
    Address 1: 10.0.0.74 frontend.default.svc.cluster.local
$ kubectl get svc
    NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
    frontend       ClusterIP   10.0.0.74    <none>        80/TCP         33m

$ kubectl get pods --all-namespaces | grep -i dns
    kube-system   kube-dns-1326421443-0f77k      3/3       Running   0          34m
$ kubectl get svc --all-namespaces | grep -i dns
    kube-system   kube-dns               ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP       35m

# Deployments
$ kubectl delete deployments frontend redis-master redis-slave
    deployment "frontend" deleted
    deployment "redis-master" deleted
    deployment "redis-slave" deleted
$ kubectl get pods
    NAME                           READY     STATUS        RESTARTS   AGE
    busybox-2125412808-dlzl6       1/1       Running       0          13m
    foo                            1/1       Running       0          36m
    frontend-1768566195-pn4fx      1/1       Running       0          44m
    redis-slave-3837281623-5fzc1   0/1       Terminating   0          44m
    redis-slave-3837281623-plmms   0/1       Terminating   0          44m
$ kubectl delete pods frontend-1768566195-pn4fx foo
    pod "frontend-1768566195-pn4fx" deleted
    pod "foo" deleted
$ kubectl run ghost --image=ghost
    deployment "ghost" created
$ kubectl get rs
    NAME                 DESIRED   CURRENT   READY     AGE
    busybox-2125412808   1         1         1         14m
    ghost-1255708890     1         1         0         25s

$ kubectl scale deployment ghost --replicas=3
    deployment "ghost" scaled
$ kubectl get rs
    NAME                 DESIRED   CURRENT   READY     AGE
    busybox-2125412808   1         1         1         15m
    ghost-1255708890     3         3         3         1m

Now we've got new image, how to update?
$ kubectl set image deployment ghost ghost=ghost:0.17
    deployment "ghost" image updated

$ kubectl get pods
    NAME                       READY     STATUS             RESTARTS   AGE
    busybox-2125412808-dlzl6   1/1       Running            0          18m
    ghost-1255708890-3krrw     1/1       Running            0          2m
    ghost-1255708890-3vzcd     1/1       Running            0          1m
    ghost-1255708890-82rxl     1/1       Running            0          4m
    ghost-1255708890-w7wcf     1/1       Running            0          2m
    ghost-1777248768-0z5rr     0/1       ImagePullBackOff   0          41s
    ghost-1777248768-n2zbn     0/1       ErrImagePull       0          41s
What's happend?

Create new replica set:
$ kubectl get rs
    NAME                 DESIRED   CURRENT   READY     AGE
    busybox-2125412808   1         1         1         19m
    ghost-1255708890     4         4         4         5m
    ghost-1777248768     2         2         0         1m
1777248768 — hash by image version.
After 5 minutes docker stop trying.

Change version:
$ kubectl set image deployment ghost ghost=ghost:1.7
    deployment "ghost" image updated
$ kubectl get pods
    NAME                       READY     STATUS              RESTARTS   AGE
    busybox-2125412808-dlzl6   1/1       Running             0          22m
    ghost-1255708890-3krrw     1/1       Running             0          6m
    ghost-1255708890-3vzcd     1/1       Running             0          5m
    ghost-1255708890-82rxl     1/1       Running             0          7m
    ghost-1255708890-w7wcf     1/1       Running             0          6m
    ghost-3038644648-6153m     0/1       ContainerCreating   0          25s
    ghost-3038644648-fww41     0/1       ContainerCreating   0          25s

Show containers:
$ kubectl get pods -o json | jq -r .items[].spec.containers[0].image
    busybox
    ghost:1.7
    ghost:1.7
    ghost:1.7
    ghost:1.7
    ghost:1.7

$ kubectl get rs
    NAME                 DESIRED   CURRENT   READY     AGE
    busybox-2125412808   1         1         1         25m
    ghost-1255708890     0         0         0         11m
    ghost-1777248768     0         0         0         7m
    ghost-3038644648     5         5         5         4m
$ kubectl rollout history deployment ghost
    deployments "ghost"
    REVISION  CHANGE-CAUSE
    1         <none>
    2         <none>
    3         <none>
Each revision - each replicaset.

Set specific key:
$ kubectl annotate deployments ghost kubernetes.io/change-cause=foobar
    deployment "ghost" annotated
$ kubectl rollout history deployment ghost
    deployments "ghost"
    REVISION  CHANGE-CAUSE
    1         <none>
    2         <none>
    3         foobar

## Rollback
$ kubectl rollout undo deployment ghost --to-revision=3 # new version
    deployment "ghost" rolled back
$ kubectl get rs --watch
    NAME                 DESIRED   CURRENT   READY     AGE
    busybox-2125412808   1         1         1         28m
    ghost-1255708890     4         4         4         14m
    ghost-1777248768     0         0         0         11m
    ghost-3038644648     2         2         0         7m
    ghost-3038644648   2         2         1         7m
    ghost-1255708890   3         4         4         14m
    ghost-1255708890   3         4         4         14m
    ghost-3038644648   3         2         1         7m
    ghost-1255708890   3         3         3         14m
    ghost-3038644648   3         2         1         7m
    ghost-3038644648   3         3         1         7m
    ghost-3038644648   3         3         2         7m
    ghost-1255708890   2         3         3         14m
    ghost-3038644648   4         3         2         7m
    ghost-1255708890   2         3         3         14m
    ghost-1255708890   2         2         2         14m
    ghost-3038644648   4         3         2         7m
    ghost-3038644648   4         4         2         7m
$ kubectl get rs
    NAME                 DESIRED   CURRENT   READY     AGE
    busybox-2125412808   1         1         1         30m
    ghost-1255708890     0         0         0         15m
    ghost-1777248768     0         0         0         12m
    ghost-3038644648     5         5         5         8m
And now we've 5 replicas.

Have error:
$ kubectl delete deployment ghost
    deployment "ghost" deleted
$ kubectl get deployment
    NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    busybox   1         1         1            1           39m
Now all replicas stopped!

$ kubectl run ghost --image=runseb/2048 --record
    deployment "ghost" created
$ kubectl get deployment
    NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    busybox   1         1         1            1           41m
    ghost     1         1         1            0           2s
$ kubectl rollout history deployment ghost
    deployments "ghost"
    REVISION  CHANGE-CAUSE
    1         kubectl run ghost game --image=runseb/2048 --record=true
$ kubectl edit deployment ghost
  searching for RollingUpdate: maxSurge and maxUnavailable.

Also read about Canary release, as kubernetes do this:
  https://hackernoon.com/canary-release-with-kubernetes-1b732f2832ac
  https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments

Create a service:
$ kubectl expose deployment ghost --port 80 --type NodePort
    service "ghost" exposed
$ kubectl get svc | grep -i ghost
    ghost          NodePort    10.0.0.8     <none>        80:30323/TCP   54s
$ minikube service ghost
  Here redirect to browsers

# Replication controllers
  Do the same things like a replica set,
  deploy and rollover do w/ controller.
  !! It's old concept, use deployments !!

# Create own WordPress
$ kubectl run mysql --image=mysql:5.5 --env=MYSQL_ROOT_PASSWORD=root
$ kubectl logs mysql-1883654754-0126r | tail
    171017 14:22:43 InnoDB: Completed initialization of buffer pool
    171017 14:22:43 InnoDB: highest supported file format is Barracuda.
    171017 14:22:43 InnoDB: 5.5.58 started; log sequence number 1595675
    171017 14:22:43 [Note] Server hostname (bind-address): '0.0.0.0'; port: 3306
    171017 14:22:43 [Note]   - '0.0.0.0' resolves to '0.0.0.0';
    171017 14:22:43 [Note] Server socket created on IP: '0.0.0.0'.
    171017 14:22:43 [Warning] 'proxies_priv' entry '@ root@mysql-1883654754-0126r' ignored in --skip-name-resolve mode.
    171017 14:22:43 [Note] Event Scheduler: Loaded 0 events
    171017 14:22:43 [Note] mysqld: ready for connections.
    Version: '5.5.58'  socket: '/tmp/mysql.sock'  port: 3306  MySQL Community Server (GPL)
$ kubectl expose deployment mysql --port 3306
    service "mysql" exposed
$ kubectl exec -ti busybox-2125412808-dlzl6 -- nslookup mysql
    Server:    10.0.0.10
    Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      mysql
    Address 1: 10.0.0.142 mysql.default.svc.cluster.local
$ kubectl exec -ti mysql-1883654754-0126r -- mysql -p -uroot

$ kubectl run wordpress --image=wordpress --env WORDPRESS_DB_HOST=mysql --env WORDPRESS_DB_PASSWORD=root
    deployment "wordpress" created
$ kubectl expose deployment wordpress --port 80 --type NodePort
    service "wordpress" exposed
$ kubectl get endpoints | grep word
    wordpress      172.17.0.7:80     18m
$ kubectl get pods | grep word
    wordpress-2118809867-4lprn   1/1       Running            1          19m
$ minikube service wordpress

$ kubectl exec -ti  mysql-1883654754-0126r -- mysql -p -uroot
    mysql> use wordpress;
    mysql> select * from wp_users;
    +----+------------+------------------------------------+---------------+-------------------------+----------+---------------------+---------------------+-------------+--------------+
    | ID | user_login | user_pass                          | user_nicename | user_email              | user_url | user_registered     | user_activation_key | user_status | display_name |
    +----+------------+------------------------------------+---------------+-------------------------+----------+---------------------+---------------------+-------------+--------------+
    |  1 | Andrey     | $P$BDgQlV52WYIerAC.8ggvlpeiOIu98V. | andrey        | kiselevandrew@yandex.ru |          | 2017-10-17 14:41:02 |                     |           0 | Andrey       |
    +----+------------+------------------------------------+---------------+-------------------------+----------+---------------------+---------------------+-------------+--------------+
    1 row in set (0.00 sec)

Ta-daaam!
Full desc of service:
  https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/wordpress/wp.yaml

## Create key
$ kubectl create -f github/oreilly-kubernetes/manifests/wordpress/wp.yaml
    namespace "wordpress" created
    resourcequota "counts" created
    service "mysql" created
    service "wordpress" created
    ingress "wordpress" created
    deployment "mysql" created
    deployment "wordpress" created
$ kubectl create secret generic mysql --from-literal=password=foobar -n wordpress
    secret "mysql" created
$ kubectl get pods -n wordpress
    NAME                         READY     STATUS                          RESTARTS   AGE
    mysql-15637851-5ppqt         1/1       Running                         0          18m
    wordpress-1233699284-z3qqd   0/1       secrets "wordpress" not found   0          18m
$ kubectl create secret generic wordpress --from-literal=password=foobar -n wordpress
    secret "wordpress" created
$ kubectl get pods -n wordpress
    NAME                         READY     STATUS    RESTARTS   AGE
    mysql-15637851-5ppqt         1/1       Running   0          19m
    wordpress-1233699284-z3qqd   1/1       Running   0          19m

$ kubectl get secrets -n wordpress -o yaml | grep -i password
    password: Zm9vYmFy
$ echo Zm9vYmFy | base64 -D
    foobar
Password unencrypted! It's a big problem!

# Ingress Controller
It's an nginx proxy, driven by kubernets API:
dynamically reconfigure controller w/ nginx.
Continiously monitored endpoints.

$ minikube addons enable ingress
    ingress was successfully enabled
$ kubectl get pods -n kube-system
    nginx-ingress-controller-1wfrh   1/1       Running   0          18m
$ kubectl get pods nginx-ingress-controller-1wfrh -o yaml -n kube-system
    <skip>
    image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.14 # unstable build
    <skip>
    name: nginx-ingress-controller
    ports:
    - containerPort: 80
      hostPort: 80
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      protocol: TCP
    - containerPort: 18080
      hostPort: 18080
      protocol: TCP
    <skip>
$ kubectl get ingress -n wordpress
    NAME        HOSTS                             ADDRESS          PORTS     AGE
    wordpress   wordpress.192.168.99.100.nip.io   192.168.99.100   80        35m

Simple example:
  https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/ingress-controller/wordpress.yaml

$ curl 192.168.99.100
    default backend - 404
But wordpress.192.168.99.100.nip.io works!

Check scaling:
$ kubectl get pods -n wordpress
$ kubectl scale deployment wordpress -n wordpress --replicas 5
    deployment "wordpress2" scaled
$ kubectl exec -ti nginx-ingress-controller-1wfrh -n kube-system -- /bin/bash
    $ cat /etc/nginx/nginx.conf
        <skip>
        upstream wordpress-wordpress-80 {
            # Load balance algorithm; empty for round robin, which is the default
            least_conn;

            keepalive 32;

            server 172.17.0.21:80 max_fails=0 fail_timeout=0;
            server 172.17.0.13:80 max_fails=0 fail_timeout=0;
            server 172.17.0.20:80 max_fails=0 fail_timeout=0;
        }
        <skip>
            server {
            server_name wordpress.192.168.99.100.nip.io;
            listen 80;
            listen [::]:80;
        <skip>
$ kubectl get endpoints -n wordpress
    NAME        ENDPOINTS                                      AGE
    mysql       172.17.0.12:3306                               48m
    wordpress   172.17.0.13:80,172.17.0.20:80,172.17.0.21:80   48m


# Canary
  https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/canary/red-deploy.yaml
    <skip>
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: red
      volumes:
      - name: red
        configMap:
          name: red
    <skip>

# Multiple containers
    apiVersion: v1
    kind: Pod
    metadata:
    name: multi
    spec:
    containers:
      - name: busybox
        image: busybox
      - name: redis
        image: redis

$ kubectl exec -ti multi -c redis -- redis-cli
    127.0.0.1:6379>
$ kubectl get pods multi -o yaml
    <skip>
    name: busybox
    ready: false
    restartCount: 6
    state:
      waiting:
         message: Back-off 5m0s restarting failed container=busybox pod=multi_default(f2c46a54-b3dc-11e7-a319-0800275e6490)
         reason: CrashLoopBackOff
*It's make simplify to work between pods, like a localhost.*
https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/

# Basic schemes of Day 1 discuss
Pic.1 - summary scheme of architecture namespaces.
Pic.2 — Pod scheme.
Pic.3 — How Node works w/ containers.

# Volumes
Create pod:
$ kubectl run nginx --image=nginx
$ kubectl expose deployment nginx --port 80 --type NodePort
$ kubectl get pods --show-labels
    NAME                     READY     STATUS             RESTARTS   AGE       LABELS
    ghost-1255708890-bgk83   1/1       Running            0          30m       pod-template-hash=1255708890,run=ghost
    multi                    1/2       CrashLoopBackOff   8          20m       <none>
    nginx-4217019353-t8rt0   1/1       Running            0          2m        pod-template-hash=4217019353,run=nginx
$ kubectl get pods -Lrun
    NAME                     READY     STATUS             RESTARTS   AGE       RUN
    ghost-1255708890-bgk83   1/1       Running            0          30m       ghost
    multi                    1/2       CrashLoopBackOff   8          20m
    nginx-4217019353-t8rt0   1/1       Running            0          3m        nginx
$ kubectl get pods -l run=nginx
    NAME                     READY     STATUS    RESTARTS   AGE
    nginx-4217019353-t8rt0   1/1       Running   0          3m

$ kubectl get pods nginx-4217019353-t8rt0 -o yaml
    <skip>
    volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: default-token-8pf01
    <skip>
    volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: default-token-8pf01
          readOnly: true
    <skip>

$ kubectl get secrets
    NAME                  TYPE                                  DATA      AGE
    default-token-8pf01   kubernetes.io/service-account-token   3         51m
$ kubectl exec -ti nginx-4217019353-t8rt0 -- /bin/bash
root@nginx-4217019353-t8rt0:/# ls -la /var/run/secrets/kubernetes.io/serviceaccount/

*Volume shared space between containers.*
emptyDir and hostPath volumes are extremely easy to use (and understand). emptyDir is an empty directory that gets erased when the Pod dies (but survives container restarts), HostPath volumes survive Pod deletion.

Example:
https://github.com/sebgoa/oreilly-kubernetes/tree/master/manifests/volumes/volumes.yaml
$ kubectl create -f volumes.yaml
    pod "vol" created
Two containers used one volume — test.

$ kubectl get pods|grep vol
    vol                      2/2       Running            0          1m

How does it work:
$ kubectl exec -ti vol -c busy -- ls -la /busy
    total 8
    drwxrwxrwx    2 root     root          4096 Oct 18 09:04 .
    drwxr-xr-x    1 root     root          4096 Oct 18 09:04 ..
$ kubectl exec -ti vol -c busy -- touch /busy/busy
$ kubectl exec -ti vol -c busy -- ls -la /busy
    total 8
    drwxrwxrwx    2 root     root          4096 Oct 18 09:07 .
    drwxr-xr-x    1 root     root          4096 Oct 18 09:04 ..
    -rw-r--r--    1 root     root             0 Oct 18 09:07 busy
$ kubectl exec -ti vol -c box -- ls -la /box
    total 8
    drwxrwxrwx    2 root     root          4096 Oct 18 09:07 .
    drwxr-xr-x    1 root     root          4096 Oct 18 09:04 ..
    -rw-r--r--    1 root     root             0 Oct 18 09:07 busy

What about configmap:
$ kubectl create configmap foobar --from-file=volumes.yaml
    configmap "foobar" created

Update file:
manifests/volumes/configmap.yaml
-    - mountPath: /kubecon
+    - mountPath: /velocity

$ kubectl create -f configmap.yaml
    pod "cm-test" created
$ kubectl exec -ti cm-test -- ls -la /velocity/volumes.yaml
    lrwxrwxrwx    1 root     root            19 Oct 18 09:11 /velocity/volumes.yaml -> ..data/volumes.yaml
$ kubectl get cm
    NAME      DATA      AGE
    foobar    1         3m

# Presistent Volumes and Claims
Persistent Volumes are a storage abstraction, which provides a standard volume type for Pod: Claims. You define PersistentVolumes Objects backed by an underlying storage provider. Pods mount volumes based on claims they make on the persistent storage.

https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/volumes/pvc.yaml

$ kubectl create -f pvc.yaml
    persistentvolumeclaim "foobar" created
$ kubectl get pvc
    NAME      STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    foobar    Bound     pvc-5c2a7727-b3e5-11e7-a319-0800275e6490   1Gi        RWO            standard       4s

PV volume created automatically.
$ kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM            STORAGECLASS   REASON    AGE
    pvc-5c2a7727-b3e5-11e7-a319-0800275e6490   1Gi        RWO            Delete           Bound     default/foobar   standard                 3s

$ kubectl get pv pvc-5c2a7727-b3e5-11e7-a319-0800275e6490 -o yaml
    path: /tmp/hostpath-provisioner/pvc-5c2a7727-b3e5-11e7-a319-0800275e6490
It's inside minikibe.

https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/volumes/mysql.yaml
MySQL using this pv.
$ kubectl create -f mysql.yaml
    pod "data" created
$ kubectl get pods | grep data
    data                     1/1       Running   0          35s
$ kubectl exec -ti data -- mysql -uroot -p
    mysql>

$ minikube ssh
    $ ls -la /tmp/hostpath-provisioner/pvc-5c2a7727-b3e5-11e7-a319-0800275e6490/
        total 28692
        drwxrwxrwx 4  999  999     4096 Oct 18 09:23 .
        drwxr-xr-x 3 root root     4096 Oct 18 09:18 ..
        -rw-rw---- 1  999  999        2 Oct 18 09:23 data.pid
        -rw-rw---- 1  999  999  5242880 Oct 18 09:23 ib_logfile0
        -rw-rw---- 1  999  999  5242880 Oct 18 09:23 ib_logfile1
        -rw-rw---- 1  999  999 18874368 Oct 18 09:23 ibdata1
        drwx------ 2  999  999     4096 Oct 18 09:23 mysql
        drwx------ 2  999  999     4096 Oct 18 09:23 performance_schema

*kubernetes doesn't monitor volumes automatically.
So PV doesn't grow automatic.*

$ kubectl top node
    NAME       CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%
    minikube   216m         10%       1283Mi          67%

Get provisioner;
$ kubectl get storageclass
    NAME                 PROVISIONER
    standard (default)   k8s.io/minikube-hostpath

# Helm
is the first package manager for kubernetes.
Pic.4.

$ helm create oreilly
    Creating oreilly
$ ls -l oreilly/
    total 16
    -rw-r--r--  1 ezhichek  LD\Domain Users    85 18 окт 11:08 Chart.yaml
    drwxr-xr-x  2 ezhichek  LD\Domain Users    68 18 окт 11:08 charts
    drwxr-xr-x  7 ezhichek  LD\Domain Users   238 18 окт 11:08 templates
    -rw-r--r--  1 ezhichek  LD\Domain Users  1091 18 окт 11:08 values.yaml

Helm deploy charts.
$ kubectl get pods -n kube-system
    tiller-deploy-1936853538-8bj55   1/1       Running   0          16m
$ kubectl get deployment -n kube-system
    NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    kube-dns        1         1         1            1           2h
    tiller-deploy   1         1         1            1           17m

$ helm install stable/redis
$ REDIS_PASSWORD=$(kubectl get secret --namespace default hissing-manta-redis -o jsonpath="{.data.redis-password}" | base64 --decode)
$ kubectl run hissing-manta-redis-client --rm --tty -i     --env REDIS_PASSWORD=$REDIS_PASSWORD  --image bitnami/redis:3.2.9-r2 -- bash
$ kubectl get pods
    hissing-manta-redis-2930991512-xslw1          1/1       Running   0          18m
    hissing-manta-redis-client-3793932574-hj3xn   1/1       Running   0          17m

$ echo $REDIS_PASSWORD
    fxWY3qosRN
$ kubectl exec -ti hissing-manta-redis-2930991512-xslw1 -- redis-cli -a fxWY3qosRN
    127.0.0.1:6379>

$ helm repo list
    NAME    URL
    stable  https://kubernetes-charts.storage.googleapis.com
    local   http://127.0.0.1:8879/charts

$ curl https://kubernetes-charts.storage.googleapis.com/index.yaml -o index.yaml
index.yaml search redis:
  `engine: gotpl` # type of template
  and others

$ helm ls
    NAME            REVISION    UPDATED                     STATUS      CHART           NAMESPACE
    hissing-manta   1           Wed Oct 18 10:58:18 2017    DEPLOYED    redis-0.10.2    default
$ helm delete hissing-manta
    release "hissing-manta" deleted
$ helm ls
$ kubectl get pods # no redis here
    NAME                     READY     STATUS    RESTARTS   AGE
    cm-test                  1/1       Running   0          1h
    data                     1/1       Running   0          58m
    ghost-1255708890-bgk83   1/1       Running   0          2h
    nginx-4217019353-t8rt0   1/1       Running   0          1h
    vol                      2/2       Running   2          1h

Redis chart: https://github.com/kubernetes/charts/tree/master/stable/redis
  https://github.com/kubernetes/charts/tree/master/stable/redis/templates — manifest for make up the package
  https://github.com/kubernetes/charts/blob/master/stable/redis/values.yaml — .Values for templates

## Build your own package
$ cd oreilly-kubernetes/manifests
$ mkdir chart && cd chart
$ helm create velocity
    Creating velocity
$ ls velocity/
    Chart.yaml  charts      templates   values.yaml

// edit velocity/values.yaml: nginx > ghost
$ helm install ./velocity/
$ kubectl get pods | grep velocity
    willing-beetle-velocity-4000694782-zvrjk   0/1       Running   0          16m

Update replica count:
$ helm ls
    NAME            REVISION    UPDATED                     STATUS      CHART           NAMESPACE
    willing-beetle  6           Wed Oct 18 11:25:00 2017    DEPLOYED    velocity-0.1.0  default
$ helm upgrade --set replicaCount=3 willing-beetle ./velocity/
    RESOURCES:
    ==> v1/Service
    NAME                     CLUSTER-IP  EXTERNAL-IP  PORT(S)       AGE
    willing-beetle-velocity  10.0.0.49   <nodes>      80:31658/TCP  8m

    ==> v1beta1/Deployment
    NAME                     DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
    willing-beetle-velocity  3        3        3           0          8m

This pod have trouble w/ memory, comment limits in velocity/values.yaml and recreate:
$ kubectl get pods
    willing-beetle-velocity-4000694782-0jrzt   0/1       Terminating   4          19m
    willing-beetle-velocity-4000694782-p00nd   0/1       Terminating   4          19m

$ helm delete willing-beetle
$ helm install ./velocity/
$ helm ls
    NAME        REVISION    UPDATED                     STATUS      CHART           NAMESPACE
    worn-sponge 1           Wed Oct 18 11:28:06 2017    DEPLOYED    velocity-0.1.0  default
$ helm upgrade --set replicaCount=3 worn-sponge ./velocity/
    RESOURCES:
    ==> v1/Service
    NAME                  CLUSTER-IP  EXTERNAL-IP  PORT(S)       AGE
    worn-sponge-velocity  10.0.0.213  <nodes>      80:31520/TCP  23s

    ==> v1beta1/Deployment
    NAME                  DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
    worn-sponge-velocity  3        3        3           1          23s

$ kubectl get pods | grep velocity
    worn-sponge-velocity-986117239-nm003   1/1       Running   0          18m
    worn-sponge-velocity-986117239-pz04j   1/1       Running   0          18m
    worn-sponge-velocity-986117239-w1nd1   1/1       Running   0          18m

Depences resolv into https://github.com/kubernetes/charts/blob/master/stable/joomla/requirements.yaml — requirements.yaml
$ kubectl get deployment
    NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    ghost                  1         1         1            1           2h
    nginx                  1         1         1            1           2h
    worn-sponge-velocity   3         3         3            3           26m

Now we changing state:
$ kubectl scale deployment worn-sponge-velocity --replicas=10
    deployment "worn-sponge-velocity" scaled
$ kubectl get deployment
    NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    ghost                  1         1         1            1           2h
    nginx                  1         1         1            1           2h
    worn-sponge-velocity   10        10        10           3           26m

*How many replicas changed 3 or 10?*
$ helm upgrade worn-sponge --set image.tag=ghost:0.17 ./velocity/
$ kubectl get pods | grep velocity
    worn-sponge-velocity-584389613-6nxgl   0/1       InvalidImageName   0          16m
    worn-sponge-velocity-986117239-039ck   0/1       Terminating        2          18m
    worn-sponge-velocity-986117239-7t5xs   0/1       Terminating        2          18m
    worn-sponge-velocity-986117239-9fdp3   0/1       Terminating        2          18m
    worn-sponge-velocity-986117239-gmzzs   0/1       Terminating        2          18m
    worn-sponge-velocity-986117239-nm003   0/1       Terminating        2          28m
    worn-sponge-velocity-986117239-pz04j   0/1       Terminating        2          28m
    worn-sponge-velocity-986117239-qb1nf   0/1       Terminating        2          18m
    worn-sponge-velocity-986117239-th9qm   0/1       Terminating        2          18m
    worn-sponge-velocity-986117239-w1nd1   0/1       Terminating        2          28m
    worn-sponge-velocity-986117239-zj92g   0/1       Terminating        2          18m

We corrupt our deplyment.
Delete all helms:
$ helm delete worn-sponge
    release "worn-sponge" deleted

# Python client
Look the PDF.
Examples U might find here:
  https://github.com/kubernetes-incubator/client-python/blob/master/examples/create_deployment.py
  https://github.com/sebgoa/oreilly-kubernetes/blob/master/scripts/create_pod.py

# Custom Resource Defintions
Pic.5.
Kubernetes lets you add your own API objects. Kubernetes can create a new custom API endpoint and provide CRUD operations as well as watch API.
https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/crd/database.yml

$ kubectl create -f database.yml
    customresourcedefinition "databases.foo.bar" created
$ kubectl get customresourcedefinition
    NAME                AGE
    databases.foo.bar   16m

Create DB:
    https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/crd/db.yml
$ kubectl create -f db.yml
    database "my-new-db" created
$ kubectl get db
    NAME        AGE
    my-new-db   16m
$ kubectl label db my-new-db super=crazy
    database "my-new-db" labeled
$ kubectl get db -Lsuper
    NAME        AGE       SUPER
    my-new-db   17m       crazy

Out API available throught proxy:
$ curl 127.0.0.1:8001/apis/foo.bar/v1/databases

Need to write controller, that we can make operations like:
- create
- delete
- create endpoint (?)

# Function deploy
Pic.6.

http://kubeless.io/install/
$ kubectl create ns kubeless
    namespace "kubeless" created
$ curl -sL https://github.com/kubeless/kubeless/releases/download/v0.2.3/kubeless-v0.2.3.yaml | kubectl create -f -
    customresourcedefinition "functions.k8s.io" created
    statefulset "kafka" created
    service "zookeeper" created
    service "kafka" created
    service "zoo" created
    statefulset "zoo" created
    deployment "kubeless-controller" created
    serviceaccount "controller-acct" created
    service "broker" created
$ kubectl get pods -n kubeless
    NAME                                   READY     STATUS              RESTARTS   AGE
    kafka-0                                0/1       ContainerCreating   0          2m
    kubeless-controller-2704426935-91cc8   0/1       ContainerCreating   0          2m
    zoo-0                                  0/1       ContainerCreating   0          2m
$ kubectl get customresourcedefinitions
    NAME                AGE
    databases.foo.bar   1h
    functions.k8s.io    2m

Apply HTTP triggered functions from
  http://kubeless.io/examples/

$ kubectl get functions
    NAME          AGE
    post-python   6s

$ kubectl get functions post-python -o yaml
$ kubectl get pods | grep pyth
    post-python-2365099997-ngk1g   1/1       Running   0          1m
$ kubeless function call post-python --data '{"echo":"velocoty"}'
    Connecting to function...
    Forwarding from 127.0.0.1:30000 -> 8080
    Forwarding from [::1]:30000 -> 8080
    Handling connection for 30000
    {"echo": "velocoty"}

$ kubectl get svc | grep python
    post-python   ClusterIP   10.0.0.131   <none>        8080/TCP         6m
$ kubectl get cm | grep -i python
    post-python   3         6m

$ kubectl get pods post-python-2365099997-ngk1g -o yaml
    volumeMounts:
        - mountPath: /kubeless
        name: post-python

# Security
*Namespaces do not give network isolations*, they just give you naming isolation for resources.
By default kubelet don't run privileged container and using priveleged is dangerous.
*Don't pus secrets in secrets, because base64 only included.*

Solutions:
- Network Policies
- Pod Security Policies
- Secret encryption at rest in 1.7 + sealed-secrets

Example:
https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/security/test.yaml
$ kubectl create -f test.yaml
    pod "redis" created
$ kubectl get pods -w
    NAME                           READY     STATUS                                                  RESTARTS   AGE
    redis                          0/1       container has runAsNonRoot and image will run as root   0          14s
95% containers runs as the root user, and it's too.

Edit security/test.yaml like a file:
    apiVersion: v1
    kind: Pod
    metadata:
      name: pawn
    spec:
      containers:
      - image: busybox
        command:
        - sleep
        - "3600"
        imagePullPolicy: IfNotPresent
        name: pawns
        securityContext:
          privileged: true
      hostNetwork: true
      hostPID: true
      restartPolicy: Always

$ kubectl create -f test.yaml
  pod "pawn" created

Detail documentation about PSP (poe security policies):
  https://docs.bitnami.com/kubernetes/how-to/secure-kubernetes-cluster-psp/

# RBAC (roll based access control)
Quota include in admisstion control (e.g. PSP).
Pic.7.

$ minikube stop
$ minikube start --extra-config=apiserver.Authorization.Mode=RBAC

$ openssl genrsa -out employee.key 2048
$ openssl req -new -key employee.key -out employee.csr -subj "/CN=employee/O=bitnami"
$ export CA_LOCATION=~/.minikube/
$ openssl x509 -req -in employee.csr -CA ${CA_LOCATION}/ca.crt -CAkey ${CA_LOCATION}/ca.key -CAcreateserial -out employee.crt -days 500

$ kubectl config set-credentials employee --client-certificate=employee.crt --client-key=employee.key
    User "employee" set.
$ kubectl config set-context employee-context --cluster=minikube --namespace=office --user=employee
    Context "employee-context" created.

 *Deny:*
$ kubectl --context=employee-context get pods
Error from server (Forbidden): User "employee" cannot list pods in the namespace "office". (get pods)

> role.yaml:
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      namespace: office
      name: deployment-manager
    rules:
      - apiGroups: ["", "extensions", "apps"]
        resources: ["deployments", "replicasets", "pods"]
        verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

> deployment.yaml:
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: deployment-manager-binding
      namespace: office
    subjects:
    - kind: User
      name: employee
      apiGroup: ""
    roleRef:
      kind: Role
      name: deployment-manager
      apiGroup: ""


$ kubectl create -f role.yaml
    role "deployment-manager" created
$ kubectl create -f deployment.yaml
    rolebinding "deployment-manager-binding" created

*Allow:*
$ kubectl --context=employee-context get pods -n office
    NAME                     READY     STATUS    RESTARTS   AGE
    redis-3073568142-qtnnr   1/1       Running   0          7m

## Network policies
https://github.com/ahmetb/kubernetes-networkpolicy-tutorial

# Sealed Secrets
https://github.com/bitnami/sealed-secrets

Trouble desc:
$ kubectl get secrets
    NAME                  TYPE                                  DATA      AGE
    default-token-8pf01   kubernetes.io/service-account-token   3         6h
$ kubectl create secret generic unsecure --from-literal=root=root
    secret "unsecure" created
$ kubectl get secrets unsecure -o yaml|grep root
    root: cm9vdA==
$ echo cm9vdA== | base64 -D
    root

Run sealed-secret:
$ release=`curl --silent "https://api.github.com/repos/bitnami/sealed-secrets/releases/latest" | sed -n 's/.*"tag_name": *"\([^"]*\)".*/\1/p')])"')`
$ kubectl create -f https://github.com/bitnami/sealed-secrets/releases/download/$release/controller.yaml
$ kubectl get pods --all-namespaces | grep sealed
    kube-system   sealed-secrets-controller-665256659-wpfmr   1/1       Running                         0          12m
$ kubectl create secret generic velocity -o json --dry-run --from-literal=foo=bar
    {
        "kind": "Secret",
        "apiVersion": "v1",
        "metadata": {
            "name": "velocity",
            "creationTimestamp": null
        },
        "data": {
           "foo": "YmFy"
        }
    }

$ kubeseal <secret.json
    {
      "kind": "SealedSecret",
      "apiVersion": "bitnami.com/v1alpha1",
      "metadata": {
        "name": "velocity",
        "namespace": "default",
        "creationTimestamp": null
      },
      "spec": {
        "data": "AgC0wM0UtR8+LXZllqnS5Ce36kcKv64aDbJ2sVLLAksle/oBcBf3Ov5Xl0s1vIWDfOnryd11jy99d/k6by0+ckyjVnXF1rWwGmpIzOxlXPJ6MVj51WC7Vdp7O6asyRynWMVEVvtUeYfYAeREyedg3IAkx4PcKhn1o7Q5gIyg9lZag3EEHCbIGV5WhGUzdX65IActvm4cuCDEZ08Bi3fOQMbb0l2TWfJ7tVgrMlOg0tCyArfZ8dpxiE/Nzv0jWjmxPSxkC3//Cvxu+9p9+ARne48vqnaNbhjK5zKCMlDG7NHsW3Sb2ua/kyplbrpFP9VQFEViXJka3t84NcPpT6ryDAMnIfEzm0vpzST555kSP0YwOJukWx96uK+XSKZQWP830IXDwEQ2xtQ/9bhwpqnec0jGCDYTz/X8mk+GVnuQABsGrtnvZ28Rqe+SqmiaIJOW7Lbhf7ebFVCXwZCVo4d5ujW5vhfjrl5S5ksY6YMY55AfeiI8uxvuPn17Btv6rLtzgoTwwtN09YLY+fAhbIVreo94ZALBqrMlnrSiO/2jCawnc4Htw5FZcsOAOnOPofRmakxiOM5LVLuQjxTLSZnETrKR3JUpzl7G6pGuaVlWVlQN0Mo1USdJ/R+ZHDCHiYrfkg5sB6VEyTbYckK3rCUpnSrKADQxM8CY28ZtQ9jnFOZ2SmH008+Sdl+Ncps2BKa67Ll32S7WJG6NQKnqhVNaEKSrZrGF41pAT2KQ2kCZ+qDcJGSmrr2RIDYvfS1hFaxEdnyU0EGvIgB/t5fT1XuHOXW8KSuPdJ8vX6wjgE+U4YAVeij7CAEJhy/PhAjTOTIjVuLlvQ5hmvbeff0NKojHk4t/I3xtIq3VomIDJoTTg2L9YK6W5FLIvryw"
      }
    }

$ kubeseal <secret.json >mysealedsecret.json
$ kubectl create -f https://github.com/bitnami/sealed-secrets/releases/download/$release/sealedsecret-tpr.yaml
    thirdpartyresource "sealed-secret.bitnami.com" created
$ kubectl create -f https://github.com/bitnami/sealed-secrets/releases/download/$release/sealedsecret-crd.yaml
    customresourcedefinition "sealedsecrets.bitnami.com" created
$ kubectl create -f mysealedsecret.json
    sealedsecret "velocity" created

Profit:
$ kubectl get secret velocity
    NAME       TYPE      DATA      AGE
    velocity   Opaque    1         13m
$ kubectl get secrets velocity -o yaml
    apiVersion: v1
    data:
      foo: YmFy
    kind: Secret
    metadata:
      creationTimestamp: 2017-10-18T14:34:46Z
      name: velocity
      namespace: default
      ownerReferences:
      - apiVersion: bitnami.com/v1alpha1
        controller: true
        kind: SealedSecret
        name: velocity
        uid: 7d5048cf-b411-11e7-950f-0800275e6490
      resourceVersion: "38108"
      selfLink: /api/v1/namespaces/default/secrets/velocity
      uid: 7d545fef-b411-11e7-950f-0800275e6490
    type: Opaque

# Scheduling Pods
https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

> foobar.yaml:
    apiVersion: v1
    kind: Pod
    metadata:
      name: foobar
        namespace: default
        spec:
          containers:
              - image: ghost
                name: ghost
          nodeSelector:
            disk: ssd
*look at nodeselector*

$ kubectl create -f foobar.yaml
    pod "foobar" created

$ kubectl get pods
    NAME                           READY     STATUS    RESTARTS   AGE
    foobar                         0/1       Pending   0          13m
$ kubectl label node minikube disk=ssd
    node "minikube" labeled
$ kubectl get pods | grep foobar
    foobar                         1/1       Running   0          14m

## Custom scheduler
> foobar2.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: foobar2
    spec:
      containers:
      - image: redis
        name: redis
      schedulerName: foobar

> cust_sched.json
    {
      "apiVersion": "v1",
      "kind": "Binding",
      "metadata": {
        "name": "foobar2"
      },
      "target": {
        "apiVersion": "v1",
        "kind": "Node",
        "name": "minikube"
      }
    }

$ kubectl create -f foobar2.yaml
    pod "foobar2" created

$ kubectl get pods
    NAME                           READY     STATUS    RESTARTS   AGE
    foobar2                        0/1       Pending   0          12m

Create our binding;
$ curl -H "Content-Type:application/json" -X POST --data @cust_sched.json localhost:8001/api/v1/namespaces/default/pods/foobar-sched/binding/
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {},
      "status": "Success",
      "code": 201
    }

$ kubectl get pods | grep foobar2
    foobar2                        1/1       Running   0          17m

Simple python sched:
  https://github.com/sebgoa/oreilly-kubernetes/blob/master/manifests/scheduling/scheduler.py

# Monitoring
https://github.com/sebgoa/oreilly-kubernetes/tree/master/monitoring

Prometheus:
$ kubectl apply -f monitoring-namespace.yaml
$ kubectl apply -f prometheus-rbac.yaml
$ kubectl apply -f prometheus-config.yaml
$ kubectl apply -f prometheus-statefulset.yaml
$ kubectl apply -f prometheus-svc.yaml
$ kubectl apply -f node-exporter-daemonset.yaml
$ kubectl apply -f node-exporter-svc.yam

Grafana:
$ kubectl create -f grafana-statefulset.yaml
$ kubectl create -f grafana-svc.yaml

Pic.8.
Cadviser collect metrics:
  http://192.168.99.100:4194/containers/
$ minikube addons list | grep -i heapster
    - heapster: enabled

# Upgrade
  https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/#upgrading-a-cluster

to make node unscailable/unschedulable:
$ kubectl drain --help
and revert this:
$ kubectl uncordon --help