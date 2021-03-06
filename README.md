# prometheus-support

Prometheus configuration for M-Lab.

# Deploying Prometheus to Kubernetes

The following instructions presume there is already a kubernetes cluster
created in a GCP project and the cluster is accessible with `kubectl`.

See the [container engine quickstart guide][quickstart] for a simple howto.

[quickstart]: https://cloud.google.com/container-engine/docs/quickstart

# Using Kubernetes config files

Kubernetes config files preserve a deployment configuration and provide a
convenient mechanism for review and automated deployments.

Some steps cannot be automated. For example, while a LoadBalancer can
automatically assign a public IP address to a service, it will not ([yet][dns])
update corresponding DNS records for that IP address. So, we must reserve a
static IP through the Cloud Console interface first.

*Also, note*: Only one GKE cluster at a time can use the static IPs allocated
in the `k8s/.../services.yml` files. If you are using an additional GKE cluster
(e.g.  in mlab-sandbox project), create a new services.yml file that uses the
new static IP allocation.

[dns]: https://github.com/kubernetes-incubator/external-dns

## ConfigMaps

Many services like prometheus provide canonical docker images published to
[Dockerhub][dockerhub] (or other registry). We can customize the deployment by
changing the configuration at run time using ConfigMaps. For detailed background
see the [official docs][configmaps].

Create the ConfigMap for prometheus (only create a ConfigMap for one of cluster
or federation, not both):

    kubectl create configmap prometheus-federation-config \
        --from-file=config/federation/prometheus
    kubectl create configmap prometheus-cluster-config \
        --from-file=config/cluster/prometheus

Although the flag is named `--from-file`, it accepts a directory. With this
flag, kubernetes creates a ConfigMap with keys equal to the filenames, and
values equal to the content of the file.

    kubectl describe configmap prometheus-cluster-config

      Name:       prometheus-cluster-config
      Namespace:  default
      Labels:     <none>
      Annotations:    <none>

      Data
      ====
      prometheus.yml: 9754 bytes

We can now refer to this ConfigMap in the "deployment" configuration later. For
example, k8s/prometheus.yml declares the prometheus configuration as a volume so
that the file `prometheus.yml` appears under `/etc/prometheus`.

For example this will look something like (with abbreviated configuration):

    - containers:
      ...
        volumeMounts:
          # /etc/prometheus/prometheus.yml should contain the M-Lab Prometheus
          # config.
          - mountPath: /etc/prometheus
            name: prometheus-config
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-cluster-config

Note: Configmaps only support text data. Secrets may be an alternative for
binary data. https://github.com/kubernetes/kubernetes/issues/32432

[dockerhub]: https://hub.docker.com/r/prom/prometheus/
[configmaps]: https://kubernetes.io/docs/user-guide/configmap/

### Verify that a ConfigMap is Mounted

When a pod has mounted a configmap (or other resource), it is visible in the
"Volume Mounts" status reported by `kubectl describe`.

For example (with abbreviated output):

    podname=$( kubectl get pods -o name --selector='run=prometheus-server' )
    kubectl describe ${podname}
      ...
      Containers:
        prometheus-server:
          ...
          Image:              prom/prometheus:v1.5.2
          ...
          Port:               9090/TCP
          ...
          Volume Mounts:
            /etc/prometheus from prometheus-cluster-config (rw)
            /legacy-targets from prometheus-storage (rw)
            /prometheus from prometheus-storage (rw)
          ...

### Update a ConfigMap

When the content of a configmap value needs to change, you can either delete and
create the configmap object (not ideal), or replace the new configuration all at
once.

    kubectl create configmap prometheus-cluster-config --from-file=prometheus \
        --dry-run -o json | kubectl apply -f -

After updating a configmap, any pods that use this configmap will need to be
restarted for the change to take effect.

    podname=$( kubectl get pods -o name --selector='run=prometheus-server' )
    kubectl delete ${podname}

