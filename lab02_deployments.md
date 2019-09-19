# Deployments

To run an application on Kubernetes, you need to provide it one or more  containers which power your workload. Historically, k8s has had several different methods of describing how to get that container in the cluster. In recent versions, however, many of those methods have converged into one k8s object type, a *Deployment*.

A Deployment describes how what containers to run for the application, how many copies, where, and more. Deployments are used as templates to create instances of your application in the cluster, known as **pods**. When you modify the deployment, Kubernetes will update or re-crete the pods to match the new definition.

In this lab we'll:
* Create a new deployment by writing a YAML file.
* Use `kubectl` to *apply* that file to our cluster.
* Inspect the application as it's deployed, edit it, and delete it.
* Create a service defintion with which to access the service externally.

## Writing a definition

While you can create a deployment definition directly using kubectl, it's often better to write a YAML file containing the definition locally. This way, you can add it to a git repository or other version control system.

1. Using a text editor of your choice, create a new text file called `web.yml`.
2. Create the basic structure of the definition as follows:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - image: ten7/flight-deck-drupal:latest
          name: web
          ports:
            - containerPort: 80
```
3. Save the file. You may want to create a directory for other files you'll create throughout this class.
4. Use `kubectl` to apply the file to your cluster:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/web.yml
```
5. The `kubectl` command will validate your definition prior deployment. If there are errors, go back, edit the file, and try to apply it again.

## Inspect the deployment

Now that the deployment has been applied to the cluster, we can use `kubectl` to interact with it.

1. List all deployments on the cluster:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    1/1     1            1           8m52s
```
2. Like with nodes, we can also describe the deployment:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" describe deployment web
```
3. Note the output, particularly the **events** section at the bottom.

## Edit the deployment

The `kubectl apply` command can be used multiple times against the same definitions, it doesn't need to only be used for creation. Sometimes, however, we need to modify a Kubernetes definition directly to make a critical change or to try out different configuration options to solve a problem. For this, we can use `kubectl edit`.

1. Edit the deployment definition:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" edit deployment web
```
2. Notice that in addition to what was in your file, multiple additional items and formatting has been applied to your deployment definition. This is normal.
3. Locate the `replicas` key. Change the value from `1` to `3`.
4. Save the file.
5. List the deployments again, this time you'll notice that multiple items are listed as `READY`:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    3/3     3            3           35m
```

## Work with underlying pods

A Deployment acts as a template in Kubernetes to create one or more instances of your application. These "pods" are the real workload of your application and are made of one or more running containers. We can view the underlying pods using `kubectl get pods`:

1. List the pods in your cluster:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-56c74df886-46wcf   1/1     Running   0          37m
web-56c74df886-gpf9c   1/1     Running   0          3m13s
web-56c74df886-kz95s   1/1     Running   0          3m13s
```
2. Notice that there are three pods, each starting with our deployment name (`web`). There's one pod for each replica in our Deployment.
3. Edit your deployment again, changing the `replicas` to `2`. Immediately list the pods again:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" edit deployment web
deployment.extensions/web edited
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods
NAME                   READY   STATUS        RESTARTS   AGE
web-56c74df886-46wcf   1/1     Running       0          46m
web-56c74df886-gpf9c   1/1     Running       0          12m
web-56c74df886-kz95s   0/1     Terminating   0          12m
```
4. Notice that now one of the pods is listed as `Terminating`. If you get the pods again, you may notice it no longer exists:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-56c74df886-46wcf   1/1     Running   0          48m
web-56c74df886-gpf9c   1/1     Running   0          13m
```
5. Let's try something fun -- let's *delete* a pod outright. Choose a pod from the list, then delete it:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" delete pod web-56c74df886-46wcf
```
6. Now list the pods again:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-56c74df886-gpf9c   1/1     Running   0          15m
web-56c74df886-m27zq   1/1     Running   0          18s
```
7. Didn't we delete that pod? We did! Notice, the pod name you deleted is no longer in the list. Instead, k8s replaced it with a new pod so our deployment remained consistent.

## Entering into the pod

## Creating a service endpoint

## Installing Drupal
