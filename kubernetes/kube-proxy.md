# kube-proxy

`kube-proxy` is a Kubernetes node component that makes **Service IP / ClusterIP** able to communicate with a Pod, so that pods can communicate with each other with a single virtual IP.

It watches **Services** and **EndpointSlices** from the Kubernetes API server.

When something changes, it creates or updates node-level rules, usually with `iptables`, so traffic can be forwarded to a Pod.

---

### Example Cluster

To understand it, let's use a three-node cluster:

```text
Node 1
Role: control-plane
IP:   172.16.0.2

Node 2
Role: worker-01
IP:   172.16.0.3

Node 3
Role: worker-02
IP:   172.16.0.4
```

### Check Available Nodes

```bash
laborant@cplane-01:~$ kubectl get nodes -o wide
NAME        STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
cplane-01   Ready    control-plane   9m2s    v1.36.0   172.16.0.2    <none>        Ubuntu 24.04.4 LTS   6.1.167 (amd64)   containerd://2.2.3
node-01     Ready    <none>          8m50s   v1.36.0   172.16.0.3    <none>        Ubuntu 24.04.4 LTS   6.1.167 (amd64)   containerd://2.2.3
node-02     Ready    <none>          8m49s   v1.36.0   172.16.0.4    <none>        Ubuntu 24.04.4 LTS   6.1.167 (amd64)   containerd://2.2.3
```

And also, let's check their CIDRs:

```bash
laborant@cplane-01:~$ kubectl describe node node-01 | grep -i PodCIDR:
PodCIDR:                      10.244.1.0/24
```

```bash
laborant@cplane-01:~$ kubectl describe node node-02 | grep -i PodCIDR:
PodCIDR:                      10.244.2.0/24
```

### A New Namespace

First, let's create a new namespace so that we can see our related namespace's `iptables` and how things will change.

```bash
laborant@cplane-01:~$ kubectl create namespace kproxy-lab
namespace/kproxy-lab created
```

```bash
laborant@cplane-01:~$ kubectl get all -n kproxy-lab
No resources found in kproxy-lab namespace.
laborant@cplane-01:~$
```

### Deploy Basic Application

We will use `podinfo` for both frontend and backend, and we will deploy 2 sets of backend and 1 frontend in the `kproxy-lab` namespace.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: kproxy-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
      - name: podinfo
        image: ghcr.io/stefanprodan/podinfo:latest
        ports:
        - containerPort: 9898
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: kproxy-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: netshoot
        image: nicolaka/netshoot:latest
        command: ["sleep", "infinity"]
EOF
```

Let's check the status of pods:

```bash
laborant@cplane-01:~$ kubectl get pods -n kproxy-lab -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
backend-api-7fd9cffcc4-mk9lc   1/1     Running   0          11m   10.244.2.2   node-02   <none>           <none>
backend-api-7fd9cffcc4-qnqk2   1/1     Running   0          11m   10.244.1.3   node-01   <none>           <none>
frontend-55567b655d-9j8w4      1/1     Running   0          11m   10.244.1.2   node-01   <none>           <none>
laborant@cplane-01:~$
```

Now we get 2 pods for backend:

```text
backend-api-7fd9cffcc4-mk9lc 10.244.2.2 # running on node-02
backend-api-7fd9cffcc4-qnqk2 10.244.1.3 # running on node-01
```

And 1 for frontend:

```text
frontend-55567b655d-9j8w4 10.244.1.2 # running on node-01
```

From the frontend pod (frontend-55567b655d-9j8w4), we curl the backend; then we should be able to curl them, as they both are running.

```bash
laborant@cplane-01:~$ kubectl exec -n kproxy-lab frontend-55567b655d-9j8w4 -- curl -s http://10.244.2.2:9898 | grep hostname
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
laborant@cplane-01:~$ kubectl exec -n kproxy-lab frontend-55567b655d-9j8w4 -- curl -s http://10.244.1.3:9898 | grep hostname
  "hostname": "backend-api-7fd9cffcc4-qnqk2",
