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

Deploy the mongodb backend for the application
~~~~~~~~~~

.. code-block:: shell

  cat << EOF >> /tmp/pxbbq-mongo-tc.yaml
  ---
  apiVersion: "v1"
  kind: "PersistentVolumeClaim"
  metadata: 
    name: "mongodb-pvc"
    namespace: "trashcan"
    labels: 
      app: "mongo-db"
  spec: 
    accessModes: 
      - ReadWriteOnce
    resources: 
      requests: 
        storage: 5Gi
    storageClassName: trash-sc
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: mongo
    labels:
      app.kubernetes.io/name: mongo
      app.kubernetes.io/component: backend
    namespace: trashcan
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
            claimName: mongodb-pvc
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: mongo
    labels:
      app.kubernetes.io/name: mongo
      app.kubernetes.io/component: backend
    namespace: trashcan
  spec:
    ports:
    - port: 27017
      targetPort: 27017
    type: ClusterIP
    selector:
      app.kubernetes.io/name: mongo
      app.kubernetes.io/component: backend
  EOF

.. code-block:: shell

  oc create -f /tmp/pxbbq-mongo-tc.yaml

Deploy the front-end components for the application
~~~~~~~~~~

.. code-block:: shell

  cat << EOF >> /tmp/pxbbq-frontend-tc.yaml
  ---
  apiVersion: apps/v1
  kind: Deployment                 
  metadata:
    name: pxbbq-web  
    namespace: trashcan
  spec:
    replicas: 3                    
    selector:
      matchLabels:
        app: pxbbq-web
    template:                      
      metadata:
        labels:                    
          app: pxbbq-web
      spec:                        
        containers:
        - name: pxbbq-web
          image: eshanks16/pxbbq:v3.2
          env:
          - name: MONGO_INIT_USER
            value: "porxie" #Mongo User with permissions to create additional databases and users. Typically "porxie" or "pds"
          - name: MONGO_INIT_PASS
            value: "porxie" #Required to connect the init user to the database. If using the mongodb yaml supplied, use "porxie"
          - name: MONGO_NODES
            value: "mongo" #COMMA SEPARATED LIST OF MONGO ENDPOINTS. Example: mongo1.dns.name,mongo2.dns.name
          - name: MONGO_PORT
            value: "27017"
          - name: MONGO_USER
            value: porxie #Mongo DB User that will be created by using the Init_User
          - name: MONGO_PASS
            value: "porxie" #Mongo DB Password for User that will be created by using the Init User
          imagePullPolicy: Always
          ports:
            - containerPort: 8080    
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: pxbbq-svc
    namespace: trashcan
    labels:
      app: pxbbq-web
  spec:
    ports:
    - port: 80
      targetPort: 8080
    type: LoadBalancer
    selector:
      app: pxbbq-web
  EOF

.. code-block:: shell

  oc apply -f /tmp/pxbbq-frontend-tc.yaml

Access the application
~~~~~~~~~~

Access the demo application using the LoadBalancer endpoint from the command below, and place some orders to store in the backend MongoDB database. If you need help placing orders, please refer to the 3.2.6 module of the workshop. 

.. code-block:: shell
   
  oc get svc -n trashcan pxbbq-svc

Delete the demo application
~~~~~~~~~~

Next, let's "accidentally" delete the postgres pod and persistent volume:

.. code-block:: shell

  oc delete -f /tmp/pxbbq-mongo-tc.yaml

Wait for the delete to complete before continuing.

Once the MongoDB is deleted, navigate back to the Demo App tab to verify that it stopped working. Click on the refresh icon to the right of the tabs just to make sure - once the DB pod has been deleted, the app should be unreachable.

Restoring volume from Volume Trashcan
~~~~~~~~~~

Let's use pxctl commands to restore our volume from the trashcan:

.. code-block:: shell
  
  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  VolMongo=$(oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume list --trashcan | grep "5 GiB" | awk '{print $8}')

.. code-block:: shell

  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume restore --trashcan $VolMongo pvc-restoredvol
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
      storage: 5Gi
    claimRef:
      apiVersion: v1
      kind: PersistentVolumeClaim
      name: mongodb-pvc
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

  oc apply -f /tmp/pxbbq-mongo-tc.yaml

Delete the old web front end:

.. code-block:: shell

  oc delete deploy pxbbq-web -n trashcan

And redeploy the web front end: 

.. code-block:: shell

  oc apply -f /tmp/pxbbq-frontend-tc.yaml

Verify the restore by accessing the app
~~~~~~~~~~

Navigate to the Demo App UI by using the LoadBalancer endpoint from the command below and see that our orders are back! 

.. code-block:: shell

  oc get svc -n trashcan pxbbq-svc

This is how Portworx allows users to use the Trash Can feature to recover accidentally deleted persistent volumes. This prevents additional downtime and reduces ticket churn for data restoration due to human error!

Wrap up this module
~~~~~~~~~~

Use the following commands to delete objects used for this specific scenario:

.. code-block:: shell

  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl cluster options update --volume-expiration-minutes 0
  
.. code-block:: shell 
  
  oc delete -f /tmp/pxbbq-frontend-tc.yaml 
  oc delete -f /tmp/pxbbq-mongo-tc.yaml
  oc delete ns trashcan
  oc wait --for=delete ns/trashcan --timeout=60s