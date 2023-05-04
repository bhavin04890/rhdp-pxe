=========================
Volume Snapshots and Group VolumeSnapshots
=========================

Portworx allows you to take standard snapshots of your persistent volumes on a per-volume basis, but also gives you the capability to take group snapshots if you have persistence across multiple volumes to enable application-consistent snapshots.

In this scenario, you will:
* Perform a single volume snapshot and restore
* Configure pre and post snapshot rules to quiesce an application
* Perform a group volume snapshot and restore, utilizing the pre and post rules

Working with single volume snapshots
-------------------------
Before we take volume snapshots, let's navigate to the application UI that we deployed in the previous step, and verify that we can see Kubernetes logos on the page, if not, generate some data by clicking randomly on the screen before proceeding with the next step. 
To find the LoadBalancer endpoint for our demo application, use the following command: 

.. code-block:: shell

  oc get svc -n demo k8s-counter-service

Create volumesnapshot config
~~~~~~~~~~
Use the following yaml file to create a volume snapshot for your Postgres PVC.

.. code-block:: shell 

  cat << EOF >> /tmp/pg-snapshot.yaml
  apiVersion: volumesnapshot.external-storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    name: px-postgres-snapshot
    namespace: demo
  spec:
    persistentVolumeClaimName: postgres-data
  EOF

Apply the yaml file to create the snapshot. 

.. code-block:: shell 

  oc apply -f /tmp/pg-snapshot.yaml

And let's look at the snapshot object:

.. code-block:: shell

  oc get stork-volumesnapshots,volumesnapshotdatas -n demo

Accidently "Drop Table" in your Postgres database
~~~~~~~~~~
Let's delete the data within our Postgres DB by exec'ing into the pod:

.. code-block:: shell

  POD=$(oc get pods -l app=postgres -n demo | grep 1/1 | awk '{print $1}')
  oc exec -it $POD -n demo -- bash

And then drop our table:

.. code-block:: shell

  psql -U $POSTGRES_USER
  \c postgres
  drop table mywhales cascade;
  \q
  exit

Verify data has been deleted 
~~~~~~~~~~
Navigate to the Demo App using the LoadBalancer endpoint from the below command. All the logos that you saw in the beginning of the module will be gone, as we dropped our backend table in the Postgres database. 

.. code-block:: shell

  oc get svc -n demo k8s-counter-service

Restore our application from snapshot 
~~~~~~~~~~

Using the following yaml file, you can create a new PVC using the snapshot we created earlier: 

.. code-block:: shell

  cat << EOF >> /tmp/pvc-from-snap.yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: px-postgres-snap-clone
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: px-postgres-snapshot
  spec:
    accessModes:
       - ReadWriteOnce
    storageClassName: stork-snapshot-sc
    resources:
      requests:
        storage: 20Gi
  EOF

Create the PVC by applying the yaml 

.. code-block:: shell 

  oc apply -f /tmp/pvc-from-snap.yaml -n demo 

Then inspect the new PVC: 

.. code-block:: shell

  oc get pvc px-postgres-snap-clone -n demo

Redeploy the Demo Application
~~~~~~~~~~

Use the following commands to redeploy the application, so that it uses the new PVC object. First, we'll delete the old Postgres instance:

.. code-block:: shell

  oc delete -f /tmp/postgres-db.yaml

Next, we can redeploy Postgres using the new PVC that was restored from the snapshot:

.. code-block:: shell

  cat << EOF >> /tmp/postgres-db-restore.yaml
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
            claimName: px-postgres-snap-clone
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

Apply the yaml file: 

.. code-block:: shell

  oc apply -f /tmp/postgres-db-restore.yaml

Now let's restart the web front end of the demo application to force a reconnection to the Postgres DB:

.. code-block:: shell

  oc scale deployment.apps/k8s-counter-deployment --replicas=0 -n demo
  sleep 5
  oc scale deployment.apps/k8s-counter-deployment --replicas=1 -n demo

And make sure that our application pod is up and running - keep watching until the old k8s-counter-deployment pod terminates and only the new one is running:

.. code-block:: shell

  watch oc get pods -n demo

Use ctrl-c to exit out of the watch command. 

Verify the application has been completely restored
~~~~~~~~~~

Access the application by navigating to the LoadBalancer endpoint and refreshing the page. All of our logos are back where they originally were!
If you need to find your LoadBalancer endpoint, use the following command: 

.. code-block:: shell
  
  oc get svc -n demo k8s-counter-service

In this step, we took a snapshot of the persistent volume, deleted the database table and then restored our application by restoring the persistent volume using the snapshot!

