# kube-proxy

`kube-proxy` is a Kubernetes node component that makes **Service IP / ClusterIP** able to communicate with a Pod, so that pods can communicate with each other with a single virtual IP.

It watches **Services** and **EndpointSlices** from the Kubernetes API server.

When something changes, it creates or updates node-level rules, in this lab with `nftables`, so traffic can be forwarded to a Pod.

---

### Example Cluster

To understand it, let's use a single-node Raspberry Pi Kubernetes cluster:

```text
Node 1
Role: control-plane
IP:   192.168.0.231
```

### Check Available Nodes

```bash
aidid@pi:~$ kubectl get nodes -o wide
NAME   STATUS   ROLES           AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION               CONTAINER-RUNTIME
pi     Ready    control-plane   162m   v1.36.2   192.168.0.231   <none>        Debian GNU/Linux 13 (trixie)   6.18.34+rpt-rpi-v8 (arm64)   containerd://2.2.5
```

Also, let's check its CIDR:

```bash
aidid@pi:~$ kubectl describe node pi | grep -i PodCIDR:
PodCIDR:                      10.244.0.0/24
```

### Change kube-proxy Mode to nftables

Now change kube-proxy mode to `nftables`.

```bash
aidid@pi:~$ kubectl -n kube-system edit configmap kube-proxy
```

Inside the config, change the mode:

```yaml
mode: nftables
```

Then restart kube-proxy so that it starts again with the new mode:

```bash
aidid@pi:~$ kubectl -n kube-system rollout restart daemonset kube-proxy
daemonset.apps/kube-proxy restarted
```

```bash
aidid@pi:~$ kubectl -n kube-system rollout status daemonset kube-proxy
daemon set "kube-proxy" successfully rolled out
```

Also, let's confirm that kube-proxy is running in `nftables` mode:

```bash
aidid@pi:~$ kubectl -n kube-system get configmap kube-proxy -o jsonpath='{.data.config\.conf}' | grep '^mode:'
mode: nftables
```

```bash
aidid@pi:~$ sudo nft list tables | grep kube-proxy
table ip kube-proxy
table ip6 kube-proxy
```

### A New Namespace

First, let's create a new namespace so that we can see our related namespace's `nftables` and how things will change.

```bash
aidid@pi:~$ kubectl create namespace kproxy-lab
namespace/kproxy-lab created
```

```bash
aidid@pi:~$ kubectl get all -n kproxy-lab
No resources found in kproxy-lab namespace.
aidid@pi:~$
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
aidid@pi:~$ kubectl get pods -n kproxy-lab -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE   NOMINATED NODE   READINESS GATES
backend-api-7fd9cffcc4-9lgb8   1/1     Running   0          8s    10.244.0.10   pi     <none>           <none>
backend-api-7fd9cffcc4-bpwvj   1/1     Running   0          9s    10.244.0.8    pi     <none>           <none>
frontend-55567b655d-jz8s8      1/1     Running   0          9s    10.244.0.9    pi     <none>           <none>
aidid@pi:~$
```

Now we get 2 pods for backend:

```text
backend-api-7fd9cffcc4-9lgb8 10.244.0.10 # running on pi
backend-api-7fd9cffcc4-bpwvj 10.244.0.8  # running on pi
```

And 1 for frontend:

```text
frontend-55567b655d-jz8s8 10.244.0.9 # running on pi
```

From the frontend pod (frontend-55567b655d-jz8s8), we curl the backend; then we should be able to curl them, as they both are running.

```bash
aidid@pi:~$ kubectl exec -n kproxy-lab frontend-55567b655d-jz8s8 -- curl -s http://10.244.0.10:9898 | grep hostname
  "hostname": "backend-api-7fd9cffcc4-9lgb8",
aidid@pi:~$ kubectl exec -n kproxy-lab frontend-55567b655d-jz8s8 -- curl -s http://10.244.0.8:9898 | grep hostname
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
aidid@pi:~$
```

NB: Though all pods are on the same node, still we can connect to them. CNI/Flannel is making the magic for us.

So it's confirmed that the frontend can talk with the backend, but as we know, there are some issues:

