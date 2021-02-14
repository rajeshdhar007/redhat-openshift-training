# MachineSets, Machines, and Nodes

Kubernetes Nodes are where containers are orchestrated and run in Pods. OpenShift 4 is fundamentally different than OpenShift 3 with respect to its focus on automated operations through the use of Operators. With respect to Nodes, there is a set of Operators and controllers that are focused on maintaining the state of the cluster size — including creating and destroying Nodes!

```
oc get machineset -n openshift-machine-api
	NAME                                   DESIRED   CURRENT   READY   AVAILABLE   AGE
	cluster-2f83-d7fvm-worker-us-east-2a   1         1         1       1           4h2m
	cluster-2f83-d7fvm-worker-us-east-2b   1         1         1       1           4h2m
	cluster-2f83-d7fvm-worker-us-east-2c   1         1         1       1           4h2m
```

# Relationship
Like:
ReplicaSet --> ReplicationController
So is:
MachineSet --> Machine

When OpenShift was installed, the installer interrogated the cloud provider to learn about the available AZs (since this is on AWS). It then ultimately created a MachineSet for each AZ and then scaled those sets, in order, until it reached the desired number of Machines. Since the default installation has 3 workers, the first 3 AZs got one worker each. The rest got zero
```
oc get machine -n openshift-machine-api
	NAME                                         PHASE     TYPE         REGION      ZONE         AGE
	cluster-2f83-d7fvm-master-0                  Running   m5.xlarge    us-east-2   us-east-2a   4h5m
	cluster-2f83-d7fvm-master-1                  Running   m5.xlarge    us-east-2   us-east-2b   4h5m
	cluster-2f83-d7fvm-master-2                  Running   m5.xlarge    us-east-2   us-east-2c   4h5m
	cluster-2f83-d7fvm-worker-us-east-2a-5c2fs   Running   m5.2xlarge   us-east-2   us-east-2a   4h1m
	cluster-2f83-d7fvm-worker-us-east-2b-4665z   Running   m5.2xlarge   us-east-2   us-east-2b   4h1m
	cluster-2f83-d7fvm-worker-us-east-2c-sdlkq   Running   m5.2xlarge   us-east-2   us-east-2c   4h1m
```

Each Machine has a corresponding INSTANCE
They are AWS EC2 instance IDs. You also see Machines for the OpenShift masters. They are not part of a MachineSet because they are somewhat stateful and their management is handled by different operators and through a different process.

There is currently no protection for the master Machines. Do not accidentally or intentionally delete them, as this will potentially break your cluster. It is repairable, but it is not fun.
```
oc get nodes
	NAME                                         STATUS   ROLES    AGE     VERSION
	ip-10-0-142-163.us-east-2.compute.internal   Ready    master   4h7m    v1.16.2
	ip-10-0-142-4.us-east-2.compute.internal     Ready    worker   3h59m   v1.16.2
	ip-10-0-147-192.us-east-2.compute.internal   Ready    worker   3h59m   v1.16.2
	ip-10-0-154-52.us-east-2.compute.internal    Ready    master   4h7m    v1.16.2
	ip-10-0-164-147.us-east-2.compute.internal   Ready    worker   3h59m   v1.16.2
	ip-10-0-167-205.us-east-2.compute.internal   Ready    master   4h7m    v1.16.2
```

# Cluster Scaling
Because of the magic of Operators and the way in which OpenShift uses them to manage Machines and Nodes, scaling your cluster in OpenShift 4 is extremely trivial.
```
oc get machineset -n openshift-machine-api
	NAME                                   DESIRED   CURRENT   READY   AVAILABLE   AGE
	cluster-2f83-d7fvm-worker-us-east-2a   1         1         1       1           4h8m
	cluster-2f83-d7fvm-worker-us-east-2b   1         1         1       1           4h8m
	cluster-2f83-d7fvm-worker-us-east-2c   1         1         1       1           4h8m

oc scale machineset cluster-2f83-d7fvm-worker-us-east-2c -n openshift-machine-api --replicas=2
	machineset.machine.openshift.io/cluster-2f83-d7fvm-worker-us-east-2c scaled

oc get machines -n openshift-machine-api
	NAME                                         PHASE         TYPE         REGION      ZONE         AGE
	cluster-2f83-d7fvm-master-0                  Running       m5.xlarge    us-east-2   us-east-2a   4h11m
	cluster-2f83-d7fvm-master-1                  Running       m5.xlarge    us-east-2   us-east-2b   4h11m
	cluster-2f83-d7fvm-master-2                  Running       m5.xlarge    us-east-2   us-east-2c   4h11m
	cluster-2f83-d7fvm-worker-us-east-2a-5c2fs   Running       m5.2xlarge   us-east-2   us-east-2a   4h7m
	cluster-2f83-d7fvm-worker-us-east-2b-4665z   Running       m5.2xlarge   us-east-2   us-east-2b   4h7m
	cluster-2f83-d7fvm-worker-us-east-2c-mjcrn   Provisioned   m5.2xlarge   us-east-2   us-east-2c   23s
	cluster-2f83-d7fvm-worker-us-east-2c-sdlkq   Running       m5.2xlarge   us-east-2   us-east-2c   4h7m

oc get machines -n openshift-machine-api
	NAME                                         PHASE     TYPE         REGION      ZONE         AGE
	cluster-2f83-d7fvm-master-0                  Running   m5.xlarge    us-east-2   us-east-2a   4h15m
	cluster-2f83-d7fvm-master-1                  Running   m5.xlarge    us-east-2   us-east-2b   4h15m
	cluster-2f83-d7fvm-master-2                  Running   m5.xlarge    us-east-2   us-east-2c   4h15m
	cluster-2f83-d7fvm-worker-us-east-2a-5c2fs   Running   m5.2xlarge   us-east-2   us-east-2a   4h10m
	cluster-2f83-d7fvm-worker-us-east-2b-4665z   Running   m5.2xlarge   us-east-2   us-east-2b   4h10m
	cluster-2f83-d7fvm-worker-us-east-2c-mjcrn   Running   m5.2xlarge   us-east-2   us-east-2c   4m16s
	cluster-2f83-d7fvm-worker-us-east-2c-sdlkq   Running   m5.2xlarge   us-east-2   us-east-2c   4h10m
oc get nodes
	NAME                                         STATUS   ROLES    AGE     VERSION
	ip-10-0-142-163.us-east-2.compute.internal   Ready    master   4h14m   v1.16.2
	ip-10-0-142-4.us-east-2.compute.internal     Ready    worker   4h7m    v1.16.2
	ip-10-0-147-192.us-east-2.compute.internal   Ready    worker   4h7m    v1.16.2
	ip-10-0-154-52.us-east-2.compute.internal    Ready    master   4h15m   v1.16.2
	ip-10-0-164-147.us-east-2.compute.internal   Ready    worker   4h7m    v1.16.2
	ip-10-0-167-205.us-east-2.compute.internal   Ready    master   4h15m   v1.16.2
	ip-10-0-171-211.us-east-2.compute.internal   Ready    worker   42s     v1.16.2
```

# Scale Down
```
oc scale machineset cluster-2f83-d7fvm-worker-us-east-2c -n openshift-machine-api --replicas=1
	machineset.machine.openshift.io/cluster-2f83-d7fvm-worker-us-east-2c scaled
```






