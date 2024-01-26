# Challenge Documentation
## First Level

Setting up a MySQL Replication Cluster along with the exporter serv- [Challenge Documentation](#challenge-documentation)
- [Challenge Documentation](#challenge-documentation)
  - [First Level](#first-level)
    - [Step 1: the values file](#step-1-the-values-file)
    - [Step 2: the manifest file](#step-2-the-manifest-file)
    - [Step 3: testing the cluster](#step-3-testing-the-cluster)
  - [Tech](#tech)
  - [Installation](#installation)
  - [Plugins](#plugins)
  - [Development](#development)
      - [Building for source](#building-for-source)
  - [Docker](#docker)
  - [License](#license)

Steps:

1. configuring the values files for installing the package
2. using the generated manifest file 
2. testing the cluster
3. ✨Magic ✨

### Step 1: the values file

First up is setting up a couple of important values in the values file as is shown below:


| Key                       | Value        | Description                                                                         |
| ------------------------- | ------------ | ----------------------------------------------------------------------------------- |
| architecture              | replication  | database architecture type which in our case is going to be replication             |
| auth.createDatabase       | true         | to create a database with the name provided in auth.database when creating the pod  |
| auth.database             | "TestDB"     |
| auth.username             | testuser     | similar to auth.database , when provided will create a user with the specified name |
| auth.password             | "123456"     |
| auth.replicationUser      | Replicator   | the user to use for the replication service, created on runtime                     |
| auth.replicationPassword  | "123456"     |
| metrics.enabled           | true         | whether to enable the metrics service using mysql exporter                          |
| metrics.service.clusterIP | LoadBalancer | to assign a public ip for remote access to the exporter service                     |


next up is assigning the resources we need to these pods in the same values file,these resources need to be set under both the primary and the secondary instance:


```yaml
  ## metrics.resources.limits  The resources limits for MySQL prometheus exporter containers
  ## metrics.resources.requests The requested resources for MySQL prometheus exporter containers
resources:
    limits:     
       cpu: "2"
       memory: 4Gi
    requests:
       cpu: "2"
       memory: 4Gi
```

we won't be assigning resources to the metrics service.


now adding the above value file to ArvanCloud Container setup page will generate a Kuber manifest file which we will make some adjusments to and use for orchestrating the cluster.(the manifest file can be obtained through helm as well.)

### Step 2: the manifest file

**the genereated file is going to consist of the following:**

* **serviceaccount/arvandb-mysql** 
    -  containing the serviceaccount information
  
        ```yaml

            apiVersion: v1
            kind: ServiceAccount
            metadata:
            name: arvandb-mysql
            namespace: arvanchallenge
            labels:
                app.kubernetes.io/instance: arvandb
                app.kubernetes.io/managed-by: Helm
                app.kubernetes.io/name: mysql
                app.kubernetes.io/version: 8.0.36
                helm.sh/chart: mysql-9.18.0
            automountServiceAccountToken: false
            secrets:
            - name: arvandb-mysql
        ```

* **secret/arvandb-mysql**
    + contains the following values in the secret( the secret must contain the `mysql-replication-password` parameter to ensure the replication cluster functions ): 
      
      - `mysql-root-password`: root user password
      - `mysql-password`: password for the test user we created earlier
      - `mysql-replication-password`: password to use for the replication service
        
      ```yaml
            apiVersion: v1
            kind: Secret
            metadata:
            name: arvandb-mysql
            namespace: arvanchallenge
            labels:
                app.kubernetes.io/instance: arvandb
                app.kubernetes.io/managed-by: Helm
                app.kubernetes.io/name: mysql
                app.kubernetes.io/version: 8.0.36
                helm.sh/chart: mysql-9.18.0
            type: Opaque
            data:
            mysql-root-password: test
            mysql-password: test
            mysql-replication-password: test
      ```




* **configmap/arvandb-mysql-primary**
    - config file for the primary mysql node containing information like server port, server bind address etc..

      ```yaml
        apiVersion: v1
        kind: ConfigMap
        metadata:
        name: arvandb-mysql-primary
        namespace: arvanchallenge
        labels:
            app.kubernetes.io/instance: arvandb
            app.kubernetes.io/managed-by: Helm
            app.kubernetes.io/name: mysql
            app.kubernetes.io/version: 8.0.36
            helm.sh/chart: mysql-9.18.0
            app.kubernetes.io/component: primary
        data:
        my.cnf: |-
            [mysqld]
            default_authentication_plugin=mysql_native_password
            skip-name-resolve
            explicit_defaults_for_timestamp
            basedir=/opt/bitnami/mysql
            plugin_dir=/opt/bitnami/mysql/lib/plugin
            port=3306
            socket=/opt/bitnami/mysql/tmp/mysql.sock
            datadir=/bitnami/mysql/data
            tmpdir=/opt/bitnami/mysql/tmp
            max_allowed_packet=16M
            bind-address=*  
            pid-file=/opt/bitnami/mysql/tmp/mysqld.pid
            log-error=/opt/bitnami/mysql/logs/mysqld.log
            character-set-server=UTF8
            slow_query_log=0
            long_query_time=10.0

            [client]
            port=3306
            socket=/opt/bitnami/mysql/tmp/mysql.sock
            default-character-set=UTF8
            plugin_dir=/opt/bitnami/mysql/lib/plugin

            [manager]
            port=3306
            socket=/opt/bitnami/mysql/tmp/mysql.sock
            pid-file=/opt/bitnami/mysql/tmp/mysqld.pid
      ```

* **configmap/arvandb-mysql-secondary**
    - similar to primary config map

* **service/arvandb-mysql-metrics**
    - service for making the metrics exporter available , configured with a LoadBalancer IP on port 9104.
    
      ```yaml
           
            apiVersion: v1
            kind: Service
            metadata:
            name: arvandb-mysql-metrics
            namespace: arvanchallenge
            labels:
                app.kubernetes.io/instance: arvandb
                app.kubernetes.io/managed-by: Helm
                app.kubernetes.io/name: mysql
                app.kubernetes.io/version: 8.0.36
                helm.sh/chart: mysql-9.18.0
                app.kubernetes.io/component: metrics
            annotations:
                prometheus.io/port: "9104"
                prometheus.io/scrape: "true"
            spec:
            type: LoadBalancer
            ports:
                - port: 9104
                targetPort: metrics
                protocol: TCP
                name: metrics
            selector:
                app.kubernetes.io/instance: arvandb
                app.kubernetes.io/name: mysql                
      ```

* **service/arvandb-mysql-primary-headless**
    - Kubernetes service that does not assign an IP address to itself. instead returns the ip address of the pod it is attached to.

      ```yaml
        apiVersion: v1
        kind: Service
        metadata:
        name: arvandb-mysql-primary-headless
        namespace: arvanchallenge
        labels:
            app.kubernetes.io/instance: arvandb
            app.kubernetes.io/managed-by: Helm
            app.kubernetes.io/name: mysql
            app.kubernetes.io/version: 8.0.36
            helm.sh/chart: mysql-9.18.0
            app.kubernetes.io/component: primary
        spec:
        type: ClusterIP
        clusterIP: None
        publishNotReadyAddresses: true
        ports:
            - name: mysql
            port: 3306
            targetPort: mysql
        selector:
            app.kubernetes.io/instance: arvandb
            app.kubernetes.io/name: mysql
            app.kubernetes.io/component: primary
      ```
* **service/arvandb-mysql-primary**
    - service for the priamry node exposing the service on port 3306 in the cluster

      ```yaml
        apiVersion: v1
        kind: Service
        metadata:
        name: arvandb-mysql-primary
        namespace: arvanchallenge
        labels:
            app.kubernetes.io/instance: arvandb
            app.kubernetes.io/managed-by: Helm
            app.kubernetes.io/name: mysql
            app.kubernetes.io/version: 8.0.36
            helm.sh/chart: mysql-9.18.0
            app.kubernetes.io/component: primary
        spec:
        type: ClusterIP
        sessionAffinity: None
        ports:
            - name: mysql
            port: 3306
            protocol: TCP
            targetPort: mysql
            nodePort: null
        selector:
            app.kubernetes.io/instance: arvandb
            app.kubernetes.io/name: mysql
            app.kubernetes.io/component: primary   
      ```

* **service/arvandb-mysql-secondary-headless**
    - similar to primary
* **service/arvandb-mysql-secondary**
    - similar to primary

* **statefulset.apps/arvandb-mysql-primary** 
    - the statefulset for setting up the primary pod along with its metric exporter, the use of a stateful set instead of deployment to ensure:
        
        1. the ordering and uniqueness for the mysql and its exporter are respected
        2. the storage claim is persistent across this pod and any that would replace it

    - certain values in this stateful set are required for the replication cluster to work correctly:
      + `MYSQL_REPLICATION_MODE`: which should be set to master in the primary statefulset                          
      + `MYSQL_REPLICATION_USER`: as it was set in the values files       
      + `MYSQL_REPLICATION_PASSWORD`: which is taken from the secret defined at the begining
      ```yaml
        apiVersion: apps/v1
        kind: StatefulSet
        metadata:
        name: arvandb-mysql-primary
        namespace: arvanchallenge
        labels:
            app.kubernetes.io/instance: arvandb
            app.kubernetes.io/managed-by: Helm
            app.kubernetes.io/name: mysql
            app.kubernetes.io/version: 8.0.36
            helm.sh/chart: mysql-9.18.0
            app.kubernetes.io/component: primary
        spec:
        replicas: 1
        podManagementPolicy: ""
        selector:
            matchLabels:
            app.kubernetes.io/instance: arvandb
            app.kubernetes.io/name: mysql
            app.kubernetes.io/component: primary
        serviceName: arvandb-mysql-primary
        updateStrategy:
            type: RollingUpdate
        template:
            metadata:
            annotations:
                checksum/configuration: 2688c511fffe3429037df6855884a0838cf3dce40e2767364a7b05627e39c3d9
            labels:
                app.kubernetes.io/instance: arvandb
                app.kubernetes.io/managed-by: Helm
                app.kubernetes.io/name: mysql
                app.kubernetes.io/version: 8.0.36
                helm.sh/chart: mysql-9.18.0
                app.kubernetes.io/component: primary
            spec:
            serviceAccountName: arvandb-mysql
            automountServiceAccountToken: false
            affinity:
                nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                    nodeSelectorTerms:
                    - matchExpressions:
                        - key: node-role.kubernetes.io/cloud-container-g2
                            operator: In
                            values:
                            - "true"
            tolerations:
                - effect: NoSchedule
                key: role
                operator: Equal
                value: cloud-container-g2
            securityContext:
                fsGroup: 1001
                fsGroupChangePolicy: Always
                supplementalGroups: []
                sysctls: []
            initContainers: null
            containers:
                - name: mysql
                image: docker.io/bitnami/mysql:8.0.36-debian-11-r0
                imagePullPolicy: IfNotPresent
                securityContext:
                    allowPrivilegeEscalation: false
                    capabilities:
                    drop:
                        - ALL
                    runAsNonRoot: true
                    runAsUser: 1001
                    seLinuxOptions: {}
                    seccompProfile:
                    type: RuntimeDefault
                env:
                    - name: BITNAMI_DEBUG
                    value: "false"
                    - name: MYSQL_ROOT_PASSWORD
                    valueFrom:
                        secretKeyRef:
                        name: arvandb-mysql
                        key: mysql-root-password
                    - name: MYSQL_USER
                    value: testuser
                    - name: MYSQL_PASSWORD
                    valueFrom:
                        secretKeyRef:
                        name: arvandb-mysql
                        key: mysql-password
                    - name: MYSQL_DATABASE
                    value: my_database
                    - name: MYSQL_REPLICATION_MODE
                    value: master
                    - name: MYSQL_REPLICATION_USER
                    value: replicator
                    - name: MYSQL_REPLICATION_PASSWORD
                    valueFrom:
                        secretKeyRef:
                        name: arvandb-mysql
                        key: mysql-replication-password
                envFrom: null
                ports:
                    - name: mysql
                    containerPort: 3306
                resources:
                    limits:
                    cpu: "2"
                    memory: 4Gi
                    requests:
                    cpu: "2"
                    memory: 4Gi
                volumeMounts:
                    - name: data
                    mountPath: /bitnami/mysql
                    - name: config
                    mountPath: /opt/bitnami/mysql/conf/my.cnf
                    subPath: my.cnf
                - name: metrics
                image: docker.io/bitnami/mysqld-exporter:0.15.1-debian-11-r2
                imagePullPolicy: IfNotPresent
                securityContext:
                    runAsNonRoot: true
                    runAsUser: 1001
                    seLinuxOptions: {}
                env:
                    - name: MYSQL_ROOT_PASSWORD
                    valueFrom:
                        secretKeyRef:
                        name: arvandb-mysql
                        key: mysql-root-password
                command:
                    - /bin/bash
                    - -ec
                    - >
                    password_aux="${MYSQL_ROOT_PASSWORD:-}"

                    if [[ -f "${MYSQL_ROOT_PASSWORD_FILE:-}" ]]; then
                        password_aux=$(cat "$MYSQL_ROOT_PASSWORD_FILE")
                    fi

                    MYSQLD_EXPORTER_PASSWORD=${password_aux} /bin/mysqld_exporter --mysqld.address=localhost:3306 --mysqld.username=root
                ports:
                    - name: metrics
                    containerPort: 9104
                resources:
                    limits: {}
                    requests: {}
            volumes:
                - name: config
                configMap:
                    name: arvandb-mysql-primary
        persistentVolumeClaimRetentionPolicy:
            whenDeleted: Retain
            whenScaled: Retain
        volumeClaimTemplates:
            - metadata:
                name: data
                labels:
                app.kubernetes.io/instance: arvandb
                app.kubernetes.io/name: mysql
                app.kubernetes.io/component: primary
            spec:
                accessModes:
                - ReadWriteOnce
                resources:
                requests:
                    storage: 20Gi
      ```
* **statefulset.apps/arvandb-mysql-secondary**
    - similar to primary with a few exceptions:
      - `MYSQL_REPLICATION_MODE`: should have its value set to  "slave"
      - `MYSQL_MASTER_HOST`: the name of our primary pod in this case  *arvandb-mysql-primary*
      - `MYSQL_MASTER_PORT_NUMBER`
        value: "3306"
      - `MYSQL_MASTER_ROOT_USER`
        value: root
      - `MYSQL_REPLICATION_USER`: the replication user defined previously
      - `MYSQL_MASTER_ROOT_PASSWORD`: set in the secret
      - `MYSQL_REPLICATION_PASSWORD` : set in the secret 
    
* **ingress.networking.k8s.io/mysql-route-primary**
  - we need to define this ingress to expose the reporter service using a hostname since the public ip address can change each time the cluster is initialized

    ```yaml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
        name: mysql-route-primary
        spec:
        rules:
        - host: mysql-reporter-arvanchallenge.apps.ir-thr-ba1.arvancaas.ir
            http:
            paths:
            - backend:
                service:
                    name: arvandb-mysql-metrics
                    port:
                    number: 9104
                path: /
                pathType: Prefix
    ```

the final manifest file is available as [manifest-file]

now we need to apply this manifest file using kubectl:

```
kubectl apply -f "path-to-manifest\manifest.yml"
```

### Step 3: testing the cluster

 using the following command we can connect to the mysql node:

 ```
 kubectl run arvandb-mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.36-debian-11-r0 --namespace arvanchallenge --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash
 ```

replacing the root password with the password set in the manifest, this will create a temporary pod in the same cluster since the database cluster isnt available remotely by default.

 we can connect to the database using :

 ```
 mysql -h arvandb-mysql-primary.arvanchallenge.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
 ```



- Import a HTML file and watch it magically convert to Markdown
- Drag and drop images (requires your Dropbox account be linked)
- Import and save files from GitHub, Dropbox, Google Drive and One Drive
- Drag and drop markdown and HTML files into Dillinger
- Export documents as Markdown, HTML and PDF

Markdown is a lightweight markup language based on the formatting conventions
that people naturally use in email.
As [John Gruber] writes on the [Markdown site][df1]

> The overriding design goal for Markdown's
> formatting syntax is to make it as readable
> as possible. The idea is that a
> Markdown-formatted document should be
> publishable as-is, as plain text, without
> looking like it's been marked up with tags
> or formatting instructions.

This text you see here is *actually- written in Markdown! To get a feel
for Markdown's syntax, type some text into the left window and
watch the results in the right.

## Tech

Dillinger uses a number of open source projects to work properly:

- [AngularJS] - HTML enhanced for web apps!
- [Ace Editor] - awesome web-based text editor
- [markdown-it] - Markdown parser done right. Fast and easy to extend.
- [Twitter Bootstrap] - great UI boilerplate for modern web apps
- [node.js] - evented I/O for the backend
- [Express] - fast node.js network app framework [@tjholowaychuk]
- [Gulp] - the streaming build system
- [Breakdance](https://breakdance.github.io/breakdance/) - HTML
to Markdown converter
- [jQuery] - duh

And of course Dillinger itself is open source with a [public repository][dill]
 on GitHub.

## Installation

Dillinger requires [Node.js](https://nodejs.org/) v10+ to run.

Install the dependencies and devDependencies and start the server.

```sh
cd dillinger
npm i
node app
```

For production environments...

```sh
npm install --production
NODE_ENV=production node app
```

## Plugins

Dillinger is currently extended with the following plugins.
Instructions on how to use them in your own application are linked below.

| Plugin           | README                                    |
| ---------------- | ----------------------------------------- |
| Dropbox          | [plugins/dropbox/README.md][PlDb]         |
| GitHub           | [plugins/github/README.md][PlGh]          |
| Google Drive     | [plugins/googledrive/README.md][PlGd]     |
| OneDrive         | [plugins/onedrive/README.md][PlOd]        |
| Medium           | [plugins/medium/README.md][PlMe]          |
| Google Analytics | [plugins/googleanalytics/README.md][PlGa] |

## Development

Want to contribute? Great!

Dillinger uses Gulp + Webpack for fast developing.
Make a change in your file and instantaneously see your updates!

Open your favorite Terminal and run these commands.

First Tab:

```sh
node app
```

Second Tab:

```sh
gulp watch
```

(optional) Third:

```sh
karma test
```

#### Building for source

For production release:

```sh
gulp build --prod
```

Generating pre-built zip archives for distribution:

```sh
gulp build dist --prod
```

## Docker

Dillinger is very easy to install and deploy in a Docker container.

By default, the Docker will expose port 8080, so change this within the
Dockerfile if necessary. When ready, simply use the Dockerfile to
build the image.

```sh
cd dillinger
docker build -t <youruser>/dillinger:${package.json.version} .
``` 

This will create the dillinger image and pull in the necessary dependencies.
Be sure to swap out `${package.json.version}` with the actual
version of Dillinger.

Once done, run the Docker image and map the port to whatever you wish on
your host. In this example, we simply map port 8000 of the host to
port 8080 of the Docker (or whatever port was exposed in the Dockerfile):

```sh
docker run -d -p 8000:8080 --restart=always --cap-add=SYS_ADMIN --name=dillinger <youruser>/dillinger:${package.json.version}
```

> Note: `--capt-add=SYS-ADMIN` is required for PDF rendering.

Verify the deployment by navigating to your server address in
your preferred browser.

```sh
127.0.0.1:8000
```

## License

MIT

**Free Software, Hell Yeah!**

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)

   [dill]: <https://github.com/joemccann/dillinger>
   [manifest-file]: <https://github.com/AzureLeMoon/ArvanChallenge/blob/main/MySql%20Cluster/manifest.yml>
   [john gruber]: <http://daringfireball.net>
   [df1]: <http://daringfireball.net/projects/markdown/>
   [markdown-it]: <https://github.com/markdown-it/markdown-it>
   [Ace Editor]: <http://ace.ajax.org>
   [node.js]: <http://nodejs.org>
   [Twitter Bootstrap]: <http://twitter.github.com/bootstrap/>
   [jQuery]: <http://jquery.com>
   [@tjholowaychuk]: <http://twitter.com/tjholowaychuk>
   [express]: <http://expressjs.com>
   [AngularJS]: <http://angularjs.org>
   [Gulp]: <http://gulpjs.com>

   [PlDb]: <https://github.com/joemccann/dillinger/tree/master/plugins/dropbox/README.md>
   [PlGh]: <https://github.com/joemccann/dillinger/tree/master/plugins/github/README.md>
   [PlGd]: <https://github.com/joemccann/dillinger/tree/master/plugins/googledrive/README.md>
   [PlOd]: <https://github.com/joemccann/dillinger/tree/master/plugins/onedrive/README.md>
   [PlMe]: <https://github.com/joemccann/dillinger/tree/master/plugins/medium/README.md>
   [PlGa]: <https://github.com/RahulHP/dillinger/blob/master/plugins/googleanalytics/README.md>

