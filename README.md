# OpenShift Container Platform Upgrade Guide

This guide is meant to be a reference guide using a real example to help people understand how to upgrade OpenShift.

The example shows following items:
*  Host Preparation for installation
*  Install OpenShift 3.10
*  Host Preparation for 3.11 upgrade
*  Upgrade OpenShift 3.10 to 3.11


The sample environment is a `6` node OpenShift environment with `3 master`, `2 infra`, `1 worker` nodes.

Here's the required system information:

| Node | Hostname | IP | OS | Resource |
|--------------|----------------|--------------------|--------------------|-----------------|
| Master 1 | ocp-test-m.example.com, ocp-test-m1.example.com | 192.168.0.201 | RHEL 7.x minimal | 4c,16g |
| Master 2 | ocp-test-m2.example.com | 192.168.0.202 | RHEL 7.x minimal | 4c,16g |
| Master 3 | ocp-test-m3.example.com | 192.168.0.203 | RHEL 7.x minimal | 4c,16g |
| Infra 1 | ocp-test-i1.example.com, *.apps.example.com | 192.168.0.204 | RHEL 7.x minimal | 2c,8g |
| Infra 2 | ocp-test-i2.example.com | 192.168.0.205 | RHEL 7.x minimal | 2c,8g |
| Worker 1 | ocp-test-w1.example.com | 192.168.0.206 | RHEL 7.x minimal | 2c,4g |
| Bastion | rhlab1.example.com | 192.168.0.41 | RHEL 7.x N/A | N/A |

Wildcard Domain:
````
*.apps.example.com
````
Red Hat Subscription Account:
````
Account: sampleSubAcct
Password: sampleSubPW
Pool ID: 1234567890qwertyuiop
````

A. Host Preparation for installation
------------

1. Configure `/etc/sysconfig/network-scripts/ifcfg-eth0` on all nodes, add following network config to the file.
	```
	BOOTPROTO=none
	ONBOOT=yes
	IPADDR=192.168.0.201
	GATEWAY=192.168.0.254
	DNS1=192.168.0.201
	```
2. set hostname with following command example
	```
	hostnamectl set-hostname ocp-test-m1.example.com
	```
------------

3. Git clone this repo to `Bastion` working folder

4. Execute following command on `Bastion`.
	```
	ansible nodes -m "copy" -a "src=./materials/hosts dest=/etc/hosts"
	ansible nodes -m "copy" -a "src=./materials/node-custom.conf dest=/etc/dnsmasq.d/node-custom.conf"

	ansible nodes -m "lineinfile" -a "dest=/etc/sysconfig/network-scripts/ifcfg-eth0 line='NM_CONTROLLED=yes'"
	ansible nodes -m "lineinfile" -a "dest=/etc/dnsmasq.conf line='server=8.8.8.8'"
	ansible nodes -a "systemctl enable dnsmasq"
	ansible nodes -a "systemctl start dnsmasq"
	ansible nodes -a "systemctl restart NetworkManager"
	ansible nodes -a "nslookup google.com"

	ansible nodes -a "subscription-manager register --username=sampleSubAcct --password=sampleSubPW"

	ansible nodes -a "subscription-manager attach --pool=1234567890qwertyuiop"

	ansible nodes -a "subscription-manager repos --disable='*'"

	ansible nodes -a "subscription-manager repos --enable='rhel-7-server-rpms' --enable='rhel-7-server-extras-rpms' --enable='rhel-7-server-ose-3.10-rpms' --enable='rhel-7-fast-datapath-rpms' --enable='rhel-7-server-ansible-2.4-rpms'"

	ansible nodes -a "yum install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct -y"

	ansible nodes -a "yum install openshift-ansible -y"

	ansible nodes -a "yum install docker-1.13.1 -y"
	ansible nodes -a "yum update -y"
	ansible nodes -a "reboot -h"

	yum install openshift-ansible -y
	```
------------

B. Install OpenShift 3.10
------------
1. Execute following command on `Bastion`.
	```
	ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml

	ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
	```
2. Edit `/etc/origin/master/master-config.yaml` on every master node, replace `DenyAllPasswordIdentityProvider` with `AllowAllPasswordIdentityProvider`
3. Verify following items:
* login OpenShift console with user `admin`
* `oc new-app` an example app and test app through route.
* `oc adm diagnostics` should only shows logging related errors.
* `oc adm policy add-cluster-role-to-user cluster-admin admin`

C. Host Preparation for 3.11 upgrade
------------
1. Execute following command on `Bastion`.
	```
	oc login -u admin
	oc adm migrate storage --include=* --loglevel=2 --confirm --config /etc/origin/master/admin.kubeconfig
	
	ansible nodes -a "subscription-manager refresh"

	## Do Backup - the steps only cover ETCD v3
	ansible nodes -a "mkdir ~/OCP3_10_BK"
	ansible masters -a "cp /etc/origin/master/master-config.yaml ~/OCP3_10_BK/master-config.yaml"
	ansible masters -a "cp /etc/origin/master/master.env ~/OCP3_10_BK/master.env"
	ansible masters -a "cp /etc/origin/master/scheduler.json ~/OCP3_10_BK/scheduler.json"
	ansible nodes -a "cp /etc/origin/node/node-config.yaml ~/OCP3_10_BK/node-config.yaml"
	ansible masters -a "cp /etc/etcd/etcd.conf ~/OCP3_10_BK/etcd.conf"

	## Backup ETCD config
	ansible masters -a "mkdir -p /backup/etcd-config-$(date +%Y%m%d)/"
	ansible masters -a "cp -R /etc/etcd/ /backup/etcd-config-$(date +%Y%m%d)/"
	```
