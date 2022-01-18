# Flightdeck Operator

A Kubernetes operator to support LAMP applications in a multitenant environment.

## Features

* Builds on production-proven Flightdeck containers.
* Supports MySQL replication, Memcache load balancing, and PHP applications.
* Autogenerates passwords by default.
* Relies on Kubernetes Custom Resource Definitions (CRDs) to provision and manage applications and shared services.

## Installation

You can install the Flightdeck Operator using Helm 3.

```shell
helm repo add flightdeck https://ten7.github.io/flightdeck-operator/
helm install flightdeck-operator-system flightdeck/flightdeck-operator
```

## Managing MySQL

The Flightdeck Operator provides three CRDs to work with MySQL:
* MySQLCluster
* MySQLDatabase
* MySQLUser

### Defining MySQL Clusters

```yaml
apiVersion: flightdeck.t7.io/v1
kind: MySQLCluster
metadata:
  name: mysqlcluster-sample
  namespace: my-database-namespace
spec:
  fullnameOverride: myCluster
  replicas: 3
  mysql_admin_secret: mysql-admin
  mysql_readers_secret: mysql-reader
```

Where:

* **fullnameOverride** is the full name override to use when naming cluster containers and services. Optional, defaults to `mysqlcluster-` followed by the name of the `MySQLCluster` definiton.
* **replicas** is the total number of MySQL containers to create. The first of which will act as the writer/master, and all following contianers will be configured as a reader/replica.
* **mysql_admin_secret** is the name of the secret to use to store the MySQL root password. Optional, defaults to the value of `fullnameOverride` followed by -root.
* **mysql_readers_secret** is the name of the secret to use to store Flightdeck Operator user credentials. Optional, defaults to the value of `fullnameOverride` followed by -operator.
* **mysql_readers_secret** is the name of the secret to use to store replication user credentials. Optional, defaults to the value of `fullnameOverride` followed by -reader.
* **nodeSelector** specifies a node selector by which to place containers. Must have both a `key` and a `value`. Optional.
* **affinity** is a Kubernetes affinity statement by which to place containers. Optional.

Note, if the secrets do not exist, they will be created and populated with a randomly generated password.

### Persistency

To ensure that your databases are preserved if a container is deleted or replaced, use the `persistence` key:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: MySQLCluster
metadata:
  name: mysqlcluster-sample
  namespace: my-database-namespace
spec:
  persistence:
    enabled: true
    name: "mysql-data"
    existingClaim: "my-mysql-pvc"
    size: 20Gi
    accessModes:
      - ReadWriteOnce
    storageClass: "rook-ceph"
```

Where:
* **enabled** specifies if persistent storage is enabled (`true`), or disabled (`false`). Optional, defaults to `false`.
* **name** is the name of the PersistentVolumeClaim (PVC) to use when allocating storage. Optional, defaults to the value of `fullnameOverride`. Ignored when using `existingClaim`.
* **existingClaim** is the name of an existing PersistentVolumeClaim (PVC) in the same namespace as the `MySQLCluster` definition. Optional.
* **size** is the size of the persistent storage to allocate for each MySQL replica. Required when `enabled` is `true`, and not using `existingClaim`.
* **accessModes** is a list of access modes by which to mount the persistent storage. Optional, defaults to `ReadWriteOnce`.
* **storageClass** is the storage class to use to allocate persistent storage. Optional.

Note that replication may not function when using an existing claim or an `accessMode` of `ReadWriteMany`.

### Controlling container placement

You can control where in your cluster the MySQL containers are placed using the `nodeSelector` and/or `affinity` keys:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: MySQLCluster
metadata:
  name: mysqlcluster-sample
  namespace: my-database-namespace
spec:
  nodeSelector:
    key: doks.digitalocean.com/node-pool
    value: web-pool
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: flightdeck.ten7.io/phpapplication
                operator: In
                values:
                  - drupal
          topologyKey: kubernetes.io/hostname
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
              - linux
```
Where:

* **nodeSelector** specifies a node selector by which to place containers. Must have both a `key` and a `value`. Optional.
* **affinity** is a Kubernetes affinity statement by which to place containers. Optional.

### Controlling resource allocation

By default, all containers for the MySQL cluster are run without any requests or limits as to memory and CPU resources. You define those requests and limits using the `resources` key:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: MySQLCluster
metadata:
  name: mysqlcluster-sample
  namespace: my-database-namespace
