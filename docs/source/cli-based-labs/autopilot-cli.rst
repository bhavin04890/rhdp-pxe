========================================
Automated Volume Expansion using Portworx Autopilot
========================================

Portworx Autopilot is a rule-based engine that responds to changes from a monitoring source. Autopilot allows you to specify monitoring conditions along with actions it should take when those conditions occur.

Configure Autopilot Rule
~~~~~~~~~~

Autopilot rules allow users to create IFTTT (IF This Then That) rules, where Autopilot will monitor for a condition and then perform an action on your behalf.

Let's create a simple rule that will monitor persistent volumes associated with objects that have the app: postgres label and in namespaces that have the label ``type: db``.

It will keep volumes underneath 30% used capacity, and if capacity usage grows to or above 30%, it will automatically grow the persistent volume and underlying filesystem by 100% of the current volume size, up to a maximum volume size of 50Gi:

Keep in mind, an AutoPilot Rule has 4 main parts.

-  ``Selector`` Matches labels on the objects that the rule should monitor.
-  ``Namespace Selector`` Matches labels on the Kubernetes namespaces the rule should monitor. This is optional, and the default is all namespaces.
-  ``Conditions`` The metrics for the objects to monitor.
-  ``Actions`` to perform once the metric conditions are met.


.. code-block:: shell

  cat << EOF >> /tmp/autopilotrule.yaml
  apiVersion: autopilot.libopenstorage.org/v1alpha1
  kind: AutopilotRule
  metadata:
    name: volume-resize
  spec:
    ##### selector filters the objects affected by this rule given labels
    selector:
      matchLabels:
        app: postgres
    ##### namespaceSelector selects the namespaces of the objects affected by this rule
    namespaceSelector:
      matchLabels:
        type: db
    ##### conditions are the symptoms to evaluate. All conditions are AND'ed
    conditions:
      # volume usage should be less than 30%
      expressions:
      - key: "100 * (px_volume_usage_bytes / px_volume_capacity_bytes)"
        operator: Gt
        values:
          - "30"
    ##### action to perform when condition is true
    actions:
    - name: openstorage.io.action.volume/resize
      params:
        # resize volume by scalepercentage of current size
        scalepercentage: "100"
        # volume capacity should not exceed 50GiB
        maxsize: "50Gi"
  EOF

Apply the yaml to create the Portworx Autopilot rule:

.. code-block:: shell

  oc create -f /tmp/autopilotrule.yaml

Create a namespace for demo application
~~~~~~~~~~

Since our Portworx Autopilot rule only targets namespaces that have the label type: db, let's create the yaml for the namespace:

.. code-block:: shell

  cat << EOF >> /tmp/namespaces.yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: pg1
    labels:
      type: db
  EOF

And apply the yaml to create the namespace:

.. code-block:: shell

  oc apply -f /tmp/namespaces.yaml

Deploy PVCs for Postgres
~~~~~~~~~~

Create the yaml for the Postgres DB, and note the label ``app: postgres``:
 
.. code-block:: shell

  cat << EOF >> /tmp/autopilot-postgres.yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pgbench-data
    labels:
      app: postgres
  spec:
    storageClassName: block-sc
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
  ---
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pgbench-state
  spec:
    storageClassName: block-sc
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  EOF

And then apply it:

.. code-block:: shell

  oc create -f /tmp/autopilot-postgres.yaml -n pg1

Ensure that the PVCs are created and bound before deploying Postgres:

.. code-block:: shell

  oc get pvc -n pg1

Deploy Postgres and pgbench pods
~~~~~~~~~~

Create the yaml for Postgres and pgbench, which we'll use to grow the underlying data disk by generating random data:

.. code-block:: shell

  cat << EOF >> /tmp/autopilot-app.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: pgbench
    labels:
      app: pgbench
  spec:
    selector:
      matchLabels:
        app: pgbench
    strategy:
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 1
      type: RollingUpdate
    replicas: 1
    template:
      metadata:
        labels:
          app: pgbench
      spec:
        schedulerName: stork
        containers:
          - image: postgres:9.5
            name: postgres
            ports:
            - containerPort: 5432
            env:
            - name: POSTGRES_USER
              value: pgbench
            - name: POSTGRES_PASSWORD
              value: superpostgres
            - name: PGBENCH_PASSWORD
              value: superpostgres
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: pgbenchdb
          - name: pgbench
            image: portworx/torpedo-pgbench:latest
            imagePullPolicy: "Always"
            env:
              - name: PG_HOST
                value: 127.0.0.1
              - name: PG_USER
                value: pgbench
              - name: SIZE
                value: "15"
            volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: pgbenchdb
            - mountPath: /pgbench
              name: pgbenchstate
        volumes:
        - name: pgbenchdb
          persistentVolumeClaim:
            claimName: pgbench-data
        - name: pgbenchstate
          persistentVolumeClaim:
            claimName: pgbench-state
  EOF

Now deploy the pods:

.. code-block:: shell

  oc create -f /tmp/autopilot-app.yaml -n pg1

Verify write operations from pgbench
~~~~~~~~~~

Wait 10-20 seconds, then take a look at the logs from pgbench to ensure it is generating data:

.. code-block:: shell

  POSTGRES_POD=$(oc get pods -n pg1 | grep 2/2 | awk '{print $1}')
  oc logs $POSTGRES_POD -n pg1 pgbench

Note: If you get an error that the pod does not exist, run the command above until you see the output from pgbench on your terminal screen.

Observe the Portworx Autopilot events
~~~~~~~~~~

Run the following command to observe the state changes for Portworx Autopilot:

.. code-block:: shell

  watch oc get events --field-selector involvedObject.kind=AutopilotRule,involvedObject.name=volume-resize --all-namespaces --sort-by .lastTimestamp

You will see Portworx Autopilot move through the following states as it monitors volumes and takes actions defined in Portworx Autopilot rules:

1. Initializing (Detected a volume to monitor via applied rule conditions)
2. Normal (Volume is within defined conditions and no action is necessary)
3. Triggered (Volume is no longer within defined conditions and action is necessary)
4. ActiveActionsPending (Corrective action is necessary but not executed yet)
5. ActiveActionsInProgress (Corrective action is under execution)
6. ActiveActionsTaken (Corrective action is complete)

Once you see ActiveActionsTaken in the event output, click CTRL+C to exit the watch command.

Verify the Volume Expansion
~~~~~~~~~~

Now let's take a look at our PVCs - note the automatic expansion of the volume occurred with no human interaction and no application interruption:

.. code-block:: shell

  oc get pvc -n pg1

You've just configured Portworx Autopilot and observed how it can perform automated capacity management based on rules you configure, and be able to "right size" your underlying persistent storage as it is needed!

Wrap up this module 
~~~~~~~~~~

Use the following commands to delete objects used for this specific scenario:

.. code-block:: shell
  
  oc delete -f /tmp/autopilot-app.yaml -n pg1
  oc delete -f /tmp/autopilot-postgres.yaml -n pg1
  oc delete -f /tmp/autopilotrule.yaml
  oc delete ns pg1
  oc wait --for=delete ns/pg1 --timeout=60s

