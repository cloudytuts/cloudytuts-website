---
title: "How to Deploy WordPress and MySQL on Kubernetes"
date: 2020-08-19T22:18:46-04:00
draft: false
author: serainville
tags:
    - kubernetes
    - wordpress
    - network-policies
    - mysql
description: |
    Learn how to run WordPress and MySQL on Kubernetes in production with persistent storage, network policies, configmaps, secrets, and regular backups.
repo: https://github.com/cloudytuts/kubernetes-in-action
---

WordPress continues to be the most popular content management system used by the internet. It continues to evolve with the internet and will remain a popular choice for many more years.

Running WordPress in a stable, containerized environment requires much more than just deploying the container. There are a number of additional considerations to be made.

## What's Covered
* Deploying MySQL in Kubernetes
* Deploying WordPress in Kubernetes
* Using Ingress controllers to route web traffic
* Using CronJob to run scheduled tasks, such as backups.
* Network policies to protect network services and pods
* Persistent storage to persist state for the database and WordPress
* Updating WordPress

## Getting Started
### Persistent Storage
Containers are designed to be ephemeral, which means their state is lost once the container reaches the end of its lifecycle. For a database server, this is clear not desirably, as we expect our data to continue persistenting beyond the life of our service. Otherwise, our data would be lost when a pod stops. 

### Network Policies
Network policies allow us to restrict access to our services and pods using ingress (incoming) and egress (outgoing) traffic rules. Services in a production environment should only be explicitly allowed to be accessible by the public or other services. An application that does not use a database server should not be able to connect to one, for example.

In this tutorial, we are going to implement network policies between our WordPress service and its backend database server. We are going to allow ingress traffic from WordPress to the database, and egress traffic from the database to WordPress.

### Ingress Controllers
Before a service in Kubernetes can be publicly exposed a `service` resource. The service must be either exposed via a `loadBalancer` type service or through an ingress controller, in order to support advanced service features, such as round-robin traffic to bachend pods.

For web traffic it is recommended to use Ingress Controllers. An ingress controller keeps costs down by only requiring a single cloud loadbalancer, and routing traffic to services by matching domain name and URI requests.

### CronJobs
Keeping your database and WordPress environment protected is critical in an production environment. CronJobs, which are regularly scheduled jobs that execute commands, allow us to routinely backup our data. 

Two cronjobs will be created for protecting the WordPress database and our WordPress content.

## Deploying MySQL
The following steps will show you how to deploy a single MySQL instance for your WordPress site.

### Persistent Storage
In order to support a persistent database a `PersistentVolumeClaim` needs to be defined. The following creates a writable volume claim for 10GB of storage. 

Create a new YAML filed named `wordpress-pvc.yaml` with the following contents.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

To create the `PersistentVolumeClaim` we must apply our manifest against our Kubernetes cluster using `kubectl`.
```shell
kubectl apply -f wordpress-pvc.yaml
```

### Secrets
Secrets are classified as any data that should not be publicly exposed. Certificates and passwords, for example, our common secrets required by applications. For MySQL we will need a **root** password and we most definitly do not want something as sensitive as this leaking.

There are two methods for creating secrets in Kubernetes: using the `kubectl create` command or by using a manifest file. The following example shows how to create a secret for a MySQL root password.
{{< warning >}}
**Data Exposure**: Secrets created from the command-line can be viewed in your shell's history
{{< /warning >}}

```shell
kubectl create secret wordpress-mysql --literal-string=MYSQL_ROOT_PASSWORD=super-secret-password
```

Alternatively, secrets can be defined in a manifest file. Secrets are defined under one of two keys in a secret manifest: `data` and `stringData`. Secrets defined under `data` must be base64 encoded, while secrets under `stringData` may be written in clear text. Either way, these values are stored base64 encoded in the etcd database and encrypted on disk.'

Strings can be easily base64 encoded on OSX and Linux using the following command.

```shell
echo -n "my-secret-text" | base64
```

Create a new manifest file named `wordpress-mysql-secrets.yaml` with the following contents. The value for `db.root.password` will be used to set our MySQL server's root password, and is base64 encoded. The value for our database name is in clear text, as it is less senstive to being exposed.

{{< warning >}}
**Data Exposure**: Secret manifests should not be stored in version control
{{< /warning >}}

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: wordpress-mysql
data: 
    db.root.password: c3VwZXItc2VjcmV0LXBhc3N3b3Jk
    db.password: UEBzc3cwcmQx
stringData:
    db.user: wp
