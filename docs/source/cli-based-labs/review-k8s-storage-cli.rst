=======================================
Lab 02 - StorageClasses, RWO, and RWX Volumes
=======================================

In this scenario, you'll learn about Portworx Enterprise StorageClass parameters and deploy demo applications that use RWO (ReadWriteOnce) and RWX (ReadWriteMany) Persistent Volumes provisioned by Portworx Enterprise.

Deploying Portworx Storage Classes
-----------------------

Portworx provides the ability for users to leverage a unified storage pool to dynamically provision both Block-based (ReadWriteOnce) and File-based (ReadWriteMany) volumes for applications running on your Kubernetes cluster without having to provision multiple CSI drivers/plugins, and without the need for specific backing storage devices!

Deploy StorageClass for Block (ReadWriteOnce) volumes
~~~~~~~~~~
Run the following command to create a new yaml file for the block-based StorageClass configuration:


.. code-block:: shell

  cat << EOF >> /tmp/block-sc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: block-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "3"
    priority_io: "high"
    io_profile: "auto"
  allowVolumeExpansion: true
  EOF

PVCs provisioned using the above StorageClass will have a replication factor of 3, which means there will be three replicas of the PVC spread across the Kubernetes worker nodes.

Now, let's use the following command to apply this yaml file and deploy the StorageClass on our Kubernetes cluster:


.. code-block:: shell

  oc create -f /tmp/block-sc.yaml

Deploy StorageClass for File (ReadWriteMany) volumes
~~~~~~~~~~

Run the following command to create a new yaml file for the file-based StorageClass configuration:


.. code-block:: shell

  cat << EOF >> /tmp/file-sc.yaml
  kind: StorageClass
  apiVersion: storage.k8s.io/v1
  metadata:
    name: file-sc
  provisioner: pxd.portworx.com
  parameters:
    repl: "2"
    priority_io: "high"
    sharedv4: "true"
    sharedv4_svc_type: "ClusterIP"
    sharedv4_failover_strategy: "aggressive"
  EOF

PVCs provisioned using the above StorageClass can be accessed by multiple pods at the same time (ReadWriteMany) and will have a replication factor of 2.

Now, let's use the following command to apply this yaml file and deploy the StorageClass on our Kubernetes cluster:


.. code-block:: shell

  oc create -f /tmp/file-sc.yaml

Deploying demo application for ReadWriteOnce volumes
-----------------------
In this step, we will deploy a demo application that provisions a PostgreSQL database that uses a ReadWriteOnce volume to store data.

Deploy StorageClass for Block (ReadWriteOnce) volumes
~~~~~~~~~~


.. code-block:: shell

  oc create ns demo

Deploy the PostgreSQL database resources in the "demo" namespace
~~~~~~~~~~


.. code-block:: shell 

  cat << EOF >> /tmp/postgres-db.yaml
  ---   
  ##### Portworx persistent volume claim
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: postgres-data
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

  oc create -f /tmp/postgres-db.yaml -n demo

Deploy the front-end components for the application in the `demo` namespace
~~~~~~~~~~


.. code-block:: shell

  cat << EOF >> /tmp/k8s-webapp.yaml
  # DEMO APP
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
      name: k8s-counter-web
  EOF


.. code-block:: shell

  oc apply -f /tmp/k8s-webapp.yaml -n demo

Monitor the application deployment using the following command:
~~~~~~~~~~


.. code-block:: shell

  watch oc get all -n demo

When all of the pods are running, press `CTRL+C` to exit.

Create some data using the app:
~~~~~~~~~~

Use the following commnad to fetch the LoadBalancer endpoint for the k8s-counter-web service in the demo namespace and navigate to it using a new browser tab. 


.. code-block:: shell

  oc get svc -n demo k8s-counter-service

Click anywhere on the blank screen to generate Kubernetes logos. The (X,Y) pixel coordinates for these logos are stored in the backend Postgres database.

Inspect the Postgres volume
~~~~~~~~~~

Use the following command to inspect the Postgres volume and look at the Portworx parameters configured for the volume:


.. code-block:: shell

  VOL=`oc get pvc -n demo | grep postgres-data | awk '{print $3}'`
  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume inspect ${VOL}

Observe how Portworx creates volume replicas, and spreads them across your Kubernetes worker nodes.

List entries from the PostgreSQL database
~~~~~~~~~~

To look at the Postgres entries generated because of your interaction with the demo application, first get a bash shell on the Postgres pod:


.. code-block:: shell

  POD=$(oc get pods -l app=postgres -n demo | grep 1/1 | awk '{print $1}')
  oc exec -it $POD -n demo -- bash

Then, let's use psql to take a look at the contents of our database, where you should see the x/y coordinates of the logos you generated:


.. code-block:: shell

  psql -U $POSTGRES_USER
  \c postgres
  select * from mywhales;
  \q
  exit

In this step, you saw how Portworx can dynamically provisions a highly available ReadWriteOnce persistent volume for your application.

Deploying demo application for ReadWriteMany volumes
-----------------------

Portworx offers a `sharedv4 service` volume which allows applications to connect to the shared persistent volume either using a ClusterIP or a LoadBalancer endpoint. This is advantageous as even if one of the worker node goes down, the shared volume is still accessible without any interruption of the application utilizing the data on the shared volume.