spec:
  nodeSelector:
    key: doks.digitalocean.com/node-pool
    value: web-pool
  resources:
    requests:
      memory: "1024Mi"
      cpu: "250m"
    limits:
      memory: "2048Mi"
      cpu: "1000m"
```

See the [Kubernetes documentation on resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for a complete description of use.

### Defining MySQL databases

```yaml
apiVersion: flightdeck.t7.io/v1
kind: MySQLDatabase
metadata:
  name: my-database
spec:
  cluster:
    name: mysqlcluster-sample
    namespace: my-database-namespace
  dbName: "my_database"
  encoding: "utf8mb4"
  collation: "utf8mb4_unicode_ci"
```

* **cluster.name** is the name of the `MySQLCluster` on which to provision this database.
* **cluster.namespace** is the namespace of the `MySQLCluster`. Optional, defaults to the current namespace.
* **dbName** is the name of the database to create. Optional, defaults to the name of the `MySQLDatabase` definition.
* **encoding** is the encoding to use to create the database. Optional, defaults to `utf8`.
* **collation** is the collation to use when creating the database. Optional, defaults to `utf8_general_ci`.

### Defining MySQL Users

```yaml
apiVersion: flightdeck.t7.io/v1
kind: MySQLUser
metadata:
  name: my-user
spec:
  cluster:
    name: fdo
    namespace: flightdeck-operator-system
  username: "my_user"
  host: "%"
  password_secret: my-user-pass
  privileges:
    - database: "my_database"
      table: "*"
      grant: "ALL"
```

Where:

* **cluster.name** is the name of the `MySQLCluster` on which to provision this database.
* **cluster.namespace** is the namespace of the `MySQLCluster`. Optional, defaults to the current namespace.
* **username** is the username of the MySQL user to create. Optional, defaults to the name of the `MySQLUser` definition.
* **host* is the host pattern for the MySQL user. Optional, defaults to `%`.
* **password_secret** is the Secret name containing the MySQL password for the user. Optional, creates a Secret named `mysqluser-TheMySQLClusterDefName-TheMySQLUserDefName` by default. If non-existent or unspecified, the password itself is autogenerated.

The `privileges` key defines the user's privileges:
* **database** is the database name for the grant.
* **table** is the table name pattern for the grant.
* **grant** is a list of permissions, comma delimited.

## PHP applications

The most basic way to deploy a PHP application with the Flightdeck Operator is to create a `PhpApplication` definition.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  replicas: 3
  image: "ten7/flightdeck-web-7.4"
  mysqlDatabases:
    - name: my-database
      namespace: "flightdeck-operator-system"
  mysqlUsers:
    - name: my-user
      namespace: "flightdeck-operator-system"
  docroot: "/var/www/html"
  storage:
    name: "muffy-live-files"
    class: "rook-cephfs"
    mode: "ReadWriteMany"
    size: "5Gi"
    path: "/var/www/files"
```

Where:

* **fullnameOverride** is the full name override to use when naming PHP application containers and services. Optional, defaults to `phpapp-` followed by the name of the `PhpApplication` definiton.
* **replicas** is the number of replicas to run the application. Optional, defaults to 1.
* **image** is the container image to use. Optional, defaults to `ten7/flightdeck-web-7.4`. It is highly recommended you override this to your custom container!
* **mysqlDatabases** is a list of `MySQLDatabase` definitions utilized by this application. Optional.
* **mysqlUsers** is a list of `MySQLUser` definitions utilized by this application. Optional.
* **docroot** is the path to the docroot of the application. Optional, defaults to `/var/www/html` inside the `image` container.


### Persistency

If your application requires persistent file storage, you can use the `persistence` key:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  persistence:
    enabled: true
    name: "muffy-live-files"
    existingClaim: "my-php-pvc"
    size: "5Gi"
    accessModes:
      - "ReadWriteOnce"
    storageClass: "rook-ceph"
    path: "/var/www/files"
