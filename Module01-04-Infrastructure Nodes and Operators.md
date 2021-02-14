# Infrastructure Nodes and Operators

## OpenShift components that fall into the infrastructure categorization include:

* kubernetes and OpenShift control plane services ("masters")
* router
* container image registry
* cluster metrics collection ("monitoring")
* cluster aggregated logging
* service brokers

```
oc get machineset -n openshift-machine-api -o yaml cluster-2f83-d7fvm-worker-us-east-2c
	apiVersion: machine.openshift.io/v1beta1
	kind: MachineSet
	metadata:
	  creationTimestamp: "2020-12-10T14:52:41Z"
	  generation: 3
	  labels:
	    machine.openshift.io/cluster-api-cluster: cluster-2f83-d7fvm
	  name: cluster-2f83-d7fvm-worker-us-east-2c
	  namespace: openshift-machine-api
	  resourceVersion: "85307"
	  selfLink: /apis/machine.openshift.io/v1beta1/namespaces/openshift-machine-api/machinesets/cluster-2f8
	3-d7fvm-worker-us-east-2c
	  uid: 0918b101-24f0-40fd-991b-7ef9f9ab87c2
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      machine.openshift.io/cluster-api-cluster: cluster-2f83-d7fvm
	      machine.openshift.io/cluster-api-machineset: cluster-2f83-d7fvm-worker-us-east-2c
	  template:
	    metadata:
	      creationTimestamp: null
	      labels:
	        machine.openshift.io/cluster-api-cluster: cluster-2f83-d7fvm
	        machine.openshift.io/cluster-api-machine-role: worker
	        machine.openshift.io/cluster-api-machine-type: worker
	        machine.openshift.io/cluster-api-machineset: cluster-2f83-d7fvm-worker-us-east-2c
	    spec:
	      metadata:
	        creationTimestamp: null
	      providerSpec:
	        value:
	          ami:
	            id: ami-0d8f77b753c0d96dd
	          apiVersion: awsproviderconfig.openshift.io/v1beta1
	          blockDevices:
	          - ebs:
	              iops: 0
	              volumeSize: 120
	              volumeType: gp2
	          credentialsSecret:
	            name: aws-cloud-credentials
	          deviceIndex: 0
	          iamInstanceProfile:
	            id: cluster-2f83-d7fvm-worker-profile
	          instanceType: m5.2xlarge
	          kind: AWSMachineProviderConfig
	          metadata:
	            creationTimestamp: null
	          placement:
	            availabilityZone: us-east-2c
	            region: us-east-2
	          publicIp: null
	          securityGroups:
	          - filters:
	            - name: tag:Name
	              values:
	              - cluster-2f83-d7fvm-worker-sg
	          subnet:
	            filters:
	            - name: tag:Name
	              values:
	              - cluster-2f83-d7fvm-private-us-east-2c
	          tags:
	          - name: kubernetes.io/cluster/cluster-2f83-d7fvm
	            value: owned
	          - name: guid
	            value: 2f83
	          - name: owner
	            value: sramanat+generic@redhat.com
	          - name: platform
	            value: RHPDS
	          - name: uuid
	            value: 4ec688cb-d57a-4b18-bfd5-3cc82e9f6345
	          - name: Stack
	            value: project ocp4-workshop-2f83
	          - name: catalog_item
	            value: OCP4_and_Container_Storage_for_Admins_L_
	          - name: email
	            value: sramanat_generic@redhat.com
	          - name: env_type
	            value: ocp4-workshop
	          userDataSecret:
	            name: worker-user-data
	status:
	  availableReplicas: 1
	  fullyLabeledReplicas: 1
	  observedGeneration: 3
	  readyReplicas: 1
	  replicas: 1
```	
# Defining a Custom MachineSet
Now that you’ve analyzed an existing MachineSet it’s time to go over the rules for creating one, at least for a simple change like we’re making:
* Don’t change anything in the providerSpec
* Don’t change any instances of machine.openshift.io/cluster-api-cluster: <clusterid>
* Give your MachineSet a unique name
* Make sure any instances of machine.openshift.io/cluster-api-machineset match the name
* Add labels you want on the nodes to .spec.template.spec.metadata.labels
* Even though you’re changing MachineSet name references, be sure not to change the subnet.
This sounds complicated, but we have a little program and some steps that will do the hard work for you:
```
bash /opt/app-root/src/support/machineset-generator.sh 1 infra 0 | oc create -f -
export MACHINESET=$(oc get machineset -n openshift-machine-api -l machine.openshift.io/cluster-api-machine-role=infra -o jsonpath='{.items[0].metadata.name}')
oc patch machineset $MACHINESET -n openshift-machine-api --type='json' -p='[{"op": "add", "path": "/spec/template/spec/metadata/labels", "value":{"node-role.kubernetes.io/worker":"", "node-role.kubernetes.io/infra":""} }]'
oc scale machineset $MACHINESET -n openshift-machine-api --replicas=3

	[~] $ bash /opt/app-root/src/support/machineset-generator.sh 1 infra 0 | oc create -f -
		machineset.machine.openshift.io/cluster-2f83-d7fvm-infra-us-east-2a created
	[~] $ export MACHINESET=$(oc get machineset -n openshift-machine-api -l machine.openshift.io/cluster-ap
		machine-role=infra -o jsonpath='{.items[0].metadata.name}')
	[~] $ oc patch machineset $MACHINESET -n openshift-machine-api --type='json' -p='[{"op": "add", "path":
	/spec/template/spec/metadata/labels", "value":{"node-role.kubernetes.io/worker":"", "node-role.kubernet
	.io/infra":""} }]'
		machineset.machine.openshift.io/cluster-2f83-d7fvm-infra-us-east-2a patched
	[~] $ oc scale machineset $MACHINESET -n openshift-machine-api --replicas=3
		machineset.machine.openshift.io/cluster-2f83-d7fvm-infra-us-east-2a scaled
	
oc get machineset -n openshift-machine-api
	NAME                                   DESIRED   CURRENT   READY   AVAILABLE   AGE
	cluster-2f83-d7fvm-infra-us-east-2a    3         3                             2m9s
	cluster-2f83-d7fvm-worker-us-east-2a   1         1         1       1           4h40m
	cluster-2f83-d7fvm-worker-us-east-2b   1         1         1       1           4h40m
	cluster-2f83-d7fvm-worker-us-east-2c   1         1         1       1           4h40m

oc get nodes
	NAME                                         STATUS     ROLES          AGE     VERSION
	ip-10-0-128-95.us-east-2.compute.internal    NotReady   infra,worker   6s      v1.16.2
	ip-10-0-139-107.us-east-2.compute.internal   NotReady   infra,worker   5s      v1.16.2
	ip-10-0-139-54.us-east-2.compute.internal    NotReady   infra,worker   5s      v1.16.2
	ip-10-0-142-163.us-east-2.compute.internal   Ready      master         4h41m   v1.16.2
	ip-10-0-142-4.us-east-2.compute.internal     Ready      worker         4h33m   v1.16.2
	ip-10-0-147-192.us-east-2.compute.internal   Ready      worker         4h34m   v1.16.2
	ip-10-0-154-52.us-east-2.compute.internal    Ready      master         4h41m   v1.16.2
	ip-10-0-164-147.us-east-2.compute.internal   Ready      worker         4h34m   v1.16.2
	ip-10-0-167-205.us-east-2.compute.internal   Ready      master         4h41m   v1.16.2
```
# Query for labels on a node
```
oc get node ip-10-0-128-95.us-east-2.compute.internal --show-labels
	NAME                                        STATUS   ROLES          AGE     VERSION   LABELS
	ip-10-0-128-95.us-east-2.compute.internal   Ready    infra,worker   2m57s   v1.16.2   beta.kubernetes.io/arch
	=amd64,beta.kubernetes.io/instance-type=m5.4xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes
	.io/region=us-east-2,failure-domain.beta.kubernetes.io/zone=us-east-2a,kubernetes.io/arch=amd64,kubernetes.io
	/hostname=ip-10-0-128-95,kubernetes.io/os=linux,node-role.kubernetes.io/infra=,node-role.kubernetes.io/worker
	=,node.openshift.io/os_id=rhcos
```
# Router
The OpenShift router is managed by an Operator called openshift-ingress-operator. Its Pod lives in the openshift-ingress-operator project:
```
oc get pod -n openshift-ingress-operator
	NAME                                READY   STATUS    RESTARTS   AGE
	ingress-operator-7854d9fd4f-9q5rn   2/2     Running   0          4h41m

oc get pods -n openshift-ingress -o wide
	NAME                              READY   STATUS    RESTARTS   AGE     IP           NODE
	                    NOMINATED NODE   READINESS GATES
	router-default-7897696f67-7ppjt   1/1     Running   0          4h42m   10.128.2.8   ip-10-0-147-192.us-east-2
	.compute.internal   <none>           <none>
	router-default-7897696f67-ptrvz   1/1     Running   0          4h42m   10.131.0.6   ip-10-0-164-147.us-east-2
	.compute.internal   <none>           <none>

oc get node ip-10-0-147-192.us-east-2.compute.internal
	NAME                                         STATUS   ROLES    AGE     VERSION
	ip-10-0-147-192.us-east-2.compute.internal   Ready    worker   4h47m   v1.16.2
```
The OpenShift router operator uses a custom resource definition (CRD) called ingresses.config.openshift.io to define the default routing subdomain for the cluster:
```
oc get ingresses.config.openshift.io cluster -o yaml
	apiVersion: config.openshift.io/v1
	kind: Ingress
	metadata:
	  creationTimestamp: "2020-12-10T14:52:14Z"
	  generation: 1
	  name: cluster
	  resourceVersion: "429"
	  selfLink: /apis/config.openshift.io/v1/ingresses/cluster
	  uid: f6f3eca4-8c37-4b25-ab52-e36084b660a2
	spec:
	  domain: apps.cluster-2f83.2f83.sandbox603.opentlc.com
	status: {}
```
Individual router deployments are managed via the ingresscontrollers.operator.openshift.io CRD. There is a default one created in the openshift-ingress-operator namespace:
```
oc get ingresscontrollers.operator.openshift.io default -n openshift-ingress-operator -o yaml
	apiVersion: operator.openshift.io/v1
	kind: IngressController
	metadata:
	  creationTimestamp: "2020-12-10T15:00:50Z"
	  finalizers:
	  - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
	  generation: 1
	  name: default
	  namespace: openshift-ingress-operator
	  resourceVersion: "12486"
	  selfLink: /apis/operator.openshift.io/v1/namespaces/openshift-ingress-operator/ingresscontrollers/default
	  uid: bc7ea39b-11f3-4d6c-8914-f5165ab00f32
	spec:
	  replicas: 2
	status:
	  availableReplicas: 2
	  conditions:
	  - lastTransitionTime: "2020-12-10T15:00:50Z"
	    reason: Valid
	    status: "True"
	    type: Admitted
	  - lastTransitionTime: "2020-12-10T15:01:32Z"
	    status: "True"
	    type: Available
	  - lastTransitionTime: "2020-12-10T15:01:32Z"
	    message: The deployment has Available status condition set to True
	    reason: DeploymentAvailable
	    status: "False"
	    type: DeploymentDegraded
	  - lastTransitionTime: "2020-12-10T15:00:57Z"
	    message: The endpoint publishing strategy supports a managed load balancer
	    reason: WantedByEndpointPublishingStrategy
	    status: "True"
	    type: LoadBalancerManaged
	  - lastTransitionTime: "2020-12-10T15:01:01Z"
	    message: The LoadBalancer service is provisioned
	    reason: LoadBalancerProvisioned
	    status: "True"
	    type: LoadBalancerReady
	  - lastTransitionTime: "2020-12-10T15:00:57Z"
	    message: DNS management is supported and zones are specified in the cluster DNS
	      config.
	    reason: Normal
	    status: "True"
	    type: DNSManaged
	  - lastTransitionTime: "2020-12-10T15:01:04Z"
	    message: The record is provisioned in all reported zones.
	    reason: NoFailedZones
	    status: "True"
	    type: DNSReady
	  - lastTransitionTime: "2020-12-10T15:01:32Z"
	    status: "False"
	    type: Degraded
	  domain: apps.cluster-2f83.2f83.sandbox603.opentlc.com
	  endpointPublishingStrategy:
	    loadBalancer:
	      scope: External
	    type: LoadBalancerService
	  observedGeneration: 1
	  selector: ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
	  tlsProfile:
	    ciphers:
	    - TLS_AES_128_GCM_SHA256
	    - TLS_AES_256_GCM_SHA384
	    - TLS_CHACHA20_POLY1305_SHA256
	    - ECDHE-ECDSA-AES128-GCM-SHA256
	    - ECDHE-RSA-AES128-GCM-SHA256
	    - ECDHE-ECDSA-AES256-GCM-SHA384
	    - ECDHE-RSA-AES256-GCM-SHA384
	    - ECDHE-ECDSA-CHACHA20-POLY1305
	    - ECDHE-RSA-CHACHA20-POLY1305
	    - DHE-RSA-AES128-GCM-SHA256
	    - DHE-RSA-AES256-GCM-SHA384
	    minTLSVersion: VersionTLS12
```

