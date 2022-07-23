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

Retrieve back the file inside POD
```
$ echo "clarictl and config.yaml" >index.html

$ oc create configmap index-html --from-file=index.html=./index.html

$ cat deploy-http.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
    app.kubernetes.io/component: web
    app.kubernetes.io/instance: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      deployment: web
  template:
    metadata:
      labels:
        deployment: web
    spec:
      containers:
      - image: registry.redhat.io/rhel8/httpd-24:1-161.1638356842
        name: web
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 8443
          protocol: TCP
        resources: {}
        volumeMounts:
        - mountPath: "/var/www/html/data" 
          name: data
        - name: index-html
          mountPath: /var/www/html/index.html
          readOnly: true
          subPath: index.html
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: index.html
            path: index.html
          name: index-html
        name: index-html
      - name: data
        persistentVolumeClaim:
          claimName: share-date-01

$ oc apply -f deploy-http.yaml

$ oc get pod
web-dc6d675c9-lhjh5                                    1/1     Running     0             2m39s

$ oc expose deploy web

$ oc expose service/web

$ oc get route
web   web-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com   web port-1   None

$ wget http://web-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com/data/clairctl

$ wget http://web-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com/data/config.yaml

$ chmod u+x ./clairctl

$ ll clairctl config.yaml
-rwxr--r--. 1 jinzha-redhat.com users 24910609 Jul 23 02:34 clairctl
-rw-r--r--. 1 jinzha-redhat.com users     1055 Jul 23 02:35 config.yaml
```


Exporting the updaters bundle

Export RHEL and Oracle updaters
```
$ mv config.yaml resource-config.yaml

$ vim resource-config.yaml

updaters:
  sets:
    - rhel
    - oracle

$ ./clairctl --config ./resource-config.yaml export-updaters updates.gz

$ ll updates.gz
-rw-r--r--. 1 jinzha-redhat.com users 132385639 Jul 23 03:09 updates.gz

$ file updates.gz
updates.gz: gzip compressed data, original size 1556964611

```

Importing the updaters bundle into the air-gapped environment

```
$ cp resource-config.yaml dest--config.yaml

indexer:
    connstring: host=example-registry-clair-postgres port=5432 dbname=postgres user=postgres password=postgres sslmode=disable pool_max_conns=33
    layer_scan_concurrency: 5
    migrations: true
    scanlock_retry: 10
    scanner:
        package: {}
        dist: {}
        repo: {}
    airgap: ture

:%s#example-registry-clair-postgres#localhost#g

```

On other session, run port forward
```
$ oc port-forward service/example-registry-clair-postgres 5432:5432
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
```

http_listen_addr

