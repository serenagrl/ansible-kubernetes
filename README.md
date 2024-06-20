# Kubernetes Lab on Hyper-V
A collection of Ansible playbooks and roles to provision a bare-metal Kubernetes cluster on Hyper-V, either for testing/learning purposes or SIT environments. These playbooks and add-ons were tailored to my working environment and were not intended to be an all-purpose "installer" for Kubernetes. Therefore, please feel free to customize them as you see fit.

## Quick Notes
* Playbooks are tested on **Ubuntu 22.04** and **Red Hat 9.3** Linux.
* Playbooks are defaulted to setup a DNS server, 2 HAProxy load balancers and 3 control-plane VMs.
* The Kubernetes cluster uses **Containerd** as container runtime and **Calico** as CNI.
* Configure the IP addresses, host names and domain to your environment in the host files located in `\inventories`
* An infrastructure service VM is dedicated to host DNS, NFS and Minio for testing purposes. NFS and Minio will not provision VMs by default.
* Some roles are deploying sample/example configurations only (although some may deploy production environments).
* Add-ons may conflict with each other (i.e. Kube-Prometheus vs. Metrics Server). Adjust the order of installation if needed.
* Version incompatibility may occur i.e. new Kubernetes version may break everything or some add-ons version may not be compatible with each other. Configure the desired versions in the `vars\main.yaml` of the role.
* If an add-on fails to install, try using an older release of the add-on since the playbooks may not be compatible with the latest releases.
* Be patient if you are running on a slow internet connection. Installation may take some time. Increase the timeout of tasks if your environment needs more time to complete certain tasks.
* VM Checkpoints will be created to provide safety just in case your installation breaks. These checkpoints may consume a lot of disk space. You may remove the checkpoints when you feel everything is stable to recover the disk space.

## Host and Virtual Machine Requirements
### 1. Host Requirements

A Windows machine with sufficient processing power, RAM and disk space (i7 CPU, 128GB RAM, SSD recommended) with the Hyper-V feature enabled and [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install) installed. These playbooks have been tested on Windows 11 and Windows Server 2022 hosts with Ubuntu 22.04 running in WSL.

Two folders are expected to be available on the Windows Hyper-V host:
* A folder where the playbooks will place temporary iso images i.e. `Installation Files`
* A folder where the Virtual Machines will be created in i.e. `Virtual Machines` 

### 2. Virtual Machine Requirements

The computing resources for each VM is depending on how much add-ons you are planning to install and how much capacity your Windows host can provide. Remember that you can pool multiple Windows Hyper-V servers together to host the VMs to distribute the load.

a. The recommendations for the **Kubernetes** VMs are as follows:

  | Lab Type                    | Control Plane | vCPU | Min. RAM  | Recommended RAM | Worker Node | vCPU | Min. RAM  | Recommended RAM |
  | --------------------------- | :-----------: | :--: | :-------: | :-------------: | :---------: | :--: | :-------: | :-------------: |
  | Barebones - Cluster Only    | 3             | 4    | 4GB       | 6GB             | Optional    | 4    | 4GB       | 4GB             |
  | Basic (NFS)                 | 3             | 4    | 8GB       | 12GB            | Optional    | 4    | 4GB       | 4GB             |
  | Basic (Longhorn/Rook-ceph)  | 3             | 6    | 12GB      | 16GB            | Optional    | 4    | 4GB       | 4GB             |
  | DevOps/DevSecOps            | 3             | 6    | 16GB      | 24GB            | Optional    | 4    | 4GB       | 4GB             |
  | Test/SIT Environment        | 3             | 8    | 24GB      | 32GB            | 2           | 4    | 4GB       | 8GB             |

  > [!NOTE]
  > Generally, worker nodes are only provisioned for learning kubernetes concepts or in SIT environments where workload isolation is required for applications.

  > [!WARNING]
  > Both Longhorn and Rook-Ceph CSIs can consume a lot of resources but offers a good learning path and discipline for managing clustered storage. You may need to utilize more than 1 Hyper-V hosts with more RAM if you ran out of resources.


