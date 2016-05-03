---
---

These instructions are very similar to the master set-up above, but they are duplicated for clarity.
You need to repeat these instructions for each node you want to join the cluster.
We will assume that you have the IP address of the master in `${MASTER_IP}` that you created in the [master instructions](/docs/getting-started-guides/docker-multinode/master/).  We'll need to run several versioned Kubernetes components, so we'll assume that the version we want
to run is `${K8S_VERSION}`, which should hold a released version of Kubernetes >= "1.2.0-alpha.6"

Enviroinment variables used:

```shell
export MASTER_IP=<the_master_ip_here>
export K8S_VERSION=<your_k8s_version (e.g. 1.2.0-alpha.6)>
export FLANNEL_VERSION=<your_flannel_version (e.g. 0.5.5)>
export FLANNEL_IFACE=<flannel_interface (defaults to eth0)>
export FLANNEL_IPMASQ=<flannel_ipmasq_flag (defaults to true)>
```

For each worker node, there are three steps:

   * [Set up `flanneld` on the worker node](#set-up-flanneld-on-the-worker-node)
   * [Start Kubernetes on the worker node](#start-kubernetes-on-the-worker-node)
   * [Add the worker to the cluster](#add-the-node-to-the-cluster)

### Set up Flanneld on the worker node

As before, the Flannel daemon is going to provide network connectivity.

_Note_:
This guide expects **Docker 1.7.1 or higher**.


#### Set up a bootstrap docker

As previously, we need a second instance of the Docker daemon running to bootstrap the flannel networking.

Run:

```shell
sudo sh -c 'docker -d -H unix:///var/run/docker-bootstrap.sock -p /var/run/docker-bootstrap.pid --iptables=false --ip-masq=false --bridge=none --graph=/var/lib/docker-bootstrap 2> /var/log/docker-bootstrap.log 1> /dev/null &'
```

_If you have Docker 1.8.0 or higher run this instead_

```shell
sudo sh -c 'docker daemon -H unix:///var/run/docker-bootstrap.sock -p /var/run/docker-bootstrap.pid --iptables=false --ip-masq=false --bridge=none --graph=/var/lib/docker-bootstrap 2> /var/log/docker-bootstrap.log 1> /dev/null &'
```

_Important Note_:
If you are running this on a long running system, rather than experimenting, you should run the bootstrap Docker instance under something like SysV init, upstart or systemd so that it is restarted
across reboots and failures.

#### Bring down Docker

To re-configure Docker to use flannel, we need to take docker down, run flannel and then restart Docker.

Turning down Docker is system dependent, it may be:

```shell
sudo /etc/init.d/docker stop
```

or

```shell
sudo systemctl stop docker
```

or it may be something else.

#### Run flannel

Now run flanneld itself, this call is slightly different from the above, since we point it at the etcd instance on the master.

```shell
sudo docker -H unix:///var/run/docker-bootstrap.sock run -d \
    --net=host \
    --privileged \
    -v /dev/net:/dev/net \
    quay.io/coreos/flannel:${FLANNEL_VERSION} \
    /opt/bin/flanneld \
        --ip-masq=${FLANNEL_IPMASQ} \
        --etcd-endpoints=http://${MASTER_IP}:4001 \
        --iface=${FLANNEL_IFACE}
```

The previous command should have printed a really long hash, the container id, copy this hash.

Now get the subnet settings from flannel:

```shell
sudo docker -H unix:///var/run/docker-bootstrap.sock exec <really-long-hash-from-above-here> cat /run/flannel/subnet.env
```


#### Edit the docker configuration

You now need to edit the docker configuration to activate new flags.  Again, this is system specific.

This may be in `/etc/default/docker` or `/etc/systemd/service/docker.service` or it may be elsewhere.

Regardless, you need to add the following to the docker command line:

```shell
--bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```

#### Remove the existing Docker bridge

Docker creates a bridge named `docker0` by default.  You need to remove this:

```shell
sudo /sbin/ifconfig docker0 down
sudo brctl delbr docker0
```

You may need to install the `bridge-utils` package for the `brctl` binary.

#### Restart Docker

Again this is system dependent, it may be:

```shell
sudo /etc/init.d/docker start
```

or it may be:

```shell
systemctl start docker
```

### Start Kubernetes on the worker node

#### Run the kubelet

Again this is similar to the above, but the `--api-servers` now points to the master we set up in the beginning.

```shell
sudo docker run \
    --volume=/:/rootfs:ro \
    --volume=/sys:/sys:ro \
    --volume=/dev:/dev \
    --volume=/var/lib/docker/:/var/lib/docker:rw \
    --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
    --volume=/var/run:/var/run:rw \
    --net=host \
    --privileged=true \
    --pid=host \
    -d \
    gcr.io/google_containers/hyperkube-amd64:v${K8S_VERSION} \
    /hyperkube kubelet \
        --allow-privileged=true \
        --api-servers=http://${MASTER_IP}:8080 \
        --v=2 \
        --address=0.0.0.0 \
        --enable-server \
        --containerized \
        --cluster-dns=10.0.0.10 \
        --cluster-domain=cluster.local
```

#### Run the service proxy

The service proxy provides load-balancing between groups of containers defined by Kubernetes `Services`

```shell
sudo docker run -d \
    --net=host \
    --privileged \
    gcr.io/google_containers/hyperkube-amd64:v${K8S_VERSION} \
    /hyperkube proxy \
        --master=http://${MASTER_IP}:8080 \
        --v=2
```

### Next steps

Move on to [testing your cluster](/docs/getting-started-guides/docker-multinode/testing/) or [add another node](#adding-a-kubernetes-worker-node-via-docker)
