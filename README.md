# clair-update

```
$ git config --global credential.helper 'cache --timeout 72000'
$ git add . ; git commit -a -m "update README" ; git push -u origin main
```

Cluser And Quay version
```
$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.10.22   True        False         2d19h   Cluster version is 4.10.22

$ oc get sub -n openshift-operators
NAME            PACKAGE         SOURCE             CHANNEL
quay-operator   quay-operator   redhat-operators   stable-3.7

$ oc get csv
NAME                   DISPLAY        VERSION   REPLACES               PHASE
quay-operator.v3.7.4   Red Hat Quay   3.7.4     quay-operator.v3.7.3   Succeeded

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

**Workround 01**

```
$ ll /home/jinzha-redhat.com/clair/config/config.yaml /home/jinzha-redhat.com/clair/updaters
-rwxrwxrwx. 1 jinzha-redhat.com users 0 Jul 23 09:56 /home/jinzha-redhat.com/clair/config/config.yaml

/home/jinzha-redhat.com/clair/updaters:
total 0

$ chmod -R 777 /home/jinzha-redhat.com/clair

$ podman run -it --name clairv4 \
  -p 8081:8081 -p 8089:8089 \
  -e CLAIR_CONF=/clair/config.yaml -e CLAIR_MODE=combo \
  -v /home/jinzha-redhat.com/clair/config:/clair:Z \
  -v /home/jinzha-redhat.com/clair/updaters:/updaters:Z \
  --entrypoint /bin/bash registry.redhat.io/quay/clair-rhel8:v3.6.0
bash-4.4$ cp /bin/clairctl /updaters/.
bash-4.4$ exit
exit

$ mv /home/jinzha-redhat.com/clair/updaters/clairctl /home/jinzha-redhat.com/clair/updaters/clairctl-3-6

$ podman rm clairv4

$ podman run -it --name clairv4 \
  -p 8081:8081 -p 8089:8089 \
  -e CLAIR_CONF=/clair/config.yaml -e CLAIR_MODE=combo \
  -v /home/jinzha-redhat.com/clair/config:/clair:Z \
  -v /home/jinzha-redhat.com/clair/updaters:/updaters:Z \
  --entrypoint /bin/bash registry.redhat.io/quay/clair-rhel8:v3.7.3
bash-4.4$ cp /bin/clairctl /updaters/.
bash-4.4$ exit

$ mv /home/jinzha-redhat.com/clair/updaters/clairctl /home/jinzha-redhat.com/clair/updaters/clairctl-3-7

$ ll /home/jinzha-redhat.com/clair/updaters/clairctl*
-rwxr-xr-x. 1 231069 231069 22595592 Jul 23 10:37 /home/jinzha-redhat.com/clair/updaters/clairctl-3-6
-rwxr-xr-x. 1 231069 231069 24910609 Jul 23 10:41 /home/jinzha-redhat.com/clair/updaters/clairctl-3-7

```

```
cd ./clair/updaters

$ ll
total 177420
-rwxr-xr-x. 1            231069 231069  22595592 Jul 23 10:37 clairctl-3-6
-rwxr-xr-x. 1            231069 231069  24910609 Jul 23 10:41 clairctl-3-7
-rw-r--r--. 1 jinzha-redhat.com users       1282 Jul 23 10:47 dest-config.yaml
-rw-r--r--. 1 jinzha-redhat.com users  134163351 Jul 23 10:46 updates.gz

log_level: info
indexer:
    connstring: host=localhost port=5432 dbname=postgres user=postgres password=postgres sslmode=disable 
    scanlock_retry: 10
    layer_scan_concurrency: 5
    migrations: true
    scanner:
        package: {}
        dist: {}
        repo: {}
    airgap: true
matcher:
    connstring: host=localhost port=5432 dbname=postgres user=postgres password=postgres sslmode=disable 
    max_conn_pool: 100
    indexer_addr: ""
    migrations: true
    period: null
    disable_updaters: false
notifier:
    connstring: host=localhost port=5432 dbname=postgres user=postgres password=postgres sslmode=disable 
    delivery_interval: 1m0s
    migrations: true
    indexer_addr: ""
    matcher_addr: ""
    poll_interval: 5m
    webhook:
        callback: http://example-registry-clair-app/notifier/api/v1/notifications
        target: https://example-registry-quay-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com/secscan/notification
auth:
    psk:
        iss:
            - quay
            - clairctl
        key: YWlybkxReDhPQTlvTk1wYWVtVDVwam5ZZHBXM01JVjM=
metrics:
    name: prometheus
    prometheus:
      endpoint: null
    dogstatsd:
      url: ""
