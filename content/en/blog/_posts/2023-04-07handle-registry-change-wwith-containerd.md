---
layout: blog
title: "Use containerd to handle k8s.gcr.io deprecation"
date: 2023-04-07
slug: handle-registry-deprecation-with-containerd
---


The Kubernetes community recently made the last step in another major change. Until fall of 2022, `k8s.gcr.io`
container registry was the official place to find many Kubernetes community-managed containers images;
for example, kube-proxy, the cluster autoscaler, the metrics server, or the cluster-proportional autoscaler.
The GCR in “gcr.io” is Google Cloud Registry. The Kubernetes project
[switched](/blog/2022/11/28/registry-k8s-io-faster-cheaper-ga/) in November 2022
to a new registry with a different implementation.

As a result, starting March 20th this year, traffic from the old `k8s.gcr.io` registry is being redirected to `registry.k8s.io`. The deprecated `k8s.gcr.io` registry will remain functioning for some time but may one day be shut down entirely.

{{< figure src="2023-04-07handle-registry-change-with-containerd/00-registry-deprecation-announcement.png" alt="Legacy k8s.gcr.io container image registry is being redirected to registry.k8s.io" class="diagram-large" caption="k8s.gcr.io registry redirection notice" >}}

## What’s the impact?

The change that occurred six days after Pi day is unlikely to cause major problems. There are some edge cases. But unless you operate in an airgapped or highly restrictive environment that applies strict domain name access controls, you won’t notice the change.

This doesn’t mean that there’s no action required. Now is the time to scan code repositories and clusters for usage of the old registry. Failing to act will result in cluster components failing.

Once the old registry goes away, **Kubernetes will not be able to create new Pods** (unless image is cached) if the container uses an image hosted on `k8s.gcr.io`.

## What do you need to change?

Cluster owners and development teams have to ensure they are not using any images stored in the old registry. The change is fairly simple. It's pretty simple, really.

You need to change your manifests to use the `registry.k8s.io` container registry instead of `k8s.gcr.io`.

{{< figure src="2023-04-07handle-registry-change-with-containerd/01-registry-change.png" alt="Changing image registry from k8s.gcr.io to registry.k8s.io" class="diagram-large" caption="Changing image registry from k8s.gcr.io to registry.k8s.io" >}}

You can find out which Pods use the old registry using `kubectl`:

```bash
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" |\0;31;37M0;31;38m
tr -s '[[:space:]]' '\n' |\
sort |\
uniq -c | grep -i gcr.io
```

Here are the Pods in my test cluster that use the old registry:

{{< figure src="2023-04-07handle-registry-change-with-containerd/02-images-from-gcr.png" alt="Containers using k8s.gcr.io registry" class="diagram-large" caption="Containers using k8s.gcr.io hosted images" >}}

These are the at-risk Pods. I’ll have to update the container registry used in the Pods.

When hunting for references to old registry, be sure to include containers that may not be currently running in your cluster.

## What if I don’t control the workloads?

One of my colleagues raised an intriguing question. Is there’s a way to handle this change at a cluster level?
He had a valid concern. Many large enterprises might not be able to implement this change in time before
the Kubernetes project fully sunsets `k8s.gcr.io`.

I work with many customers that manage large Kubernetes clusters, but have little control over
the workloads that get deployed into the cluster. Some of these clusters are shared by hundreds
of development teams. The burden is on central platform engineering teams to dissipate this
information to individual development teams (who are busy writing code and not checking Kubernetes news!).

So, what can these organizations' SRE, platform or infrastructure teams do to make sure when the old registry
finally croaks, they don’t get paged for in the middle of the night for `ErrImagePull` and `ImagePullBackOff`
errors?

Turns out you can use containerd to handle this redirection at node level. Let’s find out how.

## Using mirrors in containerd

Ever since Docker Hub started rate limiting image pulls, many have opted to store images in local registries. Mirrors save network bandwidth, reduce image pull time, and don’t rate-limit.

You can configure `registry.k8s.io` as a mirror to `k8s.gcr.io` in containerd. This configuration will **automatically pull images from `registry.k8s.io`** whenever a Pod uses an image stored in `k8s.gcr.io`.

On your worker node, append these lines in the containerd config file at `/etc/containerd/config.toml`:

```bash
[plugins."io.containerd.grpc.v1.cri".registry]
   config_path = "/etc/containerd/certs.d"
```