1. This is actually communicating with one pod. If the pod (backend) crashes, then communication is lost.
2. Pods are not static. They can be deleted and recreated, and also their IP changes.
3. It is not feasible to give the frontend all backend pod IPs because sometimes there can be hundreds of pods running for backend.

So, to solve this issue, we will create a ClusterIP Service, which will give the backend Pods one stable internal IP/name.

But before creating that, let's check the `nftables`:

```bash
aidid@pi:~$ sudo nft list table ip kube-proxy > /tmp/before-service.nft
aidid@pi:~$ sudo nft list table ip kube-proxy | grep kproxy-lab
aidid@pi:~$
```

All `nftables` rules on the node are empty for this namespace (`kproxy-lab`).

Also, if we check the EndpointSlices, we will find them empty:

```bash
aidid@pi:~$ kubectl get endpointslices -n kproxy-lab -o wide
No resources found in kproxy-lab namespace.
```

Now let's add a ClusterIP to the backend:

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

Let's check our ClusterIP:

```bash
aidid@pi:~$ kubectl get svc -n kproxy-lab -o wide
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
backend-api   ClusterIP   10.100.192.23   <none>        80/TCP    13s   app=backend-api
```

Now let's check the `nftables` again:

```bash
aidid@pi:~$ sudo nft list table ip kube-proxy
table ip kube-proxy {
	comment "rules for kube-proxy"
	set hairpin-connections {
		type ipv4_addr . ipv4_addr . inet_proto . inet_service
		comment "service hairpin connections"
		elements = { 10.244.0.8 . 10.244.0.8 . tcp . 9898,
			     10.244.0.10 . 10.244.0.10 . tcp . 9898 }
	}

	set cluster-ips {
		type ipv4_addr
		comment "Active ClusterIPs"
		elements = { 10.96.0.1, 10.96.0.10,
			     10.100.192.23 }
	}

	set nodeport-ips {
		type ipv4_addr
		comment "IPs that accept NodePort traffic"
		elements = { 192.168.0.231 }
	}

	map service-ips {
		type ipv4_addr . inet_proto . inet_service : verdict
		comment "ClusterIP, ExternalIP and LoadBalancer IP traffic"
		elements = { 10.100.192.23 . tcp . 80 : goto service-YVSKLJHA-kproxy-lab/backend-api/tcp/http,
			     10.96.0.1 . tcp . 443 : goto service-2QRHZV4L-default/kubernetes/tcp/https }
	}

	chain services {
		ip daddr @cluster-ips ip saddr != 10.244.0.0/16 meta mark set meta mark | 0x00004000 comment "masquerade clusterIP traffic from outside cluster"
		ip daddr . meta l4proto . th dport vmap @service-ips
		ip daddr @nodeport-ips meta l4proto . th dport vmap @service-nodeports
	}

	chain masquerading {
		meta mark & 0x00004000 != 0x00000000 meta mark set meta mark ^ 0x00004000 masquerade fully-random
		ct status dnat ip saddr . ip daddr . meta l4proto . th dport @hairpin-connections masquerade fully-random
	}

	chain service-2QRHZV4L-default/kubernetes/tcp/https {
		meta l4proto tcp dnat ip to numgen random mod 1 map { 0 : 192.168.0.231 . 6443 }
	}

	chain service-YVSKLJHA-kproxy-lab/backend-api/tcp/http {
		meta l4proto tcp dnat ip to numgen random mod 2 map { 0 : 10.244.0.10 . 9898, 1 : 10.244.0.8 . 9898 }
	}
}
aidid@pi:~$
```

Now let's check the EndpointSlices as well:

```bash
aidid@pi:~$ kubectl get endpointslices -n kproxy-lab -o wide
NAME                ADDRESSTYPE   PORTS   ENDPOINTS                AGE
backend-api-k47lx   IPv4          9898    10.244.0.8,10.244.0.10   13s
```

It has also changed. Now, let's keep it as it is and move to curl the backend pods from the frontend:

