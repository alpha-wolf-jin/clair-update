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

## 7.2.2. Retrieving the Clair config


```
$ oc get secret -n quay-enterprise example-registry-clair-config-secret-kk9h966cfh -o "jsonpath={$.data['config\.yaml']}" | base64 -d > clair-config.yaml

# use localhost to replace example-registry-clair-postgres
$ vim clair-config.yaml

:%s#example-registry-clair-postgres#localhost#g

# limit updater to RHEL & Oracle
$ cat clair-config.yaml
auth:
    psk:
        iss:
            - quay
            - clairctl
        key: YWlybkxReDhPQTlvTk1wYWVtVDVwam5ZZHBXM01JVjM=
http_listen_addr: :8080
indexer:
    connstring: host=localhost port=5432 dbname=postgres user=postgres password=postgres sslmode=disable pool_max_conns=33
    layer_scan_concurrency: 5
    migrations: true
    scanlock_retry: 10
log_level: info
matcher:
    connstring: host=localhost port=5432 dbname=postgres user=postgres password=postgres sslmode=disable pool_max_conns=33
    migrations: true
metrics:
    name: prometheus
notifier:
    connstring: host=localhost port=5432 dbname=postgres user=postgres password=postgres sslmode=disable pool_max_conns=33
    delivery_interval: 1m0s
    migrations: true
    poll_interval: 5m0s
    webhook:
        callback: http://example-registry-clair-app/notifier/api/v1/notifications
        target: https://example-registry-quay-quay-enterprise.apps.cluster-ptscz.ptscz.sandbox878.opentlc.com/secscan/notification
updaters:
  sets:
    - rhel
    - oracle


```

## 7.2.3. Exporting the updaters bundle

```
cd ./clair/updaters

$ ./clairctl-3-7 --config ./clair-config.yaml export-updaters updates.gz

$ ll
total 177420
-rwxr-xr-x. 1            231069 231069  22595592 Jul 23 10:37 clairctl-3-6
-rwxr-xr-x. 1            231069 231069  24910609 Jul 23 10:41 clairctl-3-7
-rw-r--r--. 1 jinzha-redhat.com users       1282 Jul 23 10:47 clair-config.yaml
-rw-r--r--. 1 jinzha-redhat.com users  134163351 Jul 23 10:46 updates.gz
```

## 7.2.4. Configuring access to the Clair database in the air-gapped OpenShift cluster

On other session run port forward
```
$ oc port-forward service/example-registry-clair-postgres 5432:5432

```

## 7.2.5. Importing the updaters bundle into the air-gapped environment

**clairctl import-updaters failed**
```
$ ./clairctl-3-7 -D --config ./clair-config.yaml import-updaters -g updates.gz 
2022-07-23T10:53:02Z ERR  error="gzip: invalid header"

```

**Workround 02**
```
$ ./clairctl-3-6 -D --config ./clair-config.yaml import-updaters  updates.gz 
2022-07-23T10:53:43Z INF fingerprint match, skipping component=libvuln/OfflineImporter updater=RHEL7-dotnet-3.0
2022-07-23T10:53:43Z INF fingerprint match, skipping component=libvuln/OfflineImporter updater=RHEL7-openstack-9
2022-07-23T10:53:43Z INF fingerprint match, skipping component=libvuln/OfflineImporter updater=RHEL7-openshift-3.2

```

> **Guess there is a bug in clairctl from quay3.7**

# Others

```
$ oc get pod | grep postgres
example-registry-clair-postgres-65d984b49f-swcww       1/1     Running     1 (11m ago)   12m

$ oc rsh example-registry-clair-postgres-65d984b49f-swcww
sh-4.4$ psql -U postgres;
psql (10.21)
Type "help" for help.

postgres=# TABLE scanner;
 id |          name           | version |     kind     
----+-------------------------+---------+--------------
  1 | dpkg                    | 4       | package
  2 | apk                     | v0.0.1  | package
  3 | rpm                     | 4       | package
  4 | python                  | 0.1.0   | package
  5 | java                    | 3       | package
  6 | rhel_containerscanner   | 1       | package
  7 | debian                  | v0.0.2  | distribution
  8 | ubuntu                  | v0.0.1  | distribution
  9 | alpine                  | v0.0.1  | distribution
 10 | rhel                    | v0.0.1  | distribution
 11 | aws                     | v0.0.1  | distribution
 12 | oracle                  | v0.0.1  | distribution
 13 | suse                    | v0.0.1  | distribution
 14 | photon                  | v0.0.1  | distribution
 15 | rhel-repository-scanner | 1.1     | repository
 16 | pip                     | 0.0.1   | repository
 17 | maven                   | 0.0.1   | repository
 18 | rhel_containerscanner   | 1       | repository
(18 rows)

postgres=# 
postgres=# \q
sh-4.4$ exit
exit

$ oc get pod | grep clair-app
example-registry-clair-app-5b6b694cb5-2tftm            1/1     Running     0             19m

$ oc rsh example-registry-clair-app-5b6b694cb5-2tftm
sh-4.4$ cat^Cconfig/ 
sh-4.4$ ls /
bin  boot  clair  config  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
sh-4.4$ cat /clair/config.yaml 
auth:
    psk:
        iss:
            - quay
            - clairctl
        key: elh5SjJnYmNBS2tRM3c5MjE0eUlLNXp1U3NOUGdzSEI=
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
        target: https://example-registry-quay-openshift-operators.apps.cluster-k8twm.k8twm.sandbox1612.opentlc.com/secscan/notification
sh-4.4$ exit
exit

```
