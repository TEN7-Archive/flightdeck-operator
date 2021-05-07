# Flight Deck Operator

A Kubernetes operator to provide a LAMP-based web hosting solution.

## Defining MySQL Clusters

```yaml
apiVersion: flightdeck.t7.io/v1
kind: MySQLCluster
metadata:
  name: mysqlcluster-sample
  namespace: my-database-namespace
spec:
  mysql_admin_secret: mysql-admin
  mysql_readers_secret: mysql-reader
  size: 20Gi
```

## Defining MySQL databases

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
  state: present
```