Portworx Group Volume Snapshots
-------------------------
In this step, we will look at how you can use Portworx Group Volume Snapshots and 3D snapshots - to take application consistent multi-PVC snapshots for your application.

Create StorageClass for group volume snapshots
~~~~~~~~~~

Review the yaml of the StorageClass we are creating: 

.. code-block:: shell

  cat << EOF >> /tmp/group-sc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: group-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "2"
  EOF

Then apply the yaml to create it: 

.. code-block:: shell

  oc apply -f /tmp/group-sc.yaml

Create a new namespace 
~~~~~~~~~~

.. code-block:: shell

  oc create ns groupsnaps

Create pre-snap rule for Cassandra
~~~~~~~~~~

Portworx allows users to specify pre- and post-snapshot rules to ensure that the snapshots are application consistent and not crash consistent.

For this example, we are creating a pre-snapshot rule that flushes all the Cassandra data to the persistent volumes using the command nodetool flush before taking the snapshot.

Review the yaml for the snapshot rule:

.. code-block:: shell

  cat << EOF >> /tmp/cassandra-presnap-rule.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: Rule
  metadata:
    name: cassandra-presnap-rule
  rules:
    - podSelector:
        app: cassandra
      actions:
      - type: command
        value: nodetool flush
  EOF

Then apply the yaml file to create the rule: 

.. code-block:: shell
  
  oc apply -f /tmp/cassandra-presnap-rule.yaml -n groupsnaps

Deploy Cassandra 
~~~~~~~~~~

Deploy a Cassandra statefulset with 2 replicas to learn how Portworx GroupVolumeSnapshots work: 

.. code-block:: shell

  cat << EOF >> /tmp/cassandra-app.yaml
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: cassandra
    name: cassandra
  spec:
    clusterIP: None
    ports:
      - port: 9042
    selector:
      app: cassandra
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: cassandra
  spec:
    serviceName: cassandra
    replicas: 2
    selector:
      matchLabels:
        app: cassandra
    template:
      metadata:
        labels:
          app: cassandra
      spec:
        schedulerName: stork
        terminationGracePeriodSeconds: 1800
        containers:
        - name: cassandra
          image: gcr.io/google-samples/cassandra:v14
          imagePullPolicy: Always
          ports:
          - containerPort: 7000
            name: intra-node
          - containerPort: 7001
            name: tls-intra-node
          - containerPort: 7199
            name: jmx
          - containerPort: 9042
            name: cql
          resources:
            limits:
              cpu: "500m"
              memory: 1Gi
            requests:
             cpu: "500m"
             memory: 1Gi
          securityContext:
            capabilities:
              add:
                - IPC_LOCK
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "PID=$(pidof java) && kill $PID && while ps -p $PID > /dev/null; do sleep 1; done"]
          env:
            - name: MAX_HEAP_SIZE
              value: 512M
            - name: HEAP_NEWSIZE
              value: 100M
            - name: CASSANDRA_SEEDS
              value: "cassandra-0.cassandra.groupsnaps.svc.cluster.local"
            - name: CASSANDRA_CLUSTER_NAME
              value: "K8Demo"
            - name: CASSANDRA_DC
              value: "DC1-K8Demo"
            - name: CASSANDRA_RACK
              value: "Rack1-K8Demo"
            - name: CASSANDRA_AUTO_BOOTSTRAP
              value: "false"
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          readinessProbe:
            exec:
              command:
              - /bin/bash
              - -c
              - /ready-probe.sh
            initialDelaySeconds: 15
            timeoutSeconds: 5
          # These volume mounts are persistent. They are like inline claims,
          # but not exactly because the names need to match exactly one of
          # the stateful pod volumes.
          volumeMounts:
          - name: cassandra-data
            mountPath: /cassandra_data
    # These are converted to volume claims by the controller
    # and mounted at the paths mentioned above.
    volumeClaimTemplates:
    - metadata:
        name: cassandra-data
        annotations:
          volume.beta.kubernetes.io/storage-class: group-sc
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 2Gi
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: cqlsh
  spec:
    containers:
    - name: cqlsh
      image: mikewright/cqlsh
      command:
        - sh
        - -c
        - "exec tail -f /dev/null"
  EOF

Apply the yaml file to create the Cassandra deployment

.. code-block:: shell

  oc apply -f /tmp/cassandra-app.yaml -n groupsnaps

Watch until you see two Cassandra pods up and running with Ready 1/1 status:

.. code-block:: shell
  
  watch oc get pods,pvc -n groupsnaps

