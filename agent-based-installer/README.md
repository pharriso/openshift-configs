Agent Based install
=========

Some steps specific to agent based install.

With this method, Ansible will create the VMs and the agent configs. It generates the ISO, mounts it and boots the servers.

Run the playbook as follows for single node openshift:

```
ansible-playbook -i sno-inventory ocp_install_agent.yml
```

For 3 node compact cluster:

```
ansible-playbook -i compact-inventory ocp_install_agent.yml
```

A variable controls whether the Ansible Playbook attempts the install or just creates the servers. If you want to manually play with agent based installer then set agent_install variable to false. You can then edit and copy your own configs:

Manual install steps include:

Copy the agent-config.yaml and install-config.yaml into /opt/{{ ocp_name }} or the right directory.

Generate the ISO.

```bash
openshift-install --dir /opt/agent/ agent create image
```

Mount the ISO and power on the VMs.


To monitor the install run:

```
export KUBECONFIG=/opt/sno/auth/kubeconfig
openshift-install --dir /opt/sno agent wait-for install-complete
```

Finding device details
=========

If the install fails on disk, then you can work out the available disks by logging onto the rendevouz host. The USER_AUTH_TOKEN for the assisted service is in /etc/assisted/rendezvous-host.env



```
source /etc/assisted/rendezvous-host.env
```

Get the cluster ID:

```
curl -sS -H "Authorization: $USER_AUTH_TOKEN" <rendevouz_ip>:8090/api/assisted-install/v2/clusters |  jq -r '.[].id'
```

Query hosts in cluster. Disk information can be found in inventory.

```
curl -sS -H "Authorization: $USER_AUTH_TOKEN" <rendevouz_ip>:8090/api/assisted-install/v2/clusters/<cluster_id> | jq .hosts
```