Here's an example that I tried out on an Amazon EKS cluster, which is also using
Elastic Container Registry (ECR) as a mirror.

```yaml
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"

[grpc]
address = "/run/containerd/containerd.sock"

[plugins."io.containerd.grpc.v1.cri".containerd]
default_runtime_name = "runc"

[plugins."io.containerd.grpc.v1.cri"]
sandbox_image = "602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/pause:3.5"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".cni]
bin_dir = "/opt/cni/bin"
conf_dir = "/etc/cni/net.d"

[plugins."io.containerd.grpc.v1.cri".registry]
config_path = "/etc/containerd/certs.d"
```

Next, create a directory called `k8s.gcr.io` and a `hosts.toml` file inside it:

```bash
mkdir -p /etc/containerd/certs.d/k8s.gcr.io

cat << EOF > /etc/containerd/certs.d/k8s.gcr.io/hosts.toml
server = "https://k8s.gcr.io"

[host."https://registry.k8s.io"]
capabilities = ["pull", "resolve"]
EOF
```

Image pull requests for `k8s.gcr.io` will now be sent to `registry.k8s.io`.

Restart containerd and kubelet for the change to take effect.

```bash
# This assumes that you use Linux nodes and use systemd for init
systemctl restart containerd kubelet
```

Let’s validate that images are indeed getting pulled from the new registry. I added an entry to my `/etc/hosts` file to break `k8s.gcr.io`:

{{< figure src="2023-04-07handle-registry-change-with-containerd/03-hosts-file-change.png" alt="Break k8s.gcr.io" class="diagram-large" caption="Stop traffic to k8s.gcr.io in the node" >}}

Containerd can no longer pull an image from `k8s.gcr.io`:

{{< figure src="2023-04-07handle-registry-change-with-containerd/04-registry-broken.png" alt="Conatinerd cannot pull images from k8s.gcr.io" class="diagram-large" caption="Connections k8s.gcr.io blocked at node level" >}}

I can tell `ctr` to use the mirror by specifying the `—hosts-dir` parameter:

```bash
ctr images pull --hosts-dir "/etc/containerd/certs.d" k8s.gcr.io/kube-proxy:v1.24.0
```

This time the operation succeeds.

{{< figure src="2023-04-07handle-registry-change-with-containerd/05-registry-succeeds.png" alt="Conatinerd can pull images from the mirror" class="diagram-large" caption="Containerd can pull images from the mirror" >}}

Any Pods I create now onwards will use the new registry even though the manifests reference old registry. Here’s a test using a pause container that uses an image from `k8s.gcr.io`.

```bash
kubectl create deployment pause --image k8s.gcr.io/pause
```

{{< figure src="2023-04-07handle-registry-change-with-containerd/06-cotainer works.png" alt="Kubernetes can create Pods by pulling images from mirror" class="diagram-large" caption="Kubernetes can create Pods by pulling images from mirror" >}}

Perfect! Kubernetes could create Pods even though I blocked `k8s.gcr.io` on the node.

## What’s the best way to implement this in production?

In my little demo, I changed a single node in the cluster. What about the rest of the nodes?

There are three ways you can use to implement this change on every node in your cluster:

1. Probably the easiest way is to use a DaemonSet to change to containerd `config.toml` and add a `hosts.toml` file. IBM Cloud has shared an [example](https://raw.githubusercontent.com/IBM-Cloud/kube-samples/master/containerd-registry-daemonset-example) on GitHub.
1. In the cloud, you can use your provider's server management tooling to make this change automatically when
   a node gets created.
1. You can create (_bake_) your own server or virtual machine image that includes these customizations, 
   and then launch nodes based on that image.

## What if I use Docker as runtime?

Starting Kubernetes version `1.24`, containerd is the only runtime available in Amazon EKS AMIs. If you have an edge case that requires using Docker, there's still hope.

Docker also has support for registry mirrors. [Registry as a pull through cache](https://docs.docker.com/registry/recipes/mirror/#configure-the-docker-daemon) has the details you need.

## Don’t rely on stop gaps

While the solution included in this post works, I recommend only using as a safety measure. The main reason is that you’ll need to customize the node image on your Kubernetes cluster, or your configuration management tool,
to set up this extra detail.

You’ll have less operational overhead with fewer node-level customisations. The best way to handle
this registry deprecation is to update manifests (and Helm charts, etc).

Oh, and by the way, you can also use mirrors to enforce pull through cache.
