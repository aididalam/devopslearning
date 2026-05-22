`kube-proxy` is a Kubernetes node component that makes **Service IP / ClusterIP** able to communicate with a Pod.

It watches **Services** and **EndpointSlices** from the Kube API server.

On changes it creates/updates node level rules, usually `iptables`, so traffic can forward to a pod.