updaters:
  sets:
    - rhel
    - oracle

```

On other session run port forward
```
$ oc port-forward service/example-registry-clair-postgres 5432:5432

```

Import Updaters
```
$ ./clairctl-3-7 -D --config ./dest-config.yaml import-updaters -g updates.gz 
2022-07-23T10:53:02Z ERR  error="gzip: invalid header"

$ ./clairctl-3-6 -D --config ./dest-config.yaml import-updaters  updates.gz 
2022-07-23T10:53:43Z INF fingerprint match, skipping component=libvuln/OfflineImporter updater=RHEL7-dotnet-3.0
2022-07-23T10:53:43Z INF fingerprint match, skipping component=libvuln/OfflineImporter updater=RHEL7-openstack-9
2022-07-23T10:53:43Z INF fingerprint match, skipping component=libvuln/OfflineImporter updater=RHEL7-openshift-3.2

```
**Workround 02**

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

Find out IPs used in quay project

- 3.133.250.207
- 3.13.168.216
- 10.128.0.0/14
- 172.30.0.0/16

```
$ oc get route
NAME                                  HOST/PORT                                                                                             PATH   SERVICES                              PORT     TERMINATION     WILDCARD
example-registry-quay                 example-registry-quay-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com                        example-registry-quay-app             http     edge/Redirect   None
example-registry-quay-builder         example-registry-quay-builder-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com                example-registry-quay-app             grpc     edge/Redirect   None
example-registry-quay-config-editor   example-registry-quay-config-editor-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com          example-registry-quay-config-editor   http     edge/Redirect   None

$ dig example-registry-quay-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com

;; ANSWER SECTION:
example-registry-quay-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com. 60 IN A 3.133.250.207
example-registry-quay-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com. 60 IN A 3.13.168.216

$ dig example-registry-quay-builder-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com

;; ANSWER SECTION:
example-registry-quay-builder-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com. 60 IN A 3.133.250.207
example-registry-quay-builder-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com. 60 IN A 3.13.168.216


$ dig example-registry-quay-config-editor-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com

;; ANSWER SECTION:
example-registry-quay-config-editor-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com. 60	IN A 3.133.250.207
example-registry-quay-config-editor-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com. 60	IN A 3.13.168.216

$ oc describe network.config/cluster

Spec:
  Cluster Network:
    Cidr:         10.128.0.0/14
    Host Prefix:  23
  External IP:
    Policy:
  Network Type:  OpenShiftSDN
  Service Network:
    172.30.0.0/16

```

Delete the QuayRegistries instance

Set Egress restriction to simulate disconnection environment

```
$ cat egress.yaml 
apiVersion: network.openshift.io/v1
kind: EgressNetworkPolicy
metadata:
  name: default
spec:
  egress: 
  - type: Allow
    to:
      cidrSelector: 172.30.0.0/16
  - type: Allow
    to:
      cidrSelector: 10.128.0.0/14
  - type: Allow
    to:
      cidrSelector: 3.133.250.207/32
  - type: Allow
    to:
      cidrSelector: 3.13.168.216/32
  - type: Allow
    to:
      dnsName: example-registry-quay-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com
  - type: Deny
    to:
      cidrSelector: 0.0.0.0/0

$ oc apply -f egress.yaml

```

Redeploy QuayRegistry again. The images were pulled locally during the previous installation.

Confirm the CVE data is empty

```
$ oc get pod
NAME                                                   READY   STATUS      RESTARTS      AGE
example-registry-clair-postgres-5f9c6cd797-pn5mq       1/1     Running     1 (88s ago)   109s

$ oc rsh example-registry-clair-postgres-5f9c6cd797-pn5mq
sh-4.4$ psql -U postgres;
psql (10.21)
Type "help" for help.
postgres=# \dt
                   List of relations
 Schema |           Name            | Type  |  Owner   
--------+---------------------------+-------+----------
 public | dist                      | table | postgres
 public | dist_scanartifact         | table | postgres
 public | enrichment                | table | postgres
 public | indexreport               | table | postgres
 public | key                       | table | postgres
 public | layer                     | table | postgres
 public | libindex_migrations       | table | postgres
 public | libvuln_migrations        | table | postgres
 public | manifest                  | table | postgres
 public | manifest_index            | table | postgres
 public | manifest_layer            | table | postgres
 public | notification              | table | postgres
 public | notification_body         | table | postgres
 public | notifier_migrations       | table | postgres
 public | notifier_update_operation | table | postgres
 public | package                   | table | postgres
 public | package_scanartifact      | table | postgres
 public | receipt                   | table | postgres
 public | repo                      | table | postgres
 public | repo_scanartifact         | table | postgres
 public | scanned_layer             | table | postgres
 public | scanned_manifest          | table | postgres
 public | scanner                   | table | postgres
 public | scannerlist               | table | postgres
 public | uo_enrich                 | table | postgres
 public | uo_vuln                   | table | postgres
 public | update_operation          | table | postgres
 public | vuln                      | table | postgres