To specify a nodeSelector that tells the router pods to hit the infrastructure nodes, we can apply the following configuration:
```
oc apply -f /opt/app-root/src/support/ingresscontroller.yaml
	Warning: oc apply should be used on resource created by either oc create --save-config or oc apply
	ingresscontroller.operator.openshift.io/default configured
	
cat /opt/app-root/src/support/ingresscontroller.yaml
	apiVersion: operator.openshift.io/v1
	kind: IngressController
	metadata:
	  finalizers:
	  - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
	  name: default
	  namespace: openshift-ingress-operator
	spec:
	  nodePlacement:
	    nodeSelector:
	      matchLabels:
	        node-role.kubernetes.io/infra: ""
```	
	
# Registry
The registry uses a similar CRD mechanism to configure how the operator deploys the actual registry pods. That CRD is configs.imageregistry.operator.openshift.io. You will edit the cluster CR object in order to add the nodeSelector. First, take a look at it:
```
oc get configs.imageregistry.operator.openshift.io/cluster -o yaml
	apiVersion: imageregistry.operator.openshift.io/v1
	kind: Config
	metadata:
	  creationTimestamp: "2020-12-10T15:00:46Z"
	  finalizers:
	  - imageregistry.operator.openshift.io/finalizer
	  generation: 2
	  name: cluster
	  resourceVersion: "12063"
	  selfLink: /apis/imageregistry.operator.openshift.io/v1/configs/cluster
	  uid: 34f84d1c-72c1-4635-8c2c-b78c89681cdb
	spec:
	  defaultRoute: false
	  disableRedirect: false
	  httpSecret: f2524f48a66650d63e1063de7dd746c25d7426f47a5768394857bbbd95a80d6e68add6bf0ccabf8ed20120c41f80
	b9bebc5b61e93edc44ebb5901a0af399c26f
	  logging: 2
	  managementState: Managed
	  proxy:
	    http: ""
	    https: ""
	    noProxy: ""
	  readOnly: false
	  replicas: 1
	  requests:
	    read:
	      maxInQueue: 0
	      maxRunning: 0
	      maxWaitInQueue: 0s
	    write:
	      maxInQueue: 0
	      maxRunning: 0
	      maxWaitInQueue: 0s
	  storage:
	    s3:
	      bucket: cluster-2f83-d7fvm-image-registry-us-east-2-ffcldlnwuyoipuglve
	      encrypt: true
	      keyID: ""
	      region: us-east-2
	      regionEndpoint: ""
	status:
	  conditions:
	  - lastTransitionTime: "2020-12-10T15:00:46Z"
	    reason: S3 Bucket Exists
	    status: "True"
	    type: StorageExists
	  - lastTransitionTime: "2020-12-10T15:00:47Z"
	    message: Public access to the S3 bucket and its contents have been successfully
	      blocked.
	    reason: Public Access Block Successful
	    status: "True"
	    type: StoragePublicAccessBlocked
	  - lastTransitionTime: "2020-12-10T15:00:47Z"
	    message: Tags were successfully applied to the S3 bucket
	    reason: Tagging Successful
	    status: "True"
	    type: StorageTagged
	  - lastTransitionTime: "2020-12-10T15:00:47Z"
	    message: Default AES256 encryption was successfully enabled on the S3 bucket
	    reason: Encryption Successful
	    status: "True"
	    type: StorageEncrypted
	  - lastTransitionTime: "2020-12-10T15:00:47Z"
	    message: Default cleanup of incomplete multipart uploads after one (1) day was
	      successfully enabled
	    reason: Enable Cleanup Successful
	    status: "True"
	    type: StorageIncompleteUploadCleanupEnabled
	  - lastTransitionTime: "2020-12-10T15:01:16Z"
	    message: The registry is ready
	    reason: Ready
	    status: "True"
	    type: Available
	  - lastTransitionTime: "2020-12-10T15:01:22Z"
	    message: The registry is ready
	    reason: Ready
	    status: "False"
	    type: Progressing
	  - lastTransitionTime: "2020-12-10T15:00:48Z"
	    status: "False"
	    type: Degraded
	  - lastTransitionTime: "2020-12-10T15:00:48Z"
	    status: "False"
	    type: Removed
	  observedGeneration: 2
	  readyReplicas: 0
	  storage:
	    s3:
	      bucket: cluster-2f83-d7fvm-image-registry-us-east-2-ffcldlnwuyoipuglve
	      encrypt: true
	      keyID: ""
	      region: us-east-2
	      regionEndpoint: ""
	  storageManaged: true
	[~] $ oc get configs.imageregistry.operator.openshift.io/cluster -o yaml
	apiVersion: imageregistry.operator.openshift.io/v1
	kind: Config
	metadata:
	  creationTimestamp: "2020-12-10T15:00:46Z"
	  finalizers:
	  - imageregistry.operator.openshift.io/finalizer
	  generation: 2
	  name: cluster
	  resourceVersion: "12063"
	  selfLink: /apis/imageregistry.operator.openshift.io/v1/configs/cluster
	  uid: 34f84d1c-72c1-4635-8c2c-b78c89681cdb
	spec:
	  defaultRoute: false
	  disableRedirect: false
	  httpSecret: f2524f48a66650d63e1063de7dd746c25d7426f47a5768394857bbbd95a80d6e68add6bf0ccabf8ed20120c41f80
	b9bebc5b61e93edc44ebb5901a0af399c26f
	  logging: 2
	  managementState: Managed
	  proxy:
	    http: ""
	    https: ""
	    noProxy: ""
	  readOnly: false
	  replicas: 1
	  requests:
	    read:
	      maxInQueue: 0
	      maxRunning: 0
	      maxWaitInQueue: 0s
	    write:
	      maxInQueue: 0
	      maxRunning: 0
	      maxWaitInQueue: 0s
	  storage:
	    s3:
	      bucket: cluster-2f83-d7fvm-image-registry-us-east-2-ffcldlnwuyoipuglve
	      encrypt: true
	      keyID: ""
	      region: us-east-2
	      regionEndpoint: ""
	status:
	  conditions:
	  - lastTransitionTime: "2020-12-10T15:00:46Z"
	    reason: S3 Bucket Exists
	    status: "True"
	    type: StorageExists
	  - lastTransitionTime: "2020-12-10T15:00:47Z"
	    message: Public access to the S3 bucket and its contents have been successfully
	      blocked.
	    reason: Public Access Block Successful
	    status: "True"
	    type: StoragePublicAccessBlocked
	  - lastTransitionTime: "2020-12-10T15:00:47Z"
	    message: Tags were successfully applied to the S3 bucket
	    reason: Tagging Successful
	    status: "True"
	    type: StorageTagged
	  - lastTransitionTime: "2020-12-10T15:00:47Z"
	    message: Default AES256 encryption was successfully enabled on the S3 bucket
	    reason: Encryption Successful
	    status: "True"
	    type: StorageEncrypted
	  - lastTransitionTime: "2020-12-10T15:00:47Z"
	    message: Default cleanup of incomplete multipart uploads after one (1) day was
	      successfully enabled
	    reason: Enable Cleanup Successful
	    status: "True"
	    type: StorageIncompleteUploadCleanupEnabled
	  - lastTransitionTime: "2020-12-10T15:01:16Z"
	    message: The registry is ready
	    reason: Ready
	    status: "True"
	    type: Available
	  - lastTransitionTime: "2020-12-10T15:01:22Z"
	    message: The registry is ready
	    reason: Ready
	    status: "False"
	    type: Progressing
	  - lastTransitionTime: "2020-12-10T15:00:48Z"
	    status: "False"
	    type: Degraded
	  - lastTransitionTime: "2020-12-10T15:00:48Z"
	    status: "False"
	    type: Removed
	  observedGeneration: 2
	  readyReplicas: 0
	  storage:
	    s3:
	      bucket: cluster-2f83-d7fvm-image-registry-us-east-2-ffcldlnwuyoipuglve
	      encrypt: true
	      keyID: ""
	      region: us-east-2
	      regionEndpoint: ""
	  storageManaged: true
```