```

Create the secret by applying your manifest.

```shell
kubectl apply -f wordpress-mysql-secrets.yaml
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-mysql
              key: db.root.password
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: wordpress-mysql
              key: db.name
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: wordpress-mysql
              key: db.user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-mysql
              key: db.password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```

## Deploying WordPress
### ConfigMap
The `ConfigMap` for our WordPress instance needs to define a few parameters for connecting to a backend database. In the example below, we are defining values for the database host, database name, and prefix to be used by WordPress for the database tables.

{{< note >}}
Sensitive information such as database credentials should not be added to a `ConfigMap`. Secrets should be stored in `Secret` resources.
{{< /note >}}

Create a new manifest file named `wordpress-configmap.yaml` and add the following to it.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress
data:
  db.host: wordpress-mysql
  db.name: myblog
  db.prefix: mb_
```

Create the `ConfigMap` by applying the manifest.
```shell
kubectl apply -f wordpress-configmap.yaml
```

### Secrets
The database user and password have already been defined in our `mysql-secret` secret resource above. We will pull these values in from the existing resource rather than define two secret resources.

### Persistent Storage
WordPress will need enough storage to store our installed theme, plugins, and any upload content. In the example `PersistentVolumeClaim` below will create a 10GB volume.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: wordpress
    labels:
        app: wordpress
spec:
    replicas: 1
    selector:
        matchLabels:
            app: wordpress
    template:
        metadata:
            labels:
                app: wordpress
        spec:
            containers:
                - name: wordpress
                  image: wordpress:5.5.0-php7.2-apache
                  ports:
                    - containerPort: 80
                  volumeMounts:
                  - name: wordpress-persistent-storage
                    mountPath: /var/www/html
                  env:
                  - name: WORDPRESS_DB_HOST
                    valueFrom:
                      configMapKeyRef:
                         name: wordpress
                         key: db.host
                  - name: WORDPRESS_DB_USER
                    valueFrom:
                      secretKeyRef:
                        name: wordpress-mysql
                        key: db.user
                  - name: WORDPRESS_DB_PASSWORD
                    valueFrom:
                      secretKeyRef:
                        name: wordpress-mysql
                        key: db.password
                  - name: WORDPRESS_DB_NAME
                    valueFrom:
                      configMapKeyRef:
                        name: wordpress
                        key: db.name
                  - name: WORDPRESS_TABLE_PREFIX
                    valueFrom:
                      configMapKeyRef:
                        name: wordpress
                        key: db.prefix
            volumes:
            - name: wordpress-persistent-storage
              persistentVolumeClaim:
                claimName: wp-pv-claim
```

### Service
Exposing pods directly to public traffic is possible but strongly discouraged. A pod are not expected to be stable and typically have relatively short lifespans. A service, on the other hand, is expected to be fairly static. Therefore, a service should be exposed through a Kubernetes service, rather than directly from a pod.

The following service example creates a service for our WordPress site and exposes it over TCP port 80. Create a file named `wordpress-service.yaml` and add the following contents to it.

```yaml
apiVersion: v1
kind: Service
metadata:
    name: wordpress
spec:
    selector:
        app: wordpress
    ports:
        - protocol: TCP
          port: 80
```

Create the service resource by applying the YAML manifest.

```shell
kubectl apply -f wordpress-service.yaml
```

### Ingress Controller
An Ingress controller handles traffic to web services running your Kubernetes cluster. It is essentially a reverse-proxy serve that receives HTTP and HTTPS traffic, and then routes it to backend services based on ingress rules, which are covered by `ingress` manifests (see below).

The most common Ingress controller is NGINX, with Traefik growing in popularity. You will need to understand the strengths and weaknesses of both before determining which one to us. The default ingress controller is NGINX, and it is the simplist to deploy.

Download and apply the NGINX IngressController
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/cloud/deploy.yaml
```

When the Ingress Controller is created it will request a load balancer from your cloud provider. Check your providor's pricing to understand what the extra costs our.


### Ingress
An ingress resource is used in concert with an Ingress Controller. It defines the hostnames and paths the ingress controller should route web traffic. Each host maps to a Kuberntes service and the service's port.

The following ingress manifest configures two hosts: www.myblog.com and myblog.com. Both route to our backend `wordpress` service on port 80.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: wordpress
spec:
  rules:
  - host: www.myblog.com
    http:
      paths:
      - backend:
          serviceName: wordpress
          servicePort: 80
        path: /
  - host: myblog.com
    http:
      paths:
      - backend:
          serviceName: wordpress
          servicePort: 80
```

Create the ingress resource by applying your manifest.

```shell
kubectl apply -f wordpress-ingress.yaml
```

## Network Policy
### WordPress
The only port traffic should ever reach our WordPress server is through TCP port 80. Let's write a network policy to enforce this as a rule.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: wordpress-ingress
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    ports:
    - protocol: TCP
      port: 80
```

### MySQL
Our MySQL instance should not be able to accept connections to it, unless explicitly allowed. The only application that should have network access to the MySQL instance is our WordPress blog. The following `NetworkPolicy` permits access from our `wordpress` pods to the `mysql` pod over TCP port 3306.

