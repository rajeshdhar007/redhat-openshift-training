In this lab you will learn how to: 
* Configure and deploy containerized Ceph and NooBaa
* Validate deployment of containerized Ceph and NooBaa
* Deploy the Rook toolbox to run Ceph and RADOS commands
* Creating a Read-Write-Once (RWO) PVC that is based on Ceph RBDs
* Creating a Read-Write-Many (RWX) PVC that is based on CephFS
* Use OCS for Prometheus and AlertManager storage
* Use the MCG to create a bucket and use in an application
* Add more storage to the Ceph cluster
* Use must-gather to collect support information

Scale OCP cluster and add 3 new nodes
```
oc get nodes -l node-role.kubernetes.io/worker -l '!node-role.kubernetes.io/infra','!node-role.kubernetes.io/master'
	NAME                                         STATUS   ROLES    AGE   VERSION
	ip-10-0-142-4.us-east-2.compute.internal     Ready    worker   25h   v1.16.2
	ip-10-0-147-192.us-east-2.compute.internal   Ready    worker   25h   v1.16.2
	ip-10-0-164-147.us-east-2.compute.internal   Ready    worker   25h   v1.16.2
```
Add 3 more OCP compute nodes to cluster using machinesets
```
oc get machinesets -n openshift-machine-api | grep -v infra
	NAME                                   DESIRED   CURRENT   READY   AVAILABLE   AGE
	cluster-2f83-d7fvm-worker-us-east-2a   1         1         1       1           25h
	cluster-2f83-d7fvm-worker-us-east-2b   1         1         1       1           25h
	cluster-2f83-d7fvm-worker-us-east-2c   1         1         1       1           25h
```
Create new MachineSets that will run storage-specific nodes for your OCP cluster
```
[~] $ bash /opt/app-root/src/support/machineset-generator.sh 3 workerocs 0 | oc create -f -
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2a created
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2b created
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2c created
	
[~] $ oc get machineset -n openshift-machine-api -l machine.openshift.io/cluster-api-machine
-role=workerocs -o name | xargs oc patch -n openshift-machine-api --type='json' -p '[{"op":
"add", "path": "/spec/template/spec/metadata/labels", "value":{"node-role.kubernetes.io/work
er":"", "role":"storage-node", "cluster.ocs.openshift.io/openshift-storage":""} }]'
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2a patched
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2b patched
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2c patched
	
[~] $ oc get machineset -n openshift-machine-api -l machine.openshift.io/cluster-api-machine
-role=workerocs -o name | xargs oc scale -n openshift-machine-api --replicas=1
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2a scaled
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2b scaled
	machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2c scaled
```
Check that you have new machines created
```
oc get machines -n openshift-machine-api | egrep 'NAME|workerocs'
	NAME                                            PHASE         TYPE         REGION      ZONE
	        AGE
	cluster-2f83-d7fvm-workerocs-us-east-2a-zt2gl   Provisioned   m5.4xlarge   us-east-2   us-ea
	st-2a   3m36s
	cluster-2f83-d7fvm-workerocs-us-east-2b-sdb6t   Provisioned   m5.4xlarge   us-east-2   us-ea
	st-2b   3m36s
	cluster-2f83-d7fvm-workerocs-us-east-2c-4dnpk   Provisioned   m5.4xlarge   us-east-2   us-ea
	st-2c   3m36s

oc get machines -n openshift-machine-api | egrep 'NAME|workerocs'
	NAME                                            PHASE     TYPE         REGION      ZONE
	    AGE
	cluster-2f83-d7fvm-workerocs-us-east-2a-zt2gl   Running   m5.4xlarge   us-east-2   us-east-2
	a   4m42s
	cluster-2f83-d7fvm-workerocs-us-east-2b-sdb6t   Running   m5.4xlarge   us-east-2   us-east-2
	b   4m42s
	cluster-2f83-d7fvm-workerocs-us-east-2c-4dnpk   Running   m5.4xlarge   us-east-2   us-east-2
	c   4m42s


watch "oc get machinesets -n openshift-machine-api | egrep 'NAME|workerocs'"
	Every 2.0s: oc get machinesets -n openshift-machine-api | egrep...  Fri Dec 11 16:11:31 2020
	
	NAME                                      DESIRED   CURRENT   READY   AVAILABLE   AGE
	cluster-2f83-d7fvm-workerocs-us-east-2a   1         1         1       1           6m57s
	cluster-2f83-d7fvm-workerocs-us-east-2b   1         1         1       1           6m57s
	
oc get nodes -l node-role.kubernetes.io/worker -l '!node-role.kubernetes.io/infra','!node-role.kubernetes.io/master'
	NAME                                         STATUS   ROLES    AGE     VERSION
	ip-10-0-138-119.us-east-2.compute.internal   Ready    worker   3m42s   v1.16.2
	ip-10-0-142-4.us-east-2.compute.internal     Ready    worker   25h     v1.16.2
	ip-10-0-147-192.us-east-2.compute.internal   Ready    worker   25h     v1.16.2
	ip-10-0-151-151.us-east-2.compute.internal   Ready    worker   3m41s   v1.16.2
	ip-10-0-164-147.us-east-2.compute.internal   Ready    worker   25h     v1.16.2
	ip-10-0-170-7.us-east-2.compute.internal     Ready    worker   3m42s   v1.16.2
```
check to make sure the new OCP nodes have the new OCS label.
```
oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
	NAME                                         STATUS   ROLES    AGE     VERSION
	ip-10-0-138-119.us-east-2.compute.internal   Ready    worker   5m41s   v1.16.2
	ip-10-0-151-151.us-east-2.compute.internal   Ready    worker   5m40s   v1.16.2
	ip-10-0-170-7.us-east-2.compute.internal     Ready    worker   5m41s   v1.16.2
```
# Installing the OCS operator
The following will be installed:
* Groups and sources for the OCS operators
* An OCS subscription
* All OCS resources (Operators, Ceph pods, Noobaa pods, StorageClasses)

Start with creating the openshift-storage namespace.
```
oc create namespace openshift-storage
	namespace/openshift-storage created
```
Add the monitoring label to this namespace. This is required to get prometheus metrics and alerts for the OCP storage dashboards. To label the openshift-storage namespace use the following command:
```
oc label namespace openshift-storage "openshift.io/cluster-monitoring=true"
	namespace/openshift-storage labeled
```
<img src=/images/mod01/001.png>
<img src=/images/mod01/002.png>
<img src=/images/mod01/003.png>

```
watch oc -n openshift-storage get csv
	Every 2.0s: oc -n openshift-storage get csv                         Fri Dec 11 16:31:55 2020
	
	NAME                  DISPLAY                       VERSION   REPLACES   PHASE
	ocs-operator.v4.5.2   OpenShift Container Storage   4.5.2                Succeeded
```

The resource csv is a shortened word for clusterserviceversions.operators.coreos.com.

New operator pods in openshift-storage namespace:
```
oc -n openshift-storage get pods
	NAME                                  READY   STATUS    RESTARTS   AGE
	noobaa-operator-5484d8f45c-rfzr7      1/1     Running   0          11m
	ocs-operator-5df89454f7-7nbw2         1/1     Running   0          11m
	rook-ceph-operator-689fd87b48-jgc65   1/1     Running   0          11m
```
<img src=/images/mod01/004.png>
<img src=/images/mod01/005.png>
<img src=/images/mod01/006.png>
<img src=/images/mod01/007.png>
<img src=/images/mod01/008.png>

