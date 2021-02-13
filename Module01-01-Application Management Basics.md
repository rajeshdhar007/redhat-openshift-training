```
oc new-project app-management

oc new-app quay.io/thoraxe/mapit

oc get pods
	NAME             READY   STATUS      RESTARTS   AGE
	mapit-1-5qmgw    1/1     Running     0          24s
	mapit-1-deploy   0/1     Completed   0          32s
```
The -deploy pod is related to the DeploymentConfig which will be discussed later.
	
```
oc describe pod mapit-1-5qmgw

oc get services

oc describe service mapit

oc describe svc mapit

oc get service mapit -o yaml

oc get pod mapit-1-5qmgw -o yaml

oc get endpoints mapit

oc expose service mapit
	route.route.openshift.io/mapit exposed
	
oc get route
	NAME    HOST/PORT                                                           PATH   SERVICES   PORT       TERMINATION   WILDCARD
	mapit   mapit-app-management.apps.cluster-2f83.2f83.sandbox603.opentlc.com              mapit      8080-tcp
```	
Now that we know the background of what a ReplicatonController and DeploymentConfig are, we can explore how they work and are related. Take a look at the DeploymentConfig (DC) that was created for you when you told OpenShift to stand up the mapit image
```
oc get dc

oc get rc
```
## Scaling the Application
```
oc scale --replicas=2 dc/mapit
```
## Application "Self-Healing"
Because OpenShift’s RCs are constantly monitoring to see that the desired number of Pods are actually running, you might also expect that OpenShift will "fix" the situation if it is ever not right. You would be correct!
```
oc delete pod mapit-1-6lczv && oc get pods
```
## Scale Down
```
oc scale --replicas=1 dc/mapit
```

## Liveness Probe
A liveness probe checks if the container in which it is configured is still running. If the liveness probe fails, the container is killed, which will be subjected to its restart policy.

## Readiness Probe
A readiness probe determines if a container is ready to service requests. If the readiness probe fails, the endpoint’s controller ensures the container has its IP address removed from the endpoints of all services that should match it. A readiness probe can be used to signal to the endpoint’s controller that even though a container is running, it should not receive any traffic.

## Add Probes to the Application
The oc set command can be used to perform several different functions, one of which is creating and/or modifying probes. The mapit application exposes an endpoint which we can check to see if it is alive and ready to respond. You can test it using curl:
```
curl mapit-app-management.apps.cluster-2f83.2f83.sandbox603.opentlc.com/health
	{"status":"UP","diskSpace":{"status":"UP","total":128283815936,"free":116883525632,"threshold":10485
	760}}
```	
## Liveness Probe
```
oc set probe dc/mapit --liveness --get-url=http://:8080/health --initial-delay-seconds=30
	{"status":"UP","diskSpace":{"status":"UP","total":128283815936,"free":116883525632,"threshold":10485
	760}}[~] $ oc set probe dc/mapit --liveness --get-url=http://:8080/health --initial-delay-seconds=30
	deploymentconfig.apps.openshift.io/mapit probes updated
	

oc describe dc mapit
	…
	…
	mapit:
	    Image:              quay.io/thoraxe/mapit@sha256:8c7e0349b6a016e3436416f3c54debda4594ba09fd34b8a
	0dee0c4497102590d
	    Ports:              8080/TCP, 8778/TCP, 9779/TCP
	    Host Ports:         0/TCP, 0/TCP, 0/TCP
	    Liveness:           http-get http://:8080/health delay=30s timeout=1s period=10s #success=1 #fai
	lure=3
	…
	…
```	
## Readiness Probe
```
oc set probe dc/mapit --readiness --get-url=http://:8080/health --initial-delay-seconds=30
	deploymentconfig.apps.openshift.io/mapit probes updated
```

## Examining DeploymentConfigs and ReplicationControllers
```
oc get pods
	NAME             READY   STATUS      RESTARTS   AGE
	mapit-1-deploy   0/1     Completed   0          63m
	mapit-2-deploy   0/1     Completed   0          5m35s
	mapit-3-98tqg    1/1     Running     0          2m17s
	mapit-3-deploy   0/1     Completed   0          2m20s
```	
Notice that there are 3 -deploy pods? And that the name of the current mapit pod has the number 3 in it? This is because each change to the DeploymentConfig is counted as a configuration change, which triggered a new deployment. The -deploy pod is responsible for ensuring the new deployment happens

```	
oc get deploymentconfigs
	NAME    REVISION   DESIRED   CURRENT   TRIGGERED BY
	mapit   3          1         1         config,image(mapit:latest)

oc get replicationcontrollers
	NAME      DESIRED   CURRENT   READY   AGE
	mapit-1   0         0         0       65m
	mapit-2   0         0         0       7m16s
	mapit-3   1         1         1       4m2s
```	
Each time a new deployment is triggered, the deployer pod creates a new ReplicationController which then is responsible for ensuring that pods exist. Notice that the old RCs have a desired scale of zero, and the most recent RC has a desired scale of 1.
If you oc describe each of these RCs you will see how -1 has no probes, and then -2 and -3 have the new probes, respectively.