```bash
aidid@pi:~$ SVC_IP=10.100.192.23
aidid@pi:~$ FRONTEND_POD=frontend-55567b655d-jz8s8
aidid@pi:~$ for i in $(seq 1 10); do
  kubectl exec -n kproxy-lab "$FRONTEND_POD" -- curl -s http://$SVC_IP:80 | grep hostname
done
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-9lgb8",
  "hostname": "backend-api-7fd9cffcc4-9lgb8",
  "hostname": "backend-api-7fd9cffcc4-9lgb8",
  "hostname": "backend-api-7fd9cffcc4-9lgb8",
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-9lgb8",
aidid@pi:~$
```

Now with one single IP and port, `10.100.192.23`, we can curl to the backends.

But this ClusterIP is quite different from the pod IPs, and also not similar to the node IP, so how is this IP actually hitting the backend and not even one backend, but all backends?

Who is actually responsible for this?

The answer is `nftables`. Now let's check the `nftables` in depth:

```bash
aidid@pi:~$ sudo nft list table ip kube-proxy
table ip kube-proxy {
	comment "rules for kube-proxy"
	set hairpin-connections {
		type ipv4_addr . ipv4_addr . inet_proto . inet_service
		comment "service hairpin connections"
		elements = { 10.244.0.8 . 10.244.0.8 . tcp . 9898,
			     10.244.0.10 . 10.244.0.10 . tcp . 9898 }
	}

	set cluster-ips {
		type ipv4_addr
		comment "Active ClusterIPs"
		elements = { 10.96.0.1, 10.96.0.10,
			     10.100.192.23 }
	}

	map service-ips {
		type ipv4_addr . inet_proto . inet_service : verdict
		comment "ClusterIP, ExternalIP and LoadBalancer IP traffic"
		elements = { 10.100.192.23 . tcp . 80 : goto service-YVSKLJHA-kproxy-lab/backend-api/tcp/http,
			     10.96.0.1 . tcp . 443 : goto service-2QRHZV4L-default/kubernetes/tcp/https }
	}

	chain services {
		ip daddr @cluster-ips ip saddr != 10.244.0.0/16 meta mark set meta mark | 0x00004000 comment "masquerade clusterIP traffic from outside cluster"
		ip daddr . meta l4proto . th dport vmap @service-ips
		ip daddr @nodeport-ips meta l4proto . th dport vmap @service-nodeports
	}

	chain service-YVSKLJHA-kproxy-lab/backend-api/tcp/http {
		meta l4proto tcp dnat ip to numgen random mod 2 map { 0 : 10.244.0.10 . 9898, 1 : 10.244.0.8 . 9898 }
	}
}
aidid@pi:~$
```

Now from this, we have a `service-ips` map and a generated Service chain, and if we remember our endpoints, then we had this:

```text
backend-api-k47lx   IPv4          9898    10.244.0.8,10.244.0.10   13s
```

```text
Service map:
service-ips

Service chain:
service-YVSKLJHA-kproxy-lab/backend-api/tcp/http

Endpoint 1:
10.244.0.10:9898

Endpoint 2:
10.244.0.8:9898
```

Now let's check each part one by one:

##### service-ips

```nft
map service-ips {
	type ipv4_addr . inet_proto . inet_service : verdict
	comment "ClusterIP, ExternalIP and LoadBalancer IP traffic"
	elements = { 10.100.192.23 . tcp . 80 : goto service-YVSKLJHA-kproxy-lab/backend-api/tcp/http,
		     10.96.0.1 . tcp . 443 : goto service-2QRHZV4L-default/kubernetes/tcp/https }
}
```

1. If traffic comes to `10.100.192.23` on TCP port `80`, then it matches this key: `10.100.192.23 . tcp . 80`.
2. That match sends traffic to the generated chain `service-YVSKLJHA-kproxy-lab/backend-api/tcp/http`.

##### service-YVSKLJHA-kproxy-lab/backend-api/tcp/http

```nft
chain service-YVSKLJHA-kproxy-lab/backend-api/tcp/http {
	meta l4proto tcp dnat ip to numgen random mod 2 map { 0 : 10.244.0.10 . 9898, 1 : 10.244.0.8 . 9898 }
}
```

1. It uses `numgen random mod 2` to choose one backend endpoint.
2. It DNATs the traffic to `10.244.0.10:9898` or `10.244.0.8:9898`.

##### hairpin-connections