```
oc -n openshift-storage get pods
	NAME                                                              READY   STATUS      RESTARTS   AGE
	csi-cephfsplugin-4n8z6                                            3/3     Running     0          53m
	csi-cephfsplugin-7wmgr                                            3/3     Running     0          53m
	csi-cephfsplugin-hmdbp                                            3/3     Running     0          53m
	csi-cephfsplugin-m2n2r                                            3/3     Running     0          53m
	csi-cephfsplugin-n5gnx                                            3/3     Running     0          53m
	csi-cephfsplugin-provisioner-7b66d96d9c-47tqv                     5/5     Running     0          53m
	csi-cephfsplugin-provisioner-7b66d96d9c-jx6pv                     5/5     Running     0          53m
	csi-cephfsplugin-sntxm                                            3/3     Running     0          53m
	csi-cephfsplugin-xzbbj                                            3/3     Running     0          53m
	csi-cephfsplugin-zdjp7                                            3/3     Running     0          53m
	csi-cephfsplugin-zhnzq                                            3/3     Running     0          53m
	csi-rbdplugin-6mg26                                               3/3     Running     0          53m
	csi-rbdplugin-6wtwb                                               3/3     Running     0          53m
	csi-rbdplugin-bv6qr                                               3/3     Running     0          53m
	csi-rbdplugin-cml8r                                               3/3     Running     0          53m
	csi-rbdplugin-fd4mp                                               3/3     Running     0          53m
	csi-rbdplugin-kbj7l                                               3/3     Running     0          53m
	csi-rbdplugin-lplnj                                               3/3     Running     0          53m
	csi-rbdplugin-plnk2                                               3/3     Running     0          53m
	csi-rbdplugin-provisioner-6bf9d5db4c-fl5z9                        5/5     Running     0          53m
	csi-rbdplugin-provisioner-6bf9d5db4c-lmxht                        5/5     Running     0          53m
	csi-rbdplugin-wnz56                                               3/3     Running     0          53m
	noobaa-core-0                                                     1/1     Running     0          48m
	noobaa-db-0                                                       1/1     Running     0          48m
	noobaa-endpoint-6f94dcc4f6-82bqb                                  1/1     Running     0          46m
	noobaa-operator-5484d8f45c-rfzr7                                  1/1     Running     0          80m
	ocs-operator-5df89454f7-7nbw2                                     1/1     Running     0          80m
	rook-ceph-crashcollector-ip-10-0-138-119-586f74948f-jmg8t         1/1     Running     0          52m
	rook-ceph-crashcollector-ip-10-0-151-151-db5c76f4d-7nxv6          1/1     Running     0          49m
	rook-ceph-crashcollector-ip-10-0-170-7-559d5db597-nszf5           1/1     Running     0          51m
	rook-ceph-drain-canary-0c597d959957bccde739a8a71b0c9d74-f627ph4   1/1     Running     0          48m
	rook-ceph-drain-canary-32e194071d0d4db6607535d9a307e05d-6brzl29   1/1     Running     0          48m
	rook-ceph-drain-canary-ip-10-0-170-7.us-east-2.compute.intvrbgh   1/1     Running     0          48m
	rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-866d4c64cw6q5   1/1     Running     0          48m
	rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-9cdb6b4bsdcbh   1/1     Running     0          48m
	rook-ceph-mgr-a-6f9685f5d9-vcjbs                                  1/1     Running     0          49m
	rook-ceph-mon-a-56944467c8-4gscb                                  1/1     Running     0          52m
	rook-ceph-mon-b-b9cd6cc58-pd66d                                   1/1     Running     0          51m
	rook-ceph-mon-c-5c8567756-np6vx                                   1/1     Running     0          49m
	rook-ceph-operator-689fd87b48-jgc65                               1/1     Running     0          80m
	rook-ceph-osd-0-7545bdff4b-dgqcj                                  1/1     Running     0          48m
	rook-ceph-osd-1-6f44b68f99-c9j5h                                  1/1     Running     0          48m
	rook-ceph-osd-2-74c787d6fd-csbxr                                  1/1     Running     0          48m
	rook-ceph-osd-prepare-ocs-deviceset-0-data-0-xd7qf-tx42m          0/1     Completed   0          48m
	rook-ceph-osd-prepare-ocs-deviceset-1-data-0-cgqmz-rp6vm          0/1     Completed   0          48m
	rook-ceph-osd-prepare-ocs-deviceset-2-data-0-g8lx9-cscd5          0/1     Completed   0          48m

oc -n openshift-storage get sc
	NAME                          PROVISIONER                             AGE
	gp2 (default)                 kubernetes.io/aws-ebs                   26h
	ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      62m
	ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   62m
	openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         55m
```
<img src=/images/mod01/009.png>

Using the Rook-Ceph toolbox to check on the Ceph backing storage
Since the Rook-Ceph toolbox is not shipped with OCS, we need to deploy it manually.
```
oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch  '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'
	ocsinitialization.ocs.openshift.io/ocsinit patched
```
After the rook-ceph-tools Pod is Running you can access the toolbox like this:
```
[~] $ TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
[~] $ oc rsh -n openshift-storage $TOOLS_POD
	sh-4.4#
```

Try out the following Ceph commands:
```
sh-4.4# ceph status
  cluster:
    id:     1141df56-ed91-429d-aa21-863d7043bb3e
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 62m)
    mgr: a(active, since 62m)
    mds: ocs-storagecluster-cephfilesystem:1 {0=ocs-storagecluster-cephfilesystem-a=up:active} 1 up:
standby-replay
    osd: 3 osds: 3 up (since 61m), 3 in (since 61m)

  task status:
    scrub status:
        mds.ocs-storagecluster-cephfilesystem-a: idle
        mds.ocs-storagecluster-cephfilesystem-b: idle

  data:
    pools:   3 pools, 96 pgs
    objects: 95 objects, 93 MiB
    usage:   3.1 GiB used, 6.0 TiB / 6 TiB avail
    pgs:     96 active+clean

  io:
    client:   853 B/s rd, 6.7 KiB/s wr, 1 op/s rd, 0 op/s wr

sh-4.4# ceph osd status
	+----+--------------------------------------------+-------+-------+--------+---------+--------+---------+-----------+
	| id |                    host                    |  used | avail | wr ops | wr data | rd ops | rd data |   state   |
	+----+--------------------------------------------+-------+-------+--------+---------+--------+---------+-----------+
	| 0  |  ip-10-0-170-7.us-east-2.compute.internal  | 1064M | 2046G |    0   |  2457   |    0   |     0   | exists,up |
	| 1  | ip-10-0-138-119.us-east-2.compute.internal | 1064M | 2046G |    0   |  3276   |    2   |   106   | exists,up |
	| 2  | ip-10-0-151-151.us-east-2.compute.internal | 1064M | 2046G |    0   |     0   |    0   |     0   | exists,up |
	+----+--------------------------------------------+-------+-------+--------+---------+--------+---------+-----------+

sh-4.4# ceph osd tree
	ID  CLASS WEIGHT  TYPE NAME                                     STATUS REWEIGHT PRI-AFF
	 -1       6.00000 root default
	 -5       6.00000     region us-east-2
	-10       2.00000         zone us-east-2a
	 -9       2.00000             host ocs-deviceset-2-data-0-g8lx9
	  1   ssd 2.00000                 osd.1                             up  1.00000 1.00000
	-14       2.00000         zone us-east-2b
	-13       2.00000             host ocs-deviceset-1-data-0-cgqmz
	  2   ssd 2.00000                 osd.2                             up  1.00000 1.00000
	 -4       2.00000         zone us-east-2c
	 -3       2.00000             host ocs-deviceset-0-data-0-xd7qf
	  0   ssd 2.00000                 osd.0                             up  1.00000 1.00000

sh-4.4# ceph df
	RAW STORAGE:
	    CLASS     SIZE      AVAIL       USED        RAW USED     %RAW USED
	    ssd       6 TiB     6.0 TiB     122 MiB      3.1 GiB          0.05
	    TOTAL     6 TiB     6.0 TiB     122 MiB      3.1 GiB          0.05
	
	POOLS:
	    POOL                                           ID     STORED      OBJECTS     USED        %USED     MAX AVAIL
	    ocs-storagecluster-cephblockpool                1      41 MiB          73     122 MiB         0       1.7 TiB
	    ocs-storagecluster-cephfilesystem-metadata      2     2.2 KiB          22      96 KiB         0       1.7 TiB
	    ocs-storagecluster-cephfilesystem-data0         3         0 B           0         0 B         0       1.7 TiB

sh-4.4# rados df
	POOL_NAME                                     USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS      R
	D WR_OPS     WR USED COMPR UNDER COMPR
	ocs-storagecluster-cephblockpool           122 MiB      74      0    222                  0       0        0    114 1.3 Mi
	B   6470 76 MiB        0 B         0 B
	ocs-storagecluster-cephfilesystem-data0        0 B       0      0      0                  0       0        0      0     0
	B      0    0 B        0 B         0 B
	ocs-storagecluster-cephfilesystem-metadata  96 KiB      22      0     66                  0       0        0   7692 3.8 Mi
	B     45 13 KiB        0 B         0 B
	
	total_objects    96
	total_used       3.1 GiB
	total_avail      6.0 TiB
	total_space      6 TiB

sh-4.4# ceph versions
	{
	    "mon": {
	        "ceph version 14.2.8-91.el8cp (75b4845da7d469665bd48d1a49badcc3677bf5cd) nautilus (stable)": 3
	    },
	    "mgr": {
	        "ceph version 14.2.8-91.el8cp (75b4845da7d469665bd48d1a49badcc3677bf5cd) nautilus (stable)": 1
	    },
	    "osd": {
	        "ceph version 14.2.8-91.el8cp (75b4845da7d469665bd48d1a49badcc3677bf5cd) nautilus (stable)": 3
	    },
	    "mds": {
	        "ceph version 14.2.8-91.el8cp (75b4845da7d469665bd48d1a49badcc3677bf5cd) nautilus (stable)": 2
	    },
	    "overall": {
	        "ceph version 14.2.8-91.el8cp (75b4845da7d469665bd48d1a49badcc3677bf5cd) nautilus (stable)": 9
	    }
	}
```
# Create a new OCP application deployment using Ceph RBD volume
```
oc new-project my-database-app
oc new-app -f /opt/app-root/src/support/ocslab_rails-app.yaml -p STORAGE_CLASS=ocs-storagecluster-ceph-rbd -p VOLUME_CAPACITY=5Gi

	[~] $ oc new-project my-database-app
	Now using project "my-database-app" on server "https://172.30.0.1:443".
```	
	You can add applications to this project with the 'new-app' command. For example, try:
```	
	    oc new-app django-psql-example
```	
	to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:
```	
	    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```	
	ter-ceph-rbd -p VOLUME_CAPACITY=5Girc/support/ocslab_rails-app.yaml -p STORAGE_CLASS=ocs-storageclus
	--> Deploying template "my-database-app/rails-pgsql-persistent-storageclass" for "/opt/app-root/src/support/ocslab_rails-a
	pp.yaml" to project my-database-app

Rails + PostgreSQL + Congigigurable StorageClass
An example Rails application with a PostgreSQL database. For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/rails-ex/blob/master/README.md.

The following service(s) have been created in your project: rails-pgsql-persistent, postgresql.
	
For more information about using this template, including OpenShift considerations, see ```https://github.com/sclorg/rails-ex/blob/master/README.md.```
	     * With parameters:
        * Name=rails-pgsql-persistent
        * Namespace=openshift
        * Memory Limit=512Mi
        * Memory Limit (PostgreSQL)=512Mi
        * Volume Capacity=5Gi
        * Volume Storage Class=ocs-storagecluster-ceph-rbd
        * Git Repository URL=https://github.com/sclorg/rails-ex.git
        * Git Reference=
        * Context Directory=
        * Application Hostname=
        * GitHub Webhook Secret=vEsN617Gge1BK7yyeT7KueUBaGb6sQbNOYtc5WiY # generated
        * Secret Key=ywrp1j3g87ju77fajsfg6jbbrn16ui5gkupvas8ynlvwrq7ue2gkfk168fedcqmvuqrljffg5204vj1b6aa2xqte0686j4580jqiagyyukv8fpad4cgkvuj17krjsen # generated
	      * Application Username=openshift
	      * Application Password=secret
	      * Rails Environment=production
	      * Database Service Name=postgresql
	      * Database Username=user013 # generated
	      * Database Password=k7D2RMN8 # generated
	      * Database Name=root
	      * Maximum Database Connections=100
	      * Shared Buffer Amount=12MB
	      * Custom RubyGems Mirror URL=
	
  --> Creating resources ...

      secret "rails-pgsql-persistent" created
	    service "rails-pgsql-persistent" created
	    route.route.openshift.io "rails-pgsql-persistent" created
	    imagestream.image.openshift.io "rails-pgsql-persistent" created
	    buildconfig.build.openshift.io "rails-pgsql-persistent" created
	    deploymentconfig.apps.openshift.io "rails-pgsql-persistent" created
	    persistentvolumeclaim "postgresql" created
	    service "postgresql" created
	    deploymentconfig.apps.openshift.io "postgresql" created
	--> Success
	    Access your application via route 'rails-pgsql-persistent-my-database-app.apps.cluster-2f83.2f83.sandbox603.opentlc.co
	m'
	    Build scheduled, use 'oc logs -f bc/rails-pgsql-persistent' to track its progress.
	    Run 'oc status' to view your app.

oc status
	In project my-database-app on server https://172.30.0.1:443
	
	svc/postgresql - 172.30.178.189:5432
	  dc/postgresql deploys openshift/postgresql:10
	    deployment #1 deployed 2 minutes ago - 1 pod
	
	http://rails-pgsql-persistent-my-database-app.apps.cluster-2f83.2f83.sandbox603.opentlc.com (svc/rails-pgsql-persistent)
	  dc/rails-pgsql-persistent deploys istag/rails-pgsql-persistent:latest <-
	    bc/rails-pgsql-persistent source builds https://github.com/sclorg/rails-ex.git on openshift/ruby:2.5
	    deployment #1 running for 30 seconds - 0/1 pods
	
	View details with 'oc describe <resource>/<name>' or list everything with 'oc get all'.

oc get pvc -n my-database-app
	NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
	postgresql   Bound    pvc-f87cf8d9-f2af-4bb0-a77c-b3e10a120190   5Gi        RWO            ocs-storagecluster-ceph-rbd   3m8s


watch oc get pods -n my-database-app
	Every 2.0s: oc get pods -n my-database-app                                              Fri Dec 11 18:06:51 2020
	
	NAME                                READY   STATUS      RESTARTS   AGE
	postgresql-1-b5f24                  1/1     Running     0          4m23s
	postgresql-1-deploy                 0/1     Completed   0          4m25s
	rails-pgsql-persistent-1-build      0/1     Completed   0          4m25s
	rails-pgsql-persistent-1-deploy     0/1     Completed   0          2m35s
	rails-pgsql-persistent-1-hook-pre   0/1     Completed   0          2m31s

oc get route -n my-database-app
	NAME                     HOST/PORT                                                                              PATH   SERVICES                 PORT    TERMINATION   WILDCARD
	rails-pgsql-persistent   rails-pgsql-persistent-my-database-app.apps.cluster-2f83.2f83.sandbox603.opentlc.com          rails-pgsql-persistent   <all>                 None
```
Open a browser and navigate to:
```
http://rails-pgsql-persistent-my-database-app.apps.cluster-2f83.2f83.sandbox603.opentlc.com/articles
```
<img src=/images/mod01/010.png>
```
username: openshift
password: <secret>
```

Now take another look at the Ceph ocs-storagecluster-cephblockpool created by the ocs-storagecluster-ceph-rbd Storage Class. Log into the toolbox pod again.
```
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD
	sh-4.4#

sh-4.4# ceph df
	RAW STORAGE:
	    CLASS     SIZE      AVAIL       USED        RAW USED     %RAW USED
	    ssd       6 TiB     6.0 TiB     304 MiB      3.3 GiB          0.05
	    TOTAL     6 TiB     6.0 TiB     304 MiB      3.3 GiB          0.05
	
	POOLS:
	    POOL                                           ID     STORED      OBJECTS     USED        %USED     MAX AVAIL
	    ocs-storagecluster-cephblockpool                1     101 MiB         113     303 MiB         0       1.7 TiB
	    ocs-storagecluster-cephfilesystem-metadata      2     2.2 KiB          22      96 KiB         0       1.7 TiB
	    ocs-storagecluster-cephfilesystem-data0         3         0 B           0         0 B         0       1.7 TiB

sh-4.4# rados df
	POOL_NAME                                     USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS      RD WR_OPS      WR USED COMPR UNDER COMPR
	ocs-storagecluster-cephblockpool           304 MiB     114      0    342                  0       0        0    224 2.4 MiB   9935 151 MiB        0 B         0 B
	ocs-storagecluster-cephfilesystem-data0        0 B       0      0      0                  0       0        0      0     0 B      0     0 B        0 B         0 B
	ocs-storagecluster-cephfilesystem-metadata  96 KiB      22      0     66                  0       0        0  10190 5.0 MiB     45  13 KiB        0 B         0 B
	
	total_objects    136
	total_used       3.3 GiB
	total_avail      6.0 TiB
	total_space      6 TiB
	
sh-4.4# rbd -p ocs-storagecluster-cephblockpool ls | grep vol
	csi-vol-06eaf0a3-3bdb-11eb-af7f-0a580a800605
	csi-vol-97ba3ed4-3bd1-11eb-af7f-0a580a800605
