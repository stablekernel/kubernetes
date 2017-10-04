# k8s bootstrapping

## nginx-ingress-controller

Add to cluster to use `nginx` as a load balancer.

- Useful for environments where cost of a cloud provided load balancer is untenable.
- Deploy once per cluster, use one or more Ingress resources for routing.
- Requires allocating static IP address from cloud provider and entering in `nginx-ingress-controller/ingress-controller-lb.yaml`.
- Assumes there already exists a default-http-backend service in kube-system. This is true of GKE.


## https://github.com/jetstack/kube-lego/tree/master/examples/nginx