# Statefulsets

Deployments fulfill most needs we have for running workloads on Kubernetes. Deployments, however, assume that each pod is temporary, replaceable, and arbitrarily scaleable. This is why pod names created from a Deployment each start with the Deployment name, but end in a random series of characters. Deployments are said to be "stateless" by default. Sometimes, however, we *do* need something stateful, such as a file server or a database. For that, we have Statefulsets.

## Creating a Statefulset

A Statefulset works like a Deployment in most cases. The only difference is in how it is provisioned and what capabilities are available to it. We can create Statefulsets using `kubectl`.

1. Using a text editor, create a new file named `mysql.yml`.
2. Edit the file, adding the following:
```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: mysql
  spec:
    selector:
      matchLabels:
        app: mysql
    serviceName: mysql
    replicas: 1
    template:
      metadata:
        labels:
          app: mysql
      spec:
        containers:
          - name: "db"
            image: "ten7/flight-deck-db:develop"
            ports:
              - containerPort: 3306
                name: mysql
                protocol: TCP
```
3. Save the file. Now, apply it to the cluster:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/mysql.yml
```
4. Just like every definition we've worked with thusfar, `kubectl` will validate your file before applying it. If there are errors, go back and correct them, and re-apply the file.

## Working with Statefulsets

Like a Deployment, a Statefulset acts as a template that creates one or more pods. The number of pods created is consistent with the `replicas` field in the Statefulset definition. We can work with our Statefulset using `kubectl`.

1. First, let's list the number of Statefulsets on our cluster:
```yaml
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" get statefulsets  

  NAME    READY   AGE
  mysql   1/1     19s
```
2. Now, let's list all the pods on the cluster:
```yaml
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods

  NAME                   READY   STATUS    RESTARTS   AGE
  mysql-0                1/1     Running   0          87s
  web-56c74df886-gpf9c   1/1     Running   0          20h
```
3. Notice how once applied, a pod managed from a Deployment and a pod managed from a Statefulset are the same, with one key difference: The Statefulset pod has a deterministic hostname, `mysql-0`.
4. Let's get the details for the statefulset:
```yaml
  kubectl --kubeconfig="/path/to/kubeconfig.yml" describe statefulset mysql
```
5. Examine the command output. Notice in particular the `Update strategy` and the `Events` section.
6. Edit the Statefulset, increasing the `replicas` field from `1` to `3`:
```
  kubectl --kubeconfig="/path/to/kubeconfig.yml" edit statefulset mysql
```
7. List the pods again. Notice that as expected, there are now three `mysql-` pods, each with a deterministic number:
```yaml
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods  

  NAME                   READY   STATUS    RESTARTS   AGE
  mysql-0                1/1     Running   0          16m
  mysql-1                1/1     Running   0          10s
  mysql-2                1/1     Running   0          7s
  web-56c74df886-gpf9c   1/1     Running   0          20h
```
8. Next, let's delete the `mysql-1` pod, then list the pods again.
```yaml
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" delete pod mysql-1
  pod "mysql-1" deleted

  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods

  NAME                   READY   STATUS              RESTARTS   AGE
  mysql-0                1/1     Running             0          21m
  mysql-1                0/1     ContainerCreating   0          3s
  mysql-2                1/1     Running             0          5m11s
  web-56c74df886-gpf9c   1/1     Running             0          20h
```
9. Notice that instead of creating a new pod, `mysql-3`, it recreates the original `mysql-1` pod in place. This is another key feature of Statefulsets.
10. Finally, let's scale down the Statefulset to it's original size of `1`. We can do this by using `kubectl edit`, or even `kubectl apply`, but we can also use another command:
```shell
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" scale statefulset/mysql --replicas=1

  statefulset.apps/mysql scaled
```
10. List the pods again. Notice that they are removed, not randomly, but it an explicit reverse order, and only one at a time:
```shell
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods                            

  NAME                   READY   STATUS        RESTARTS   AGE
  mysql-0                1/1     Running       0          28m
  mysql-1                1/1     Running       0          6m54s
  mysql-2                0/1     Terminating   0          12m
  web-56c74df886-gpf9c   1/1     Running       0          20h
```

## Persistent volumes

When we delete a pod, we delete everything with it, including all users, files, even databases. To prevent this, we need to create a *persistent volume* to store the data permanently. While we can allocate persistent volumes independent of any other k8s object, it's often useful to allocate them as part of a Statefulset. This way, each pod created by the Statefulset will also have the same size volume at the same mountpoints. The disks cannot be shared, but they will be preserved in the case the pod is deleted and needs to be recreated.

1. Using a text editor, edit the `mysql.yml` file we created earlier.
2. Update the Statefulset definition to the following:
```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: mysql
  spec:
    selector:
      matchLabels:
        app: mysql
    serviceName: mysql
    replicas: 1
    template:
      metadata:
        labels:
          app: mysql
      spec:
        initContainers:
          - name: "fix-pvc-permissions"
            image: "alpine:3.9"
            command:
              - "sh"
              - "-c"
              - "chown -R 1000:1000 /var/lib/mysql"
            volumeMounts:
              - mountPath: /var/lib/mysql
                name: vol-mysql
        containers:
          - name: "db"
            image: "ten7/flight-deck-db:develop"
            ports:
              - containerPort: 3306
                name: mysql
                protocol: TCP
            volumeMounts:
              - mountPath: /var/lib/mysql
                name: vol-mysql
        volumes:
          - name: vol-mysql
            persistentVolumeClaim:
              claimName: mysql
    volumeClaimTemplates:
      - metadata:
          name: vol-mysql
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
```
3. Save the file, apply it to the cluster.
```shell
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/mysql.yml

  The StatefulSet "mysql" is invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'template', and 'updateStrategy' are forbidden