```
# Matching PVs to RBDs
```
oc get pv -o 'custom-columns=NAME:.spec.claimRef.name,PVNAME:.metadata.name,STORAGECLASS:.spec.storageClassName,VOLUMEHANDLE:.spec.csi.volumeHandle'
	NAME                           PVNAME                                     STORAGECLASS                  VOLUMEHANDLE
	rook-ceph-mon-a                pvc-065a653b-d350-44d0-9c6a-7ae6beb04c14   gp2                           <none>
	rook-ceph-mon-b                pvc-2ba5e4e7-e8ab-4838-9297-e000060371eb   gp2                           <none>
	rook-ceph-mon-c                pvc-3155286a-a6e1-4416-a41f-d00920b1c147   gp2                           <none>
	mapit-storage                  pvc-36f49fbf-de88-40c8-90f7-e7f918334e1c   gp2                           <none>
	ocs-deviceset-2-data-0-g8lx9   pvc-3f37cec3-b195-4c81-a071-deff88108758   gp2                           <none>
	ocs-deviceset-0-data-0-xd7qf   pvc-7395251b-c08b-4659-8fd5-2af43e5c298b   gp2                           <none>
	ocs-deviceset-1-data-0-cgqmz   pvc-78dbcba1-0470-4808-a787-ea4e998cb036   gp2                           <none>
	db-noobaa-db-0                 pvc-b0728837-1d80-44b7-81f6-25682460e63b   ocs-storagecluster-ceph-rbd   0001-0011-openshift-storage-0000000000000001-97ba3ed4-3bd1-11eb-af7f-0a580a800605
	postgresql                     pvc-f87cf8d9-f2af-4bb0-a77c-b3e10a120190   ocs-storagecluster-ceph-rbd   0001-0011-openshift-storage-0000000000000001-06eaf0a3-3bdb-11eb-af7f-0a580a800605
```

Get the full RBD name and the associated information for our postgreSQL PV
```
CSIVOL=$(oc get pv $(oc get pv | grep my-database-app | awk '{ print $1 }') -o jsonpath='{.spec.csi.volumeHandle}' | cut -d '-' -f 6- | awk '{print "csi-vol-"$1}')
echo $CSIVOL
	csi-vol-06eaf0a3-3bdb-11eb-af7f-0a580a800605
	
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD rbd -p ocs-storagecluster-cephblockpool info $CSIVOL
	rbd image 'csi-vol-06eaf0a3-3bdb-11eb-af7f-0a580a800605':
	        size 5 GiB in 1280 objects
	        order 22 (4 MiB objects)
	        snapshot_count: 0
	        id: 5ee57863109a
	        block_name_prefix: rbd_data.5ee57863109a
	        format: 2
	        features: layering
	        op_features:
	        flags:
	        create_timestamp: Fri Dec 11 18:02:26 2020
	        access_timestamp: Fri Dec 11 18:02:26 2020
	        modify_timestamp: Fri Dec 11 18:02:26 2020
```
# Create a new OCP application deployment using CephFS volume
## Create a new project:
```
oc new-project my-shared-storage
	Now using project "my-shared-storage" on server "https://172.30.0.1:443".
```	
	You can add applications to this project with the 'new-app' command. For example, try:
```	
	    oc new-app django-psql-example
```	
	to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes
	application:
```	
	    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```
Next deploy the example PHP application called file-uploader:
```
oc new-app openshift/php:7.2~https://github.com/christianh814/openshift-php-upload-demo --name=file-uploader
	--> Found image 83343c8 (4 weeks old) in image stream "openshift/php" under tag "7.2" for "openshift/php:
	7.2"
	
	    Apache 2.4 with PHP 7.2
	    -----------------------
	    PHP 7.2 available as container is a base platform for building and running various PHP 7.2 applicatio
	ns and frameworks. PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developer
	s to write dynamically generated web pages. PHP also offers built-in database integration for several com
	mercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is
	 fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.
	
	    Tags: builder, php, php72, rh-php72
	
	    * A source build using source code from https://github.com/christianh814/openshift-php-upload-demo wi
	ll be created
	      * The resulting image will be pushed to image stream tag "file-uploader:latest"
	      * Use 'oc start-build' to trigger a new build
	    * This image will be deployed in deployment config "file-uploader"
	    * Ports 8080/tcp, 8443/tcp will be load balanced by service "file-uploader"
	      * Other containers can access this service through the hostname "file-uploader"
	
	--> Creating resources ...
	    imagestream.image.openshift.io "file-uploader" created
	    buildconfig.build.openshift.io "file-uploader" created
	    deploymentconfig.apps.openshift.io "file-uploader" created
	    service "file-uploader" created
	--> Success
	    Build scheduled, use 'oc logs -f bc/file-uploader' to track its progress.
	    Application is not exposed. You can expose services to the outside world by executing one or more of
	the commands below:
	     'oc expose svc/file-uploader'
	    Run 'oc status' to view your app.

Watch and wait for the application to be deployed:
oc logs -f bc/file-uploader -n my-shared-storage
	Cloning "https://github.com/christianh814/openshift-php-upload-demo" ...
	        Commit: 288eda3dff43b02f7f7b6b6b6f93396ffdf34cb2 (trying to modularize)
	        Author: Christian Hernandez <christian.hernandez@yahoo.com>
	        Date:   Sun Oct 1 17:15:09 2017 -0700
	Caching blobs under "/var/cache/blobs".
	Getting image source signatures
	Copying blob sha256:2b63304f1869b42027cc113af46692a963e05ef7867c88d36ef9c75ffc2fdfc7
	Copying blob sha256:3c8d5133e4cc879adaecc6a57976bbfe19286f3c66f9ac80642631bfca25da9d
	Copying blob sha256:3811c17826fed41d93a6e6bcfb6a048828f5f72e3032bb61e8f7cd0785995bfc
	Copying blob sha256:d518f7038c56d881774a7a29c5eb0c5dab9a9c5223a989b818ec4f2ca864867c
	Copying blob sha256:58ef20b02aca8ea0795927176625aef3850209d126949927908234c97bc3c2c4
	Copying config sha256:83343c85274ce68b4d5d857a63c26f6d8f424269ac150ff1d9637c23ffcdc433
	Writing manifest to image destination
	Storing signatures
	Generating dockerfile with builder image image-registry.openshift-image-registry.svc:5000/openshift/php@s
	ha256:cee55e4fa48f3536b478789018d466eb417ddb8360d0608f3d48cb5b72a77fca
	STEP 1: FROM image-registry.openshift-image-registry.svc:5000/openshift/php@sha256:cee55e4fa48f3536b47878
	9018d466eb417ddb8360d0608f3d48cb5b72a77fca
	STEP 2: LABEL "io.openshift.build.commit.message"="trying to modularize" "io.openshift.build.source-locat
	ion"="https://github.com/christianh814/openshift-php-upload-demo" "io.openshift.build.image"="image-regis
	try.openshift-image-registry.svc:5000/openshift/php@sha256:cee55e4fa48f3536b478789018d466eb417ddb8360d060
	8f3d48cb5b72a77fca" "io.openshift.build.commit.author"="Christian Hernandez <christian.hernandez@yahoo.co
	m>" "io.openshift.build.commit.date"="Sun Oct 1 17:15:09 2017 -0700" "io.openshift.build.commit.id"="288e
	da3dff43b02f7f7b6b6b6f93396ffdf34cb2" "io.openshift.build.commit.ref"="master"
	STEP 3: ENV OPENSHIFT_BUILD_NAME="file-uploader-1" OPENSHIFT_BUILD_NAMESPACE="my-shared-storage" OPENSHIF
	T_BUILD_SOURCE="https://github.com/christianh814/openshift-php-upload-demo" OPENSHIFT_BUILD_COMMIT="288ed
	a3dff43b02f7f7b6b6b6f93396ffdf34cb2"
	STEP 4: USER root
	STEP 5: COPY upload/src /tmp/src
	STEP 6: RUN chown -R 1001:0 /tmp/src
	STEP 7: USER 1001
	STEP 8: RUN /usr/libexec/s2i/assemble
	---> Installing application source...
	=> sourcing 20-copy-config.sh ...
	---> 18:34:03     Processing additional arbitrary httpd configuration provided by s2i ...
	=> sourcing 00-documentroot.conf ...
	=> sourcing 50-mpm-tuning.conf ...
	=> sourcing 40-ssl-certs.sh ...
	STEP 9: CMD /usr/libexec/s2i/run
	STEP 10: COMMIT temp.builder.openshift.io/my-shared-storage/file-uploader-1:9849dca2
	Getting image source signatures
	Copying blob sha256:1bdfedf126f0037cb8ed6b6f06f329492d1049d629f1d31dc7ba2d17010dea2d
	Copying blob sha256:b545558594f3e5e5e4ef3335bd145938cd471b8bc77497d02ea0d22da2a6c3ee
	Copying blob sha256:9f8a253a7475bf2051c441a77a096f341bcc01f2e2df6077449b392a27898932
	Copying blob sha256:6af478d2ff9903d89e17aa6fdd62ac32af3d29107d1f11be3a00fcb6ddba067c
	Copying blob sha256:08d6ff662692986faa71bcc6621d31aa95eb0b82c00705b0da9a803c2b20eab2
	Copying blob sha256:e4c2e30752b8f5d791a77ebf22b2b5cde34f69746abbe5d1c05324cd32080519
	Copying config sha256:36ad2b27621db06285b4d6628b5814a7fa338846a62022e3c0eede334d08b840
	Writing manifest to image destination
	Storing signatures
	36ad2b27621db06285b4d6628b5814a7fa338846a62022e3c0eede334d08b840
	36ad2b27621db06285b4d6628b5814a7fa338846a62022e3c0eede334d08b840
	
	Pushing image image-registry.openshift-image-registry.svc:5000/my-shared-storage/file-uploader:latest ...
	Getting image source signatures
	Copying blob sha256:e4c2e30752b8f5d791a77ebf22b2b5cde34f69746abbe5d1c05324cd32080519
	Copying blob sha256:2b63304f1869b42027cc113af46692a963e05ef7867c88d36ef9c75ffc2fdfc7
	Copying blob sha256:3c8d5133e4cc879adaecc6a57976bbfe19286f3c66f9ac80642631bfca25da9d
	Copying blob sha256:d518f7038c56d881774a7a29c5eb0c5dab9a9c5223a989b818ec4f2ca864867c
	Copying blob sha256:3811c17826fed41d93a6e6bcfb6a048828f5f72e3032bb61e8f7cd0785995bfc
	Copying blob sha256:58ef20b02aca8ea0795927176625aef3850209d126949927908234c97bc3c2c4
	Copying config sha256:36ad2b27621db06285b4d6628b5814a7fa338846a62022e3c0eede334d08b840
	Writing manifest to image destination
	Storing signatures
	Successfully pushed image-registry.openshift-image-registry.svc:5000/my-shared-storage/file-uploader@sha2
	56:414575663f5093de3f706034199ebf297c28a020ffd46cd0a4076aa29ec9f0b8
	Push successful