Modify the .spec of the registry CR in order to add the desired nodeSelector
```
oc patch configs.imageregistry.operator.openshift.io/cluster -p '{"spec":{"nodeSelector":{"node-role.kubernetes.io/infra": ""}}}' --type=merge
	config.imageregistry.operator.openshift.io/cluster patched
	
oc get pod -n openshift-image-registry
	NAME                                            READY   STATUS    RESTARTS   AGE
	cluster-image-registry-operator-9754995-9gjc7   2/2     Running   0          24h
	image-registry-cf5bc5f67-ggf82                  1/1     Running   0          103s
	node-ca-5pwz5                                   1/1     Running   0          24h
	node-ca-9pcvz                                   1/1     Running   0          24h
	node-ca-dgzm5                                   1/1     Running   0          24h
	node-ca-f5swv                                   1/1     Running   0          19h
	node-ca-frpkq                                   1/1     Running   0          24h
	node-ca-gbwqs                                   1/1     Running   0          19h
	node-ca-jwmzs                                   1/1     Running   0          24h
	node-ca-lgql9                                   1/1     Running   0          19h
	node-ca-vl6gl                                   1/1     Running   0          24h
```

# Monitoring
The Cluster Monitoring operator is responsible for deploying and managing the state of the Prometheus+Grafana+AlertManager cluster monitoring stack. It is installed by default during the initial cluster installation. Its operator uses a ConfigMap in the openshift-monitoring project to set various tunables and settings for the behavior of the monitoring stack.

