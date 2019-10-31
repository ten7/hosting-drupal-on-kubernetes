# Configuration

When working with pods in Kubernetes, there are three basic ways to manage configuration. We could build the configuration into the pod's image, but this is limiting and results in shared credentials. We could use Docker Compose-style environment variables, but this is insecure. The last option is to mount configuration files when the pod starts. This is the most complex option, but it offers acceptable security with broad and straightforward support.

It's this latter method that Kubernetes uses to manage and provide configuration. This is accomplished through *Secrets* and *ConfigMaps*.

In this lab we'll:

* Create Secrets and Configmaps.
* Update our Deployment and Statefulset to use them.
* Observe how applying Secrets and Configmaps affects pod availability.

## Create secret for database credentials

Secrets aren't considered a part of a Deployment or a Statefulset. They are their own independent definitions which must be created and managed separately.

1. Using a text editor, create a new file `secrets.yml`.
2. Edit the file to define a new Secret. Use a **new** password of your own choosing. Do not use `drupal`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: drupal-db
type: Opaque
stringData:
  drupal-db-password.txt: "abetterpasswordthanthis"
```
3. Use `kubectl` to apply the file to the cluster:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/secrets.yml
```
4. Although our `secrets.yml` file is pretty simple, it's still subject to the same validation as Deployments, Services, and Statefulsets. If there are errors, go back and correct them, then reapply the file.
5. List all the secrets on the cluster:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get secrets

NAME                  TYPE                                  DATA   AGE
default-token-8kf7b   kubernetes.io/service-account-token   3      3d19h
drupal-db             Opaque                                1      7s
```
6. Now, edit the Secret:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" edit secret drupal-db
```
7. Notice that the password we entered earlier is no longer in plaintext. It's been base64 encoded. Also, the `stringData` key was replaced with a `data` key.
8. Close the editor.

## Update web deployment

With our Secret created, let's put it to use on our web deployment.

1. Using a text editor, open the `web.yml` file you created earlier in the class.
2. Update the `Deployment` definition to add the `volumes` section. It should be at the indent level as `containers`:
```yaml
volumes:
  - name: "vol-drupal-db"
    secret:
      secretName: "drupal-db"
```
3. Update the `Deployment` definition to add the `volumeMounts` section. It should be at the same indent level as `ports`:
```yaml
volumeMounts:
  - name: "vol-drupal-db"
    mountPath: "/config/drupal-db"
```
4. When finished, your `web.yml` file should look like this:
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
          volumeMounts:
            -  name: "vol-drupal-db"
               mountPath: "/config/drupal-db"
      volumes:
        - name: "vol-drupal-db"
          secret:
            secretName: "drupal-db"
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
5. Save the file, then apply the updated deployment to the cluster:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/web.yml
```
6. List the pods in the cluster. Notice that the pod(s) for your `web` deployment were recently recreated:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods

NAME                  READY   STATUS    RESTARTS   AGE
mysql-0               1/1     Running   0          2d10h
web-5ddcb78d8-p5lqb   1/1     Running   0          19m
```

## Examine files

At first, re-creatng the pods seems like an excessive response to a configuration change. Yet, this is an intentional behavior. Pods are meant to be ephemeral. Even Statefulset pods should be designed to survive a pod being deleted and recreated. Let's examine the new web pod to see how our Secret is presented inside the pod:

1. Examine the `web.yml` file. Notice that under `volumeMounts`, we mounted the Secret under `/config/drupal-db`.
2. Return to the list of the pods in your cluster:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods
```
3. Locate the pod starting with **web-**, and note the full pod name.
4. Enter into the web pod using `kubectl`:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" exec -it web-abcdef-12345 /bin/bash
```
Where:
  * **web-abcdef-12345** is the full web pod name.
5. Change to the directory specified by the `volumeMounts` line in `web.yml`:
```shell
$ cd /config/drupal-db/
```
6. List the contents of the directory. You should see one file:
```shell
$ ls
drupal-db-password.txt
```
7. Use `less drupal-db-password.txt` to view the contents of the file.
8. Notice that inside the container, the contents of the file are **not** encrypted or base64 encoded. This is intentional to make them easier to use for your application.
10. Press the `q` key to exit the `less` command.
11. Use `exit` to leave the pod.

## Create configmap

Typically, Secrets are used for highly secure credentials such as passwords and API keys, rather than to store entire configuration files. For the latter case, Kubernetes offers *Configmaps*. A Configmap represents one or more files inside a single directory. They aren't subject to the 1Mb size limitation or base64 encoding of Secrets, but they aren't encrypted either. Configmaps are created using `kubectl`.