```
Make our application production ready by exposing it via a Route and scale to 3 instances for high availability:
```
oc expose svc/file-uploader -n my-shared-storage
	route.route.openshift.io/file-uploader exposed

oc scale --replicas=3 dc/file-uploader -n my-shared-storage
	deploymentconfig.apps.openshift.io/file-uploader scaled

oc get pods -n my-shared-storage
	NAME                     READY   STATUS      RESTARTS   AGE
	file-uploader-1-2lnzv    1/1     Running     0          5m57s
	file-uploader-1-42ckn    1/1     Running     0          24s
	file-uploader-1-79pfg    1/1     Running     0          24s
	file-uploader-1-build    0/1     Completed   0          7m7s
	file-uploader-1-deploy   0/1     Completed   0          6m
```
Create a PersistentVolumeClaim and attach it into an application with the oc set volume command. Execute the following
```
oc set volume dc/file-uploader --add --name=my-shared-storage \
> -t pvc --claim-mode=ReadWriteMany --claim-size=1Gi \
> --claim-name=my-shared-storage --claim-class=ocs-storagecluster-cephfs \
> --mount-path=/opt/app-root/src/uploaded \
> -n my-shared-storage
	deploymentconfig.apps.openshift.io/file-uploader volume updated
```
This command will:
* create a PersistentVolumeClaim
* update the DeploymentConfig to include a volume definition
* update the DeploymentConfig to attach a volumemount into the specified mount-path
* cause a new deployment of the 3 application Pods
```
oc get pvc -n my-shared-storage
	NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
	my-shared-storage   Bound    pvc-258b8dc8-e9e0-43b2-bb5c-93be25601cdd   1Gi        RWX            ocs-storagecluster-cephfs   99s

oc get route file-uploader -n my-shared-storage -o jsonpath --template="{.spec.host}"
	file-uploader-my-shared-storage.apps.cluster-2f83.2f83.sandbox603.opentlc.com
```


Discover what Pods and PVCs are installed in the openshift-monitoring namespace
```
oc get pods,pvc -n openshift-monitoring
	NAME                                               READY   STATUS    RESTARTS   AGE
	pod/alertmanager-main-0                            3/3     Running   0          3h39m
	pod/alertmanager-main-1                            3/3     Running   0          3h39m
	pod/alertmanager-main-2                            3/3     Running   0          3h40m
	pod/cluster-monitoring-operator-7b898649bd-fvvgz   1/1     Running   0          28h
	pod/grafana-58b94d6b88-wnlqm                       2/2     Running   0          3h40m
	pod/kube-state-metrics-7f6c88d446-s8xfk            3/3     Running   0          3h40m
	pod/node-exporter-6ns4f                            2/2     Running   0          3h2m
	pod/node-exporter-7sfzz                            2/2     Running   0          3h2m
	pod/node-exporter-8b9l6                            2/2     Running   0          28h
	pod/node-exporter-92wdp                            2/2     Running   0          28h
	pod/node-exporter-9l9j8                            2/2     Running   0          28h
	pod/node-exporter-bzvnd                            2/2     Running   0          23h
	pod/node-exporter-cc8rk                            2/2     Running   0          23h
	pod/node-exporter-hn9dz                            2/2     Running   0          28h
	pod/node-exporter-p2wqp                            2/2     Running   0          28h
	pod/node-exporter-tvmdc                            2/2     Running   0          3h2m
	pod/node-exporter-xbn6m                            2/2     Running   0          23h
	pod/node-exporter-ztp9c                            2/2     Running   0          28h
	pod/openshift-state-metrics-785dd5b54c-qtmg8       3/3     Running   0          28h
	pod/prometheus-adapter-5f9c68d46c-6jzh7            1/1     Running   0          3h40m
	pod/prometheus-adapter-5f9c68d46c-qc88h            1/1     Running   0          3h40m
	pod/prometheus-k8s-0                               7/7     Running   1          3h39m
	pod/prometheus-k8s-1                               7/7     Running   1          3h39m
	pod/prometheus-operator-c78dc59f4-l5lrp            1/1     Running   0          171m
	pod/telemeter-client-568bb4569b-dvqhc              3/3     Running   0          3h40m
	pod/thanos-querier-65466f7b48-dtdfj                4/4     Running   0          28h
	pod/thanos-querier-65466f7b48-tx5cv                4/4     Running   0          28h
```
At this point there are no PVC resources because Prometheus and AlertManager are both using ephemeral (EmptyDir) storage

Modifying your Prometheus environment
For Prometheus every supported configuration change is controlled through a central ConfigMap, which needs to exist before we can make changes
```
oc -n openshift-monitoring get configmap cluster-monitoring-config
	NAME                        DATA   AGE
	cluster-monitoring-config   1      3h43m

oc -n openshift-monitoring get configmap cluster-monitoring-config -o yaml | more
	apiVersion: v1
	data:
	  config.yaml: |
	    alertmanagerMain:
	      nodeSelector:
	        node-role.kubernetes.io/infra: ""
	    prometheusK8s:
	      nodeSelector:
	        node-role.kubernetes.io/infra: ""
	    prometheusOperator:
	      nodeSelector:
	        node-role.kubernetes.io/infra: ""
	    grafana:
	      nodeSelector:
	        node-role.kubernetes.io/infra: ""
	    k8sPrometheusAdapter:
	      nodeSelector:
	        node-role.kubernetes.io/infra: ""
	    kubeStateMetrics:
	      nodeSelector:
	        node-role.kubernetes.io/infra: ""
	    telemeterClient:
	      nodeSelector:
	        node-role.kubernetes.io/infra: ""
	kind: ConfigMap
	metadata:
	  creationTimestamp: "2020-12-11T15:29:57Z"
	  name: cluster-monitoring-config
	  namespace: openshift-monitoring
	  resourceVersion: "444487"
	  selfLink: /api/v1/namespaces/openshift-monitoring/configmaps/cluster-monitoring-config
	  uid: af399e55-19ca-4d87-8868-4ece3cd97cb8

