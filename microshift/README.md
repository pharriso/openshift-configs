# Microshift setup

This repo contains instructions for deploying microshift.

## Prerequisites

* RHEL 9 server deployed
* RHEL server registered with CDN or an internal repository
* Access to rhocp-4.22-for-rhel-9-x86_64-rpms and fast-datapath-for-rhel-9-x86_64-rpms
* OpenShift pull secret

## Enable repos and install microshift

```
subscription-manager repos --enable rhocp-4.22-for-rhel-9-$(uname -m)-rpms --enable fast-datapath-for-rhel-9-$(uname -m)-rpms
dnf install -y microshift microshift-olm
```

## Setup dynamic LVM storage for persistent volumes

If you want persistent storage you can use LVM dynamic storage. This requires additional disks configured for LVM. In this example the extra disk is /dev/vdb

```
dnf install lvm2 -y
pvcreate /dev/vdb
vgcreate microshift /dev/vdb
```

Create an lvm config file. Copy the sample file:

```
cp /etc/microshift/lvmd.yaml.default /etc/microshift/lvmd.yaml
```

Edit /etc/microshift/lvmd.yaml file with the correct details. Example config:

```
device-classes:
  - name: default
    volume-group: microshift
    spare-gb: 0
    default: true
```

## Configure microshift

You can further customise microshift by creating /etc/microshift/config.yaml. For this example I didn't make any changes, but you can find details in the [documentation](https://docs.redhat.com/en/documentation/red_hat_build_of_microshift/4.22/html/configuring/microshift-default-config-yaml)

## Pull secret

Download pull secret from https://console.redhat.com/openshift/install/pull-secret

Copy the pull secret to the microshift VM in /etc/crio/openshift-pull-secret

```
chown root:root /etc/crio/openshift-pull-secret
chmod 600 /etc/crio/openshift-pull-secret
```

## Start / Install microshift

To start microshift run:

```
systemctl start microshift
```

To interact with the deployment you can use the kubeconfig file that is provided. Validate the node is ready and check pods are starting:

```
export KUBECONFIG=/var/lib/microshift/resources/kubeadmin/kubeconfig
oc get no
oc get po -A
```

Node should show Ready and pods should reach a running state.

## Define OLM catalogsource

Define required operator catalog sources. An example catalogsource CR for Red Hat operators is in this repo.

```
oc apply -f olm-catalogsource/redhat-operators.yaml
```

Wait for the associated pod to start:


```
oc get po -n openshift-marketplace 
redhat-operators-j859n                 1/1     Running     0          3h58m
```

## Install sample apps - Ansible Automation Platform example

Install the AAP operator:


```
oc apply -f aap/aap-operator-install.yaml
```

Check that the relevant operator controllers are running for the different AAP components:


```
oc get po -n ansible-automation-platform | grep operator-controll
aap-gateway-operator-controller-manager-768897bf97-lqbg8          1/1     Running     0          8h
ansible-lightspeed-operator-controller-manager-844766cc94-pjxj5   1/1     Running     0          8h
automation-controller-operator-controller-manager-7944c879msm2z   1/1     Running     0          8h
automation-hub-operator-controller-manager-54b56497d9-rzkzg       1/1     Running     0          8h
```

Deploy an instance of AAP. The example disables all components except for controller:


```
oc apply -f aap/aap-instance.yaml
```

Expose the AAP gateway with a custom hostname e.g. aap.pharriso.co.uk. This DNS record should point to the microshift VM IP.

```
oc create route edge aap-public --service=aap --hostname aap.pharriso.co.uk
```

Check you can login to the AAP instance. Grab the admin password from the secret:

```
oc extract secret/aap-admin-password --to=-
```

Browse to your route and login!


## Deploying Automation Orchestrator

Deploy the orchestrator operator:

```
oc apply -f aap-orchestrator-operator/aap-orchestrator-operator.yaml
```

Check the operator controller is running

```
oc get po -n automation-orchestrator
automation-orchestrator-operator-controller-manager-965ddbr8gvh   1/1     Running     0          7h15m
```

Create secrets for automation orchestrator database. You can use the example in aap-orchestrator/db-secrets.yaml  and just add the relevant details.

```
oc apply -f aap-orchestrator/db-secrets.yaml 
```

Now deploy an instance of orchestrator. Edit the example yaml with relevant hostname and DB connection details and then apply it:

```
oc apply -f aap-orchestrator/aap-orchestrator-instance.yaml
```