```nft
set hairpin-connections {
	type ipv4_addr . ipv4_addr . inet_proto . inet_service
	comment "service hairpin connections"
	elements = { 10.244.0.8 . 10.244.0.8 . tcp . 9898,
		     10.244.0.10 . 10.244.0.10 . tcp . 9898 }
}
```

1. If a backend pod reaches the Service and kube-proxy selects the same backend pod, then this set helps identify that hairpin connection.
2. The `masquerading` chain can then MASQUERADE that traffic.

The above `nftables` output is on the node, so hitting the ClusterIP will be redirected to available pods in a random order using `numgen random mod 2`.

Now, to inspect more, let's delete this backend pod `backend-api-7fd9cffcc4-9lgb8`:

```bash
aidid@pi:~$ kubectl delete pod backend-api-7fd9cffcc4-9lgb8 -n kproxy-lab
pod "backend-api-7fd9cffcc4-9lgb8" deleted from kproxy-lab namespace
```

```bash
aidid@pi:~$ kubectl get pods -n kproxy-lab -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE   NOMINATED NODE   READINESS GATES
backend-api-7fd9cffcc4-bg8wn   1/1     Running   0          33m   10.244.0.11   pi     <none>           <none>
backend-api-7fd9cffcc4-bpwvj   1/1     Running   0          98m   10.244.0.8    pi     <none>           <none>
frontend-55567b655d-jz8s8      1/1     Running   0          98m   10.244.0.9    pi     <none>           <none>
aidid@pi:~$
```

*New pod has been created.*

Let's check the ClusterIP:

```bash
aidid@pi:~$ kubectl get svc -n kproxy-lab -o wide
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
backend-api   ClusterIP   10.100.192.23   <none>        80/TCP    91m   app=backend-api
aidid@pi:~$
```

Let's check the EndpointSlices:

```bash
aidid@pi:~$ kubectl get endpointslices -n kproxy-lab -o wide
NAME                ADDRESSTYPE   PORTS   ENDPOINTS                AGE
backend-api-k47lx   IPv4          9898    10.244.0.8,10.244.0.11   91m
```

So one backend pod has been removed, and one has been added with a new IP, so still we can curl using the ClusterIP:

```bash
aidid@pi:~$ for i in $(seq 1 10); do
  kubectl exec -n kproxy-lab "$FRONTEND_POD" -- curl -s http://$SVC_IP:80 | grep hostname
done
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-bg8wn",
  "hostname": "backend-api-7fd9cffcc4-bg8wn",
  "hostname": "backend-api-7fd9cffcc4-bg8wn",
  "hostname": "backend-api-7fd9cffcc4-bg8wn",
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-bpwvj",
  "hostname": "backend-api-7fd9cffcc4-bg8wn",
aidid@pi:~$
```

Now let's check the `nftables`:

```bash
aidid@pi:~$ sudo nft list table ip kube-proxy
table ip kube-proxy {
	comment "rules for kube-proxy"
	set hairpin-connections {
		type ipv4_addr . ipv4_addr . inet_proto . inet_service
		comment "service hairpin connections"
		elements = { 10.244.0.8 . 10.244.0.8 . tcp . 9898,
			     10.244.0.11 . 10.244.0.11 . tcp . 9898 }
	}

	set cluster-ips {
		type ipv4_addr
		comment "Active ClusterIPs"
		elements = { 10.96.0.1, 10.96.0.10,
			     10.100.192.23 }
	}

	map service-ips {
		type ipv4_addr . inet_proto . inet_service : verdict
		comment "ClusterIP, ExternalIP and LoadBalancer IP traffic"
		elements = { 10.100.192.23 . tcp . 80 : goto service-YVSKLJHA-kproxy-lab/backend-api/tcp/http,
			     10.96.0.1 . tcp . 443 : goto service-2QRHZV4L-default/kubernetes/tcp/https }
	}

	chain service-YVSKLJHA-kproxy-lab/backend-api/tcp/http {
		meta l4proto tcp dnat ip to numgen random mod 2 map { 0 : 10.244.0.11 . 9898, 1 : 10.244.0.8 . 9898 }
	}
}
aidid@pi:~$
```