The following ConfigMap definition will configure the monitoring solution to be redeployed onto infrastructure nodes.
```	
  apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: cluster-monitoring-config
	  namespace: openshift-monitoring
	data:
	  config.yaml: |+
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


oc get configmap cluster-monitoring-config -n openshift-monitoring
	Error from server (NotFound): configmaps "cluster-monitoring-config" not found

oc create -f /opt/app-root/src/support/cluster-monitoring-configmap.yaml
{{cluster-monitoring-configmap.yaml --> Contains content above ConfigMap}}
	configmap/cluster-monitoring-config created

Watch the monitoring pods move from worker to infra Nodes with:
watch 'oc get pod -n openshift-monitoring'
	Every 2.0s: oc get pod -n openshift-monitoring                                                 Fri Dec 11 15:32:19 2020
	
	NAME                                           READY   STATUS    RESTARTS   AGE
	alertmanager-main-0                            3/3     Running   0          77s
	alertmanager-main-1                            3/3     Running   0          85s
	alertmanager-main-2                            3/3     Running   0          105s
	cluster-monitoring-operator-7b898649bd-fvvgz   1/1     Running   0          24h
	grafana-58b94d6b88-wnlqm                       2/2     Running   0          96s
	kube-state-metrics-7f6c88d446-s8xfk            3/3     Running   0          118s
	node-exporter-8b9l6                            2/2     Running   0          24h
	node-exporter-92wdp                            2/2     Running   0          24h
	node-exporter-9l9j8                            2/2     Running   0          24h
	node-exporter-bzvnd                            2/2     Running   0          19h
	node-exporter-cc8rk                            2/2     Running   0          19h
	node-exporter-hn9dz                            2/2     Running   0          24h
	node-exporter-p2wqp                            2/2     Running   0          24h
	node-exporter-xbn6m                            2/2     Running   0          19h
	node-exporter-ztp9c                            2/2     Running   0          24h
	openshift-state-metrics-785dd5b54c-qtmg8       3/3     Running   0          24h
	prometheus-adapter-5f9c68d46c-6jzh7            1/1     Running   0          102s
	prometheus-adapter-5f9c68d46c-qc88h            1/1     Running   0          99s
	prometheus-k8s-0                               7/7     Running   1          67s
	prometheus-k8s-1                               7/7     Running   1          85s
	prometheus-operator-5fd87f4c7d-jzjxv           1/1     Running   0          116s
	telemeter-client-568bb4569b-dvqhc              3/3     Running   0          107s
	thanos-querier-65466f7b48-dtdfj                4/4     Running   0          24h
	thanos-querier-65466f7b48-tx5cv                4/4     Running   0          24h
```	
--OR--
```
oc get pod -w -n openshift-monitoring
	NAME                                           READY   STATUS    RESTARTS   AGE
	alertmanager-main-0                            3/3     Running   0          112s
	alertmanager-main-1                            3/3     Running   0          2m
	alertmanager-main-2                            3/3     Running   0          2m20s
	cluster-monitoring-operator-7b898649bd-fvvgz   1/1     Running   0          24h
	grafana-58b94d6b88-wnlqm                       2/2     Running   0          2m11s
	kube-state-metrics-7f6c88d446-s8xfk            3/3     Running   0          2m33s
	node-exporter-8b9l6                            2/2     Running   0          24h
	node-exporter-92wdp                            2/2     Running   0          24h
	node-exporter-9l9j8                            2/2     Running   0          24h
	node-exporter-bzvnd                            2/2     Running   0          19h
	node-exporter-cc8rk                            2/2     Running   0          19h
	node-exporter-hn9dz                            2/2     Running   0          24h
	node-exporter-p2wqp                            2/2     Running   0          24h
	node-exporter-xbn6m                            2/2     Running   0          19h
	node-exporter-ztp9c                            2/2     Running   0          24h
	openshift-state-metrics-785dd5b54c-qtmg8       3/3     Running   0          24h
	prometheus-adapter-5f9c68d46c-6jzh7            1/1     Running   0          2m17s
	prometheus-adapter-5f9c68d46c-qc88h            1/1     Running   0          2m14s
	prometheus-k8s-0                               7/7     Running   1          102s
	prometheus-k8s-1                               7/7     Running   1          2m
	prometheus-operator-5fd87f4c7d-jzjxv           1/1     Running   0          2m31s
	telemeter-client-568bb4569b-dvqhc              3/3     Running   0          2m22s
	thanos-querier-65466f7b48-dtdfj                4/4     Running   0          24h
	thanos-querier-65466f7b48-tx5cv                4/4     Running   0          24h
```


