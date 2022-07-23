# clair-update

```
$ git config --global credential.helper 'cache --timeout 72000'
$ git add . ; git commit -a -m "update README" ; git push -u origin main
```

https://access.redhat.com/documentation/en-us/red_hat_quay/3.7/html-single/deploy_red_hat_quay_on_openshift_with_the_quay_operator/index#quay_operator_features

# 7.2. Manually updating the vulnerability databases for Clair in an air-gapped OpenShift cluster

## 7.2.1. Obtaining clairctl

**The command from Doc does not work**
```
$ oc get pod | grep example-registry-clair-app
example-registry-clair-app-84448c9b78-mzcq2            1/1     Running     2 (21m ago)   21h
example-registry-clair-app-84448c9b78-svdnk            1/1     Running     2 (21m ago)   21h

$ oc -n quay-enterprise cp example-registry-clair-app-84448c9b78-mzcq2:/usr/bin/clairctl ./clairctl
time="2022-07-23T01:42:51Z" level=error msg="exec failed: unable to start container process: exec: \"tar\": executable file not found in $PATH"
command terminated with exit code 255

```

**Workround**

Create Storage
```
cat clair-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: share-date-01
  namespace: quay-enterprise
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem

$ oc apply -f clair-pvc.yaml 

$ oc get pvc share-date-01
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
share-date-01   Bound    pvc-9abacc62-96bb-460c-bd8c-c6259f64a173   1Gi        RWX            ocs-storagecluster-cephfs   19s

```

Update Deploy to use the storage
```
$ oc edit deploy example-registry-clair-app

spec:

  template:

    spec:
      containers:

        volumeMounts:
        - mountPath: "/updaters"             
          name: data

      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: share-date-01

$ oc rsh example-registry-clair-app-67659d9b88-sfp4b
sh-4.4$ ls -ld /updaters
drwxrwxrwx. 2 root root 0 Jul 23 02:19 /updaters

sh-4.4$ cp /bin/clairctl /updaters/.
sh-4.4$ cp /clair/config.yaml /updaters/.

sh-4.4$ ls -l /updaters  
total 24329
-rwxr-xr-x. 1 1000670000 root 24910609 Jul 23 02:34 clairctl
-rw-r--r--. 1 1000670000 root     1055 Jul 23 02:35 config.yaml

sh-4.4$ exit
exit

```

**Revert back conf on deploy example-registry-clair-app**


