# clair-update

'''
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