oc get pods,pvc -n openshift-monitoring
NAME                                               READY   STATUS    RESTARTS   AGE
	pod/alertmanager-main-0                            3/3     Running   0          3h47m
	pod/alertmanager-main-1                            3/3     Running   0          3h48m
	pod/alertmanager-main-2                            3/3     Running   0          3h48m
	pod/cluster-monitoring-operator-7b898649bd-fvvgz   1/1     Running   0          28h
	pod/grafana-58b94d6b88-wnlqm                       2/2     Running   0          3h48m
	pod/kube-state-metrics-7f6c88d446-s8xfk            3/3     Running   0          3h48m
	pod/node-exporter-6ns4f                            2/2     Running   0          3h10m
	pod/node-exporter-7sfzz                            2/2     Running   0          3h10m
	pod/node-exporter-8b9l6                            2/2     Running   0          28h
	pod/node-exporter-92wdp                            2/2     Running   0          28h
	pod/node-exporter-9l9j8                            2/2     Running   0          28h
	pod/node-exporter-bzvnd                            2/2     Running   0          23h
	pod/node-exporter-cc8rk                            2/2     Running   0          23h
	pod/node-exporter-hn9dz                            2/2     Running   0          28h
	pod/node-exporter-p2wqp                            2/2     Running   0          28h
	pod/node-exporter-tvmdc                            2/2     Running   0          3h10m
	pod/node-exporter-xbn6m                            2/2     Running   0          23h
	pod/node-exporter-ztp9c                            2/2     Running   0          28h
	pod/openshift-state-metrics-785dd5b54c-qtmg8       3/3     Running   0          28h
	pod/prometheus-adapter-5f9c68d46c-6jzh7            1/1     Running   0          3h48m
	pod/prometheus-adapter-5f9c68d46c-qc88h            1/1     Running   0          3h48m
	pod/prometheus-k8s-0                               7/7     Running   1          3h47m
	pod/prometheus-k8s-1                               7/7     Running   1          3h48m
	pod/prometheus-operator-c78dc59f4-l5lrp            1/1     Running   0          179m
	pod/telemeter-client-568bb4569b-dvqhc              3/3     Running   0          3h48m
	pod/thanos-querier-65466f7b48-dtdfj                4/4     Running   0          28h
	pod/thanos-querier-65466f7b48-tx5cv                4/4     Running   0          28h

