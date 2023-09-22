# Ansible Kubernetes Lab on Hyper-V
A collection of amateurish-crafted Ansible playbooks and roles to provision a bare metal Kubernetes cluster on Hyper-V for testing/learning purposes. The playbooks and add-ons were tailored to my lab environment and is not intended to be an all-purpose "installer" for Kubernetes. Therefore, please feel free to customize or enhance it to your needs if you find them useful.

### Pre-requisites
1. **Knowledge of Ansible is required**.
2. A Windows machine with lots of processing power, RAM and disk space.

    Recommended specifications for 3 Control Plane configuration:

    | Hosts          | vCPU | Min. RAM  | Recommended RAM |
    | -------------- |:----:|  :----:   |     :----:      |
    | Control planes | 6    | 12GB      | 16GB            |

    Recommended specifications for 3 Control Planes with Worker Nodes configuration:

    | Hosts          | vCPU | Min. RAM  | Recommended RAM |
    | -------------- |:----:|  :----:   |     :----:      |
    | Control planes | 6    | 12GB      | 16GB            |
    | Worker Nodes   | 4    | 4GB       | 8GB             |

3. **Ansible must be installed** - These playbooks were tested on a Windows host with Ubuntu running in WSL.
   Note: You can run on a Windows 11 or Windows Server 2022 with Hyper-V enabled.
4. **Windows Remote Management (WinRM) must be configured** - The Windows host that manages the Hyper-V VMs must be configured with WinRM.
5. **A user with proper access rights** must be created in the **Windows host and WSL**.
6. Some useful links:
   - [Install Linux on Windows with WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
   - [Installing Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
   - [Setting up Ansible on a Windows Host](https://docs.ansible.com/ansible/latest/os_guide/windows_setup.html)
   - [Using WinRM in Ansible Playbooks](https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html)
     
    **Important!**: Setting up WinRM is usually the hardest part of the pre-requisites. Make sure you have it configured correctly before you attempt to run the playbooks.

### Notes
* The Kubernetes cluster is currently setup to use Calico and Containerd.
* Both control-planes only or control-planes with worker nodes configuration are supported.
* Playbooks are defaulted to setup 2 HAProxy load balancers and 3 control-planes. 
* Configure the IP addresses, host names and domain to your environment in the `\inventories\hosts.yaml`
* The nfs server is configured to not provision currently. You need to manually enable it.
* The playbook for creating 2 nodes with HAProxy and keepalived for load-balancing is available.
* Some roles are deploying sample/example configurations only (although some may deploy production environments)
* Add-ons may conflict with each other (i.e. Kube-Prometheus vs. Metrics Server). Adjust the order of installation if needed.
* Version incompatibility may occur i.e. new Kubernetes version may break everything or some add-ons version may not be compatible with each other. Configure the desired versions in the `vars\main.yaml` of the role.
* If a component fail to install, try using an older release of the component since the playbooks may not be compatible with the latest releases.
* Be patient if you are running on a slow internet connection. Installation may take some time. Increase the timeout of tasks if your environment needs more time to complete certain tasks.
* VM Checkpoints will be created to provide safety just in case your installation breaks. These checkpoint may consume a lot of disk space. You may remove the checkpoints when you feel everything is stable to recover disk space.

### Playbooks

* The following is a list of playbooks that you can run:

    | Name | Description |
    | ---- | ---- |
    | `setup-load-balancers.yaml` | Provisions the VMs and configure HAProxy and Keepalived. |
    | `setup-nfs.yaml` | Installs NFS server on a designated server. Optionally, provisions a VM for it. |
    | `setup-control-plane-servers.yaml` | Provisions VMs for the kubernetes control planes. |
    | `setup-control-plane-primary.yaml` | Creates the primary control plane in the kubernetes cluster. |
    | `setup-control-plane-other.yaml` | Creates and joins secondary control planes to the kubernetes cluster. |
    | `setup-control-planes.yaml` | Creates a kubernetes cluster with both primary and secondary control planes at-one-go. |
    | `setup-worker-node-servers.yaml` | Provisions VMs for the kubernetes worker nodes. |
    | `setup-worker-nodes.yaml` | Creates/Joins worker nodes into the kubernetes cluster. |
    | `setup-all.yaml` | Sets up all kubernetes cluster VMs, creates kubernetes cluster, configures nfs server and install add-on components; all at-one-go. |
    | `setup-components.yaml` | Install add-on components listed in `_kube.add_ons` in the `group_vars/all.yaml`. |
    | `setup-kubernetes.yaml` | Creates a kubernetes cluster with all control planes and worker nodes at-one-go. |
    | `setup-semaphore.yaml` | Installs Ansible Semaphore on a designated server. Optionally, provisions a VM for it. |
    | `setup-semaphore-project.yaml` | Creates a new Semaphore project based on this repository. |

#### Advice
You are advised not to be so ambitious to run `setup-all.yaml` on your first try. Start with these in the following order:
1. `setup-load-balancers.yaml` - (Optional) Provision the VMs and sets up a pair of HAProxy load balancers. 
2. `setup-server.yaml` - Provisions the Ubuntu VMs on Hyper-V.
3. `setup-nfs.yaml` - Sets up a basic nfs server.
4. Set `_kube.register_to_load_balancer: no` and `_kube.cluster.address` to your cp1's IP address if you skipped the creation of load balancers earlier. 
5. `setup-kubernetes.yaml` - Sets up the Kubernetes cluster without any add-ons. 
6. Specify the addons to install in the `_kube.add_ons` collection in `group_vars\all.yaml`.
7. `setup-components.yaml` to install the add-ons that you want.

Note: You may run the more granula playbooks if you want to setup things slowly and steadily.

You can rerun the `setup-components.yaml` after you have modified the `_kube.add_ons` collection to install the roles you want. The installation order is important as some add-ons have dependencies on the others. 

### Add-ons stack
* Metallb
* Ingress
  * Nginx
  * Contour
* Cert Manager
* Kubernetes Dashboard
* Metrics Server
* Container Storage Interface (CSI)
  * NFS Provisioner
  * Rook Ceph
* Kube Prometheus (monitoring)
  * Prometheus
  * Grafana
  * AlertManager
* EFK (logging)
  * Elastic Operator
  * Elasticsearch
  * Kibana
  * FluentD
* DevOps
  * ArgoCD
  * Gitlab
  * Gitlab Runner
  * Harbor
  * Minio
  * Sonarqube
* Istio
* Ansible Tower (AWX)
* RabbitMQ
  * Rabbitmq Operator
  * Rabbitmq Cluster
* Redis  
* Database
  * Microsoft SQL Server 
* Velero

### Roles Guide

The roles folders are structured in the following manner:

| Folder         | Description |
| -------------- |----         |
| defaults       | Default values that you should not change for the role. |
| tasks          | The tasks that performs the installation and configuration work. |
| vars           | Variable values that you can change to customize the role. |
| templates      | The jinja2 templates that you can modify to add extra configurations. |
| meta           | Role dependencies that you should not touch. |

To install the specific add-on, specify them in the `_kube.add_ons` collection in `group_vars\all.yaml` before running `setup-components.yaml`. 

### Disclaimer
Everything provided here is 'as-is' and with no warranty or support. 