```
4. Wait, what? Why can't we update it? It turns out that this is another key feature of Statefulsets; once created, they may only be modified in a few specific ways.
5. To make the change we need, we need to delete the Statefulset first:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" delete statefulset mysql
```
6. Once deleted, apply the `mysql.yml` file as you did before.
7. Open the DigitalOcean web portal. Navigate to **Manage** &gt; **Volumes**.
8. Notice that a new volume appears in the list. Many hosting providers will charge additionally for persistent storage, so you need to keep this in mind when choosing a storage size.
9. List our pods again, our mysql container should be running again.
```yaml
  $ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods
  NAME                   READY   STATUS    RESTARTS   AGE
  mysql-0                1/1     Running   0          107s
  web-56c74df886-gpf9c   1/1     Running   0          21h
````

## Examine databases

Now that we have a MySQL container, we should be able to access the database using `kubectl exec`.

1. Enter into the running mysql pod:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml" exec -it mysql-0 /bin/bash
````
2. Once inside the container, let's list all the disk mounts using `df -h`:
```shell
  $ df -h

  Filesystem                Size      Used Available Use% Mounted on
  overlay                  78.7G     25.3G     50.2G  34% /
  tmpfs                    64.0M         0     64.0M   0% /dev
  tmpfs                     1.9G         0      1.9G   0% /sys/fs/cgroup
  /dev/disk/by-id/scsi-0DO_Volume_pvc-de3d9fab-dbde-11e9-8a7e-daaf2df25133
                            9.8G    157.3M      9.1G   2% /var/lib/mysql
  tmpfs                     1.9G     12.0K      1.9G   0% /run/secrets/kubernetes.io/serviceaccount
```
3. If you read the list carefully, you'll notice a disk mounted at `/var/lib/mysql` of the size we specified in our Statefulset definition.
4. Enter the `mysql` command to access the database.
5. Enter `show databases;`.
6. Notice that in addition to normal MySQL databases, there's also a `drupal` database. If we examine the database, however, we'll find it empty:
```shell
  MariaDB [(none)]> show databases;
  +---------------------+
  | Database            |
  +---------------------+
  | #mysql50#db-data    |
  | #mysql50#lost+found |
  | drupal              |
  | information_schema  |
  | mysql               |
  | performance_schema  |
  +---------------------+
  6 rows in set (0.001 sec)

  MariaDB [(none)]> use drupal;
  Database changed
  MariaDB [drupal]> show tables;
  Empty set (0.000 sec)
```
7. Let's check for users. Enter `use mysql;` to change to the `mysql` database;
8. Then enter `select Host, User from user;`:
```shell
  MariaDB [drupal]> use mysql;
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A

  Database changed
  MariaDB [mysql]> select Host, User from user;
  +-----------+--------+
  | Host      | User   |
  +-----------+--------+
  | %         | drupal |
  | 127.0.0.1 | root   |
  | ::1       | root   |
  | localhost | drupal |
  | localhost | root   |
  | mysql-0   | root   |
  +-----------+--------+
  6 rows in set (0.021 sec)
```
9. Notice that there is also a `drupal` user.
10. Enter `quit;` to exit MySQL, and then `exit` to leave the pod.

## Making the database pod accessible

Like with Deployments, we also need to create a Service definition for our Statefulset. Unlike our Deployment, we do **not** need to expose the database to the public internet; it only needs to be accessible to the web pod! For this, we create a special *headless Service*.

1. Using a text editor, open the `mysql.yml` file you created earlier.
2. Add the following to the end of the file:
```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: mysql
  spec:
    clusterIP: None
    ports:
      - name: mysql
        port: 3306
        protocol: TCP
    selector:
      app: mysql
```
3. Take special note of the `---`. This is necessary so that `kubectl` separates your Statefulset from your Service definition.
4. Also note the `clusterIP` line. The `None` type is what makes this definition a headless service.
5. Use `kubectl apply` to create the Service:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/mysql.yml
```
6. List all the services on your cluster, there should now be two:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml" get services      

  NAME    TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)        AGE
  mysql   ClusterIP      None             <none>            3306/TCP       8s
  web     LoadBalancer   10.245.178.180   104.248.104.157   80:30403/TCP   5h8m
```

## Connecting to the database from the web pod

Now that we have a database backed by a persistent disk, we should be able to connect to it from our web pod. Our mysql pod by default creates a `drupal` database, with a `drupal` user. The password is `drupal` by default. Obviously we'll need to change this, but we'll do that in a future lab.

1. First, use `kubectl get pods` to get all the pods in your cluster.
2. Note the name of the `web-` pod. Now, `exec` into it:
```shell
  kubectl --kubeconfig="/path/to/kubeconfig.yml" exec -it web-123456-abc /bin/bash
```
Where:
  * **web-123456-abc** is the name of your web pod.
3. Once inside the web pod, let's try to connect to MySQL remotely:
```shell
mysql -u drupal -p -h mysql-0
```
4. When prompted, enter the password as `drupal`:
```shell
  $ mysql -u drupal -p -h mysql-0
  Enter password:

  ERROR 2005 (HY000): Unknown MySQL server host 'mysql-0' (-2)
```
5. Huh. That isn't what we expected. That's because in Kubernetes, Statefulset pods have hostnames of the form *podName.StatefulsetName*. So:
```shell
  $ mysql -u drupal -p -h mysql-0.mysql
  Enter password:

  Welcome to the MariaDB monitor.  Commands end with ; or \g.
  Your MariaDB connection id is 10
  Server version: 10.3.17-MariaDB MariaDB Server

  Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

  MariaDB [(none)]>
````

Perfect! Now we have everything we need to install Drupal using MySQL if we want. The database name, username, and password, however, aren't very secure. We'll change that in future labs.
