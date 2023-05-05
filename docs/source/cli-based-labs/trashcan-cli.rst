=========================================
Volume Trashcan
=========================================

Ever had that sinking feeling after deleting a critical volume, disk, or file? The Portworx Volume Trashcan feature provides protection against accidental or inadvertent volume deletions which could result in loss of data. Volume Trashcan is disabled by default, but can be enabled using a simple pxctl command. Let's test it out!


Setting Volume expiration minutes
~~~~~~~~~~

First, configure the Portworx cluster to enable volume deletion and tell it how long you want to retain volumes after a delete. In this example, we'll tell Portworx to retain volumes for 720 minutes after a deletion:

.. code-block:: shell

  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl cluster options update --volume-expiration-minutes 720

Configuring a StorageClass
~~~~~~~~~~

Review the yaml for the StorageClass that we'll use - note the reclaimPolicy is set to Delete:

.. code-block:: shell

  cat << EOF >> /tmp/trashcan-sc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: trash-sc
  provisioner: pxd.portworx.com
  reclaimPolicy: Delete
  parameters:
    repl: "2"
  allowVolumeExpansion: true
  EOF

Then apply it to create the StorageClass:

.. code-block:: shell

  oc apply -f /tmp/trashcan-sc.yaml

Create a new namespace
~~~~~~~~~~

Let's create a namespace to use:

.. code-block:: shell

  oc create ns trashcan

Deploy the app
~~~~~~~~~~

Let's deploy a simple demo application to use for the volume trash can demo:

.. code-block:: shell
  cat << EOF >> /tmp/postgres-db-tc.yaml
  ---   
  ##### Portworx persistent volume claim
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: postgres-data
    labels:
      app: postgres
  spec:
    storageClassName: trash-sc
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 25Gi
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: example-config
  data:
    EXAMPLE_DB_HOST: postgres://postgres@postgres/example?sslmode=disable
    EXAMPLE_DB_KIND: postgres
    PGDATA: /var/lib/postgresql/data/pgdata
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: admin123
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: postgres
  spec:
    selector:
      matchLabels:
        app: postgres
    template:
      metadata:
        labels:
          app: postgres
      spec:
        containers:
        - image: "postgres:10.1"
          name: postgres
          envFrom:
          - configMapRef:
              name: example-config
          ports:
          - containerPort: 5432
            name: postgres
          volumeMounts:
          - name: postgres-data
            mountPath: /var/lib/postgresql/data
        volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-data
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: pg-service
  spec:
    selector:
      app: postgres
    ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
  EOF

.. code-block:: shell


MySQL Deployment

.. code-block:: shell

  cat <<EOF > /tmp/create-mysql.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
      name: px-db-sc
  provisioner: pxd.portworx.com
  parameters:
     repl: "3"
     io_profile: "db"
     io_priority: "high"
  ---
  apiVersion: v1
  kind: Namespace
  metadata:
    name: mysql-app
  spec: {}
  status: {}
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
     name: px-mysql-pvc
     labels:
       app: mysql
     namespace: mysql-app
  spec:
    storageClassName: px-db-sc
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: mysql
    namespace: mysql-app
  spec:
    selector:
      matchLabels:
        app: mysql
    replicas: 1
    template:
      metadata:
        labels:
          app: mysql
      spec:
        schedulerName: stork
        containers:
        - name: mysql
          image: mysql:5.6
          imagePullPolicy: "Always"
          env:
          - name: MYSQL_ALLOW_EMPTY_PASSWORD
            value: "1"
          ports:
          - containerPort: 3306
          volumeMounts:
          - mountPath: /var/lib/mysql
            name: mysql-data
        volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: px-mysql-pvc
  EOF