Our database container, `ten7/flight-deck-db:develop`, is built to [expect a YAML file](https://github.com/ten7/flight-deck-db/blob/develop/README.md) to create users, databases, and more. Let's create a minimal Configmap to provide that file to the `mysql` Statefulset:

1. Using a text editor, create a new file `configmaps.yml`.
2. Edit the file to be the following:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
data:
  flight-deck-db.yml: |
    mysql_databases:
    - name: "drupal"
    mysql_users:
    - name: "drupal"
      host: "%"
      passwordFile: "/config/drupal-db/drupal-db-password.txt"
      priv: "drupal.*:ALL"
```
3. Notice that like Secrets, a Configmap can host multiple files; each is an item under the `data` list.
4. Apply the `configmaps.yml` file to the cluster:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/configmaps.yml
```
5. Repeat the editing process if the validation fails.
6. List all the Configmaps on the cluster:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get configmaps         

NAME    DATA   AGE
mysql   1      10s
```
7. Edit the Configmap:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" edit configmap mysql
```
8. Notice that you may edit the contents of the Configmap in plain text.
9. Close the editor.

## Add the configmap to mysql Statefulset

Mounting a Configmap inside a Statefulset is the largely the same process as adding a Secret to a Deployment:

1. Using a text editor, open the `mysql.yml` file you created earlier in the class.
2. Update the `Statefulset` definition to add the `volumes` section. It should be at the same indent level as `containers`:
```yaml
volumes:
  - name: "vol-flightdeck-db"
    configMap:
      name: "mysql"
```
3. Update the `Statefulset` definition to add the following under the `volumeMounts` section:
```yaml
- mountPath: "/config/mysql"
  name: "vol-flightdeck-db"
```
4. Save the file, but do **not** apply it yet! We have more we need to do first.

## Reusing secrets

Notice that when we created the Configmap, we didn't actually put the password for the database in the file. That's because the `ten7/flight-deck-db:develop` container can be instructed to look for the password in another file using `passwordFile`. Since we already have the password in a Secret, we're going to use the same one we created for our `web` deployment.


1. Using a text editor, open the `mysql.yml` file if you haven't already done so.
2. Update the `Statefulset` definition. Add the following under the `volumes` section:
```yaml
- name: "vol-drupal-db"
  secret:
    secretName: "drupal-db"
```
3. Update the `Statefulset` definition to add the following under the `volumeMounts` section:
```yaml
- mountPath: "/config/drupal-db"
  name: "vol-drupal-db"
```
4. When finished, the file should look like this:
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
            - mountPath: "/var/lib/mysql"
              name: "vol-mysql"
            - mountPath: "/config/mysql"
              name: "vol-flightdeck-db"
            - mountPath: "/config/drupal-db"
              name: "vol-drupal-db"
      volumes:
        - name: "vol-flightdeck-db"
          configMap:
            name: "mysql"
        - name: "vol-drupal-db"
          secret:
            secretName: "drupal-db"
  volumeClaimTemplates:
    - metadata:
        name: vol-mysql
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
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
5. Apply the changes to the cluster:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/mysql.yml
```
6. List the pods in the cluster; notice that the mysql pod is being terminated and recreated:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods

NAME                  READY   STATUS        RESTARTS   AGE
mysql-0               0/1     Terminating   0          2d13h
web-5ddcb78d8-p5lqb   1/1     Running       0          141m
```
7. After a minute or two, repeat the `get pods`. You'll notice that the container is now running again:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods

NAME                  READY   STATUS    RESTARTS   AGE
mysql-0               1/1     Running   0          56s
web-5ddcb78d8-p5lqb   1/1     Running   0          142m
```

## Configuring the web container

When we installed Drupal earlier, a new `settings.php` was generated to host configurations for the site. This includes the database credentials. Now, however, we have two new problems. Applying the Secret to the web Deployment caused the container to be recreated. With no persistent volume attached to the web pod, the `settings.php` file was lost. We also changed the database password, and now we need to fix both issues.

Fortunately, our `ten7/flight-deck-drupal:latest` container expects a YAML file as configuration. This is used by the container to create the `settings.php` with updated settings on pod startup. So, all we need to do is add a new Configmap to provide the YAML file to our `web` deployment.

1. Using a text editor, open the `configmaps.yml` file.
2. Add the following to the end of the file, noting the `---` separator:
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: drupal
data:
  flight-deck-web.yml: |
    MYSQL_NAME: "drupal"
    MYSQL_USER: "drupal"
    MYSQL_PASS_FILE: "/config/drupal-db/drupal-db-password.txt"
```
3. Save and apply the file to the cluster:
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/configmaps.yml
```
4. Using a text editor, open the `web.yml` file.
5. Update the `Deployment` definition to add the following under the `volumes` section:
```yaml
- name: "vol-drupal-config"
  configMap:
    name: "drupal"
```
3. Add the following under the `volumeMounts` section:
```yaml
- name: "vol-drupal-config"
  mountPath: "/config/web"
```
4. When finished, the file should look like this:
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
          volumeMounts:
            - name: "vol-drupal-db"
              mountPath: "/config/drupal-db"
            - name: "vol-drupal-config"
              mountPath: "/config/web"
      volumes:
        - name: "vol-drupal-db"
          secret:
            secretName: "drupal-db"
        - name: "vol-drupal-config"
          configMap:
            name: "drupal"
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
5. Apply the `web.yml` file to the cluster.
```shell
kubectl --kubeconfig="/path/to/kubeconfig.yml" apply -f /path/to/web.yml
```
6. List the pods in the cluster. As expected, the `web-` pod was recently recreated:
```shell
$ kubectl --kubeconfig="/path/to/kubeconfig.yml" get pods        

NAME                  READY   STATUS    RESTARTS   AGE
mysql-0               1/1     Running   0          25m
web-57dd9c55b-tgw6l   1/1     Running   0          28s
```

## Validate changes

With everything done, we can now do the final check. Does Drupal have the right database credentials? Let's find out!

1. Using a web browser, visit the DigitalOcean web portal.
2. Navigate to **Manage** &gt; **Networking** and open the **Load Balancers** tab.
3. Copy the IP address of your load balancer.
4. Using a web browser, open the IP address. Notice that your Drupal site is still there!
