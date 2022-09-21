# Overview
A collection of Ansible playbooks for provisioning a bare metal Kubernetes cluster on Hyper-V for testing and learning purposes. The playbooks and add-ons were tailored to my learning environment and is not intended to be an all-purpose "installer" for Kubernetes. Therefore, please feel free to customize it to your needs if you find them useful.

### Pre-requisites
1. A Windows machine with lots of processing power, RAM and disk space.
2. Ansible must be installed - These playbooks were tested on a Windows host with Ubuntu running in WSL. 
3. WinRM must be configured - The Windows host that manages the Hyper-V VMs must be configured with WinRM.
4. A user with proper access rights must be created in the Windows host and WSL.

### Notes
* The Kubernetes cluster is currently setup to use Calico and Containerd.
* Both control-planes only or control-planes with worker nodes configuration are supported.
* Configure the IP addresses, host names and domain to your environment.
* The nfs server is not provisioned currently. You need to manually create it.
* Some playbooks are deploying sample/example configurations only (although some may deploy production environments)
* Installation of add-ons are controlled in the `config-kubernetes.yaml` file under the `kube.install` section.
* Add-ons may conflict with each other (i.e. Kube-Prometheus vs. Metrics Server). Adjust the order of installation if needed.
* Version incompatibility may occur i.e. new Kubernetes version may break everything. Adjust the versions in `global.yaml` if needed.

### Add-ons List
* Ingress Nginx
* Kubernetes Dashboard
* Metrics Server
* Metallb
* NFS Provisioner
* Cert Manager
* Rook Ceph
* Kube Prometheus (monitoring)
* Elasticsearch-Fluentd-Kibana (logging)
* Harbor
* Istio
* Ansible Tower (AWX)

### Advice
You are advised not to be so ambitious to run `setup-all.yaml` on your first try. Start with these in the following order:
1. setup-server.yaml
2. setup-kubernetes.yaml

or you can run the playbooks individually to setup the add-ons you want. The installation order is important as some add-ons have dependencies on the others.
