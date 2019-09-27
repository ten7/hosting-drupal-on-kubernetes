# Interacting with Kubernetes

Kubernetes is an orchestrator which allows you to run your container-based workloads dynamically in a clustered environment. It provides provisioning, scheduling, and management so that you can focus on your workload, and not your infrastructure. The primary means we interact with Kubernetes -- often called "k8s" for short -- is the `kubectl` command. This command allows you to create, inspect, edit, and delete any object within your Kubernetes cluster.

In this lab we'll:

* Install the `kubectl` command using one of several methods.
* Create a cluster on DigitalOcean using the web portal.
* Download the kubeconfig authority file.
* Interact with your cluster using `kubectl`.

## Installing kubectl

There are several methods to install `kubectl` depending on your preference and platform. The [Kubernetes website](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-macos) specifies several different methods with which you may install `kubectl`: 

* Use a downloaded binary ("dist")
* Use a package manager such as Homebrew or Chocolatey.
* Install Docker for Mac or Docker for Windows.

Your instructor may recommend a particular method depending on your class. 

Often, however, you will want to also install Docker so that you may build and run your containers locally. Docker for Mac and Docker for Windows include `kubectl` out of the box.

On MacOS:

1. Open a web browser to the following address:
```
https://hub.docker.com/editions/community/docker-ce-desktop-mac
```
2. Click **Get Docker**.
3. Install the application as you would any MacOS app.

On Windows:

1. Open a web browser to the following address:
```
https://hub.docker.com/editions/community/docker-ce-desktop-windows
```
2. Click **Get Docker**.
3. Run the installer, follow on-screen instructions.

For Linux, typically you must install Docker and kubectl separately using your distribution's default package manager. Consult your distribution's documentation for instructions.

## Create hosting provider account or login

Kubernetes is **not** a product which creates clusters, it acts as the cluster orchestrator once installed and configured. While k8s is an open source product which can be self-hosted, you may wish to "outsource" that work to a Kubernetes-centric hosting provider.

This lab assumes we will be using DigitalOcean, although any Kubernetes hosting provider should work.

1. If so directed, use your instructor's provided login link for this class.
2. If not, navigate to `http://digitalocean.com` and follow on-screen instructions to login or create an account.

## Configure the cluster version and location

Many hosting providers will allow you to create k8s clusters using an API. This is considered best practice for highly-automated, git-driven environments. For this class, however, we'll use the hosting provider's web portal to create the cluster using the UI.

1. Using a web browser, navigate to DigitalOcean's web portal:
```
https://cloud.digitalocean.com
```
2. Using the green **Create** button at the top of the page, select **Create** &gt; **Cluster**.
3. Select a **Kubernetes Version** if so directed. By default, the newest version is selected.
4. Choose a **datacenter region**. While this can be anywhere in the world, it is best if you choose one physically proximate to where you are taking the class.
5. Keep the page open, and do **not** complete it. We will need it for the next section.

## Configure node pools

Many hosting providers will provision individual worker machines which comprise your cluster as *node pools*. Each machine ("node") in the pool has the same hardware configuration, allowing you to create sets of machines to run particular workloads.

1. Under **Choose a cluster capacity** enter a **Node Pool Name** of your choosing. You can **not** change this later.
2. Select **Standard nodes** for the **Machine type**.
3. For **Node Plan** select the smallest plan available.
4. Enter `3` for the **Number nodes** to create three machines of the selected configuration.
5. Notice the **Monthly rate** is listed.

## Create cluster

1. Scroll to the bottom of the create cluster page.
2. Under **Choose a name** enter a cluster name of your choosing, avoiding special characters.
3. Click **Create cluster**.
4. You will be directed to page displaying the cluster provisioning status. Keep this open and continue to the next section.

## Authorizing kubectl

The `kubectl` application interacts with Kubernetes through an API. As such, we often need to authenticate with that API to be certain we have the correct credentials and we are who we say we are. Different hosting providers have different authentication methods for `kubectl`, consult your hosting provider's documentation to find out which one is appropriate.

For DigitalOcean, a "kubeconfig" file may be downloaded from the web portal. This file contains the necessary authentication tokens. Note that this file also has a limited livespan. After 24 hours, credentials are often invalidated, requiring you to download a new kubeconfig file.

1. Using a web browser, login to the DigitalOcean web portal.
2. Using the sidebar, navigate to **Manage** &gt; **Kubernetes**.
3. Locate the cluster you created earlier in the list and click the cluster name.
4. Scroll down and click the **Download Config File** button. Note the download location of this file.

## List nodes

Now that we have the kubeconfig file, we can finally use `kubectl`. Even though we haven't deployed any application to the cluster yet, we can still inspect the state of the cluster by using the `get nodes` command.

1. Using the command line, enter the following:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" get nodes
```
Where:
    * **/path/to/kubeconfig.yml** is the full path to the kubeconfig file you downloaded earlier.
2. Note the command output, it should list all the worker machines ("node") provisioned when you created the cluster:
```
NAME            STATUS   ROLES    AGE     VERSION
web-pool-b64y   Ready    <none>   2d13h   v1.13.10
web-pool-b6hs   Ready    <none>   2d13h   v1.13.10
web-pool-b6hu   Ready    <none>   2d14h   v1.13.10
```
3. You may also list only the node names by using `--output=name`:
```shell
$ kubectl --kubeconfig="path/to/kubeconfig.yml" --output=name get nodes

node/web-pool-b64y
node/web-pool-b6hs
node/web-pool-b6hu
```

## Describe a node

In addition to `get`, you can also get the details of a particular node using the `describe node` command. This is useful for browsing the CPU and memory load of a particular worker machine.

1. Using the command line, list the nodes in your cluster:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" get nodes
```
2. From the list, choose a particular node name.
3. Enter the following command:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" describe node the-node-name
```
Where:
  * **the-node-name** is the name of the node.
4. Examine the command output. Note in particular the **labels**, **CPU Requests**, and **Memory Requests**.
5. Repeat the process for the other nodes as desired.

## Add a node label

In Kubernetes, individual nodes may be assigned one or more key-value pairs to act as a grouping mechanism. These labels can be used to target `kubectl` at particular pools of hardware within your cluster. You can apply labels using `kubectl`:

1. Using the command line, list the nodes in your cluster. Select a particular node to apply a label.
2. Enter the following command:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" label nodes the-node-name my-key=my-value
```
Where:
  * **the-node-name** is the name of the node.
  * **my-key** is a label key of your choosing.
  * **my-value** is a label value of your choosing.
3. Describe the node, and confirm the label is now applied.
4. Next, we'll list only the nodes in our cluster with our custom label. Enter the following, using the label key and value you just applied:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" get nodes -l my-key=my-value
```
5. Note that the output now shows only one node.
6. Apply the label to a second node and repeat the get nodes command with your custom label. Note the output.
