# Table of Contents
- [Table of Contents](#table-of-contents)
- [Challenge Documentation](#challenge-documentation)
  - [First Level](#first-level)
    - [Step 1: the values file](#step-1-the-values-file)
    - [Step 2: the manifest file](#step-2-the-manifest-file)
    - [Step 3: testing the cluster](#step-3-testing-the-cluster)
  - [Second Level](#second-level)
    - [Step 1: Setting up Ansible](#step-1-setting-up-ansible)
    - [Step 2: setting up Prometheus](#step-2-setting-up-prometheus)
    - [Step 3 : Setting up Grafana](#step-3--setting-up-grafana)
  - [Third Level](#third-level)
    - [Step 1: setting up filebeat on the observer nodes](#step-1-setting-up-filebeat-on-the-observer-nodes)
    - [Step 2: setting up Elasticsearch](#step-2-setting-up-elasticsearch)
    - [Step 3: Setting up Kibana](#step-3-setting-up-kibana)
    - [Step 4: Setting up logstash](#step-4-setting-up-logstash)


# Challenge Documentation
## First Level

Setting up a MySQL Replication Cluster along with the exporter server



Steps:

1. configuring the values files for installing the package
2. using the generated manifest file 
3. testing the cluster

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
 mysql -h arvandb-mysql-primary -uroot -p"$MYSQL_ROOT_PASSWORD"
 ```

we can list the available databases using :

```
SHOW DATABASES;
```

## Second Level


next up is setting up a monitoring cluser stack using [Ansible] to setup [Prometheus] and [Grafana].

first up we are going to need two VMs(Abraks),   
two small-g2 instances will be enough for this use-case.

the reason for seperating this stack into two parts is that by putting these two VMs into a private network they will be able to communicate with eachother however only the grafana node need to accesible remotely using a public ip address

### Step 1: Setting up Ansible

first we need to [install Ansible]

then we need to make sure we've got an ssh key pair to use for connecting to the VMs, this can be done either automatically when the VM is created in the panel or later on using ssh.

now we need to setup our Ansible Playbook

we are going to make three roles as 
  + Prometheus: which is going to scrape our mysql-exporter
  + Grafana: which is going to scrape Prometheus
  + FileBeat: for the Third Level later on

### Step 2: setting up Prometheus

first up is Prometheus which we are going to configure using the official docker image:

```yaml
---
- name: Create Folder /srv/prometheus if not exist
  file:
    path: /srv/prometheus
    mode: 0755
    state: directory


- name: Create prometheus configuration file
  copy:
    dest: /srv/prometheus/prometheus.yml
    src: prometheus_main.yml
    mode: 0644


- name: Create Prometheus container
  docker_container:
    name: prometheus
    restart_policy: always
    image: prom/prometheus:{{ prometheus_version }}
    volumes:
      - /srv/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - /srv/prometheus/prometheus_alerts_rules.yml:/etc/prometheus/prometheus_alerts_rules.yml
      - prometheus_main_data:/prometheus
    command: >
      --config.file=/etc/prometheus/prometheus.yml
      --storage.tsdb.path=/prometheus
      --web.console.libraries=/etc/prometheus/console_libraries
      --web.console.templates=/etc/prometheus/consoles
      --web.enable-lifecycle
    published_ports: "9090:9090"
```

the container is going to use the following Prometheus configuration file:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    scrape_interval: 30s
    static_configs:
    - targets: ["localhost:9090"]

  - job_name: mysql_primary
    scrape_interval: 30s
    static_configs:
    - targets: ["mysql-reporter-arvanchallenge.apps.ir-thr-ba1.arvancaas.ir"]
```

the target for scraping mysql is going to be the domain we set up in the ingress rule for the first level.


### Step 3 : Setting up Grafana

next up we setup the Grafana container: 

```yaml
---

- name: Create Folder /srv/grafana if not exist
  file:
    path: /srv/grafana
    mode: 0755
    state: directory


- name: Create grafana configuration files
  copy:
    dest: /srv/
    src: grafana
    mode: 0644

- name: Create Grafana container
  docker_container:
    name: grafana
    restart_policy: always
    image: grafana/grafana:{{ grafana_version }}
    volumes:
      - grafana-data:/var/lib/grafana
      - /srv/grafana/provisioning:/etc/grafana/provisioning
      - /srv/grafana/dashboards:/var/lib/grafana/dashboards
    env:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Admin"
    published_ports: "3000:3000"

```


and we are going to configure grafana as follows:


```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://10.0.1.179:9090
```


the target url here is going to be the private ip for the prometheus VM,  
this can be set manualy or better yet by using the Arvan API to fetch the VM ip using its name.

after running the playbook (which the complete format including the variables and what not is available [here](https://github.com/AzureLeMoon/ArvanChallenge/tree/main/observer-stack) ),   
the observer stack should be up and running with Grafana being available at `http://{Grafana host ip}:3000`

by logging into grafana we can see that prometheus is sending Data scarped from the mysql cluster.


## Third Level

now we are going to set up an elk node to monitor the logs from our observer stack.

### Step 1: setting up filebeat on the observer nodes

in the previous level we defined the FileBeat role for our observer stack, FileBeat is part of the elk stack for gathering various file and logs from the system.

we are going to setup filebeat as follows:

```yaml
---

- name: install filebeat
  ansible.builtin.apt:
    deb: https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.0-amd64.deb
    only_upgrade: true

- name: deploy filebeat configuration
  copy:
    src: filebeat.yml
    dest: "/etc/filebeat/filebeat.yml"

- name: enable system  plugin filebeat
  shell: filebeat modules enable system

- name: deploy filebeat configuration
  copy:
    src: system.yml
    dest: "/etc/filebeat/modules.d/system.yml"


- name: restart filebeat
  systemd:
    daemon_reload: yes
    name: filebeat
    state: restarted

```

its important to note that when installing componets from the elk stack it's best to have the same version across the stack, in our case its `8.12.0`.

as for collecting logs using file beat we are going to use the following config:

```yaml
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
  #reload.period: 10s

filebeat.inputs:
- type: filestream
  id: kernel-logs
  paths:
    - "/var/log/kern.log"
    - "/var/log/dmesg"
  fields:
    kernel: true

output.logstash:
  hosts: ['10.0.1.116:5044']
```

by using the builtin system plugin we can collect auth and syslog and send them to our logstash endpoint which is in the same private network using its ip adress, again this can be done either manually or preferebly using the ArvanCloud API.


### Step 2: setting up Elasticsearch

Elasticsearch is the core module in our log monitoring stack, we are going to use the official deb package from the elk apt repository to install Elasticsearch

first up we need to do some tasks to set up the requirements:

```yaml
- name: Update nameserver IP address
  ansible.builtin.lineinfile:
    path: /etc/resolv.conf
    regexp: '^nameserver (\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})$'
    line: 'nameserver 10.202.10.202'
    


- name: Install apt package requirements
  become: true
  become_user: root
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
    update_cache: true
  with_items:
    - gpg-agent
    - curl
    - procps
    - net-tools
    - gnupg
    - rpm
    - apt-transport-https
    - resolvconf


- name: Update nameserver IP address
  ansible.builtin.lineinfile:
    path: /etc/resolvconf/resolv.conf.d/head
    regexp: '^nameserver (\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})$'
    line: 'nameserver 10.202.10.202'    



- name: Import elastic keyring
  ansible.builtin.apt_key:
    state: present
    url: https://artifacts.elastic.co/GPG-KEY-elasticsearch
    keyring: /usr/share/keyrings/elasticsearch-keyring.gpg

- name: Add ELK APT Repository
  ansible.builtin.apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main"
    state: present
    filename: "elastic-8.x"    
    #update_cache: true
```

these tasks include setting up an anti sanction solution to bypass geological limitaions,  
and then using the keyring from elk to sign our apt repository to be able to install all the elk components we are going to need.

installing elastic is easy now using:

```yaml
- name: Install Elasticsearch
  block:
    - name: Install Elasticsearch package
      ansible.builtin.apt:
        name: elasticsearch={{ ELK_stack_version }}
        state: present
```

with the version being `8.12.0` as stated before.

we are not going to make any changes to the default elastisearch configuration.

### Step 3: Setting up Kibana

next up is kibana which is the UI on top of elasticsearch to ease management and provide data visualizations.

kibana can be installed same as Elasticsearch.

as for configuring it:

```yaml
- name: Update Elasticsearch Configuration File
  lineinfile:
    path: /etc/kibana/kibana.yml
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
  loop:
    - { regexp: '^server\.port:', line: 'server.port: {{ kibana.server_port }}' }
    - { regexp: '^server\.host:', line: 'server.host: "{{ kibana.server_host }}"' }
    - { regexp: '^server\.publicBaseUrl:', line: 'server.publicBaseUrl: {{ kibana.publicBaseUrl }}' }

- name: Run Elasticsearch Create Enrollment Token Command
  shell: /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
  register: token_response
- name: Extract Token from Response
  set_fact:
    enrollment_token: "{{ token_response.stdout }}"
- name: Run Kibana Enrollment Token Command
  shell: /usr/share/kibana/bin/kibana-setup --enrollment-token {{ enrollment_token }}
  args:
    stdin: "y\n"

- name: Enable and start kibana service
  service:
    name: kibana
    state: started
    enabled: yes
```

we are going to use the enrollment token generated by elastic search to connect our kibana node to the elastic service.



### Step 4: Setting up logstash


logstash is the log gathering and filtering pipeline which sends the logs into elastic search to be managed and indexed.

we are going to configure logstash to connect to our elastic service securely using the ca-cert generated by elastic:

```yaml
- name: Copy Logstash Output Configuration
  ansible.builtin.template:
    src: templates/logstash.conf
    dest: /etc/logstash/conf.d/beats.conf
    mode: 0640
    owner: logstash
    group: logstash

- name: copy elastic ca cert to logstash dir
  copy: 
    src: /etc/elasticsearch/certs/http_ca.crt    
    dest: /etc/logstash/config/certs/http_ca.crt
    owner: logstash
    group: logstash
    mode: '0644'

- name: Enable and start logstash service
  service:
    name: logstash
    state: started
    enabled: yes
```

and we are going to collect and filter the logs sent by our observer stack as below:

```c
input {
  beats {
    port => 5044
  }
}

filter {
  if [@metadata][beat] == "filebeat" {
    if [fields][kernel] == "true" {
      mutate { add_field => { "[@metadata][index]" => "kernel-%{[host][name]}" } }
    } else if [event][dataset] == "system.syslog" {
      mutate { add_field => { "[@metadata][index]" => "syslog-%{[host][name]}" } }
    } else if [event][dataset] == "system.auth" {
      mutate { add_field => { "[@metadata][index]" => "auth-%{[host][name]}" } }
    }
  }
}

output {
  if [@metadata][beat] in ["heartbeat", "metricbeat", "filebeat"] {
    elasticsearch {
      hosts => ["https://127.0.0.1:9200"]
      user => "elastic"
      password => "{{ elastic_password }}"
      ssl_certificate_authorities => "/etc/logstash/config/certs/http_ca.crt"
      index => "%{[@metadata][index]}"
    }
  }
}

```

this configuration listens on port 5044 for beats type log inputs and then filters our kernel,Auth and syslog into seperate indexes marked by the host it was collected from.





[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)

 
   [manifest-file]: <https://github.com/AzureLeMoon/ArvanChallenge/blob/main/MySql%20Cluster/manifest.yml>
   [Ansible]: <https://www.ansible.com/>
   [Prometheus]: <https://prometheus.io/>
   [Grafana]: <https://grafana.com/>
   [install Ansible]: <https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-and-upgrading-ansible-with-pip>

