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
        app: fio
    ##### namespaceSelector selects the namespaces of the objects affected by this rule
    namespaceSelector:
      matchLabels:
        type: fio
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
    name: fio
    labels:
      type: fio
  EOF

And apply the yaml to create the namespace:

.. code-block:: shell

  oc apply -f /tmp/namespaces.yaml

Deploy ConfigMap for FIO
~~~~~~~~~~

Create the yaml for the FIO configuration:
 
.. code-block:: shell

  cat << EOF >> /tmp/fio-cm.yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: fio-job-config
  data:
      fio.job: |
          [global]
          name=integrity-test
          directory=/scratch/
          rw=write
          blocksize_range=4k-512k
          direct=1
          do_verify=1
          verify=meta
          verify_pattern=0xdeadbeef
          end_fsync=1
          time_based=1
          filename=pxdtest
          runtime=99999999
          [file1]
          filesize=70G
          ioengine=libaio
          iodepth=128
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: grok-exporter
  data:
    config.yml: |-
      global:
        config_version: 3
      input:
        type: file
        path: /logs/fio.log
        readall: false
        fail_on_missing_logfile: true
      imports:
      - type: grok_patterns
        dir: ./patterns
      grok_patterns:
      - 'FIO_IOPS [0-9]*[.][0-9]k$|[0-9]*'
      metrics:
          - type: gauge
            name: iops
            help: FIO IOPS Write Gauge Metrics
            match: '  write: %{GREEDYDATA}, iops=%{NUMBER:val1}%{GREEDYDATA:thsd}, %{GREEDYDATA}'
            value: '{{`{{if eq .thsd "k"}}{{multiply .val1 1000}}{{else}}{{.val1}}{{end}}`}}'
            labels:
                iops_suffix: '{{`{{.thsd}}`}}'
            cumulative: false
            retention: 1s
          - type: gauge
            name: bandwidth
            help: FIO Bandwidth Write Gauge Metrics
            match: '  write: io=%{GREEDYDATA}, bw=%{NUMBER:val2}%{GREEDYDATA:kbs}, %{GREEDYDATA}, %{GREEDYDATA}'
            value: '{{`{{if eq .kbs "KB/s"}}{{divide .val2 1024}}{{else}}{{.val2}}{{end}}`}}'
            labels:
                bw_unit: '{{`{{.kbs}}`}}'
            cumulative: false
            retention: 1s
          - type: gauge
            name: avg_latency
            help: FIO AVG Latency Write Gauge Metrics
            match: '     lat (%{GREEDYDATA:nsec}): min=%{GREEDYDATA}, max=%{GREEDYDATA}, avg=%{NUMBER:val3}, stdev=%{GREEDYDATA}'
            value: '{{`{{if eq .nsec "(usec)"}}{{divide .val3 1000}}{{else}}{{.val3}}{{end}}`}}'
            labels:
                lat_unit: '{{`{{.nsec}}`}}'
            cumulative: false
            retention: 1s
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: fio-ready-probe
  data:
    ready-probe.sh: |
      #!/bin/bash
      if [ `cat /root/fio.log | grep 'error\|bad magic header' | wc -l` -ge 1 ]; then
        exit 1;
      else
        exit 0;
      fi  
  EOF

And then apply it:

.. code-block:: shell

  oc create -f /tmp/fio-cm.yaml -n fio

Ensure that the configmap is deployed: 

.. code-block:: shell

  oc get pvc -n fio

Deploy FIO pods 
~~~~~~~~~~

Create the yaml for fio, which we'll use to grow the underlying data disk by generating random data:

.. code-block:: shell

  cat << EOF >> /tmp/fio-app.yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: fio
  spec:
    serviceName: fio
    replicas: 1
    selector:
      matchLabels:
        app: fio
    template:
      metadata:
        labels:
          app: fio
      spec:
        schedulerName: stork
        containers:
        - name: fio
          image: portworx/fio_drv
          command: ["fio"]
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: "1"
              memory: 4Gi
          args: ["/configs/fio.job", "--status-interval=1", "--eta=never", "--output=/logs/fio.log"]
          volumeMounts:
          - name: fio-config-vol
            mountPath: /configs
          - name: fio-data
            mountPath: /scratch
          - name: fio-log
            mountPath: /logs
        - name: grok
          image: pwxvin/grok-exporter:v1.0.0-RC4
          imagePullPolicy: IfNotPresent
          ports:
          - name: grok-port
            containerPort: 9144
            protocol: TCP
          volumeMounts:
          - name: grok-config-volume
            mountPath: /etc/grok_exporter
          - name: fio-log
            mountPath: /logs
        volumes:
        - name: fio-config-vol
          configMap:
            name: fio-job-config
        - name: grok-config-volume
          configMap:
            name: grok-exporter
    volumeClaimTemplates:
    - metadata:
        name: fio-data
      spec:
        storageClassName: block-sc
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
    - metadata:
        name: fio-log
      spec:
        storageClassName: block-sc
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: grok-exporter-svc
    labels:
      app: fio
  spec:
    clusterIP: None
    selector:
      app: fio
    ports:
    - name: grok-port
      port: 9144
      targetPort: 9144
  EOF

Now deploy the pods:

.. code-block:: shell

  oc create -f /tmp/fio-app.yaml -n fio

.. code-block:: shell
  
  oc get pods,pvc -n fio

Observe the Portworx Autopilot events
~~~~~~~~~~
Wait for a couple of minutes and run the following command to observe the state changes for Portworx Autopilot:

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

  oc get pvc -n fio

You've just configured Portworx Autopilot and observed how it can perform automated capacity management based on rules you configure, and be able to "right size" your underlying persistent storage as it is needed!

Wrap up this module 
~~~~~~~~~~

Use the following commands to delete objects used for this specific scenario:

.. code-block:: shell
  
  oc delete -f /tmp/fio-app.yaml -n fio
  oc delete -f /tmp/fio-cm.yaml -n fio
  oc delete -f /tmp/autopilotrule.yaml
  oc delete ns fio
  oc wait --for=delete ns/fio --timeout=60s