Checking on the MCG status
noobaa status -n openshift-storage
	INFO[0000] CLI version: 2.3.0
	INFO[0000] noobaa-image: noobaa/noobaa-core:5.5.0
	INFO[0000] operator-image: noobaa/noobaa-operator:2.3.0
	INFO[0000] Namespace: openshift-storage
	INFO[0000]
	INFO[0000] CRD Status:
	INFO[0000] ✅ Exists: CustomResourceDefinition "noobaas.noobaa.io"
	INFO[0000] ✅ Exists: CustomResourceDefinition "backingstores.noobaa.io"
	INFO[0000] ✅ Exists: CustomResourceDefinition "bucketclasses.noobaa.io"
	INFO[0000] ✅ Exists: CustomResourceDefinition "objectbucketclaims.objectbucket.io"
	INFO[0000] ✅ Exists: CustomResourceDefinition "objectbuckets.objectbucket.io"
	INFO[0000]
	INFO[0000] Operator Status:
	INFO[0000] ✅ Exists: Namespace "openshift-storage"
	INFO[0000] ✅ Exists: ServiceAccount "noobaa"
	INFO[0000] ✅ Exists: Role "ocs-operator.v4.5.2-d49mj"
	INFO[0000] ✅ Exists: RoleBinding "ocs-operator.v4.5.2-d49mj-noobaa-z7g9j"
	INFO[0000] ✅ Exists: ClusterRole "ocs-operator.v4.5.2-j47xw"
	INFO[0000] ✅ Exists: ClusterRoleBinding "ocs-operator.v4.5.2-j47xw-noobaa-q4tw7"
	INFO[0000] ✅ Exists: Deployment "noobaa-operator"
	INFO[0000]
	INFO[0000] System Status:
	INFO[0000] ✅ Exists: NooBaa "noobaa"
	INFO[0000] ✅ Exists: StatefulSet "noobaa-core"
	INFO[0000] ✅ Exists: StatefulSet "noobaa-db"
	INFO[0000] ✅ Exists: Service "noobaa-mgmt"
	INFO[0000] ✅ Exists: Service "s3"
	INFO[0000] ✅ Exists: Service "noobaa-db"
	INFO[0000] ✅ Exists: Secret "noobaa-server"
	INFO[0000] ✅ Exists: Secret "noobaa-operator"
	INFO[0000] ✅ Exists: Secret "noobaa-endpoints"
	INFO[0000] ✅ Exists: Secret "noobaa-admin"
	INFO[0000] ✅ Exists: StorageClass "openshift-storage.noobaa.io"
	INFO[0000] ✅ Exists: BucketClass "noobaa-default-bucket-class"
	INFO[0000] ✅ Exists: Deployment "noobaa-endpoint"
	INFO[0000] ✅ Exists: HorizontalPodAutoscaler "noobaa-endpoint"
	INFO[0000] ✅ (Optional) Exists: BackingStore "noobaa-default-backing-store"
	INFO[0000] ✅ (Optional) Exists: CredentialsRequest "noobaa-aws-cloud-creds"
	INFO[0000] ⬛ (Optional) Not Found: CredentialsRequest "noobaa-azure-cloud-creds"
	INFO[0000] ⬛ (Optional) Not Found: Secret "noobaa-azure-container-creds"
	INFO[0000] ✅ (Optional) Exists: PrometheusRule "noobaa-prometheus-rules"
	INFO[0000] ✅ (Optional) Exists: ServiceMonitor "noobaa-service-monitor"
	INFO[0000] ✅ (Optional) Exists: Route "noobaa-mgmt"
	INFO[0000] ✅ (Optional) Exists: Route "s3"
	INFO[0000] ✅ Exists: PersistentVolumeClaim "db-noobaa-db-0"
	INFO[0000] ✅ System Phase is "Ready"
	INFO[0000] ✅ Exists:  "noobaa-admin"
	
	#------------------#
	#- Mgmt Addresses -#
	#------------------#
	
	ExternalDNS : [https://noobaa-mgmt-openshift-storage.apps.cluster-2f83.2f83.sandbox603.opentlc.com
	https://a56bd261b28d94484aa32c29a14aec8b-1952242030.us-east-2.elb.amazonaws.com:443]
	ExternalIP  : []
	NodePorts   : [https://10.0.139.54:31938]
	InternalDNS : [https://noobaa-mgmt.openshift-storage.svc:443]
	InternalIP  : [https://172.30.7.88:443]
	PodPorts    : [https://10.129.4.11:8443]
	
	#--------------------#
	#- Mgmt Credentials -#
	#--------------------#
	
	email    : admin@noobaa.io
	password : 1sdE19yyBcOMvoS8hNaXwQ==
	
	#----------------#
	#- S3 Addresses -#
	#----------------#
	
	ExternalDNS : [https://s3-openshift-storage.apps.cluster-2f83.2f83.sandbox603.opentlc.com https://a
	9665c72faad94a329bb702af66eae24-1941459504.us-east-2.elb.amazonaws.com:443]
	ExternalIP  : []
	NodePorts   : [https://10.0.139.54:30309]
	InternalDNS : [https://s3.openshift-storage.svc:443]
	InternalIP  : [https://172.30.186.103:443]
	PodPorts    : [https://10.129.4.12:6443]
	
	#------------------#
	#- S3 Credentials -#
	#------------------#
	
	AWS_ACCESS_KEY_ID     : wCkE3ZZhNyYSNCXNtUNx
	AWS_SECRET_ACCESS_KEY : HEZL3umBUVauWd5X4cksvwzzpAZGvy6aGGBcB5TO
	
	#------------------#
	#- Backing Stores -#
	#------------------#
	
	NAME                           TYPE     TARGET-BUCKET
	      PHASE   AGE
	noobaa-default-backing-store   aws-s3   nb.1607705669514.apps.cluster-2f83.2f83.sandbox603.opentlc.
	com   Ready   2h27m9s
	
	#------------------#
	#- Bucket Classes -#
	#------------------#
	
	NAME                          PLACEMENT
	 PHASE   AGE
	noobaa-default-bucket-class   {Tiers:[{Placement: BackingStores:[noobaa-default-backing-store]}]}
	 Ready   2h27m9s
	
	#-----------------#
	#- Bucket Claims -#
	#-----------------#
	
	No OBCs found.


Creating an Object Bucket Claim
Creating an OBC is as simple as using the NooBaa CLI:
	
noobaa obc create test21obc -n openshift-storage
	INFO[0000] ✅ Exists: StorageClass "openshift-storage.noobaa.io"
	INFO[0000] ✅ Created: ObjectBucketClaim "test21obc"
	INFO[0000]
	INFO[0000] NOTE:
	INFO[0000]   - This command has finished applying changes to the cluster.
	INFO[0000]   - From now on, it only loops and reads the status, to monitor the operator work.
	INFO[0000]   - You may Ctrl-C at any time to stop the loop and watch it manually.
	INFO[0000]
	INFO[0000] OBC Wait Ready:
	INFO[0000] ⏳ OBC "test21obc" Phase is ""
	INFO[0003] ✅ OBC "test21obc" Phase is Bound
	INFO[0003]
	INFO[0003]
	INFO[0003] ✅ Exists: ObjectBucketClaim "test21obc"
	INFO[0003] ✅ Exists: ObjectBucket "obc-openshift-storage-test21obc"
	INFO[0003] ✅ Exists: ConfigMap "test21obc"
	INFO[0003] ✅ Exists: Secret "test21obc"
	INFO[0003] ✅ Exists: StorageClass "openshift-storage.noobaa.io"
	INFO[0003] ✅ Exists: BucketClass "noobaa-default-bucket-class"
	INFO[0003] ✅ Exists: NooBaa "noobaa"
	INFO[0003] ✅ Exists: Service "noobaa-mgmt"
	INFO[0003] ✅ Exists: Secret "noobaa-operator"
	INFO[0003] ✅ Exists: Secret "noobaa-admin"
	INFO[0003] ✈️  RPC: bucket.read_bucket() Request: {Name:test21obc-d44436ae-7d2a-4ff4-b2ca-e98281af4
	f29}
	WARN[0003] RPC: GetConnection creating connection to wss://localhost:33093/rpc/ 0xc00078c320
	INFO[0003] RPC: Connecting websocket (0xc00078c320) &{RPC:0xc000099540 Address:wss://localhost:3309
	3/rpc/ State:init WS:<nil> PendingRequests:map[] NextRequestID:0 Lock:{state:1 sema:0} ReconnectDel
	ay:0s}
	INFO[0003] RPC: Connected websocket (0xc00078c320) &{RPC:0xc000099540 Address:wss://localhost:33093
	/rpc/ State:init WS:<nil> PendingRequests:map[] NextRequestID:0 Lock:{state:1 sema:0} ReconnectDela
	y:0s}
	INFO[0003] ✅ RPC: bucket.read_bucket() Response OK: took 5.0ms
	
	ObjectBucketClaim info:
	  Phase                  : Bound
	  ObjectBucketClaim      : kubectl get -n openshift-storage objectbucketclaim test21obc
	  ConfigMap              : kubectl get -n openshift-storage configmap test21obc
	  Secret                 : kubectl get -n openshift-storage secret test21obc
	  ObjectBucket           : kubectl get objectbucket obc-openshift-storage-test21obc
	  StorageClass           : kubectl get storageclass openshift-storage.noobaa.io
	  BucketClass            : kubectl get -n openshift-storage bucketclass noobaa-default-bucket-class
	
	Connection info:
	  BUCKET_NAME            : test21obc-d44436ae-7d2a-4ff4-b2ca-e98281af4f29
	  BUCKET_PORT            : 443
	  BUCKET_HOST            : s3.openshift-storage.svc
	  AWS_ACCESS_KEY_ID      : 0VQKlnWTp92nRUFLCfmH
	  AWS_SECRET_ACCESS_KEY  : sSruVV3ctCeFeFV4aC4PqmDHnRt8rKXAVOauc2rt
	
	Shell commands:
	  AWS S3 Alias           : alias s3='AWS_ACCESS_KEY_ID=0VQKlnWTp92nRUFLCfmH AWS_SECRET_ACCESS_KEY=s
	SruVV3ctCeFeFV4aC4PqmDHnRt8rKXAVOauc2rt aws s3 --no-verify-ssl --endpoint-url https://10.0.139.54:3
	0309'
	
	Bucket status:
	  Name                   : test21obc-d44436ae-7d2a-4ff4-b2ca-e98281af4f29
	  Type                   : REGULAR
	  Mode                   : OPTIMAL
	  ResiliencyStatus       : OPTIMAL
	  QuotaStatus            : QUOTA_NOT_SET
	  Num Objects            : 0
	  Data Size              : 0.000 B
	  Data Size Reduced      : 0.000 B
	  Data Space Avail       : 1.000 PB

oc get obc -n openshift-storage
	NAME        STORAGE-CLASS                 PHASE   AGE
	test21obc   openshift-storage.noobaa.io   Bound   49s

oc get obc test21obc -o yaml -n openshift-storage
NAME        STORAGE-CLASS                 PHASE   AGE
	test21obc   openshift-storage.noobaa.io   Bound   49s
	[~] $ oc get obc test21obc -o yaml -n openshift-storage
	apiVersion: objectbucket.io/v1alpha1
	kind: ObjectBucketClaim
	metadata:
	  creationTimestamp: "2020-12-11T19:31:37Z"
	  finalizers:
	  - objectbucket.io/finalizer
	  generation: 2
	  labels:
	    app: noobaa
	    bucket-provisioner: openshift-storage.noobaa.io-obc
	    noobaa-domain: openshift-storage.noobaa.io
	  name: test21obc
	  namespace: openshift-storage
	  resourceVersion: "550541"
	  selfLink: /apis/objectbucket.io/v1alpha1/namespaces/openshift-storage/objectbucketclaims/test21ob
	c
	  uid: 0844431d-ebaa-4b60-b51d-5f0741359d95
	spec:
	  ObjectBucketName: obc-openshift-storage-test21obc
	  bucketName: test21obc-d44436ae-7d2a-4ff4-b2ca-e98281af4f29
	  generateBucketName: test21obc
	  storageClassName: openshift-storage.noobaa.io
	status:
	  phase: Bound

Inside of your openshift-storage namespace, you will now find the ConfigMap and the Secret to use this OBC
oc get -n openshift-storage secret test21obc -o yaml
	apiVersion: v1
	data:
	  AWS_ACCESS_KEY_ID: MFZRS2xuV1RwOTJuUlVGTENmbUg=
	  AWS_SECRET_ACCESS_KEY: c1NydVZWM2N0Q2VGZUZWNGFDNFBxbURIblJ0OHJLWEFWT2F1YzJydA==
	kind: Secret
	metadata:
	  creationTimestamp: "2020-12-11T19:31:37Z"
	  finalizers:
	  - objectbucket.io/finalizer
	  labels:
	    app: noobaa
	    bucket-provisioner: openshift-storage.noobaa.io-obc
	    noobaa-domain: openshift-storage.noobaa.io
	  name: test21obc
	  namespace: openshift-storage
	  ownerReferences:
	  - apiVersion: objectbucket.io/v1alpha1
	    blockOwnerDeletion: true
	    controller: true
	    kind: ObjectBucketClaim
	    name: test21obc
	    uid: 0844431d-ebaa-4b60-b51d-5f0741359d95
	  resourceVersion: "550535"
	  selfLink: /api/v1/namespaces/openshift-storage/secrets/test21obc
	  uid: 012fc042-c6cf-4d24-951e-6150518957a8
	type: Opaque

oc get -n openshift-storage cm test21obc -o yaml
	apiVersion: v1
	data:
	  BUCKET_HOST: s3.openshift-storage.svc
	  BUCKET_NAME: test21obc-d44436ae-7d2a-4ff4-b2ca-e98281af4f29
	  BUCKET_PORT: "443"
	  BUCKET_REGION: ""
	  BUCKET_SUBREGION: ""
	kind: ConfigMap
	metadata:
	  creationTimestamp: "2020-12-11T19:31:37Z"
	  finalizers:
	  - objectbucket.io/finalizer
	  labels:
	    app: noobaa
	    bucket-provisioner: openshift-storage.noobaa.io-obc
	    noobaa-domain: openshift-storage.noobaa.io
	  name: test21obc
	  namespace: openshift-storage
	  ownerReferences:
	  - apiVersion: objectbucket.io/v1alpha1
	    blockOwnerDeletion: true
	    controller: true
	    kind: ObjectBucketClaim
	    name: test21obc
	    uid: 0844431d-ebaa-4b60-b51d-5f0741359d95
	  resourceVersion: "550536"
	  selfLink: /api/v1/namespaces/openshift-storage/configmaps/test21obc
	  uid: 3e820cff-9181-4902-9bbf-995938fe29c5
```
Using an OBC inside a container
we apply this YAML file
```
apiVersion: v1
kind: Namespace
metadata:
  name: obc-test
---
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: obc-test
  namespace: obc-test
spec:
  generateBucketName: "obc-test-noobaa"
  storageClassName: openshift-storage.noobaa.io
---
apiVersion: batch/v1
kind: Job
metadata:
  name: obc-test
  namespace: obc-test
  labels:
    app: obc-test
spec:
  template:
    metadata:
      labels:
        app: obc-test
    spec:
      restartPolicy: OnFailure
      containers:
        - image: mesosphere/aws-cli:latest
          command: ["sh"]
          args:
            - '-c'
            - 'set -x && s3cmd --no-check-certificate --signature-v2 --host $BUCKET_HOST:$BUCKET_PORT --host-bucket $BUCKET_HOST:$BUCKET_PORT du'
          name: obc-test
          env:
            - name: BUCKET_NAME
              valueFrom:
                configMapKeyRef:
                  name: obc-test
                  key: BUCKET_NAME
            - name: BUCKET_HOST
              valueFrom:
                configMapKeyRef:
                  name: obc-test
                  key: BUCKET_HOST
            - name: BUCKET_PORT
              valueFrom:
                configMapKeyRef:
                  name: obc-test
                  key: BUCKET_PORT
            - name: AWS_DEFAULT_REGION
              valueFrom:
                configMapKeyRef:
                  name: obc-test
                  key: BUCKET_REGION
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: obc-test
                  key: AWS_ACCESS_KEY_ID
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: obc-test
                  key: AWS_SECRET_ACCESS_KEY


oc apply -f /opt/app-root/src/support/ocslab_obc-app-example.yaml
	namespace/obc-test created
	objectbucketclaim.objectbucket.io/obc-test created
	job.batch/obc-test created

oc get pods -n obc-test -l app=obc-test
	NAME             READY   STATUS      RESTARTS   AGE
	obc-test-6b7t9   0/1     Completed   0          66s
```
check the obc-test Pod logs for the contents
```
oc logs -n obc-test $(oc get pods -n obc-test -l app=obc-test -o jsonpath='{.items[0].metadata.name}')
	+ s3cmd --no-check-certificate --signature-v2 --host s3.openshift-storage.svc:443 --host-bucket s3.
	openshift-storage.svc:443 du
	0        0 objects s3://obc-test-noobaa-410ae6b5-a6dc-4982-bc6e-d888f92f1c33/
	--------
	0        Total
```
## Adding storage to the Ceph Cluster
### Add storage worker nodes
```
oc get machinesets -n openshift-machine-api | egrep 'NAME|workerocs'
	NAME                                      DESIRED   CURRENT   READY   AVAILABLE   AGE
	cluster-2f83-d7fvm-workerocs-us-east-2a   1         1         1       1           3h42m
	cluster-2f83-d7fvm-workerocs-us-east-2b   1         1         1       1           3h42m
	cluster-2f83-d7fvm-workerocs-us-east-2c   1         1         1       1           3h42m
```
Let’s scale the workerocs machinesets up with this command:
```
oc get machinesets -n openshift-machine-api -o name | grep workerocs | xargs -n1 -t oc scale
-n openshift-machine-api --replicas=2
	oc scale -n openshift-machine-api --replicas=2 machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2a
		machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2a scaled
	oc scale -n openshift-machine-api --replicas=2 machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2b
		machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2b scaled
	oc scale -n openshift-machine-api --replicas=2 machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2c
		machineset.machine.openshift.io/cluster-2f83-d7fvm-workerocs-us-east-2c scaled

watch "oc get machinesets -n openshift-machine-api | egrep 'NAME|workerocs'"
	Every 2.0s: oc get machinesets -n openshift-machine-api | egrep 'NAME|...  Fri Dec 11 19:57:50 2020
	
	NAME                                      DESIRED   CURRENT   READY   AVAILABLE   AGE
	cluster-2f83-d7fvm-workerocs-us-east-2a   2         2         1       1           3h53m
	cluster-2f83-d7fvm-workerocs-us-east-2b   2         2         1       1           3h53m
	cluster-2f83-d7fvm-workerocs-us-east-2c   2         2         1       1           3h53m

oc get nodes -l cluster.ocs.openshift.io/openshift-storage -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}'
	ip-10-0-138-119.us-east-2.compute.internal
	ip-10-0-151-151.us-east-2.compute.internal
	ip-10-0-170-7.us-east-2.compute.internal
```
# Add storage capacity
## Verify new storage
```
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD
	sh-4.4#
	
sh-4.4# ceph status
	  cluster:
	    id:     1141df56-ed91-429d-aa21-863d7043bb3e
	    health: HEALTH_OK
	
	  services:
	    mon: 3 daemons, quorum a,b,c (age 3h)
	    mgr: a(active, since 3h)
	    mds: ocs-storagecluster-cephfilesystem:1 {0=ocs-storagecluster-cephfilesystem-a=up:active} 1 up:standby-replay
	    osd: 3 osds: 3 up (since 3h), 3 in (since 3h)
	
	  task status:
	    scrub status:
	        mds.ocs-storagecluster-cephfilesystem-a: idle
	        mds.ocs-storagecluster-cephfilesystem-b: idle
	
	  data:
	    pools:   3 pools, 96 pgs
	    objects: 158 objects, 237 MiB
	    usage:   3.5 GiB used, 6.0 TiB / 6 TiB avail
	    pgs:     96 active+clean
	
	  io:
	    client:   1.2 KiB/s rd, 45 KiB/s wr, 2 op/s rd, 3 op/s wr
```
# Check the topology of your cluster:
```
sh-4.4# ceph osd crush tree
	ID  CLASS WEIGHT  TYPE NAME
	 -1       6.00000 root default
	 -5       6.00000     region us-east-2
	-10       2.00000         zone us-east-2a
	 -9       2.00000             host ocs-deviceset-2-data-0-g8lx9
	  1   ssd 2.00000                 osd.1
	-14       2.00000         zone us-east-2b
	-13       2.00000             host ocs-deviceset-1-data-0-cgqmz
	  2   ssd 2.00000                 osd.2
	 -4       2.00000         zone us-east-2c
	 -3       2.00000             host ocs-deviceset-0-data-0-xd7qf
	  0   ssd 2.00000                 osd.0
```
# Monitoring the OCS environment

Prometheus : Alert Manager UI:


Metrics:


Using must-gather
Must-gather is a tool for collecting data about the current’y running Openshift cluster
```
oc adm must-gather
	[must-gather      ] OUT Using must-gather plugin-in image: quay.io/openshift-release-dev/ocp-v4.0-
	art-dev@sha256:a37e5bef81b80c86ab3864c3395d69c0867ab0aa58e1150eceb85be8951def49
	[must-gather      ] OUT namespace/openshift-must-gather-8n8g4 created
	[must-gather      ] OUT clusterrolebinding.rbac.authorization.k8s.io/must-gather-wkn4w created
	[must-gather      ] OUT pod for plug-in image quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha25
	6:a37e5bef81b80c86ab3864c3395d69c0867ab0aa58e1150eceb85be8951def49 created
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	[must-gather-rhc7b] POD Gathering data for ns/openshift-cluster-version...
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	[must-gather-rhc7b] POD Gathering data for ns/openshift-config...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-config-managed...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-authentication...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-authentication-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-ingress...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-cloud-credential-operator...
	oc adm must-gather
	[must-gather-rhc7b] POD Gathering data for ns/openshift-machine-api...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-console-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-console...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-dns-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-dns...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-image-registry...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-ingress-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-insights...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-kube-apiserver-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-kube-apiserver...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-kube-controller-manager...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-kube-controller-manager-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-kube-scheduler...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-kube-scheduler-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-machine-config-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-marketplace...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-monitoring...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-multus...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-sdn...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-network-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-cluster-node-tuning-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-apiserver-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-apiserver...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-controller-manager-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-controller-manager...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-cluster-samples-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-operator-lifecycle-manager...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-service-ca-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-service-ca...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-service-catalog-apiserver-operator...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-service-catalog-controller-manager-operato
	r...
	[must-gather-rhc7b] POD Gathering data for ns/openshift-cluster-storage-operator...
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	[must-gather-rhc7b] POD error: errors ocurred while gathering data:
	[must-gather-rhc7b] POD     [skipping gathering routes.route.openshift.io/oauth-openshift due to e
	rror: a resource cannot be retrieved by name across all namespaces, skipping gathering services/oa
	uth-openshift due to error: a resource cannot be retrieved by name across all namespaces, skipping
	 gathering machineautoscalers.machine.openshift.io due to error: the server doesn't have a resourc
	e type "machineautoscalers", skipping gathering clusterautoscalers.machine.openshift.io due to err
	or: the server doesn't have a resource type "clusterautoscalers", skipping gathering all/openshift
	-monitoring due to error: the server doesn't have a resource type "all", skipping gathering namesp
	aces/openshift-service-catalog-apiserver due to error: namespaces "openshift-service-catalog-apise
	rver" not found, skipping gathering apiservices.apiregistration.k8s.io/v1beta1.servicecatalog.k8s.
	io due to error: apiservices.apiregistration.k8s.io "v1beta1.servicecatalog.k8s.io" not found, ski
	pping gathering namespaces/openshift-service-catalog-controller-manager due to error: namespaces "
	openshift-service-catalog-controller-manager" not found]
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	[must-gather-rhc7b] POD Gathering data for ns/default...
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	[must-gather-rhc7b] POD Gathering data for ns/openshift...
	[must-gather-rhc7b] POD Wrote inspect data to must-gather.
	…
	…
	…
	
```