laborant@cplane-01:~$
```

NB: Though backends are in two different nodes, still we can connect to them. CNI/Flannel is making the magic for us.

So it's confirmed that frontend can talk with backend, but as we know, there are some issues:

1. This is actually communicating with one pod. If the pod (backend) crashes, then communication is lost.
2. Pods are not static. They can be deleted and recreated, and also their IP changes.
3. It is not feasible to give frontend all backend pod IPs because sometimes there can be a hundreds of pods running for backend.

For example, if we delete a backend pod, a new pod starts and its IP changes:

```bash
laborant@cplane-01:~$ kubectl delete pod backend-api-7fd9cffcc4-qnqk2 -n kproxy-lab
pod "backend-api-7fd9cffcc4-qnqk2" deleted from kproxy-lab namespace
```

```bash
laborant@cplane-01:~$ kubectl get pods -n kproxy-lab -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
backend-api-7fd9cffcc4-6kk45   1/1     Running   0          9s    10.244.1.4   node-01   <none>           <none>
backend-api-7fd9cffcc4-mk9lc   1/1     Running   0          21m   10.244.2.2   node-02   <none>           <none>
frontend-55567b655d-9j8w4      1/1     Running   0          21m   10.244.1.2   node-01   <none>           <none>
laborant@cplane-01:~$
```
A new pod (backend-api-7fd9cffcc4-6kk45) just started 9s ago, and the IP also changed.

New IP: `10.244.1.4`

If we try to connect that previous pod again it will be failed
```
laborant@cplane-01:~$ kubectl exec -n kproxy-lab frontend-55567b655d-9j8w4 -- curl --max-time 5 -s http://10.244.1.3:9898
command terminated with exit code 7
laborant@cplane-01:~$ 
```

`10.244.1.3` is not here anymore.

So, to solve this issue, we will create a ClusterIP Service, which will give the backend Pods one stable internal IP/name.

But before creating that, let's check the `iptables`:

```bash
laborant@cplane-01:~$ sudo iptables-save -t nat > /tmp/before-service.txt
laborant@cplane-01:~$ sudo iptables-save -t nat | grep kproxy-lab
laborant@cplane-01:~$
```
```bash
laborant@node-01:~$ sudo iptables-save -t nat > /tmp/before-service.txt
sudo iptables-save -t nat | grep kproxy-lab
laborant@node-01:~$ 
```
```bash
laborant@node-02:~$ sudo iptables-save -t nat > /tmp/before-service.txt
sudo iptables-save -t nat | grep kproxy-lab
laborant@node-02:~$ 
```

All `iptables` in all 3 nodes are empty for this namespace (`kproxy-lab`).

Also if we check the End point Slices we will find it empty
```bash
laborant@cplane-01:~$ kubectl get endpointslices -n kproxy-lab -o wide
No resources found in kproxy-lab namespace.
```

Now let's add a cluster ip to backend:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: kproxy-lab
spec:
  type: ClusterIP
  selector:
    app: backend-api
  ports:
  - name: http
    port: 80
    targetPort: 9898
EOF
```

Let's check our ClusterIp
```bash
laborant@cplane-01:~$ kubectl get svc -n kproxy-lab -o wide
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
backend-api   ClusterIP   10.111.5.36   <none>        80/TCP    13s   app=backend-api
```
Now let's check the Iptables again

```bash
laborant@cplane-01:~$ sudo iptables-save -t nat | grep kproxy-lab
-A KUBE-SEP-3ZR6NQOAPEGAA663 -s 10.244.1.4/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-3ZR6NQOAPEGAA663 -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.1.4:9898
-A KUBE-SEP-UQMOMZFNPTKBTHCR -s 10.244.2.2/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-UQMOMZFNPTKBTHCR -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.2.2:9898
-A KUBE-SERVICES -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-5B33ZMWP5PRQCO5X
-A KUBE-SVC-5B33ZMWP5PRQCO5X ! -s 10.244.0.0/16 -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.1.4:9898" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3ZR6NQOAPEGAA663
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.2.2:9898" -j KUBE-SEP-UQMOMZFNPTKBTHCR
laborant@cplane-01:~$ 
```

```bash
laborant@node-01:~$ sudo iptables-save -t nat | grep kproxy-lab
-A KUBE-SEP-3ZR6NQOAPEGAA663 -s 10.244.1.4/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-3ZR6NQOAPEGAA663 -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.1.4:9898
-A KUBE-SEP-UQMOMZFNPTKBTHCR -s 10.244.2.2/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-UQMOMZFNPTKBTHCR -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.2.2:9898
-A KUBE-SERVICES -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-5B33ZMWP5PRQCO5X
-A KUBE-SVC-5B33ZMWP5PRQCO5X ! -s 10.244.0.0/16 -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.1.4:9898" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3ZR6NQOAPEGAA663
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.2.2:9898" -j KUBE-SEP-UQMOMZFNPTKBTHCR
laborant@node-01:~$ 
```