(28 rows)

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

postgres=# TABLE vuln;
 id | hash_kind | hash | updater | name | description | issued | links | severity | normalized_severity | package_name | package_version | package_module | package_arch | pa
ckage_kind | dist_id | dist_name | dist_version | dist_version_code_name | dist_version_id | dist_arch | dist_cpe | dist_pretty_name | repo_name | repo_key | repo_uri | fixe
d_in_version | arch_operation | vulnerable_range | version_kind 
----+-----------+------+---------+------+-------------+--------+-------+----------+---------------------+--------------+-----------------+----------------+--------------+---
-----------+---------+-----------+--------------+------------------------+-----------------+-----------+----------+------------------+-----------+----------+----------+-----
-------------+----------------+------------------+--------------
(0 rows)

postgres=# \q
sh-4.4$ 
sh-4.4$ exit

$ oc get pod
NAME                                                   READY   STATUS      RESTARTS       AGE
example-registry-clair-app-666d94b57-b85k8             1/1     Running     0              6m9s

```

Get the key from clair config in use

```
$ oc rsh example-registry-clair-app-666d94b57-b85k8
sh-4.4$ cat /clair/config.yaml 
auth:
    psk:
        iss:
            - quay
            - clairctl
        key: YWlybkxReDhPQTlvTk1wYWVtVDVwam5ZZHBXM01JVjM=
http_listen_addr: :8080
indexer:
    connstring: host=example-registry-clair-postgres port=5432 dbname=postgres user=postgres password=postgres sslmode=disable pool_max_conns=33
    layer_scan_concurrency: 5
    migrations: true
    scanlock_retry: 10
log_level: info
matcher:
    connstring: host=example-registry-clair-postgres port=5432 dbname=postgres user=postgres password=postgres sslmode=disable pool_max_conns=33
    migrations: true
metrics:
    name: prometheus
notifier:
    connstring: host=example-registry-clair-postgres port=5432 dbname=postgres user=postgres password=postgres sslmode=disable pool_max_conns=33
    delivery_interval: 1m0s
    migrations: true
    poll_interval: 5m0s
    webhook:
        callback: http://example-registry-clair-app/notifier/api/v1/notifications
        target: https://example-registry-quay-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com/secscan/notification


```

Set up NFS
```
# yum -y install nfs-utils

# systemctl enable --now nfs-server rpcbind

# mkdir /nfs

# chmod 777 /nfs

# cat /etc/exports
/nfs *(rw)

# exportfs -r

# systemctl restart nfs-server

# ip a | grep -w inet
    inet 127.0.0.1/8 scope host lo
    inet 192.168.0.130/24 brd 192.168.0.255 scope global dynamic noprefixroute eth0
# mkdir /remote

# mount -t nfs 192.168.0.130:/nfs /remote

# df -Th /remote/
Filesystem         Type  Size  Used Avail Use% Mounted on
192.168.0.130:/nfs nfs4   30G   16G   15G  52% /remote

# echo 123 >/remote/file01

# cat /remote/file01
123

# umount /remote 

```


```
$ cat pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001 
spec:
  capacity:
    storage: 5Gi 
  accessModes:
  - ReadWriteOnce 
  nfs: 
    path: /nfs 
    server: 192.168.0.130
  persistentVolumeReclaimPolicy: Retain 

$ oc apply -f pv.yaml

$ oc get pv pv0001
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv0001   5Gi        RWO            Retain           Available                                   42s

$ cat pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-claim1
spec:
  accessModes:
    - ReadWriteOnce 
  resources:
    requests:
      storage: 5Gi 
  volumeName: pv0001
  storageClassName: ""

$ oc apply -f pvc.yaml
persistentvolumeclaim/nfs-claim1 created

$ oc get pvc nfs-claim1
NAME         STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-claim1   Bound    pv0001   5Gi        RWO                           8s

```

Add PVC to deploy
```
$ oc get deploy
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
example-registry-clair-app            2/2     2            2           94m

$ oc edit deploy example-registry-clair-app
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
          claimName: nfs-claim1


```
**Failed since there is Firewall between worker nodes and bastion in Lab which is out of my control**

Create NFS HTTP on azure, copy update.gz to nfs. clair uses curl to download

```

```
