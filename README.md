# rpi-k8s-ansible
Raspberry PI's running Kubernetes deployed with Ansible

## Assumptions

1.  there are 19 raspberry pi computers with name of
    1.  ```rpi-cluster-master.local```
    2.  ```rpi-cluster-[1:18].local```
2.  your rpis are on their own network, with address space being 10.1.1.[100:118]
3.  all rpis are using ```eth0``` for their connectivity.
4.  you have installed ```ansible```

# Verify Ansible and Host Connectivity

From within the cloned repository folder, try the following command:

```
ansible -i cluster.yml all -m ping
```

You should get output like the following:
```
rpi-cluster-3.local | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
rpi-cluster-1.local | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
rpi-cluster-2.local | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

...

rpi-cluster-16.local | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
rpi-cluster-17.local | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
rpi-cluster-18.local | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

```

# Install Kubernetes
## apt-get upgrade
```
ansible-playbook -i cluster.yml playbooks/upgrade.yml
```


## Install k8s
With the below commands, you need to include the master node (node00) in all executions for the token to be set correctly.
```
# Bootstrap the master and all slaves
ansible-playbook -i cluster.yml site.yml

# Bootstrap a single slave (rpi-cluster-5)
ansible-playbook -i cluster.yml site.yml -l rpi-cluster-master.local,rpi-cluster-5.local

# When running again, feel free to ignore the common tag as this will reboot the rpi's
ansible-playbook -i cluster.yml site.yml --skip-tags common
```

## Copy over the .kube/config
This logs into rpi-cluster-master and copies the .kube/config file back into the local users ~/.kube/config file
Allowing a locally installed kubectl/etc to be able to query the cluster

```
# Copy in your kube config
ansible-playbook -i cluster.yml playbooks/copy-kube-config.yml

# Set an alias to make it easier
alias kubectl='docker run -it --rm -v ~/.kube:/.kube -v $(pwd):/pwd -w /pwd bitnami/kubectl:1.21.3'

# Run kubectl within docker
kubectl version
```

# Running things on the cluster!
You can look in the README.md in the kubernetes subfolder to see a few examples of things to run!

# Extra misc commands
```
# Shutdown all nodes
ansible -i cluster.yml -a "shutdown -h now" all

# Ensure NFS mount is active across cluster
ansible -i cluster.yml -a "mount -a" all
```

# Upgrading your cluster
First you need to upgrade the control plane (node00).
The below example is an upgrade from v1.15.0 -> v1.15.1

## Install the target kubeadm
```
sudo apt-get install kubeadm=1.15.1-00
```

## Plan the upgrade
```
sudo kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.15.0
[upgrade/versions] kubeadm version: v1.15.1
[upgrade/versions] Latest stable version: v1.15.1
[upgrade/versions] Latest version in the v1.15 series: v1.15.1

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     7 x v1.15.0   v1.15.1

Upgrade to the latest version in the v1.15 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.15.0   v1.15.1
Controller Manager   v1.15.0   v1.15.1
Scheduler            v1.15.0   v1.15.1
Kube Proxy           v1.15.0   v1.15.1
CoreDNS              1.3.1     1.3.1
Etcd                 3.3.10    3.3.10

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.15.1

_____________________________________________________________________
```

## Apply the upgrade
```
sudo kubeadm upgrade apply v1.15.1
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/version] You have chosen to change the cluster version to "v1.15.1"
[upgrade/versions] Cluster version: v1.15.0
[upgrade/versions] kubeadm version: v1.15.1
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y

...

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.15.1". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

## Update kube_version
```
cat roles/kubernetes/defaults/main.yml
---
kube_version: "1.15.1-00"

<snip>
```

## Upgrade the kubeXYZ tooling
```
ansible-playbook -i cluster.yml site.yml --tags upgrade,kubernetes
```

----

## Final Thoughts

Setting the correct IP address of the master node on each slave node is incorrect. You will not have this problem if you are only are using one network adapter.  The way I have my cluster setup is to use the ethernet port for Power over Ethernet because the cluster is on its own wired network.  The master node acts as a *jump server*; eth0 is on the cluster's network while wlan0 is on the regular wifi network.  Using the tasks in this reposity, when setting up slave nodes, the address of wlan0 gets set on each slave instead of using the ```eth0``` address of the ```master``` node.

Second item to discuss: even though there is a task to set ```cgroup memory``` parameters for the kernel on each node (including ```master```), for some reason, the change did not work on several nodes.  I was forced to manually log into failing nodes and verify or change the kernel boot parameters.

## Acknowledgements

* The entirety of this repository is based on Michael Robbins' [rpi-k8s-ansible](https://github.com/michael-robbins/rpi-k8s-ansible) repository.

### Copyright
Original: Copyright &copy; Michael Robbins 2017-2022
Modifications: Copyright &copy; A.C. Jokela 2022 