```

Where:
* **enabled** specifies if persistent storage is enabled (`true`), or disabled (`false`). Optional, defaults to `false`.
* **name** is the name of the PersistentVolumeClaim (PVC) to use when allocating storage. Optional, defaults to the value of `fullnameOverride`. Ignored when using `existingClaim`.
* **existingClaim** is the name of an existing PersistentVolumeClaim (PVC) in the same namespace as the `PhpApplication` definition. Optional.
* **size** is the size of the persistent storage to allocate for the PHP application. Required when `enabled` is `true`, and not using `existingClaim`.
* **accessModes** is a list of access modes by which to mount the persistent storage. Optional, defaults to `ReadWriteOnce`.
* **storageClass** is the storage class to use to allocate persistent storage. Optional.
* **path** is the path inside the container to mount the PVC. Optional, defaults to `/var/www/files`.

### Hostnames

You may wish to configure various hostnames of the application using several keys:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  serverName: "muffy.example.com"
  serverAliases:
    - "docker.test"
    - "muffy.test"
  hostAliases:
    - ip: "127.0.0.1"
      hostnames:
        - "docker.test"
        - "muffy.test"
```

Where:

* **serverName** is the web server name, as reported on HTTP headers and default error pages. Optional, defaults to `flightdeck.test`.
* **serverAliases** are web server hostname aliases. Optional.
* **hostAliases** are custom entries to add to `/etc/hosts` for static host name overrides. Optional.

### Environment variables

You can set environment variables for the application using the `env` key.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  env:
    - name: "FLIGHTDECK_ENVIRONMENT"
      value: "live"
```

Where:

* **name** is the environment variable name.
* **value** is the environment variable value.

Environment variables are set both for the command line and in the web server.

### Controlling container placement

You can control where in your cluster the PHP application containers are placed using the `nodeSelector` and/or `affinity` keys:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  nodeSelector:
    key: doks.digitalocean.com/node-pool
    value: web-pool
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: flightdeck.ten7.io/phpapplication
                operator: In
                values:
                  - drupal
          topologyKey: kubernetes.io/hostname
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: doks.digitalocean.com/node-pool
            operator: In
            values:
            - web-pool
```

Where:

* **nodeSelector** specifies a node selector by which to place containers. Must have both a `key` and a `value`. Optional.
* **affinity** is a Kubernetes affinity statement by which to place containers. Optional.

### Configuring PHP

To configure the PHP engine itself, you can use the `php` key:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  php:
    upload_max_filesize: "128M"
    post_max_size: "128M"
```

See the [flightdeck-web-7.4](https://github.com/ten7/flightdeck-web-7.4#configuring-php) documentation for full options.

### Configuring Varnish

Each web server container also has an accompanying Varnish container, controlled by the `varnish` key:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  varnish:
    image: "ten7/flightdeck-varnish-6.4"
    secretName: "varnish-secret"
    memSize: "16m"
    skipCache:
      - "/update\\.php"
      - "/core/install\\.php"
      - "/admin"
      - "/admin/.*"
      - "/user"
      - "/user/.*"
      - "/users/.*"
      - "/info/.*"
      - "/flag/.*"
      - ".*/ahah/.*"
    probe:
      state: no
      probeHost: "muffy.t7test.io"
      headers:
        - name: "X-Forwarded-Proto"
          value: "https"
```

Where:

* **image** is the varnish image to use. Optional, defaults to `ten7/flightdeck-varnish-7.6`.
* **secretName** is the name of the Secret to use to store the Varnish secret. Optional, defaults to value of the `fullnameOverride` plus `-varnish`. If empty or doesn't exist, the secret will be autogenerated.
* **memSize** is the amount of memory to use for caching. Optional, defaults to `32m`.
* **skipCache** is a list of path patterns to skip caching. Optional.
* **probe** defines if varnish monitors the web server for health.

### Deployment scripts

Often, you may wish to run a series of commands when deploying a PHP application. For that, you can use the `scripts` key:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  scripts:
    preDeployCheck: |
      /path/to/command arg1 arg2
    postDeploy: |
      /path/to/command arg1 arg2
```

Where:

* **preDeployCheck** is a shell script to execute prior to doing a deploy. The deployment will fail if the script returns anything other than `0`.
* **postDeploy** is a shell script to execute when the deployment is complete.

Due to a design limitation of the operator, the above scripts are run twice.

### Controlling resource allocation

Each pod which supports a PHP application is made of two containers, one which provides a web server and PHP ("web") and a varnish container. The primary configuration which controls memory allocation for each is `spec.php.php.memory_limit` for web, and `spec.varnish.memSize` for varnish.

You may wish to impose additional restrictions on the application such that Kubernetes can better schedule and monitor the application. You can do this with the `resources` key for web, and the `varnish.resources` key for varnish:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpApplication
metadata:
  name: my-php-app
  namespace: example-com
spec:
  resources:
    requests:
      memory: "320Mi"
      cpu: "120m"
    limits:
      memory: "700Mi"
      cpu: "500m"
  varnish:
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "100Mi"
        cpu: "250m"
```