.. code-block:: shell

  cat << EOF >> /tmp/k8s-webapp-tc.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: k8s-counter-deployment
    labels:
      app: k8s-counter
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: k8s-counter
    template:
      metadata:
        labels:
          app: k8s-counter
      spec:
        containers:
        - name: k8s-counter
          image: wallnerryan/moby-counter:k8s-record-count
          imagePullPolicy: Always
          ports:
          - containerPort: 80
          env:
          - name: USE_POSTGRES_HOST
            value: "pg-service"
          - name: USE_POSTGRES_PORT
            value: "5432"
          - name: POSTGRES_USER
            value: "postgres"
          - name: POSTGRES_PASSWORD
            value: "admin123"
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: k8s-counter-service
  spec:
    type: LoadBalancer
    selector:
      app: k8s-counter
    ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30002
      name: k8s-counter-web
  EOF

.. code-block:: shell

  oc apply -f /tmp/postgres-db-tc.yaml -n trashcan
  oc apply -f /tmp/k8s-webapp-tc.yaml -n trashcan

Access the application
~~~~~~~~~~

Access the demo application using the LoadBalancer endpoint from the command below, and click around to generate some data that will be stored in the backend Postgres database.

.. code-block:: shell
   
  oc get svc -n trashcan k8s-counter-service

Delete the demo application
~~~~~~~~~~

Next, let's "accidentally" delete the postgres pod and persistent volume:

.. code-block:: shell

  oc delete -f /tmp/postgres-db-tc.yaml -n trashcan

Wait for the delete to complete before continuing.

Once the Postgres DB is deleted, navigate back to the Demo App tab to verify that it stopped working. Click on the refresh icon to the right of the tabs just to make sure - once the DB pod has been deleted, the logos should disappear.

Restoring volume from Volume Trashcan
~~~~~~~~~~

Let's use pxctl commands to restore our volume from the trashcan:

.. code-block:: shell
  
  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  VolName=$(oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume list --trashcan | grep "25 GiB" | awk '{print $8}')
  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume restore --trashcan $VolName pvc-restoredvol
  VolId=$(oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume list | grep "pvc-restoredvol" | awk '{print $1}' )

Create a persistent volume from the recovered portworx volume
~~~~~~~~~~

Now that we've restored the volume from the trashcan, let's create the yaml to tie the volume to a Kubernetes persistent volume:

.. code-block:: shell

  cat << EOF >> /tmp/recoverpv.yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    annotations:
      pv.kubernetes.io/provisioned-by: pxd.portworx.com
    finalizers:
    - kubernetes.io/pv-protection
    name: pvc-restoredvol
  spec:
    capacity:
      storage: 25Gi
    claimRef:
      apiVersion: v1
      kind: PersistentVolumeClaim
      name: postgres-data
      namespace: trashcan
    accessModes:
      - ReadWriteOnce
    storageClassName: trash-sc
    persistentVolumeReclaimPolicy: Retain
    portworxVolume:
      volumeID: "$VolId"
  EOF

And then apply the yaml:

.. code-block:: shell
  
  oc apply -f /tmp/recoverpv.yaml

Redeploy the app
~~~~~~~~~~

Let's redeploy the application which is using the recovered volume:

.. code-block:: shell

  oc apply -f /tmp/postgres-db-tc.yaml -n trashcan

Delete the old web front end:

.. code-block:: shell

  oc delete deploy k8s-counter-deployment -n trashcan

And redeploy the web front end: 

.. code-block:: shell

  oc apply -f /tmp/k8s-webapp-tc.yaml -n trashcan

Verify the restore by accessing the app
~~~~~~~~~~

Navigate to the Demo App UI by using the LoadBalancer endpoint from the command below and see that our data is back! You may have to click on the refresh icon in order to see the icons come back.

.. code-block:: shell

  oc get svc -n trashcan k8s-counter-service

This is how Portworx allows users to use the Trash Can feature to recover accidentally deleted persistent volumes. This prevents additional downtime and reduces ticket churn for data restoration due to human error!

Wrap up this module
~~~~~~~~~~

Use the following commands to delete objects used for this specific scenario:

.. code-block:: shell

  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl cluster options update --volume-expiration-minutes 0
  oc delete -f /tmp/k8s-webapp-tc.yaml -n trashcan
  oc delete -f /tmp/postgres-db-tc.yaml -n trashcan
  oc delete -f /tmp/recoverpv.yaml
  oc delete ns trashcan
  oc wait --for=delete ns/trashcan --timeout=60s