Create a new file named `mysql-network-policy.yaml` and add the following contents to it.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mysql
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 3306
```

Apply the manifest to create the `NetworkPolicy`.

```shell
kubectl apply -f mysql-network-policy.yaml
```


## Backups
### MySQL to Google Cloud Storage
It is important to regularly backup your WordPress database. To accomplish this task in an automated way we are going to introduce a new `CronJob`. 

The following example isn't the most elequent way of running a CronJob, but it effective in showing how a container can be used to perform actions. 

The CronJob below runs an Ubuntu container. When the the container is executed it will perform a number of actions, defined in an array of `args`. Our actions install Google's cloud-sdk to copy our backups to cloud storage, and MySQL Client to connect with our database server and backup our WordPress database.

{{< note >}}
Our example CronJob uses a Ubuntu distribution image and performs a number of task to install dependencies. While this may not be an issue for a cronjob that runs daily or monthly, a purpose-built image could be created with all of the dependent packages already installed.
{{< /note >}}

Create a secret to hold your credential's file for Google Cloud.
```shell
kubectl create secret generic gcloud-creds --from-file credentials.json
```

Create a new file named `wordpress-mysql-backup.yaml` with the following contents.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: wordpress-mysql-backup
spec:
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mysql-backup
            image: ubuntu:latest
            imagePullPolicy: IfNotPresent
            args:
              - echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
              - sudo apt-get install apt-transport-https ca-certificates gnupg
              - curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
              - sudo apt-get update && sudo apt-get install -y mysql-client google-cloud-sdk 
              - mysql -h ${DB_HOST} -u ${DB_USER} -p${DB_PASSWORD} ${DB_NAME} > ${DB_NAME}.backup.sql
              - gcloud auth activate-service-account --key-file credentials.json
              - gsutil cp *.sql gs:/my-bucket
            env:
            - name: MYSQL_DB_USER
              valueFrom:
                secretKeyRef:
                - name: wordpress-mysql-secrets
                  key: db.user
            - name: MYSQL_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                - name: wordpress-mysql-secrets
                  key: db.password
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                - name: wordpress-mysql-secrets
                  key: db.name
            volumeMounts:
            - name: gcloud-creds
              mountPath: "/tmp/credentials.json"
          volumes:
          - name: gcloud-creds
            secret:
              secretName: glcoud-creds
              defaultMode: 0400
          restartPolicy: OnFailure
```

Create the `CronJob` by applying the manifest against your cluster.

```shell
kubectl apply -f wordpress-mysql-backup.yaml
```

## Updating WordPress
Updating WordPress is done as you normally would. While your image tag version will fallout of sync, the WordPress installation files will persist. 

Updating the WordPress image will not update your installation files. It will, however, update any installed packages and configurations. For example, it will update the PHP and Apache package, which is important for security reasons. 

## Backup Content
Any production website should be backed up on a regular schedule. While the contents are protected by hardware redundancy, if you're running on a cloud platform like AWS, Azure, Digital Ocean, the files are not protected from accidental deletion or corruption.

Kubernetes' CronJobs can perform the task of scheduled backups. 
* Create a new CronJob.
* Store credentials for external storage bucket in a secret.
* Run CronJob nightly.
* Have the CronJob mount the WordPress persistent volume.
* Archive (tar|zip) the WordPress files with a unix timestamp.

{{< note >}}
The following example uses Google Cloud. Your CronJob should match the environment your Kubernetes cluster is hosted.
{{< /note >}}

Create a secret to hold your credential's file for Google Cloud.
```shell
kubectl create secret generic gcloud-creds --from-file credentials.json
```

Create a manifest for the WordPress backup CronJob.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: wordpress-file-backup
spec:
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: wordpress-backup
            image: gcr.io/google.com/cloudsdktool/cloud-sdk:latest
            imagePullPolicy: Always
            args:
              - gcloud auth activate-service-account --key-file /tmp/credentials.json
              - tar czvf wordpress-$(date +%s).tar.gz /var/www/html
              - gsutil cp wordpress-*.tar.gz gs:/my-bucket
            volumeMounts:
            - name: gcloud-creds
              mountPath: "/tmp/credentials.json"
          volumes:
          - name: gcloud-creds
            secret:
              secretName: glcoud-creds
              defaultMode: 0400
          restartPolicy: OnFailure
```

When using a specific image tag for a container we typically set `imagePullPolicy` to not pull every time a Pod is created. However, with Google Cloud, we want to ensure we are always using the latest version of cloud-sdk. And since our CronJob only runs once nightly, setting `imagePullPolicy` to `Always` will not put any significant amount of stress on our nodes.