```bash
laborant@node-02:~$ sudo iptables-save -t nat | grep kproxy-lab
-A KUBE-SEP-3ZR6NQOAPEGAA663 -s 10.244.1.4/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-3ZR6NQOAPEGAA663 -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.1.4:9898
-A KUBE-SEP-UQMOMZFNPTKBTHCR -s 10.244.2.2/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-UQMOMZFNPTKBTHCR -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.2.2:9898
-A KUBE-SERVICES -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-5B33ZMWP5PRQCO5X
-A KUBE-SVC-5B33ZMWP5PRQCO5X ! -s 10.244.0.0/16 -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.1.4:9898" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3ZR6NQOAPEGAA663
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.2.2:9898" -j KUBE-SEP-UQMOMZFNPTKBTHCR
laborant@node-02:~$ 
```

Interestingly all nodes ip tables also changed. 

Now let's check the  End point Slices also

```bash
laborant@cplane-01:~$ kubectl get endpointslices -n kproxy-lab -o wide
NAME                ADDRESSTYPE   PORTS   ENDPOINTS               AGE
backend-api-crs2l   IPv4          9898    10.244.2.2,10.244.1.4   4m43s
```

It has also changed. Now what happen keep it as it is and let's move to curl the backends pod from frotned

```bash
laborant@cplane-01:~$ SVC_IP=10.111.5.36
laborant@cplane-01:~$ FRONTEND_POD=frontend-55567b655d-9j8w4
laborant@cplane-01:~$ for i in $(seq 1 10); do
  kubectl exec -n kproxy-lab "$FRONTEND_POD" -- curl -s http://$SVC_IP:80 | grep hostname
done
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-6kk45",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-6kk45",
  "hostname": "backend-api-7fd9cffcc4-6kk45",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-6kk45",
  "hostname": "backend-api-7fd9cffcc4-6kk45",
laborant@cplane-01:~$ 
```

Now with one single ip and port `10.111.5.36` we can curl to backends

But this custerIp is quite diffrent from the pod ips and also not similar to any node ips also so how this Ip actully hitting the backend and not even one backend. Hitting all backend.

Who is actully responsible for this?
The answer is IPtables. Now let's check the ip tables deeply


```bash
laborant@cplane-01:~$ sudo iptables-save -t nat | grep "kproxy-lab/"
-A KUBE-SEP-3ZR6NQOAPEGAA663 -s 10.244.1.4/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-3ZR6NQOAPEGAA663 -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.1.4:9898
-A KUBE-SEP-UQMOMZFNPTKBTHCR -s 10.244.2.2/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-UQMOMZFNPTKBTHCR -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.2.2:9898
-A KUBE-SERVICES -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-5B33ZMWP5PRQCO5X
-A KUBE-SVC-5B33ZMWP5PRQCO5X ! -s 10.244.0.0/16 -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.1.4:9898" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3ZR6NQOAPEGAA663
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.2.2:9898" -j KUBE-SEP-UQMOMZFNPTKBTHCR
laborant@cplane-01:~$ 
```

Now from this we have 3 chains and if we rember our end points then we had this

```
backend-api-crs2l   IPv4          9898    10.244.2.2,10.244.1.4   4m43s
```

```
Service chain:
KUBE-SVC-5B33ZMWP5PRQCO5X

Endpoint chain 1:
KUBE-SEP-3ZR6NQOAPEGAA663 → 10.244.1.4:9898

Endpoint chain 2:
KUBE-SEP-UQMOMZFNPTKBTHCR → 10.244.2.2:9898
```

Now let's check each chain one by one

##### KUBE-SVC-5B33ZMWP5PRQCO5X
```bash
laborant@cplane-01:~$ sudo iptables -t nat -L KUBE-SVC-5B33ZMWP5PRQCO5X -n -v --line-numbers
Chain KUBE-SVC-5B33ZMWP5PRQCO5X (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 KUBE-MARK-MASQ  6    --  *      *      !10.244.0.0/16        10.111.5.36          /* kproxy-lab/backend-api:http cluster IP */ tcp dpt:80
2        0     0 KUBE-SEP-3ZR6NQOAPEGAA663  0    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kproxy-lab/backend-api:http -> 10.244.1.4:9898 */ statistic mode random probability 0.50000000000
3        0     0 KUBE-SEP-UQMOMZFNPTKBTHCR  0    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kproxy-lab/backend-api:http -> 10.244.2.2:9898 */
```