2. Execute following command on one of `Master` node.
	```
	## Backup ETCD Data
	etcdctl3 --cert="/etc/etcd/peer.crt" --key=/etc/etcd/peer.key --cacert="/etc/etcd/ca.crt" member list
	etcdctl3 --cert="/etc/etcd/peer.crt" --key=/etc/etcd/peer.key --cacert="/etc/etcd/ca.crt" \
	         --endpoints="https://ocp-test-m1.example.com:2379,https://ocp-test-m2.example.com:2379,https://ocp-test-m3.example.com:2379" \
	         endpoint health

	export ETCD_POD_MANIFEST="/etc/origin/node/pods/etcd.yaml"
	export ETCD_EP=$(grep https ${ETCD_POD_MANIFEST} | cut -d '/' -f3)
	export ETCD_POD=$(oc get pods -n kube-system | grep -o -m 1 '\S*etcd\S*')
	# Change endpoint using ETCD_EP info
	oc exec ${ETCD_POD} -c etcd -- /bin/bash -c "ETCDCTL_API=3 etcdctl --cert /etc/etcd/peer.crt --key /etc/etcd/peer.key --cacert /etc/etcd/ca.crt --endpoints '192.168.0.201:2379' snapshot save /var/lib/etcd/snapshot.db"

	cp /var/lib/etcd/snapshot.db ~/OCP3_10_BK/
	## finish Backup (skip project, PVC backup)
	## Backup single project
	# oc get -o yaml --export all > project.yaml
	# for object in rolebindings serviceaccounts secrets imagestreamtags podpreset cms egressnetworkpolicies rolebindingrestrictions limitranges resourcequotas pvcs templates cronjobs statefulsets hpas deployments replicasets poddisruptionbudget endpoints
	# do
	#  oc get -o yaml --export $object > $object.yaml
	# done
	# oc api-resources --namespaced=true -o name

	## Backup PVC content
	# using rsync download files to local, ex:
	# oc rsync demo-2-fxx6d:/opt/app-root/src/uploaded ./demo-app
	```
3. Execute following command on `Bastion`.
	```
	ansible nodes -a "subscription-manager repos --disable='rhel-7-server-ose-3.10-rpms' --disable='rhel-7-server-ansible-2.4-rpms' --enable='rhel-7-server-ose-3.11-rpms' --enable='rhel-7-server-rpms' --enable='rhel-7-server-extras-rpms' --enable='rhel-7-server-ansible-2.6-rpms'"
	ansible nodes -a "yum clean all"

	subscription-manager repos --disable='rhel-7-server-ose-3.10-rpms' --disable='rhel-7-server-ansible-2.4-rpms' --enable='rhel-7-server-ose-3.11-rpms' --enable='rhel-7-server-rpms' --enable='rhel-7-server-extras-rpms' --enable='rhel-7-server-ansible-2.6-rpms'
	yum clean all
	yum update -y openshift-ansible
	```
4. **Decision**: enable or disable new monitoring operator in hosts file, if `openshift_cluster_monitoring_operator_install=true`, then define selector key by `openshift_cluster_monitoring_operator_node_selector`, also label node using `oc label node monitoring=true`

5. Ensure `registry.access.redhat.com` to `registry.redhat.io` change on hosts and inventory file.
6. Get Serviceaccount token (a `XXXXX-OOO-secret.yml`) on `registry.redhat.io`
7. Create secret using the token (replace token `file name` and `user` & `password` in the example):
	```
	oc create -f XXXXX-OOO-secret.yaml -n openshift
	docker login -u='XXXXX-OOO-pull-secret' -p="FWNRGKJERLBJKTBKNzflkwngmler......."
	cp -r ~/.docker /var/lib/origin/
	```
8. Copy `~/.docker/config.json` to all OpenShift`node`
9. Restart all nodes
	```
	ansible nodes -a "systemctl restart atomic-openshift-node"
	```

D. Upgrade OpenShift 3.10 to 3.11
------------
Execute following command on `Bastion`.
1. Upgrade control plane first:
	```
	ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/upgrades/v3_11/upgrade_control_plane.yml
	```
2. Verify following items:
* login OpenShift console with user `admin`
* `oc new-app` an example app and test app through route.
* switch to admin console

3. Upgrade infra & worker nodes
	```
	ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/upgrades/v3_11/upgrade_nodes.yml
	```



Troubleshooting
---------------

| Problem | Probable Cause | Possible Solution |
|---------|----------------|-------------------|
| | | |
