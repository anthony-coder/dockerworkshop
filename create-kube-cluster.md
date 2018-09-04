## Building a K8s Cluster
---

#### Prerequisites needed to replicate this lab

Before installing k8 there are some things we need for this lab:
- Ubuntu 16.04
- 2 GB or more of RAM per machine
-  2 or more CPUs for master nodes
- Installed:
    - kubeadm
    - kubectl
    - Docker Version 17.0.3 (This is the latest version that has been tested with Kubernetes)
> _Note_: We have all of the above installed on your node. If you want to replicate the lab in your own environment, you'll need to make sure you have all of the pre-reqs.

### Let's get this lab started!

First things first, let's ssh into the one of nodes that was provided to you. This will be the master of your cluster.

```bash
$ ssh training@<your ip>
```
The password will be provided by your instructor.


**Type the following commands to make sure that all the required tools/software are installed on your node:**
```bash
$ kubeadm version
$ kubectl version
$ docker version
```
> _Note_: You may receive an error when running the kubectl command, and that's okay.
---

#### Here's a quick run-down on the tools we just mentioned

#### kubeadm
This is a slick little tool that lets you get a minimum viable cluster up and running **fast**. It's more of a bootstrapper, so if you want "nice-to-have" add-ons, you're going to have to install those separately.

#### kubectl
Kubectl is a command line interface for running commands against Kubernetes clusters.

#### Docker
I'm pretty sure you guys are familiar with this piece of software. Kubernetes is an orchestrator/ container management system, so it doesn't do any container creating. It needs help from the Docker engine which has a container run-time. TLDR : Docker creates the containers, Kubernetes manages them.

---

Let's get the latest version of ``kubeadm``

```bash
sudo apt-get update && sudo apt-get upgrade
```

We'll need to **Become root** for the instantiation of the cluster. So let's do that now!

```bash
$ sudo su
```

### Configuring our network
Kubernetes requires us to configure our network in advance so that our nodes can communicate with each other. We have a number of ways we can implement our network (there are tons of network plugins). For this lab we're going to use Flannel which implements an overlay network.

Flannel leverages the Linux Bridge, so in order for Flannel to work as advertised, we need to pass bridged IPv4 traffic to iptables' chains. We can do this with the following command:

```bash
sysctl net.bridge.bridge-nf-call-iptables=1
```
Now that we've got that configured we can initialize our master node.

### Initializing the master
The master is the machine where the control plane components run (this includes etcd (the cluster db) and the API server (this is what the kubectl CLI communicates with). We're going to pass in two flags when we run our ```kubeadm init``` command.

1. ``--apiserver-advertise-address`` : This configures the IP address that the master node will advertise itself on. We're going to pass in the public IP address of the node we're sshed into.
  - We can figure out our public IP and assign it to an environment variable by querying the following endpoint. Run the following command:
  ```bash
  AAA=$(curl http://169.254.169.254/latest/meta-data/public-ipv4)
  ```
2. ``--pod-network-cidr=10.244.0.0/16`` : This flag and arg are needed for Flannel to work correctly.

Now that we've got the flags lets put it all together and initialize our cluster! (we're still ``root``)
```bash
kubeadm init \
--apiserver-advertise-address=$AAA \
--pod-network-cidr=10.244.0.0/16
```

**If everything worked out you should see the following output:**
```bash
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following commands (as a regular user):

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

    kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```
---
### ¡Importante!

To avoid using complex grep commands to fetch our join token, or having to use the ``kubeadm reset`` command, copy and paste the **kubeadm join token** that was generated as a result of the ``kubeadm init`` command. ``kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>``
 **Keep it close, keep it safe**. Paste it to a stickie. We'll be using it to join your second node to your kube cluster.

---

>_Note_ : When running the following commands, make sure that they are made as a regular user (**not root**).
Run the following command to logout as the root user:
```bash
exit
```
Lets run the commands that we received in the output:
```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
We then need to deploy our Flannel network pod :
```bash
$ kubectl apply -f \
https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```
---

## Joining Our Second Node
At this point, our single cluster should be onling. Before we join our second node, let's make sure that our cluster is indeed online, and that we can interace with it via kubectl. Running the command ``$kubectl get nodes`` should produce something similar to this output:
```bash
NAME               STATUS    ROLES     AGE       VERSION
ip-<node-ip>       Ready     <master>   5m        v1.9.5
```
#### Sweet, now it's time to join our second node!

Login to the second node.
`` ssh training@<node-ip-address>``
- Ask your instructor for the password

In order to join our kube cluster, we need to paste the ``kubeadm join --token`` we saved earlier. Before we paste it, we need to make sure we're running as ```root```, so:
```bash
$ sudo su

kubeadm join --token b6d4c3.b2bc6b133d45715e <ip-address>:6443 --discovery-token-ca-cert-hash sha256:<hash>

exit # logout of root user
```
> _Note_: Your join token will be different!

#### Congrats, you've just made a two-node kubernetes cluster!!!
However, before we can get on to some awesome demos let's make sure that we do indeed have a two node cluster.
- Run ``kubectl get nodes`` on your **Master node**. If you receive something similar to the following output, then you're A Okay!
```bash
NAME               STATUS    ROLES     AGE       VERSION
ip-172-31-49-236   Ready     <none>    3m        v1.9.5
ip-172-31-59-218   Ready     master    10m       v1.9.5
```
---