The deployment replica set will automatically recreate the pod and the new
prometheus server will use the updated configmap. This is a known issue:
https://github.com/kubernetes/kubernetes/issues/13488

# Using Kubernetes Secrets

Secrets are like ConfigMaps in that they can be a source for environment
variables and volume mounts. Unlike ConfigMaps, secrets contain confidential
material, like certificates, passwords, access tokens, or similar.

## Grafana secrets

The Grafana configuration needs a pre-defined password for the admin user.
Create one using a command like this:

```
kubectl create secret generic grafana-secrets \
    --from-literal=admin-password=[redacted text]
```

To recover the password:

```
jsonpath='{.items[?(@.metadata.name=="grafana-secrets")].data.admin-password}'
kubectl get secrets -o jsonpath="${jsonpath}" | base64 --decode && echo ''
```

# Create deployment

Before beginning, verify that you are [operating on the correct kubernetes
cluster][cluster].

Then, update k8s/prometheus.yml to reference the current stable prometheus
container tag. Now, deploy the service.

Create a storage class for GCE persistent disks:

    kubectl create -f k8s/storage-class.yml

Create a persistent volume claim that Prometheus will bind to:

    kubectl create -f k8s/persistent-volumes.yml

Note: Persistent volume claims are intended to exist longer than pods. This
allows persistent disks to be dynamically allocated and preserved across pod
creations and deletions.

Create a service using the public IP address that will send traffic to pods
with the label "run=prometheus-server":

    kubectl create -f k8s/mlab-sandbox/<cluser-name>/services.yml

Create the node-exporter daemonset. A [DaemonSet][daemonset] ensures that all
nodes in the cluster run a copy of a pod. Our daemonset will run one instance of
the node-exporter on every node.

    kubectl create -f k8s/node-exporter-daemonset.yml

## Cluster deployment

The cluster deployment should be the default configuration, unless you know that
you need to setup the federation deployment. The cluster and federation
deployments are mutually exclusive.

Create the prometheus cluster deployment. This step starts the actual prometheus
server. The deployment will receive traffic from the service defined above and
binds to the persistent volume claim. If a persistent volume does not already
exist, this will create a new one. It will be automatically formatted.

    kubectl create -f k8s/cluster/prometheus.yml

## Federation deployment

The federation deployment is a super-set of the prometheus cluster deployment.
It is designed to monitor the local cluster as well as aggregate metrics from
other prometheus clusters. The cluster and federation deployments are mutually
exclusive.

    kubectl create -f k8s/federation/prometheus.yml

## Check deployment

After completing the above steps, you can view the status of all objects using
something like:

    kubectl get services,deployments,pods,configmaps,secrets,pvc,pv,events

`kubectl get` is your friend. See also `kubectl describe` for even more details.

[cluster]: https://cloud.google.com/container-engine/docs/clusters/operations
[setup]: #grafana-setup
[daemonset]: https://kubernetes.io/docs/admin/daemons/

# Custom Targets

With the M-Lab configuration, we can add or remove two kinds of targets at
runtime: legacy and federation targets.

Using the file service discovery configuration, we create a JSON or YAML input
file in the [correct form][file_sd_config], and copy the file into the pod
filesystem.

For example:

```
[
    {
        "labels": {
            "service": "sidestream"
        },
        "targets": [
            "npad.iupui.mlab4.mia03.measurement-lab.org:9090"
        ]
    }
]
```

