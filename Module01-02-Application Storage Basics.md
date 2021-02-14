# Adding Persistent Volume Claims

Here’s how you would instruct OpenShift to create a PersistentVolume object, which represents external, persistent storage, and have it mounted inside the container’s filesystem:
```
oc set volume dc/mapit --add --name=mapit-storage -t pvc --claim-mode=ReadWriteOnce --claim-size=1Gi --claim-name=mapit-storage --mount-path=/app-storage
	deploymentconfig.apps.openshift.io/mapit volume updated
```
Next the DeploymentConfig of mapit is updated to reference this storage and make it available under the /app-storage directory inside the pod.
```
oc get dc mapit
	NAME      REVISION   DESIRED   CURRENT   TRIGGERED BY
	mapit     4          1         1         config,image(mapit:latest)

oc get pod
	mapit-1-deploy   0/1     Completed   0          74m
	mapit-2-deploy   0/1     Completed   0          16m
	mapit-3-deploy   0/1     Completed   0          13m
	mapit-4-8fwcn    1/1     Running     0          2m22s
	mapit-4-deploy   0/1     Completed   0          2m26s

oc describe dc mapit
	…
	…
	    Mounts:
	      /app-storage from mapit-storage (rw)
	…
	  Volumes:
	   mapit-storage:
	    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
	    ClaimName:  mapit-storage
	    ReadOnly:   false
	…
	…
```

# Storage Classes
When OpenShift 4 was first installed, a dynamic storage provider for AWS EBS was configured. You can see this StorageClass with the following:
```
oc get storageclass
	NAME            PROVISIONER             AGE
	gp2 (default)   kubernetes.io/aws-ebs   3h6m
```

# Persistent Volume (Claims)
The command you ran earlier referenced a claim. Storage in a Kubernetes environment uses a system of volume claims and volumes. A user makes a PersistentVolumeClaim and Kubernetes tries to find a PersistentVolume that matches. In the case where a volume does not already exist, if there is a dynamic provisioner that satisfies the claim, a PersistentVolume is created.
```
oc get persistentvolume
	NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
	                          STORAGECLASS   REASON   AGE
	pvc-36f49fbf-de88-40c8-90f7-e7f918334e1c   1Gi        RWO            Delete           Bound    app-m
	anagement/mapit-storage   gp2                     7m51s
	

oc get persistentvolumeclaim -n app-management
	NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   ST
	ORAGECLASS   AGE
	mapit-storage   Bound    pvc-36f49fbf-de88-40c8-90f7-e7f918334e1c   1Gi        RWO            gp
	2            8m34s
```

# Testing Persistent Storage
Get the name of your pod using oc get pods and then log on to the pod using the remote-shell capability of the oc client:
```
oc rsh mapit-4-d872b
	sh-4.2$ ls -ahl /
	total 20K
	drwxr-xr-x.   1 root  root         81 Dec 10 18:01 .
	drwxr-xr-x.   1 root  root         81 Dec 10 18:01 ..
	-rw-r--r--.   1 root  root        16K Dec 14  2016 anaconda-post.log
	drwxrwsr-x.   3 root  1000540000 4.0K Dec 10 18:01 app-storage
	lrwxrwxrwx.   1 root  root          7 Dec 14  2016 bin -> usr/bin
	drwxrwxrwx.   1 jboss root         45 Aug  4  2017 deployments
	drwxr-xr-x.   5 root  root        360 Dec 10 18:01 dev
	drwxr-xr-x.   1 root  root         93 Jan 18  2017 etc
	drwxr-xr-x.   2 root  root          6 Nov  5  2016 home
	lrwxrwxrwx.   1 root  root          7 Dec 14  2016 lib -> usr/lib
	lrwxrwxrwx.   1 root  root          9 Dec 14  2016 lib64 -> usr/lib64
	drwx------.   2 root  root          6 Dec 14  2016 lost+found
	drwxr-xr-x.   2 root  root          6 Nov  5  2016 media
	drwxr-xr-x.   2 root  root          6 Nov  5  2016 mnt
	drwxr-xr-x.   1 root  root         19 Jan 18  2017 opt
	dr-xr-xr-x. 272 root  root          0 Dec 10 18:01 proc
	dr-xr-x---.   2 root  root        114 Dec 14  2016 root
	drwxr-xr-x.   1 root  root         21 Dec 10 18:01 run
	lrwxrwxrwx.   1 root  root          8 Dec 14  2016 sbin -> usr/sbin
	drwxr-xr-x.   2 root  root          6 Nov  5  2016 srv
	dr-xr-xr-x.  13 root  root          0 Dec 10 14:59 sys
	drwxrwxrwt.   1 root  root        121 Dec 10 18:01 tmp
	drwxr-xr-x.   1 root  root         69 Dec 16  2016 usr
	drwxr-xr-x.   1 root  root         41 Dec 14  2016 var
	sh-4.2$ echo "Hello World from OpenShift" > /app-storage/hello.txt
	exit
	
oc rsh mapit-4-8fwcn cat /app-storage/hello.txt
Hello World from OpenShift
```

Now, to verify that persistent storage really works, delete your pod:
```
oc delete pod mapit-4-8fwcn && oc get pods
	pod "mapit-4-8fwcn" deleted
	NAME             READY   STATUS      RESTARTS   AGE
	mapit-1-deploy   0/1     Completed   0          86m
	mapit-2-deploy   0/1     Completed   0          28m
	mapit-3-deploy   0/1     Completed   0          25m
	mapit-4-deploy   0/1     Completed   0          15m
	mapit-4-jjlgp    0/1     Running     0          13s

oc rsh mapit-4-jjlgp cat /app-storage/hello.txt
	Hello World from OpenShift
```



