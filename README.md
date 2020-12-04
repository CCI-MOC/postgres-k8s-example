# Creating a postgres database in Kubernetes

## Create a persistent volume claim

Use a `PersistentVolumeClaim` to dynamically allocate a new
`PersistentVolume` for storing your data. The following example will create
a `PersistentVolumeClaim` named `example-data-claim`:

```
apiVersion: "v1"
kind: "PersistentVolumeClaim"
metadata:
  name: "example-data-claim"
  labels:
    app: postgres
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "5Gi"
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
```

## Creating a secret

The [Postgres docker image][] can be configured using environment
variables. We can store these in a Kubernetes secret like this:

```
apiVersion: v1
kind: Secret
metadata:
  name: postgres
type: Opaque
stringData:
  POSTGRES_USER: example_user
  POSTGRES_PASSWORD: example_password
  POSTGRES_DB: example_db
```

When applied to a container, these settings will cause Postgres to create a
user named `example_user` with password `example_password`, and a database
named `example_db`.

[postgres docker image]: https://hub.docker.com/_/postgres

## Create a deployment

A `Deployment` wraps a `Pod`. It will take care of redeploying the pod if
the pod configuration changes. In this example, we ask for a single replica
(we're not setting up replication). We have set `strategy` to `Recreate`,
which causes Kubernetes to remove the existing pod before deploying a new
one (the alternative, `RollingUpdate`, won't work because our data volume
can only be safely mounted by a single pod at a time):

We use the `envFrom` directive to expose the values from the `Secret` as
environment variables inside the Postgres container:

```
envFrom:
  - secretRef:
      name: postgres
```

The complete deployment looks like this:


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: postgres
      name: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:13
          ports:
            - containerPort: 5432
              protocol: TCP
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: example-data
          envFrom:
            - secretRef:
                name: postgres
      volumes:
        - name: example-data
          persistentVolumeClaim:
            claimName: example-data-claim
```

## Deploying everything into Kubernetes

Using the `kubectl` (or `oc` command on OpenShift) to deploy these files
into Kubernetes. To just deploy the resources shown in this README:

```
oc apply -k base
```

To delete everything:

```
oc -o yaml apply -k base | oc delete -f-
```

If you would also like `phppgadmin` as an example of a database client:

```
oc apply -k with_phppgadmin
```

And to delete everything:

```
oc -o yaml apply -k with_phppgadmin | oc delete -f-
```
