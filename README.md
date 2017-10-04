# k8s bootstrapping

Directories in this repository contain Kubernetes configuration files for provisioning a cluster and deploying Aqueduct applications.

## Prerequisites

The following assumes that you have a Google Cloud account with billing enabled and that tools such as `docker`, `gcloud` and `kubectl` have been installed and configured. If not, see the following references:

- [Install Docker](https://www.docker.com/community-edition)
- [Install gcloud](https://cloud.google.com/sdk/downloads)
- Install `kubectl` by running `gcloud components install kubectl`.

## Cluster Provisioning

The objects in `kube-lego` and `nginx-ingress-controller` provide a cluster with low cost alternatives to load-balancing with SSL termination. These objects need only be applied when a cluster is first created and live in the `kube-system` namespace.

### SSL Certificates via Let's Encrypt

Modify `kube-lego/config-map.yaml` by replacing `<ADMIN_EMAIL>` with an administrator e-mail address. Then, apply the objects in the `kube-lego` directory to your cluster:

```bash
kubectl apply -f kube-lego/
```

`kube-lego` will monitor your cluster to request trusted SSL certificates for deployed applications.

### Nginx Ingress Controller

*The nginx ingress controller is only necessary if you prefer not to use Google Cloud Load Balancer (GCLB costs $200+/yr).*

Without the need to modify any files, apply the contents of `nginx-ingress-controller` to the cluster.

```bash
kubectl apply -f nginx-ingress-controller/
```

It will take a moment for `nginx-ingress-lb` to acquire an IP address. During that time, running the command `kubectl get services -n kube-system` will show something like the following:

```
NAME                   CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
default-http-backend   10.55.241.222   <nodes>       80:32516/TCP                 11d
heapster               10.55.255.209   <none>        80/TCP                       11d
kube-dns               10.55.240.10    <none>        53/UDP,53/TCP                11d
kubernetes-dashboard   10.55.240.49    <none>        80/TCP                       11d
nginx-ingress-lb       10.55.249.185   <pending>     80:32005/TCP,443:31623/TCP   6s
```

Where `nginx-ingress-lb`'s `EXTERNAL-IP` is `<pending>`. Once that `<pending>` flips to an IP address, note the IP address and navigate to `VPC Network->External IP adresses` in the Google Cloud console. Locate the IP address in that list and change it's type from `Ephemeral` to `Static`. (You'll be prompted for a name which can be whatever you like.)

*Important:* if you delete the `nginx-ingress-lb` service after reserving a static IP for it, simply re-creating the service won't work. You'll need to release the reserved address before the service will start again and reserve the IP address again.