b. The recommendations for the **Infrastructure Services** VMs are as follows:

  | Services                    | Unit | vCPU | Min. RAM  | Recommended RAM |
  | --------------------------- | :--: | :--: | :-------: | :-------------: |
  | Load-Balancers              | 2    | 2    | 1GB       | 1GB             |
  | K8s-svc (DNS)               | 1    | 2    | 1GB       | 1GB             |
  | K8s-svc (DNS + NFS)         | 1    | 2    | 2GB       | 2GB             |
  | K8s-svc (DNS + NFS + Minio) | 1    | 4    | 4GB       | 8GB             |

c. The recommendations for **disk size** is the VM default (127GB) but when using CSI such as Longhorn or Rook-ceph, it is recommended to set the disk size to 256GB.

## Setting Up Your Enviroment
#### 1. Install Ansible on the Ubuntu OS in your WSL.
```
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```
> [!NOTE]
> You can refer to the detail documentation [here](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html#installing-ansible-on-ubuntu).

#### 2. Configure Windows Remote Management (WinRM) on your Windows host.

2.1. Open an Ubuntu terminal in your WSL and run the following:
```
sudo apt install python3-pip
pip install ansible pywinrm kubernetes jsonpatch
```

2.2. Create a Windows user with Administrator (or proper) access rights for ansible in the Windows Hyper-V host. Open a Powershell command prompt with Administrator rights in your Windows host and run the following:
```
$username = "ansible"
$password = ConvertTo-SecureString "p@ssw0rd" -AsPlainText -Force

New-LocalUser -Name $username -Password $password -FullName $username -Description "Ansible Controller Host Account" -AccountNeverExpires -PasswordNeverExpires -UserMayNotChangePassword

Add-LocalGroupMember -Group Administrators -Member $username
```

2.3. Run the configuration script provided by ansible to configure your WinRM in the same powershell terminal:
```
$setupscript = "https://raw.githubusercontent.com/ansible/ansible-documentation/ae8772176a5c645655c91328e93196bcf741732d/examples/scripts/ConfigureRemotingForAnsible.ps1"
Invoke-WebRequest $setupscript -OutFile winrm-setup.ps1
.\winrm-setup.ps1
```
> [!NOTE]
> You can refer to the detail documentation [here](https://docs.ansible.com/ansible/latest/os_guide/windows_setup.html).

> [!IMPORTANT]
> Setting up WinRM is usually the hardest part of the pre-requisites. Make sure you have it configured correctly before you attempt to run the playbooks.

#### 3. Clone this repository into a directory of your choice in the WSL and [configure WinRM access](https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html) in the `\inventories\winrm.yaml` file. You may need to configure the necessary access rights for the directory.

```
git clone https://github.com/serenagrl/ansible-kubernetes.git
```

#### 4. Optional: Follow these [steps](docs/configure-winrm-certs.md), if you wish to connect to WinRM using certificates.

## Running the Playbooks with Semaphore UI

You can leverage on Semaphore UI to provide a User Interface on-top of the playbooks in this repository. You can run the `setup-semaphore.yaml` playbook to create and configure a separate VM for Semaphore UI and then run the `setup-semaphore-project.yaml` playbook to automate the creation of a project in the semaphore server based on the existing cloned repository on your filesystem.

This option is best for beginners who do not want to go through the internals and complexities of configuring and running the playbooks from a terminal.

To support Semaphore UI, you will need to create shared folders for the `Installation Files` and `Virtual Machines` folders on your Windows Hyper-V Host and grant full-control access to the `ansible` user which you have created in the earlier section.

## Running the Playbooks via Command Line

The playbooks can be run with the `ansible-playbook` command. i.e.
```
ansible-playbook setup-load-balancers.yaml
```
a. Run the following list of optional **infrastructure playbooks** in sequence:

| No. | Name                        | Description                                                 | Remarks |
|  :-:| ----------------------------| ----------------------------------------------------------- | ------- |
|  1. | `setup-dns.yaml`            | Provision and configure a VM for BIND (DNS) server.         | WARNING! Do not set host to existing DNS server. <br> Additional configuration required: <br> a. Set `kube.cluster.use_dns_server:` to `yes` or `no` in `inventories/group_vars/all.yaml`. |
|  2. | `setup-load-balancers.yaml` | Provision and configure two VMs for HAProxy and Keepalived. | Additional configuration required: <br> a. When deploying load-balancers: <br>&nbsp;&nbsp;&nbsp; * Set `register_to_load_balancer: yes` in `inventories/group_vars/kubernetes_cluster.yaml`. <br>&nbsp;&nbsp;&nbsp; * Set `kube.cluster.address` to point to the virtual ip address in `inventories/group_vars/all.yaml`. <br> When not deploying load-balancers: <br>&nbsp;&nbsp;&nbsp; * Set `register_to_load_balancer: no` in `inventories/group_vars/kubernetes_cluster.yaml`. <br>&nbsp;&nbsp;&nbsp; * Set `kube.cluster.address` to point to the ip address of primary control plane in `inventories/group_vars/all.yaml`. |
|  3. | `setup-nfs.yaml`            | Install NFS server on infrastructure service VM.            | Run this if you plan to use CSI NFS. Note: You must run this **before** running the csi/nfs role. |
|  4. | `setup-minio.yaml`          | Install Minio on infrastructure service VM.                 | Run this if you plan to test add-ons that requires external S3 storage for backups i.e. velero or csi/longhorn |

b. Run the following list of **kubernetes playbooks** in sequence:

| No. | Name                               | Description                                                         | Remarks |
|  :-:| ---------------------------------- | ------------------------------------------------------------------- | ------- |
|  1. | `setup-control-plane-servers.yaml` | Provision VMs for kubernetes control planes. ||
|  2. | `setup-control-plane-primary.yaml` | Create primary control plane in the kubernetes cluster. ||
|  3. | `setup-control-plane-other.yaml`   | Create and join secondary control planes to the kubernetes cluster. ||
|  4. | `setup-worker-node-servers.yaml`   | Provisions VMs for the kubernetes worker nodes. | Run this if you want to create worker nodes. |
|  5. | `setup-worker-nodes.yaml`          | Create and join worker nodes into the kubernetes cluster. | Run this if you want to create worker nodes. |
|  6. | `setup-add-ons.yaml`               | Install add-on components listed in `add_ons`. | You can run this each time you want to install a particular add-on but keep an eye on the checkpoints. The installation order is important as some add-ons have dependencies on the others. |

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
  * Longhorn
  * Rook Ceph
* Monitoring (Kube-Prometheus)
  * Prometheus
  * Grafana
  * AlertManager
* Logging
  * Elastic Operator
  * Elasticsearch
  * Kibana
  * FluentD
  * Loki (with Promtail)
* Tracing
  * Jaeger
  * OpenTelemetry Collector
* DevOps
  * ArgoCD
  * Gitlab
  * Gitlab Runner
  * Harbor
  * Minio
  * Sonarqube
  * Kyverno
* Istio
* KNative
* RabbitMQ
  * Rabbitmq Operator
  * Rabbitmq Cluster
  * Messaging Topology Operator
* Redis
  * Redis Cluster
  * Redis Sentinel
* Kafka
  * Strimzi Kafka Operator
  * Kafka Bridge
  * Kafka Connect
* Database
  * Microsoft SQL Server
* Trivy
* Keycloak
* Goldilocks
* Velero
* Ansible Tower (AWX)

### Roles Guide

The roles folders are structured in the following manner:

| Folder         | Description |
| -------------- |----         |
| defaults       | Default values that you should not change for the role. |
| tasks          | The tasks that performs the installation and configuration work. |
| vars           | Variable values that you can change to customize the role. |
| templates      | The jinja2 templates that you can modify to add extra configurations. |
| meta           | Role dependencies that you should not touch. |

To install the specific add-on, specify them in the `add_ons` collection before running `setup-add-ons.yaml`.

### Disclaimer
Everything provided here is 'as-is' and comes with no warranty or support.