Copy file(s) to the correct directory in the prometheus pod.

    DIRECTORY=/legacy-targets
    podname=$( kubectl get pods -o name --selector='run=prometheus-server' )
    kubectl cp <filename.json> ${podname##*/}:${DIRECTORY}

To look at the available files the target directory:

    kubectl exec -c prometheus-server -t ${podname##*/} -- /bin/ls -l ${DIRECTORY}

Within five minutes, any file ending with `*.json` or `*.yaml` will be scanned
and the new targets should be reported by the prometheus server.

[file_sd_config]: https://prometheus.io/docs/operating/configuration/#file_sd_config

## Legacy Targets

For legacy targets (e.g. sidestream), copy the JSON file into the prometheus
container under the `/legacy-targets` directory.

In the Prometheus server, targets are listed under:

 * Status -> Targets -> "legacy-targets"

## Federation Targets

For federation targets (i.e. other prometheus services), copy the file to the
prometheus container under the `/federation-targets` directory.

In the Prometheus server, targets are listed under:

 * Status -> Targets -> "federation-targets"

# Delete deployment

Delete the prometheus deployment:

    kubectl delete -f k8s/cluster/prometheus.yml

Or,

    kubectl delete -f k8s/federation/prometheus.yml

Since the prometheus pod is no longer running, clients connecting to the public
IP address will try to load but fail. If we also delete the service, then
traffic will stop being forwarded from the public IP altogether.

    kubectl delete -f k8s/mlab-sandbox/<cluster name>/services.yml

Even if the prometheus deployment is not running, the persistent volume keeps
the data around. If the cluster is destroyed or if the persistent volume claim
is deleted, the automatically created disk image will be garbage collected and
deleted. At that point all data will be lost.

    kubectl delete -f k8s/persistent-volumes.yml

Delete the storage class.

    kubectl delete -f k8s/storage-class.yml

Delete the node exporter daemonset.

    kubectl delete -f k8s/node-exporter-daemonset.yml

ConfigMaps are managed explicitly for now:

    kubectl delete configmap prometheus-cluster-config
    kubectl delete configmap prometheus-federation-config

Now, `kubectl get` should not include any of the above objects.

    kubectl get services,deployments,pods,configmaps,pvc,pv

# Grafana

## Create

Create ConfigMaps for grafana:

    kubectl create configmap grafana-config \
        --from-file=config/federation/grafana
    kubectl create configmap grafana-env \
        --from-literal=domain=mlab-sandbox.mlab.fyi

Note: the domain may be the public IP address, if there is no DNS name yet.

Create a secret to contain the Grafana admin password:

    kubectl create secret generic grafana-secrets \
        --from-literal=admin-password=[redacted text]

Finally, Create the grafana deployment. Like the prometheus deployment, this
step starts the grafana server.

    kubectl create -f k8s/federation/grafana.yml

## Setup

For now, we must login to the Grafana web interface and add a datasource
corresponding to the local prometheus server. It may be possible to automate
this in the future. Either way, this step should not be necessary too many
times since `/var/lib/grafana` is a persistent volume.

The steps are:

 * Login to grafana as 'admin' using the password chosen above.
 * Click "Add data source."
 * Name the source, e.g. "Prometheus"
 * Select the Type as "Prometheus"
 * Use a URI corresponding to the service name, i.e.
   http://prometheus-public-service.default.svc.cluster.local:9090
   Note: this DNS name is added automatically within the kubernetes cluster.
   The name provided must be a fully qualified URL. You may also use the public
   IP or public DNS name.
 * Leave the Access as "proxy"
 * Click "Add"


## Delete

Delete the grafana deployment.

    kubectl delete -f k8s/federation/grafana.yml

Delete the grafana configmaps:

    kubectl delete configmap grafana-config
    kubectl delete configmap grafana-env

Delete the grafana secrets:

    kubectl delete secret grafana-secrets

# Alertmanager

The alertmanager configuration depends on the shared service and persistent
volume configurations. If those are already loaded, then continue to the steps
below.

Also, note: the alertmanager service will do nothing unless the prometheus
deployment includes the command line argument that directs alerts to this
instance of the alertmanager. i.e. `-alertmanager.url=http://bobloblaw.com:9093`

## Create

If you're setting up the alertmanager for the first time, copy
`config.yml.template` to create `config.yml` and update the `api_uri` entries
with real values.

Create the configmaps for alertmanager:

    kubectl create configmap alertmanager-config \
        --from-file=config/federation/alertmanager

    kubectl create configmap alertmanager-env \
        --from-literal=external-url=http://mlab-sandbox.mlab.fyi:9093

Create the alertmanager deployment.

    kubectl create -f k8s/federation/alertmanager.yml

## Delete

Delete the deployment.

    kubectl delete -f k8s/federation/alertmanager.yml

Delete the configmaps.

    kubectl delete configmap alertmanager-config
    kubectl delete configmap alertmanage-env

# Blackbox exporter

The blackbox exporter allows probes of endpoints over HTTP, HTTPS, DNS, TCP and
ICMP.

Targets are declared in an input file like legacy targets, however
blackbox targets require specifying a "module". The module name must match a
module defined in `config/federation/blackbox/config.yml`. Different modules
have different format requirements on the targets (e.g. some with ports, some
without).

```
[
    {
        "labels": {
            "module": "ssh_v4_online",
            "service": "sshalt"
        },
        "targets": [
            "mlab1.den02.measurement-lab.org:806",
            "mlab4.den02.measurement-lab.org:806"
        ]
    }
]
```

A single input file can define a list of configurations, each with a different
"module" and independent list of targets.

## Create

Create the configmaps for the blackbox exporter:

    kubectl create configmap blackbox-config \
        --from-file=config/federation/blackbox

Create the blackbox exporter deployment.

    kubectl create -f k8s/federation/blackbox.yml

## Delete

Delete the deployment.

    kubectl delete -f k8s/federation/blackbox.yml

Delete the configmaps.

    kubectl delete configmap blackbox-config


# Debugging the steps above

## Public IP appears to hang

After `kubectl get service prometheus-server` assigns a public IP, you can visit
the service at that IP, e.g. http://[public-ip]:9090. If the service appears to
hang, the docker instance may have failed to start.

Check using:

```
$ kubectl get pods
NAME                               READY     STATUS             RESTARTS   AGE
prometheus-server-2562116152-9mdrq   0/1       CrashLoopBackOff   9          25m
```

In this case, the `READY` status is "0", meaning not yet ready. And, the
`STATUS` gives a clue that the docker instance is crashing immediately after
start.

## Viewing logs

If a docker instance is misbehaving, we can view the logs reported by that
instance using `kubectl logs`. For example, in the case above, we can ask for
the logs with the pod name.

```
$ kubectl logs prometheus-server-2562116152-9mdrq
time="2017-02-01T21:02:36Z" level=info msg="Starting prometheus (version=1.5.0, branch=master, revision=d840f2c400629a846b210cf58d65b9fbae0f1d5c)" source="main.go:75"
time="2017-02-01T21:02:36Z" level=info msg="Build context (go=go1.7.4, user=root@a04ed5b536e3, date=20170123-13:56:24)" source="main.go:76"
time="2017-02-01T21:02:36Z" level=info msg="Loading configuration file /etc/prometheus/prometheus.yml" source="main.go:248"
time="2017-02-01T21:02:36Z" level=error msg="Error loading config: couldn't load configuration (-config.file=/etc/prometheus/prometheus.yml): yaml: line 2: mapping values are not allowed in this context" source="main.go:150"
```

## Viewing events

Kubernetes events are also an good source of recent actions and their status.
For example, if we tried to create the grafana deployment before defining the
grafana-secrets secret, the pods would fail to start and the events would
include a more specific error message.

```
$ kubectl get pods,events
NAME                                    READY     STATUS    RESTARTS   AGE
po/grafana-server-1476781881-fhr5z      0/1       RunContainerError   0          6m

LASTSEEN   FIRSTSEEN   COUNT     NAME                                 KIND      SUBOBJECT                         TYPE      REASON       SOURCE                                                    MESSAGE
55s        7m          29        ev/grafana-server-1476781881-fhr5z   Pod                                         Warning   FailedSync   {kubelet gke-soltesz-test-2-default-pool-7151d92d-7dd5}   Error syncing pod, skipping: failed to "StartContainer" for "grafana-server" with RunContainerError: "GenerateRunContainerOptions: secrets \"grafana-secrets\" not found"
```