Note: use CTRL+C to exit out of the watch command once both the cassandra pods are running.

Interacting with Cassandra
~~~~~~~~~~

Let's take a look at the status of our Cassandra nodes:

.. code-block:: shell

  oc exec -it cassandra-0 -n groupsnaps -- nodetool status

And let's add some data to our Cassandra instance pods so we can take a snapshot later by exec'ing into the cqlsh pod:

.. code-block:: shell

  oc exec -it cqlsh -n groupsnaps -- cqlsh cassandra-0.cassandra.groupsnaps.svc.cluster.local --cqlversion=3.4.4

And then populate our data: 

.. code-block:: shell

  CREATE KEYSPACE portworx WITH REPLICATION = {'class':'SimpleStrategy','replication_factor':3};
  USE portworx;
  CREATE TABLE features (id varchar PRIMARY KEY, name varchar, value varchar);
  INSERT INTO portworx.features (id, name, value) VALUES ('px-1', 'snapshots', 'point in time recovery!');
  INSERT INTO portworx.features (id, name, value) VALUES ('px-2', 'cloudsnaps', 'backup/restore to/from any cloud!');
  INSERT INTO portworx.features (id, name, value) VALUES ('px-3', 'STORK', 'convergence, scale, and high availability!');
  INSERT INTO portworx.features (id, name, value) VALUES ('px-4', 'share-volumes', 'better than NFS, run wordpress on k8s!');
  INSERT INTO portworx.features (id, name, value) VALUES ('px-5', 'DevOps', 'your data needs to be automated too!');

  SELECT id, name, value FROM portworx.features;

  quit

Create and deploy a groupsnapshot object
~~~~~~~~~~

The following spec creates a snapshot of the persistent volume with the label of app: cassandra and executes the pre-snap rule that we created earlier:

.. code-block:: shell

  cat << EOF >> /tmp/cassandra-groupsnapshot.yaml
  apiVersion: stork.libopenstorage.org/v1alpha1
  kind: GroupVolumeSnapshot
  metadata:
    name: cassandra-group-snapshot
  spec:
    preExecRule: cassandra-presnap-rule
    pvcSelector:
      matchLabels:
        app: cassandra
  EOF

Apply the spec to execute the snapshot action:

.. code-block:: shell

  oc apply -f /tmp/cassandra-groupsnapshot.yaml -n groupsnaps

Note that once the snapshots have completed successfully, you should see Snapshot created successfully and it is ready for both Cassandra volumes in the oc describe output:

.. code-block:: shell

  oc get groupvolumesnapshot -n groupsnaps
  oc describe groupvolumesnapshot cassandra-group-snapshot -n groupsnaps

Drop the Portworx keyspace
~~~~~~~~~~
Now that we have a snapshot, let's exec into the cqlsh pod again:

.. code-block:: shell

  oc exec -it cqlsh -n groupsnaps -- cqlsh cassandra-0.cassandra.groupsnaps.svc.cluster.local --cqlversion=3.4.4

And then drop the Portworx keyspace from Cassandra, to see if we can restore successfully:

.. code-block:: shell

  drop keyspace Portworx;
  exit

Now, that we have dropped the Portworx keyspace, let's see how we can restore our data. 

We will start by deleting the Cassandra statefulset, Creating new PVCs using the snapshots we created earlier, and then redeploying the Cassandra statefulset. 

.. code-block:: shell
  
  oc delete sts cassandra -n groupsnaps

And let's get the snapshot names and assign them into variables

.. code-block:: shell

  SNAP0=$(oc get volumesnapshotdatas.volumesnapshot.external-storage.k8s.io -n groupsnaps -o jsonpath='{.items[0].metadata.name}')
  SNAP1=$(oc get volumesnapshotdatas.volumesnapshot.external-storage.k8s.io -n groupsnaps -o jsonpath='{.items[1].metadata.name}')

Now let's create a new yaml file for our PVC objects that will be deployed from our snapshots:

.. code-block:: shell

  cat << EOF > /tmp/restoregrouppvc.yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: cassandra-snap-data-cassandra-0
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: $SNAP0
  spec:
    accessModes:
       - ReadWriteOnce
    storageClassName: stork-snapshot-sc
    resources:
      requests:
        storage: 2Gi
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: cassandra-snap-data-cassandra-1
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: $SNAP1
  spec:
    accessModes:
       - ReadWriteOnce
    storageClassName: stork-snapshot-sc
    resources:
      requests:
        storage: 2Gi
  EOF

Now deploy the PVCs using the snapshots:

