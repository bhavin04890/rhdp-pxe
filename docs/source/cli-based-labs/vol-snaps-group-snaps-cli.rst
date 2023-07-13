=========================
Volume Snapshots and Group VolumeSnapshots
=========================

Portworx allows you to take standard snapshots of your persistent volumes on a per-volume basis, but also gives you the capability to take group snapshots if you have persistence across multiple volumes to enable application-consistent snapshots.

In this scenario, you will:

1. Perform a single volume snapshot and restore
2. Configure pre and post snapshot rules to quiesce an application
3. Perform a group volume snapshot and restore, utilizing the pre and post rules

Working with single volume snapshots
-------------------------
Before we take volume snapshots, let's navigate to the application UI that we deployed in the previous step, and verify that we can see order that we placed in the previous module.
To find the LoadBalancer endpoint for our demo application, use the following command: 

.. code-block:: shell

  oc get svc -n pxbbq pxbbq-svc

Navigate to the application and login as the Demo user and look at the `Order History`

.. image:: images/pxbbq-5.jpg
  :width: 600
  
Create volumesnapshot config
~~~~~~~~~~
Use the following yaml file to create a volume snapshot for your MongoDB PVC.

.. code-block:: shell 

  cat << EOF >> /tmp/mongo-snapshot.yaml
  apiVersion: volumesnapshot.external-storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    name: px-mongo-snapshot
    namespace: pxbbq
  spec:
    persistentVolumeClaimName: mongodb-pvc
  EOF

Apply the yaml file to create the snapshot. 

.. code-block:: shell 

  oc apply -f /tmp/mongo-snapshot.yaml

And let's look at the snapshot object:

.. code-block:: shell

  oc get stork-volumesnapshots,volumesnapshotdatas -n pxbbq

Accidently "Drop Table" in your MongoDB database
~~~~~~~~~~
Let's delete the data within our MongoDB DB by exec'ing into the pod:

.. code-block:: shell

  POD=$(oc get pods -l app.kubernetes.io/name=mongo -n pxbbq | grep 1/1 | awk '{print $1}')
  oc exec -it $POD -n pxbbq -- mongosh --quiet  

And then drop our table:

.. code-block:: shell
  
  use admin
  db.auth('porxie','porxie')
  show dbs 
  use porxbbq
  db.dropDatabase()

Use the `quit` command to exit out of the mongodb pod. 

Verify data has been deleted 
~~~~~~~~~~
Navigate to the Portworx BBQ App using the LoadBalancer endpoint from the below command. You should not be able to login using the user you created in the last module. If you were already logged in, you should not see your order from order history. 

.. code-block:: shell

  oc get svc -n pxbbq pxbbq-svc

.. image:: images/pxbbq-8.jpg
  :width: 600

Restore our application from snapshot 
~~~~~~~~~~

Using the following yaml file, you can create a new PVC using the snapshot we created earlier: 

.. code-block:: shell

  cat << EOF >> /tmp/pvc-from-snap.yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: px-mongo-snap-clone
    annotations:
      snapshot.alpha.kubernetes.io/snapshot: px-mongo-snapshot
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

  oc apply -f /tmp/pvc-from-snap.yaml -n pxbbq 

Then inspect the new PVC: 

.. code-block:: shell

  oc get pvc px-mongo-snap-clone -n pxbbq

Redeploy the Demo Application
~~~~~~~~~~

Use the following commands to redeploy the application, so that it uses the new PVC object. First, we'll delete the old MongoDB instance:

.. code-block:: shell

  oc delete -f /tmp/pxbbq-mongo.yaml

Next, we can redeploy MongoDB using the new PVC that was restored from the snapshot:

.. code-block:: shell

  cat << EOF >> /tmp/pxbbq-mongo-restore.yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: mongo
    labels:
      app.kubernetes.io/name: mongo
      app.kubernetes.io/component: backend
    namespace: pxbbq
  spec:
    selector:
      matchLabels:
        app.kubernetes.io/name: mongo
        app.kubernetes.io/component: backend
    replicas: 1
    template:
      metadata:
        labels:
          app.kubernetes.io/name: mongo
          app.kubernetes.io/component: backend
      spec:
        containers:
        - name: mongo
          image: mongo
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: porxie
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: "porxie"
          args:
            - --bind_ip
            - 0.0.0.0
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
          ports:
          - containerPort: 27017
          volumeMounts:
          - name: mongo-data-dir
            mountPath: /data/db
        volumes:
        - name: mongo-data-dir
          persistentVolumeClaim:
            claimName: px-mongo-snap-clone
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: mongo
    labels:
      app.kubernetes.io/name: mongo
      app.kubernetes.io/component: backend
    namespace: pxbbq
  spec:
    ports:
    - port: 27017
      targetPort: 27017
    type: ClusterIP
    selector:
      app.kubernetes.io/name: mongo
      app.kubernetes.io/component: backend  
  EOF

Apply the yaml file: 

.. code-block:: shell

  oc apply -f /tmp/pxbbq-mongo-restore.yaml

Verify the application has been completely restored
~~~~~~~~~~

Access the application by navigating to the LoadBalancer endpoint and refreshing the page. Our original order will be back in our order history.
If you need to find your LoadBalancer endpoint, use the following command: 

.. code-block:: shell
  
  oc get svc -n pxbbq pxbbq-svc

.. image:: images/pxbbq-5.jpg
  :width: 600

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

Create pre-snap rule for Cassandra
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

Create a ClusterRole, ClusterRoleBinding and Service Account to be used by Cassandra: 

.. code-block:: shell

  cat << EOF >> /tmp/cass-sa.yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: cass-clusterrole
  rules:
    - apiGroups: ['*']
      resources: ['*']
      verbs: ['*']
  ---
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: cass-sa
    namespace: groupsnaps
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: cass-clusterrolebinding
  subjects:
  - kind: ServiceAccount
    name: cass-sa
    namespace: groupsnaps 
    apiGroup: ""
  roleRef:
    kind: ClusterRole
    name: cass-clusterrole
    apiGroup: rbac.authorization.k8s.io
  EOF


.. code-block:: shell

  oc apply -f /tmp/cass-sa.yaml

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
        serviceAccountName: cass-sa
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
            privileged: true
            capabilities:
              add:
                - IPC_LOCK
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "PID= && kill  && while ps -p  > /dev/null; do sleep 1; done"]
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
      serviceAccountName: cass-sa
      image: mikewright/cqlsh
      command:
        - sh
        - -c
        - "exec tail -f /dev/null"
  EOF

Apply the yaml file to create the Cassandra deployment

.. code-block:: shell

  oc apply -f /tmp/cassandra-app.yaml

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
        serviceAccountName: cass-sa
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
            privileged: true
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
  kubectl delete -f /tmp/mongo-snapshot.yaml
  kubectl delete -f /tmp/pxbbq-mongo-restore.yaml -n pxbbq
  kubectl delete -f /tmp/pxbbq-frontend.yaml -n pxbbq
  kubectl delete ns pxbbq
  kubectl delete ns groupsnaps
  kubectl wait --for=delete ns/pxbbq --timeout=60s

To learn more about `Portworx <https://portworx.com/>`__, below are some useful references. 

- `Deploy Portworx on Kubernetes <https://docs.portworx.com/scheduler/kubernetes/install.html>`__
- `Create Portworx volumes <https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-pvcs/>`__
- `Use cases <https://portworx.com/use-case/kubernetes-storage/>`__