1. If traffics comes out side the pod network (`!10.244.0.0/16 `) then mark it for mark it for MASQUERADE/SNAT
2. It is sending 50% traffic to the pod having the ip `10.244.1.4` which is pod `backend-api-7fd9cffcc4-6kk45`
3. It sends remaining traffic to the pod having the ip `10.244.2.2` which is pod `backend-api-7fd9cffcc4-mk9lc`

##### KUBE-SEP-3ZR6NQOAPEGAA663
```bash
laborant@cplane-01:~$ sudo iptables -t nat -L KUBE-SEP-3ZR6NQOAPEGAA663 -n -v --line-numbers
Chain KUBE-SEP-3ZR6NQOAPEGAA663 (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 KUBE-MARK-MASQ  0    --  *      *       10.244.1.4           0.0.0.0/0            /* kproxy-lab/backend-api:http */
2        0     0 DNAT       6    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kproxy-lab/backend-api:http */ tcp to:10.244.1.4:9898
laborant@cplane-01:~$ 
```

1. If it call it self then traffic will be MASQUERADE as it will be back to itself
2. DNAT the traffic to 10.244.1.4:9898.

##### KUBE-SEP-UQMOMZFNPTKBTHCR
```bash
laborant@cplane-01:~$ sudo iptables -t nat -L KUBE-SEP-UQMOMZFNPTKBTHCR -n -v --line-numbers
Chain KUBE-SEP-UQMOMZFNPTKBTHCR (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 KUBE-MARK-MASQ  0    --  *      *       10.244.2.2           0.0.0.0/0            /* kproxy-lab/backend-api:http */
2        0     0 DNAT       6    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kproxy-lab/backend-api:http */ tcp to:10.244.2.2:9898
laborant@cplane-01:~$ 
```

1. If it call it self then traffic will be MASQUERADE as it will be back to itself
2. DNAT the traffic to 10.244.2.2:9898.

The above iptables is same for all nodes so hitting the CusterIp will be redirect to available pods in a random order using probabilty

Now to inspect more let's delete this backend pod `backend-api-7fd9cffcc4-6kk45`
```bash
laborant@cplane-01:~$ kubectl delete pod backend-api-7fd9cffcc4-6kk45 -n kproxy-lab
pod "backend-api-7fd9cffcc4-6kk45" deleted from kproxy-lab namespace
```
```bash
laborant@cplane-01:~$ kubectl get pods -n kproxy-lab -o wide
NAME                           READY   STATUS    RESTARTS   AGE    IP           NODE      NOMINATED NODE   READINESS GATES
backend-api-7fd9cffcc4-mk9lc   1/1     Running   0          114m   10.244.2.2   node-02   <none>           <none>
backend-api-7fd9cffcc4-pbn9j   1/1     Running   0          53s    10.244.1.5   node-01   <none>           <none>
frontend-55567b655d-9j8w4      1/1     Running   0          114m   10.244.1.2   node-01   <none>           <none>
laborant@cplane-01:~$ 
```

*new pod has been created*

Let's check the CusterIp
```bash
laborant@cplane-01:~$ kubectl get svc -n kproxy-lab -o wide
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE   SELECTOR
backend-api   ClusterIP   10.111.5.36   <none>        80/TCP    61m   app=backend-api
laborant@cplane-01:~$ 
```

let's check the end point slices

```
laborant@cplane-01:~$ kubectl get endpointslices -n kproxy-lab -o wide
NAME                ADDRESSTYPE   PORTS   ENDPOINTS               AGE
backend-api-crs2l   IPv4          9898    10.244.2.2,10.244.1.5   59m
```

So the backend one pod as been removed and one added with a new ip so still we can curl using the Custer Ip
```bash
laborant@cplane-01:~$ for i in $(seq 1 10); do
  kubectl exec -n kproxy-lab "$FRONTEND_POD" -- curl -s http://$SVC_IP:80 | grep hostname
done
  "hostname": "backend-api-7fd9cffcc4-pbn9j",
  "hostname": "backend-api-7fd9cffcc4-pbn9j",
  "hostname": "backend-api-7fd9cffcc4-pbn9j",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-pbn9j",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
laborant@cplane-01:~$ 
```

Now let's check the iptables

