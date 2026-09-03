# BMaaS on OCP

This repo contains instructions for using BMaaS Tech Preview in OCP:

## Prerequisites

You need a QCOW on a webserver along with a sha512sum to validate the image e.g. rhel9.qcow2

```
sha512sum rhel9.qcow2 > rhel9.qcow2.sha512sum
```

The deployment process will boot off a CoreOS live image and uses /dev/shm mapped to /tmp to store the boot image (storing it in RAM). Make sure you allocate enough RAM for the BMH in a dev/test environment to host the image.


## Enable BMH on all namespaces

```
oc patch provisioning/provisioning-configuration   --type merge -p '{"spec": {"watchAllNamespaces": true}}'
```

## Create namespace

```
oc new-project bmaas
```

## Creating configs

Create BMC secret for BMH:

```
oc create -f bmc-secret.yaml 
```

Create userdata and netdata secrets:


```
oc create secret generic bmh1-user-data   --from-file=userData=bmh1-userdata.yaml -n bmaas
oc create secret generic bmh1-network-data --from-file=networkData=bmh1-netdata.yaml -n bmaas
```


Create BMH referencing userdata, netdata and provisioning image:

```
bmh.yaml
```

## Check deploy

Validate BMH deploy status through bmh CR or web UI. 
