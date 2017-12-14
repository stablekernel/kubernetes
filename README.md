# k8s bootstrapping

Directories in this repository contain Kubernetes configuration files for provisioning a cluster and deploying Aqueduct applications.

## Prerequisites

The following assumes that you have a Google Cloud account with billing enabled and that tools such as `docker`, `gcloud` and `kubectl` have been installed and configured. If not, see the following references:

- [Install Docker](https://www.docker.com/community-edition)
- [Install gcloud](https://cloud.google.com/sdk/downloads)
- Install `kubectl` by running `gcloud components install kubectl`.

## Cluster Provisioning

*Tasks in this section need only be applied when a cluster is first created, not for each application deployed.*

The objects in `kube-lego` and `nginx-ingress-controller` provide a cluster with low cost alternatives to load-balancing with SSL termination. They live in the `kube-system` namespace.

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

*Important: if you delete the `nginx-ingress-lb` service after reserving a static IP for it, simply re-creating the service won't work. You'll need to release the reserved address before the service will start again and then reserve the IP address again.*

## Application Deployment

The `aqueduct` directory contains templates for Kubernetes objects that deploy an Aqueduct application and its PostgreSQL database. For each Aqueduct application deployed into the cluster, follow the following steps.

*Note: Kubernetes object names may not contain underscores. If your application name contains underscores, substitute a dash (`-`) when replacing the `<APP_NAME>` template variable. For example, if your application name is `my_app`, use `my-app`.*

0. Copy the *contents* of `aqueduct` into a directory named `k8s` in your project directory.

1. Create a new namespace with the name of your application.

```
kubectl create namespace <app-name>
```

2. Modify `config/configmap.yaml` and `config/secrets.yaml` by replacing all occurrences of `<APP_NAME>` with the name of your application (remembering to use dashes and not underscores), and replacing `<PASSWORD>` with a database password. Apply these files:

```
kubectl apply -f k8s/config/
```

Change your Aqueduct application's config.yaml to use the values from the configmap, secrets and db-service:

```
database:
  username: $POSTGRES_USER
  password: $POSTGRES_PASSWORD
  host: db-service
  port: 5432
  databaseName: $POSTGRES_DB
```

3. Add the `Dockerfile` in this repository to your Aqueduct project directory. Make sure that migration files are up to date by running `aqueduct db validate` (and running `aqueduct db generate` if they are not). Migration files must be a part of the docker image.

Run `docker build` in the project directory. The name of the image must have the format `gcr.io/<PROJECT_ID>/<APP_NAME>` where `PROJECT_ID` is the name of the Google Cloud project that the target cluster is in and `APP_NAME` is your application's name.

```
docker build -t gcr.io/<project-id>/<app-name>:latest .
```

Once built, push it to your project's private registry:

```
gcloud docker -- push gcr.io/<project-id>/<app-name>:latest
```

4. In both `api-deployment-and-service.yaml` and `db-deployment-and-service.yaml`, replace template variables with their appropriate values. Ensure that `<IMAGE>` is replaced with the full name of the image built with `docker`. Apply these files.

```
# Note: this will (intentionally) only apply top-level files in k8s and will not recursively apply files in subdirectories.
kubectl apply -f k8s/
```

5. Update the database schema to current version by replacing the template variables in `tasks/migration-upgrade-bare-pod.yaml` and then applying it:

```
kubectl apply -f k8s/tasks/migration-upgrade-bare-pod.yaml
```

Ensure this task completed by running `kubectl get pod -n <app-name> db-upgrade-job`. Once it has completed, delete it:

```
kubectl delete pod -n <app-name> db-upgrade-job
```

If using `ManagedAuth`, replace the template variables in `tasks/add-auth-client-bare-pod.yaml`. Apply this file, check it for completion, and delete it. Repeat for each client ID.

6. Expose your application to the world by replacing the template variables in `ingress/nginx-https.yaml` and applying it.

### Files to check in to version control

Obfuscate any secret values - those in `config/secrets.yaml` and `tasks/add-auth-client-bare-pod.yaml` and check in all files in `k8s`.

### Updating the Application

For application updates that do not require database schema changes, build the Docker image and push it to the registry with `gcloud`. If you are tagging images correctly, just set the image of your deployment:

```
kubectl set image deployment/api-deployment <app-name>=gcr.io/<project-id>/<app-name>:<tag> -n <app-name>
```

If you are reusing the latest tag, delete all *pods* in the `api-deployment`.

```
kubectl delete pod api-deployment-xxxxxxx-xxxxx -n <app-name>
```

The pods will automatically be recreated and will pull the most recent image.

If a database update is required, make sure to run `aqueduct db generate` before building and pushing the Docker image. Then follow the instructions in step 5 before deleting pods.

*Note: This scheme is useful for development. When deploying to production, a unique tag should be used for each image and that image name should be added to the `api-deployment-and-service.yaml`. Instead of deleting pods, re-apply this configuration file:

```
kubectl apply -f k8s/api-deployment-and-service.yaml
```