The previous backend endpoint `10.244.0.10:9898` no longer exists, and a new endpoint has been added: `10.244.0.11:9898`.

Before the pod was deleted, the Service chain had this endpoint:

```nft
chain service-YVSKLJHA-kproxy-lab/backend-api/tcp/http {
	meta l4proto tcp dnat ip to numgen random mod 2 map { 0 : 10.244.0.10 . 9898, 1 : 10.244.0.8 . 9898 }
}
```

After the pod was recreated, kube-proxy updated the Service chain:

```nft
chain service-YVSKLJHA-kproxy-lab/backend-api/tcp/http {
	meta l4proto tcp dnat ip to numgen random mod 2 map { 0 : 10.244.0.11 . 9898, 1 : 10.244.0.8 . 9898 }
}
```

The `hairpin-connections` set changed too.

Before:

```nft
set hairpin-connections {
	elements = { 10.244.0.8 . 10.244.0.8 . tcp . 9898,
		     10.244.0.10 . 10.244.0.10 . tcp . 9898 }
}
```

After:

```nft
set hairpin-connections {
	elements = { 10.244.0.8 . 10.244.0.8 . tcp . 9898,
		     10.244.0.11 . 10.244.0.11 . tcp . 9898 }
}
```

Both have been changed. So it's interesting who is actually changing this.

The answer is `kube-proxy`.

---

### Packet Traversal Diagram

In `iptables` mode, the packet usually walks through Service rules one by one until it finds the matching ClusterIP and port.

```text
Packet: 10.100.192.23:80

PREROUTING / OUTPUT
        |
        v
KUBE-SERVICES
        |
        +-- rule 1: service A? no
        |
        +-- rule 2: service B? no
        |
        +-- rule 3: service C? no
        |
        +-- ...
        |
        +-- rule n: kproxy-lab/backend-api? yes
                    |
                    v
              KUBE-SVC-xxxxx
                    |
                    +-- endpoint rule 1
                    |
                    +-- endpoint rule 2
                    |
                    v
              DNAT to backend Pod
```

So if there are many Services and endpoints, the packet may need to scan many rules.

```text
iptables lookup cost: O(n)
```

In `nftables` mode, kube-proxy stores Service lookups in maps and sets.

```text
Packet: 10.100.192.23:80

nat-prerouting / nat-output
        |
        v
services chain
        |
        v
lookup key:
10.100.192.23 . tcp . 80
        |
        v
service-ips map
        |
        +-- direct match:
            10.100.192.23 . tcp . 80
            goto service-YVSKLJHA-kproxy-lab/backend-api/tcp/http
                    |
                    v
              numgen random mod 2
                    |
                    +-- 0: 10.244.0.11:9898
                    |
                    +-- 1: 10.244.0.8:9898
                    |
                    v
              DNAT to selected backend Pod
```

Here the packet does not need to walk through every Service rule. It can use the map key directly.

```text
nftables lookup cost: direct map lookup instead of O(n) rule scan
```

---

### Conclusions

- kube proxy is mainly responsible for creating/updating Service networking rules, such as `nftables` rules.

- It watches the Kubernetes API for Service and EndpointSlice changes, then updates rules on the node so traffic to a ClusterIP can be forwarded to real backend Pod IPs.

- In `iptables` mode, Service traffic usually walks through many rules/chains until it finds a matching Service and endpoint.

- That means packet matching can become `O(n)`, because more Services and endpoints can mean more rules to scan.

- In `nftables` mode, kube-proxy uses maps and sets, such as `service-ips`, `cluster-ips`, and `hairpin-connections`.

- This allows nftables to look up the Service key directly, such as `10.100.192.23 . tcp . 80`, instead of checking rules one by one.

- So nftables reduces the long `O(n)` rule walk and makes Service lookup closer to a direct map lookup.

- After the Service is matched, nftables can distribute traffic across backend Pods using `numgen random mod 2`.

But:

- kube proxy is not responsible for carrying direct Pod-to-Pod traffic; it only translates Service/ClusterIP traffic to a selected real Pod IP.

- After kube proxy rewrites ClusterIP traffic to a Pod IP, the actual Pod-to-Pod delivery is handled by the CNI plugin, such as Flannel.
