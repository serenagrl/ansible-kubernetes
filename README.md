# Ansible Kubernetes Lab on Hyper-V
A collection of amateurish-crafted Ansible playbooks and roles to provision a bare metal Kubernetes cluster on Hyper-V for testing/learning purposes. The playbooks and add-ons were tailored to my lab environment and is not intended to be an all-purpose "installer" for Kubernetes. Therefore, please feel free to customize or enhance it to your needs if you find them useful.

### Pre-requisites
1. **Knowledge of Ansible is required**.
2. A Windows machine with lots of processing power, RAM and disk space.

    Recommended specifications for 3 Control Plane configuration:

    | Hosts          | vCPU | Min. RAM  | Recommended RAM |
    | -------------- |:----:|  :----:   |     :----:      |
    | Control planes | 4    | 12GB      | 16GB            |

    Recommended specifications for Control Plane with Worker Nodes configuration:

    | Hosts          | vCPU | Min. RAM  | Recommended RAM |
    | -------------- |:----:|  :----:   |     :----:      |
    | Control planes | 4    | 8GB       | 12GB            |
    | Worker Nodes   | 2    | 8GB       | 12GB            |

3. **Ansible must be installed** - These playbooks were tested on a Windows host with Ubuntu running in WSL.

    Note: You can run on a Windows 11 or Windows Server 2022 with Hyper-V enabled. 
4. **WinRM must be configured** - The Windows host that manages the Hyper-V VMs must be configured with WinRM.
5. **A user with proper access rights** must be created in the **Windows host and WSL**.

**Important!**: Setting up WinRM is usually the hardest part of the pre-requisites. Make sure you have it configured correctly before you attempt to run the playbooks.

### Notes
* The Kubernetes cluster is currently setup to use Calico and Containerd.
* Both control-planes only or control-planes with worker nodes configuration are supported.
* Configure the IP addresses, host names and domain to your environment in the `\inventories\hosts.yaml`
* The nfs server is not provisioned currently. You need to manually create it.
* Some roles are deploying sample/example configurations only (although some may deploy production environments)
* Add-ons may conflict with each other (i.e. Kube-Prometheus vs. Metrics Server). Adjust the order of installation if needed.
* Version incompatibility may occur i.e. new Kubernetes version may break everything or some add-ons version may not be compatible with each other. Configure the desired versions in the `vars\main.yaml` of the role.
* Be patient if you are running on a slow internet connection. Installation make take some time.
* VM Checkpoints will be created to provide safety just in case your installation breaks. These checkpoint may consume a lot of disk space. You may remove the checkpoints when you feel everything is stable.

### Advice
You are advised not to be so ambitious to run `setup-all.yaml` on your first try. Start with these in the following order:
1. `setup-server.yaml` - Provisions the Ubuntu VMs on Hyper-V.
2. `setup-kubernetes.yaml` - Sets up the Kubernetes cluster without any add-ons.
3. ... and then specify the roles in the `setup-components.yaml`.

You can rerun the `setup-components.yaml` after you have modified the roles to setup the add-ons you want. The installation order is important as some add-ons have dependencies on the others. Remember to comment out the NFS configuration if you are running it the second time.

### Add-ons stack
* Metallb
* Ingress Nginx or Contour
* Cert Manager
* Kubernetes Dashboard
* Metrics Server
* NFS Provisioner
* Rook Ceph
* Kube Prometheus (monitoring) - includes Prometheus, Grafana and AlertManager
* Elasticsearch-Fluentd-Kibana (logging)
* Gitlab - includes minio
* Gitlab Runner
* Sonarqube
* Harbor
* ArgoCD
* Istio
* Ansible Tower (AWX)
* RabbitMQ Cluster

### Roles Guide

The roles folders are structured in the following manner:

| Folder         | Description |
| -------------- |----         |
| defaults       | Default values that you should not change for the role. |
| tasks          | The tasks that performs the installation and configuration work. |
| vars           | Variable values that you can change to customize the role. |
| templates      | The jinja2 templates that you can modify to add extra configurations. |
| meta           | Role dependencies that you should not touch. |

To install the specific add-on, specify them in the `setup-components.yaml`. Remember to comment out the NFS configuration if you are running it the second time.

### Disclaimer
Everything provided here is 'as-is' and with no warranty or support. 