See the [Kubernetes documentation on resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for a complete description of use.

## Background tasks for PHP Applications

Often, you want to run background tasks for your PHP applications. While this can be done with normal Kubernetes `cronjob` definitions, often these tasks rely on the same image, configmaps, secrets, and volumes of the PHP application they support.

For this reason, this operator provides the `PhpCronjob` kind. It relies on a `PhpApplication` to provide key configurations:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpCronjob
metadata:
  name: "sleepy-cat"
spec:
  phpApplication: "drupal"
  image: "ten7/flightdeck-web-7.4"
  schedule: "0 * * * *"
  suspend: no
  args:
    - "/bin/bash"
    - "-c"
    - "sleep 1"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: doks.digitalocean.com/node-pool
            operator: In
            values:
            - web-pool
```
Where:

* **phpApplication** is the name of the `PhpApplication` definition. Required.
* **image** is the container image to use for the background task. Optional, defaults to the `image` value of the `PhpApplication` definition.
* **resources** is the [Kubernetes resources definition](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for the background task. Optional.
* **nodeSelector** is the key/value pair to use to place the pod in the cluster. Optional.
* **affinity** is the Kubernetes affinity statement used to place the pod in the cluster. Optional.
* **resources** is the container physical resource requests and limits.
* **schedule** is the crontab formatted schedule on which to run the job. Required.
* **command** is the command (entrypoint) to run in the pod. Optional.
* **args** are the arguments to pass to the command. Optional.
* **concurrencyPolicy** specifies if the cronjob can be run multiple times concurrently. Optional. False by default (opposite k8s' default).
* **restartPolicy** specifies what to do if the run fails. Optional, works like `tractorbeam` above.
* **suspend** specifies if the cronjob is disabled. Optional. Defaults to false (enabled).

## Defining routes for PHP applications

To define name-based networking routes to your PHP applications, you can create one or more `PhpRoute` definitions:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpRoute
metadata:
  name: "apex"
spec:
  phpApplication: "drupal"
  rules:
    - host: "muffy.t7test.io"
      paths:
        - path: "/"
          pathType: "ImplementationSpecific"
          bypassVarnish: false
```

Where:

* **phpApplication** is the name of the `PhpApplication` definition. Required.
* **rules** is a list of name path mappings. Required.

### Defining rules

The `rules` list specifies how specific paths are mapped to the PHP application.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpRoute
metadata:
  name: "apex"
spec:
  phpApplication: "drupal"
  rules:
    - host: "muffy.t7test.io"
      paths:
        - path: "/"
          pathType: "Prefix"
          bypassVarnish: false
```

Where:

* **host** is the hostname to match for the rule. Required.
* **paths** is a list of path rules to map behavior. Required.

For each item in `paths`:

* **path** is the URL path after the domain to match for the rule. Required.
* **pathType** is the [path type](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types) for the resulting ingress definition. Optional, defaults to `Prefix`.
* **bypassVarnish** is `true` to bypass the varnish caching container for the `path`, `false` to cache to varnish. Optional, defaults to `false`.

### SSL and TLS

By default, the traffic for a `PhpRoute` is unencrypted. To enabled encryption, you can use the `tls` key:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpRoute
metadata:
  name: "apex"
spec:
  phpApplication: "drupal"
  tls:
    state: true
    issuer: "lets-encrypt-ops-at-ten7-com"
    certSecret: "muffy-cert"
  rules:
    - host: "muffy.t7test.io"
      paths:
        - path: "/"
          pathType: "ImplementationSpecific"
          bypassVarnish: false
```

Where:

* **tls.state** defines if encyption is enabled for all `rules` in the `PhpRoute` definition. Optional, defaults to `false`.
* **tls.issuer** is the name of the Issuer if using Cert Manager. Optional.
* **certSecret** is the name of the certificate Secret. Optional unless if using a static certificate. Defaults to the name of the `PhpRoute` definition followed by `-cert`.

### HTTP Authentication

Often, you'll want to restrict access to a domain name or path on an existing domain name behind HTTP authentication. Flightdeck Operator supports this when using the [NGINX ingress controller](https://kubernetes.github.io/ingress-nginx/).

The first step in configuring this is to create an htpasswd secret containing one or more logins. Instead of requiring you to generate this secret yourself, you can use the `Htpasswd` definition:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: Htpasswd
metadata:
  name: "muffy-t7test-io"
spec:
  logins:
    - name: "muffy"
      secret: "muffy-auth"
    - name: "rocket"
```

The `logins` list defines which logins are added to the htpasswd. For each item:

* **name** specifies the username for the HTTPauth login. Required.
* **secret** specifies the Secret definition which contains the `password` for the login. Optional, if unspecified or does not exist, the password is autogenerated.

Once you have defined an `Htpasswd`, you can enable HTTP authentication in your `PhpRoute` using the `auth` key:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: PhpRoute
metadata:
  name: "apex"
spec:
  phpApplication: "drupal"
  auth:
    state: yes
    htpasswd: "muffy-t7test-io"
    message: "Poke cat?"
  rules:
    - host: "muffy.t7test.io"
      paths:
        - path: "/"
          pathType: "ImplementationSpecific"
          bypassVarnish: false
```

Where:

* **auth.state** specifies if HTTP authentication is enabled. Optional, defaults to `true` if `auth` is defined.
* **htpasswd** is the name of the `Htpasswd` definition. Required if `auth.state` is `true`.
* **message** is the message to display on the HTTPauth prompt. Optional, defaults to `Please enter your login`.

## Memcache key/value store

For high performance PHP application, you might turn a memory-based key/value store such as Memcache. The operator can create a load balanced, multi-tenant Memcache cluster using the `MemcacheCluster` definition:

```yaml
---
apiVersion: flightdeck.t7.io/v1
kind: MemcacheCluster
metadata:
  name: fdo
  namespace: flightdeck-operator-system
spec:
  replicas: 3
  memory: "128"
  threads: "4"
  nodeSelector:
    key: doks.digitalocean.com/node-pool
    value: cache-pool
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
           - matchExpressions:
             - key: doks.digitalocean.com/node-pool
               operator: In
               values:
                 - cache-pool
```

Where:

* **fullnameOverride** is the full name override to use when naming PHP application containers and services. Optional, defaults to `memcachecluster-` followed by the name of the `MemcacheCluster` definition.
* **replicas** is the number of Memcache instances to create in the cluster. Optional, defaults to `3`. Note, this does not affect the number of load balancing pods (twemproxy).
* **memory** is the size in megabytes to use a cache. Optional, defaults to `256`.
* **threads** is the number of threads to use for each memcache instance. Optional, defaults to `4`.
* **nodeSelector** specifies a node selector by which to place containers. Must have both a `key` and a `value`. Optional.
* **affinity** is a Kubernetes affinity statement by which to place containers. Optional.


### Controlling resource allocation

Memcache clusters are composed of a statefulset of memcache containers, and a daemonset of twemproxy load balancers. The `spec.memory` key defines how much memory to use for caching.

You may wish to impose additional restrictions on the memcache cluster such that Kubernetes can better schedule and monitor the application. You can do this with the `resources` key for the memcache statefulset, and the `twemproxy.resources` key for twemproxy load balancer daemonset:

```yaml
apiVersion: flightdeck.t7.io/v1
kind: MemcacheCluster
metadata:
  name: fdo
  namespace: flightdeck-operator-system
spec:
  resources:
    requests:
      memory: "100Mi"
      cpu: "50m"
    limits:
      memory: "200Mi"
      cpu: "200m"
  twemproxy:
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "100Mi"
        cpu: "250m"
```

See the [Kubernetes documentation on resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for a complete description of use.

## Solr search

Apache Solr is often used by PHP Applications to provide a search function. Instead of single-node Solr instances, you can stand up a multi-tenant Solr Cloud instance using a handful of definitions.

### Zookeeper Cluster

Before creating a SolrCluster, you must also create a ZookeeperCluster. Zookeeper is an Apache project which is used to manage configuration files in a multi-server environment.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: ZookeeperCluster
metadata:
  name: "fdo-sample"
spec:
  replicas: 3
  persistence:
    enabled: true
    size: "1Gi"
    class: "rook-cephfs"
```

Where:

* **fullnameOverride** is the full name override to use when naming PHP application containers and services. Optional, defaults to `-zookeepercluster` followed by the name of the `MemcacheCluster` definition.
* **replicas** is the number of instances to create in the cluster. Optional, defaults to `3`.
* **nodeSelector** specifies a node selector by which to place containers. Must have both a `key` and a `value`. Optional.
* **affinity** is a Kubernetes affinity statement by which to place containers. Optional.
* **persistence** defines how to store persistent information for the cluster. Optional, but highly recommended.


### Solr Cluster

Before creating a SolrCluster, you must also create a ZookeeperCluster. Zookeeper is an Apache project which is used to manage configuration files in a multi-server environment.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: SolrCluster
metadata:
  name: "fdo"
spec:
  zookeeperCluster:
    name: "fdo"
    namespace: "default"
  replicas: 3
  persistence:
    enabled: true
    size: "5Gi"
    class: "rook-cephfs"
  solrAdmin:
    user: solr
    secret: "solr-admin-secret"
```

Where:

* **fullnameOverride** is the full name override to use when naming PHP application containers and services. Optional, defaults to `-zookeepercluster` followed by the name of the `MemcacheCluster` definition.
* **replicas** is the number of instances to create in the cluster. Optional, defaults to `3`.
* **nodeSelector** specifies a node selector by which to place containers. Must have both a `key` and a `value`. Optional.
* **affinity** is a Kubernetes affinity statement by which to place containers. Optional.
* **persistence** defines how to store persistent information for the cluster. Optional, but highly recommended.
* **zookeeperCluster** specifies the `name` and optionally, the `namespace` of the zookeeperCluster to use to support the SolrCluster.
* **solrAdmin** specifies the `user` name and optionally, a `secret` containing the password to use for the user. If `secret` is not specified, the password is autogenerated.

### Solr Config Sets

Before creating a SolrCollection, you must create a SolrConfigset. This configset loads the collection's schema files onto the ZookeeperCluster for use.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: SolrConfigset
metadata:
  name: "fdo"
spec:
  solrCluster:
    name: "fdo"
    namespace: "default"
  configMap: "solr-conf"
```

Where:

* **solrCluster** specifies the `name` and optionally, the `namespace` of the SolrCluster. If `namespace` is not given, the current namespace is assumed.
* **configMap** is the name of a ConfigMap which contains the collection schema files.

### Solr Collection (cores)

The SolrCollection provides you an index on which to add search items and perform searches.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: SolrCollection
metadata:
  name: "fdo-operator-core"
spec:
  solrCluster:
    name: "fdo"
    namespace: "default"
  configSet:
    name: "fdo"
    namespace: "default"
```

Where:

* **solrCluster** specifies the `name` and optionally, the `namespace` of the SolrCluster. If `namespace` is not given, the current namespace is assumed.
* **configSet** specifies the `name` and optionally, the `namespace` of the SolrConfigset on which to base the index. If `namespace` is not given, the current namespace is assumed.

### Solr roles

It's highly recommended to set up a SolrRole to grant access only to needed collections.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: SolrRole
metadata:
  name: "fdo"
spec:
  solrCluster:
    name: "fdo"
    namespace: "default"
  collections:
    - name: "fdo-operator-core"
      permissions:
        - "update"
        - "read"
```

Where:

* **solrCluster** specifies the `name` and optionally, the `namespace` of the SolrCluster. If `namespace` is not given, the current namespace is assumed.
* **collections** specifies per-collection permissions. Required.

For each item under `collections`:

* **name** specifies the collection name or pattern. Required.
* **permissions* specifies the permissions to grant to the collection.

### Solr users

Once you have SolrRoles set up, you can set up SolrUsers to grant permissions specifically to key collections and operations.

```yaml
apiVersion: flightdeck.t7.io/v1
kind: SolrUser
metadata:
  name: "fdo"
spec:
  solrCluster:
    name: "fdo"
    namespace: "default"
  roles:
    - "fdo"
```

Where:

* **solrCluster** specifies the `name` and optionally, the `namespace` of the SolrCluster. If `namespace` is not given, the current namespace is assumed.
* **roles** is a list of SolrRoles to grant the user.