.. code-block:: shell

  oc apply -f /tmp/restoregrouppvc.yaml -n groupsnaps

Inspect the PVCs deployed from the snapshots:

.. code-block:: shell

  oc get pvc -n groupsnaps

Once you have these PVCs deployed, you can redeploy the Cassandra statefulset.

.. code-block:: shell

  cat << EOF >> /tmp/cassandra-restore-app.yaml
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: cassandra
  spec:
    serviceName: cassandra
    replicas: 2
    selector:
      matchLabels:
        app: cassandra
    template:
      metadata:
        labels:
          app: cassandra
      spec:
        # Use the stork scheduler to enable more efficient placement of the pods
        schedulerName: stork
        terminationGracePeriodSeconds: 1800
        containers:
        - name: cassandra
          image: gcr.io/google-samples/cassandra:v14
          imagePullPolicy: Always
          ports:
          - containerPort: 7000
            name: intra-node
          - containerPort: 7001
            name: tls-intra-node
          - containerPort: 7199
            name: jmx
          - containerPort: 9042
            name: cql
          resources:
            limits:
              cpu: "500m"
              memory: 1Gi
            requests:
             cpu: "500m"
             memory: 1Gi
          securityContext:
            capabilities:
              add:
                - IPC_LOCK
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "PID=$(pidof java) && kill $PID && while ps -p $PID > /dev/null; do sleep 1; done"]
          env:
            - name: MAX_HEAP_SIZE
              value: 512M
            - name: HEAP_NEWSIZE
              value: 100M
            - name: CASSANDRA_SEEDS
              value: "cassandra-0.cassandra.groupsnaps.svc.cluster.local"
            - name: CASSANDRA_CLUSTER_NAME
              value: "K8Demo"
            - name: CASSANDRA_DC
              value: "DC1-K8Demo"
            - name: CASSANDRA_RACK
              value: "Rack1-K8Demo"
            - name: CASSANDRA_AUTO_BOOTSTRAP
              value: "false"
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          readinessProbe:
            exec:
              command:
              - /bin/bash
              - -c
              - /ready-probe.sh
            initialDelaySeconds: 15
            timeoutSeconds: 5
          volumeMounts:
          - name: cassandra-snap-data
            mountPath: /cassandra_data
    volumeClaimTemplates:
    - metadata:
        name: cassandra-snap-data
        annotations:
          volume.beta.kubernetes.io/storage-class: group-sc
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 2Gi
  EOF

Apply the yaml file: 

.. code-block:: shell
   
  oc apply -f /tmp/cassandra-restore-app.yaml -n groupsnaps

Inspect the Pods and PVCs deployed to restore our Cassandra instance:

.. code-block:: shell

  watch oc get pods,pvc -n groupsnaps

Note: Use ctrl-c once all the pods are in running state. 

Inspect the Cassandra instance
~~~~~~~~~~

Let's verify that all of our data was restored:

.. code-block:: shell
  
  oc exec -it cqlsh -n groupsnaps -- cqlsh cassandra-0.cassandra.groupsnaps.svc.cluster.local --cqlversion=3.4.4


.. code-block:: shell

  SELECT id, name, value FROM portworx.features;
  quit

As you can see, our data has been successfully restored and is consistent due to our pre-snapshot command ensuring all data was flushed prior to the snapshots!

That's how easy it is to use Portworx snapshots, groupsnapshots and 3Dsnapshots to create application consistent snapshots for your applications running on Kubernetes.

Wrap up this module
-------------------------
Use the following commands to delete objects used for this specific scenario:

.. code-block:: shell

  kubectl delete -f /tmp/cassandra-app.yaml -n groupsnaps
  kubectl delete -f /tmp/restoregrouppvc.yaml -n groupsnaps
  kubectl delete -f /tmp/cassandra-groupsnapshot.yaml -n groupsnaps
  kubectl delete ns groupsnaps
  kubectl wait --for=delete ns/groupsnaps --timeout=60s
  kubectl delete -f /tmp/pg-snapshot.yaml
  kubectl delete -f /tmp/k8s-webapp.yaml -n demo
  kubectl delete -f /tmp/postgres-db.yaml -n demo
  kubectl delete ns demo
  kubectl wait --for=delete ns/demo --timeout=60s

To learn more about `Portworx <https://portworx.com/>`__, below are some useful references. 

- `Deploy Portworx on Kubernetes <https://docs.portworx.com/scheduler/kubernetes/install.html>`__
- `Create Portworx volumes <https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-pvcs/>`__
- `Use cases <https://portworx.com/use-case/kubernetes-storage/>`__