Create the `sharedservice` namespace:
~~~~~~~~~~


.. code-block:: shell

  oc create ns sharedservice

Deploy the sharedv4 service PVC
~~~~~~~~~~
Review the yaml for the RWX PVC:


.. code-block:: shell

  cat << EOF >> /tmp/sharedpvc.yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: px-sharedv4-pvc
    annotations:
      volume.beta.kubernetes.io/storage-class: file-sc
  spec:
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 10Gi
  EOF

Then apply the yaml to create the PVC:


.. code-block:: shell

  oc apply -f /tmp/sharedpvc.yaml -n sharedservice

Deploy the busybox pods
~~~~~~~~~~

Create a new yaml file to deploy the busybox pod yaml we'll be using:


.. code-block:: shell 

  cat << EOF >> /tmp/busyboxpod.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: shared-demo
    name: shared-busybox
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: shared-demo
    template:
      metadata:
        labels:
          app: shared-demo
      spec:
        volumes:
        - name: shared-vol
          persistentVolumeClaim:
            claimName: px-sharedv4-pvc
        terminationGracePeriodSeconds: 5
        containers:
        - image: busybox
          imagePullPolicy: Always
          name: busybox
          volumeMounts:
          - name: shared-vol
            mountPath: "/mnt"
          command:
            - sh
          args:
            - -c
            - |
              while true; do
                echo -e "{\"time\":\$(date +%H:%M:%S),\"hostname\":\$(hostname) writing to shared vol }""\n" >> /mnt/shared.log
                sleep 1
              done
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: shared-demo-reader
  spec:
    volumes:
    - name: shared-vol
      persistentVolumeClaim:
        claimName: px-sharedv4-pvc
    terminationGracePeriodSeconds: 5
    containers:
    - image: busybox
      imagePullPolicy: Always
      name: busybox
      volumeMounts:
      - name: shared-vol
        mountPath: "/mnt"
      command:
        - sh
      args:
        - -c
        - |
          while true; do
            tail -f /mnt/shared.log
          done
    EOF
  
Then apply the yaml to create the deployment and reader pod:


.. code-block:: shell 

  oc apply -f /tmp/busyboxpod.yaml -n sharedservice
 
This creates a deployment using multiple simple busybox pods that have mounted and will constantly write to the shared persistent volume. It also deploys a single busybox pod that will constantly read from the shared persistent volume.

Inspect the volume
~~~~~~~~~~

Let's take a look at what information Portworx gives us about our shared volume:


.. code-block:: shell

  VolName=$(pxctl volume list | grep "10 GiB" | awk '{print $2}' )
  PX_POD=$(oc get pods -l name=portworx -n portworx -o jsonpath='{.items[0].metadata.name}')
  oc exec -it $PX_POD -n portworx -- /opt/pwx/bin/pxctl volume inspect ${VolName}

Note that we have four pods accessing the RWX volume for our demo!

Simulate Node failure
~~~~~~~~~~

Inspect the sharedv4service Endpoint:


.. code-block:: shell

  oc describe svc -n sharedservice

Let's get the external IP of the node that is currently serving the traffic for the shared volume in the variable `NODE` - this is the "Endpoints" in the output of the command we ran above:


.. code-block:: shell

  NODEIP=$(oc describe svc -n sharedservice | grep Endpoints: | awk -F ":" '{print $2}')
  EXTERNALIP=$(oc get nodes -o wide | grep $NODEIP | awk '{print $7}')
  NODE=$(cat .ssh/config | grep -B 1 $EXTERNALIP | awk '{print $2}' | grep node)
  echo "sharedv4service is serving traffic through node: $NODE"

Now let's reboot the node that is currently set as the endpoint for the sharedv4 service:


.. code-block:: shell 

  ssh $NODE sudo reboot

Inspect the log file to ensure that there was no application interruption due to node failure
~~~~~~~~~~

Let's tail the logs of the reader pod which is reading the log file being written to by the other three pods:


.. code-block:: shell

  oc logs shared-demo-reader -n sharedservice -f

Press `CTRL-C` to exit the oc logs command.

Inspect the sharedv4 service again:
~~~~~~~~~~

Use the following commmand to verify that the sharedv4 service endpoint changed to different node in the Kubernetes cluster.


.. code-block:: shell 

  NODEIP2=$(oc describe svc -n sharedservice | grep Endpoints: | awk -F ":" '{print $2}')
  EXTERNALIP=$(oc get nodes -o wide | grep $NODEIP2 | awk '{print $7}')
  NODE2=$(cat .ssh/config | grep -B 1 $EXTERNALIP | awk '{print $2}' | grep node)
  echo "sharedv4service is serving traffic through node: $NODE2, previously served by $NODE."

You've just deployed applications with different needs on the same Kubernetes cluster without the need to install multiple CSI drivers/plugins, and it will function exactly the same way no matter what backing storage you provide for Portworx Enterprise to use!

Wrap up this module
-----------------------
Use the following commands to delete objects used for this specific scenario:


.. code-block:: shell 

  oc delete -f busyboxpod.yaml -n sharedservice
  oc delete -f sharedpvc.yaml -n sharedservice
  oc delete ns sharedservice
  oc wait --for=delete ns/sharedservice --timeout=60s