```bash
laborant@cplane-01:~$ sudo iptables-save -t nat | grep kproxy-lab
-A KUBE-SEP-DZXZXVGLX65KLHYQ -s 10.244.1.5/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-DZXZXVGLX65KLHYQ -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.1.5:9898
-A KUBE-SEP-UQMOMZFNPTKBTHCR -s 10.244.2.2/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-UQMOMZFNPTKBTHCR -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.2.2:9898
-A KUBE-SERVICES -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-5B33ZMWP5PRQCO5X
-A KUBE-SVC-5B33ZMWP5PRQCO5X ! -s 10.244.0.0/16 -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.1.5:9898" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-DZXZXVGLX65KLHYQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.2.2:9898" -j KUBE-SEP-UQMOMZFNPTKBTHCR
laborant@cplane-01:~$ 
```

The previous `KUBE-SEP-3ZR6NQOAPEGAA663` chain is no more and a new chain is added `KUBE-SEP-DZXZXVGLX65KLHYQ`

Let's check this chain:
```bash
laborant@cplane-01:~$ sudo iptables -t nat -L KUBE-SEP-DZXZXVGLX65KLHYQ -n -v --line-numbers
Chain KUBE-SEP-DZXZXVGLX65KLHYQ (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 KUBE-MARK-MASQ  0    --  *      *       10.244.1.5           0.0.0.0/0            /* kproxy-lab/backend-api:http */
2        0     0 DNAT       6    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kproxy-lab/backend-api:http */ tcp to:10.244.1.5:9898
laborant@cplane-01:~$ 
```

and the sevice chain

```bash
laborant@cplane-01:~$ sudo iptables -t nat -L KUBE-SVC-5B33ZMWP5PRQCO5X -n -v --line-numbers
Chain KUBE-SVC-5B33ZMWP5PRQCO5X (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 KUBE-MARK-MASQ  6    --  *      *      !10.244.0.0/16        10.111.5.36          /* kproxy-lab/backend-api:http cluster IP */ tcp dpt:80
2        0     0 KUBE-SEP-DZXZXVGLX65KLHYQ  0    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kproxy-lab/backend-api:http -> 10.244.1.5:9898 */ statistic mode random probability 0.50000000000
3        0     0 KUBE-SEP-UQMOMZFNPTKBTHCR  0    --  *      *       0.0.0.0/0            0.0.0.0/0            /* kproxy-lab/backend-api:http -> 10.244.2.2:9898 */
laborant@cplane-01:~$ 
```

Both has been changed. So it's intersting who is actully changing this.
The answer is it's `Kube Proxy`

Let's verify if `Kube Proxy` really responsible

So in worker node-01 (As we are send curl from frontend pod `frontend-55567b655d-9j8w4`) let's stop Kube Proxy and keep active in other nodes
```bash
laborant@node-01:~$ kubectl label node cplane-01 kube-proxy-run=true --overwrite
node/cplane-01 labeled
laborant@node-01:~$ kubectl label node node-02 kube-proxy-run=true --overwrite
node/node-02 labeled
laborant@node-01:~$ kubectl -n kube-system patch ds kube-proxy \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"kube-proxy-run":"true"}}}}}'
daemonset.apps/kube-proxy patched
laborant@node-01:~$ kubectl -n kube-system get pods -o wide | grep kube-proxy
kube-proxy-crj6h                    1/1     Running   0          18s    172.16.0.2   cplane-01   <none>           <none>
kube-proxy-r2jdk                    1/1     Running   0          18s    172.16.0.4   node-02     <none>           <none>
laborant@node-01:~$ 
```

Now Let's delete this newly created pod of backend `backend-api-7fd9cffcc4-pbn9j`

```bash
laborant@cplane-01:~$ kubectl delete pod backend-api-7fd9cffcc4-pbn9j -n kproxy-lab
pod "backend-api-7fd9cffcc4-pbn9j" deleted from kproxy-lab namespace
```

New Let's check the iptables of control plane and a worker node

Worker Node-01: the chain are same as previous though a pod was delete and added
```bash
laborant@cplane-01:~$ sudo iptables-save -t nat | grep kproxy-lab
-A KUBE-SEP-DZXZXVGLX65KLHYQ -s 10.244.1.5/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-DZXZXVGLX65KLHYQ -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.1.5:9898
-A KUBE-SEP-UQMOMZFNPTKBTHCR -s 10.244.2.2/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-UQMOMZFNPTKBTHCR -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.2.2:9898
-A KUBE-SERVICES -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-5B33ZMWP5PRQCO5X
-A KUBE-SVC-5B33ZMWP5PRQCO5X ! -s 10.244.0.0/16 -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.1.7:9898" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-POE7ZRN3ZGWUSIL2
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.2.2:9898" -j KUBE-SEP-UQMOMZFNPTKBTHCR
```

