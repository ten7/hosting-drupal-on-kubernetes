# Deployments

To run an application on Kubernetes, you need to provide it one or more  containers which power your workload. Historically, k8s has had several different methods of describing how to get that container in the cluster. In recent versions, however, many of those methods have converged into one k8s object type, a *Deployment*.

A Deployment describes how what containers to run for the application, how many copies, where, and more. Deployments are used as templates to create instances of your application in the cluster, known as **pods**. When you modify the deployment, Kubernetes will update or re-create the pods to match the new definition.

In this lab we'll:
* Create a new deployment by writing a YAML file.
* Use `kubectl` to *apply* that file to our cluster.
* Inspect the application as it's deployed, edit it, and delete it.
* Create a service definition with which to access the service externally.

## Writing a definition

While you can create a deployment definition directly using `kubectl`, it's often better to write a YAML file containing the definition locally. This way, you can add it to a git repository or other version control system.

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

We can tell from the `kubectl` output that the pods that make up our Deployment are running, but we be sure they are functioning correctly? One way is to access them via a *remote shell*. For traditional servers, this would be something like ssh. For Kubernetes, however, we can login to individual pods using `kubectl exec`.

1. First, list the pods running on our cluster:
```shell
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods
  NAME                   READY   STATUS    RESTARTS   AGE
  web-56c74df886-gpf9c   1/1     Running   0          34m
  web-56c74df886-m27zq   1/1     Running   0          37s
```
2. Select one of the pods from the list, noting the pod name.
3. Next, "exec into" the pod:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml"  exec -it web-56c74df886-gpf9c /bin/bash
```
4. Once inside the pod, we can examine the pod using a few commands. Let's start by listing the Unix name:
```shell
  uname -a
```
5. We can also see what processes are running inside the pod:
```shell
  ps -ef
```
6. Notice that Apache is running in this container. We can even use `less` to examine the virtual host configuration:
```shell
  less /etc/apache2/sites.d/000_default.conf
```
7. Use the `q` key to exit the `less` command.
8. Let's check if Apache is actually serving some pages:
```shell
  $ curl localhost
  <!DOCTYPE html>
  <html>
      <head>
          <meta charset="UTF-8" />
          <meta http-equiv="refresh" content="0;url=/core/install.php" />

          <title>Redirecting to /core/install.php</title>
      </head>
      <body>
          Redirecting to <a href="/core/install.php">/core/install.php</a>.
      </body>
```
9. Finally, let's leave the pod and return to our local system:
```shell
  exit
```

## Creating a service endpoint

We know Apache is running inside our container, but accessing it through a `kubectl exec` and then `curl` is...cumbersome at best. What we want is to access the site through a web browser! In order to do that, we need to create a *Service definition*.

1. Using a text editor, open your the `web.yml` file you created earlier.
2. Add the following to the end of the file:
```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: web
  spec:
    ports:
      - name: http
        port: 80
        protocol: TCP
    selector:
      app: web
```
3. Take special note of the `---`. This separates our Deployment definition from our Service definition while keeping them in the same file for our convenience.
4. Save, and re-`apply` the file to our cluster:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/web.yml
```
5. Like with Deployments, `kubectl` will validate the format of our Service prior to applying it to the cluster. If there are errors, go back and correct them and try to apply again.
6. Once applied, we can list our services just as we did our deployments:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml" get services
  NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
  web    ClusterIP   10.245.46.160   <none>        80/TCP    3m13s
```
7. Use `kubectl describe` to get further details on our service:
```shell
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" describe service web
  Name:              web
  Namespace:         k8s-class
  Labels:            <none>
  Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                       {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"web","namespace":"default"},"spec":{"ports":[{"name":"http",...
  Selector:          app=web
  Type:              ClusterIP
  IP:                10.245.46.160
  Port:              http  80/TCP
  TargetPort:        80/TCP
  Endpoints:         10.244.1.234:80
  Session Affinity:  None
  Events:            <none>
```

## Making the service publicly accessible

When we `describe`d our Service, you may have noticed two IP addresses listed in the output. If you try to connect to either using your browser, however, you may not get to our pods. This is because by default, the Service allocates a "cluster IP". For many hosting providers, this address only works within the firewalled environment of the clutser, and not the public internet. To make it accessible, we need to change how the service externalizes itself.

1. Using a text edit, open the `web.yml` file we created earlier.
2. Alter the Service definition, adding `type: LoadBalancer` under the `spec`:
```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: web
  spec:
    ports:
      - name: http
        port: 80
        protocol: TCP
    selector:
      app: web
    type: LoadBalancer
```
3. Save the file. Now, apply it to the cluster:
```shell
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/web.yml
  deployment.apps/web unchanged
  service/web configured
```
4. Return to the DigitalOcean web portal. Navigate to **Manage** &gt; **Networking** and open the **Load Balancers** tab.
5. Notice that a new load balancer is being allocated to power access. This external service often will have additional cost associated with it depending on your hosting provider, so we need to create `LoadBalanacer` type Services is discretion.
6. Wait for the new load balancer to finish allocating. This may take several minutes.
7. When finished, examine the **IP Address** column, there should be a new, publicly available IP address for your load balancer.
8. Let's cross check that with `kubectl`:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml" describe service web
```
9. Notice that instead of `Cluster IP`, there's a `Load Balancer Ingress` line with the same IP address as shown in the web portal.

## Installing Drupal

Everything is set, so we should be able to install Drupal, right? Let's see what happens.

1. Using a web browser, navigate to the IP address provided by the web portal.
2. After a moment, you should be redirected to the Drupal installation page. Click `Save and Continue`.
3. Continue through the installation process until you reach the *Set up database* page.

Wait, we don't have a database! That's right, we could continue to install with SQLite, but it would be much better to have a MySQL database to power our site. We'll set that up in future labs.