Control Plane The iptable chain has been updated
```bash
laborant@cplane-01:~$ sudo iptables-save -t nat | grep kproxy-lab
-A KUBE-SEP-POE7ZRN3ZGWUSIL2 -s 10.244.1.7/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-POE7ZRN3ZGWUSIL2 -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.1.7:9898
-A KUBE-SEP-UQMOMZFNPTKBTHCR -s 10.244.2.2/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-UQMOMZFNPTKBTHCR -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.2.2:9898
-A KUBE-SERVICES -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-5B33ZMWP5PRQCO5X
-A KUBE-SVC-5B33ZMWP5PRQCO5X ! -s 10.244.0.0/16 -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.1.7:9898" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-POE7ZRN3ZGWUSIL2
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.2.2:9898" -j KUBE-SEP-UQMOMZFNPTKBTHCR
laborant@node-01:~$ 
```

Now check the previous curl we use to did let's rul it 2s timeout in worker node 1 as frotnend is running there:
```bash
laborant@node-01:~$ for i in $(seq 1 10); do
  kubectl exec -n kproxy-lab "$FRONTEND_POD" -- \
  curl -s --max-time 2 http://$SVC_IP:80 | grep hostname
done
command terminated with exit code 28
command terminated with exit code 28
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
command terminated with exit code 28
command terminated with exit code 28
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
command terminated with exit code 28
```

Now bring back the Kube Proxy in worker-01 

```bash
laborant@node-01:~$ kubectl -n kube-system patch ds kube-proxy \
  -p '{"spec":{"template":{"spec":{"nodeSelector":null}}}}'
daemonset.apps/kube-proxy patched
```

Now check the curl from Worker 1
```bash
laborant@node-01:~$ for i in $(seq 1 10); do
  kubectl exec -n kproxy-lab "$FRONTEND_POD" -- \
  curl -s --max-time 2 http://$SVC_IP:80 | grep hostname
done
  "hostname": "backend-api-7fd9cffcc4-sdfrq",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-sdfrq",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-mk9lc",
  "hostname": "backend-api-7fd9cffcc4-sdfrq",
  "hostname": "backend-api-7fd9cffcc4-sdfrq",
  "hostname": "backend-api-7fd9cffcc4-sdfrq",
  "hostname": "backend-api-7fd9cffcc4-sdfrq",
  "hostname": "backend-api-7fd9cffcc4-sdfrq",
```

All the backends are now working

Finally let's check the iptables also

```bash
laborant@node-01:~$ sudo iptables-save -t nat | grep kproxy-lab
-A KUBE-SEP-POE7ZRN3ZGWUSIL2 -s 10.244.1.7/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-POE7ZRN3ZGWUSIL2 -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.1.7:9898
-A KUBE-SEP-UQMOMZFNPTKBTHCR -s 10.244.2.2/32 -m comment --comment "kproxy-lab/backend-api:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-UQMOMZFNPTKBTHCR -p tcp -m comment --comment "kproxy-lab/backend-api:http" -m tcp -j DNAT --to-destination 10.244.2.2:9898
-A KUBE-SERVICES -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-5B33ZMWP5PRQCO5X
-A KUBE-SVC-5B33ZMWP5PRQCO5X ! -s 10.244.0.0/16 -d 10.111.5.36/32 -p tcp -m comment --comment "kproxy-lab/backend-api:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.1.7:9898" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-POE7ZRN3ZGWUSIL2
-A KUBE-SVC-5B33ZMWP5PRQCO5X -m comment --comment "kproxy-lab/backend-api:http -> 10.244.2.2:9898" -j KUBE-SEP-UQMOMZFNPTKBTHCR
```

New iptable chain is also visibale and old one has been removed

Conculations:
- kube-proxy is mainly responsible for creating/updating Service networking rules, such as iptables rules.

- It watches the Kubernetes API for Service and EndpointSlice changes, then updates rules on each node so traffic to a ClusterIP can be forwarded to real backend Pod IPs.

- In iptables mode, kube-proxy can distribute traffic across multiple backend Pods using probability/random selection.

But,
- kube proxy is not responsible for carrying direct Pod-to-Pod traffic; it only translates Service/ClusterIP traffic to a selected real Pod IP.

- After kube-proxy rewrites ClusterIP traffic to a Pod IP, the actual Pod-to-Pod delivery is handled by the CNI plugin, such as Flannel.
