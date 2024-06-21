# Kubernetes Lab on Hyper-V
A collection of Ansible playbooks and roles to provision a bare-metal Kubernetes cluster for on-premise Hyper-V, either for testing/learning purposes or SIT/Proof-of-Concept environments. These playbooks and add-ons were tailored to my working environment and were not intended to be an all-purpose "installer" for Kubernetes. Therefore, please feel free to customize them as you see fit.

## Quick Notes
* Playbooks are tested on **Windows 11** and **Windows Server 2022** Hyper-V hosts.
* Playbooks can provision VMs for **Ubuntu 22.04** or **Red Hat 9.3** Linux.
* Playbooks default to setup a DNS server, 2 HAProxy load balancers and 3 control-plane VMs.
* An infrastructure service VM is dedicated to host DNS, NFS and Minio for testing purposes. NFS and Minio will not provision VMs by default.
* The Kubernetes cluster uses **Containerd** as container runtime and **Calico** as CNI.
* Configure the IP addresses, host names and domain in the host files located in `\inventories`
* Some roles are deploying sample/example configurations only (although some may deploy production environments).
* Add-ons may conflict with each other (i.e. Kube-Prometheus vs. Metrics Server). Adjust the order of installation if needed.
* Version incompatibility may occur i.e. new Kubernetes version may break everything or some add-ons version may not be compatible with each other. Configure the desired versions in the `vars\main.yaml` of the role.
* If an add-on fails to install, try using an older release of the add-on since the playbooks may not be compatible with the latest releases.
* Be patient if you are running on a slow internet connection. Installation may take some time. Increase the timeout of tasks if your environment needs more time to complete certain tasks.
* VM Checkpoints will be created to provide safety just in case your installation breaks. These checkpoints may consume a lot of disk space. You may remove the checkpoints when you feel everything is stable to recover the disk space.

## Hyper-V Host Requirements

A Windows machine or server with sufficient processing power, RAM and disk space (i7 CPU, 128GB RAM, SSD recommended) with:
* Hyper-V feature enabled.
* [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install) installed.
* Ubuntu 22.04 distribution installed in WSL.

Prepare two folders on the Windows Hyper-V host for the playbooks:
* A folder for storing temporary seed iso images i.e. `"D:\Installation Files"`
* A folder where the Virtual Machines will be created in i.e. `"D:\Virtual Machines"`

> [!CAUTION]
> You can set both to point to the same folder but you may have some residue seed iso images leftover from provisioning errors. In such cases, you may need to manually clean them up yourself.

### Virtual Machine Requirements

The computing resources for each VM is depending on how much add-ons you are planning to install and how much capacity your Windows host can provide. Remember that you can pool multiple Windows Hyper-V servers together to host the VMs to distribute the load.

> [!NOTE]
> The VMs created are using **Dynamic Memory** as the default. You can change this in **Hyper-V Manager** if you want to prevent the VMs from exceeding the assigned memory limits.

#### Infrastructure Services VMs

There are additional 3 VMs that are optional to support the Kubernetes Cluster and these are categorized as **Infrastructure Services**. These VMs need to be provisioned and configured first, before setting up the Kubernetes Cluster if you choose to include them in your lab. The Load-Balancers will each have their own VM and a spearate VM is dedicated to host DNS, NFS and Minio for testing purposes in the lab. NFS and Minio will not provision their own VMs by default.

   > [!CAUTION]
   > The Infrastructure Services VMs are purely meant to simulate existing infrastructure in a real-world network topology. While you can use them as starting points to configure a production environment, **do not target them to existing production servers** that are already running. Doing so will corrupt the existing servers.

   The recommendations for the **Infrastructure Services** VMs are as follows:

   | Services                    | Unit | vCPU | Min. RAM  | RAM |
   | --------------------------- | :--: | :--: | :-------: | :-: |
   | Load-Balancers              | 2    | 2    | 1GB       | 1GB |
   | K8s-svc (DNS)               | 1    | 2    | 1GB       | 1GB |
   | K8s-svc (DNS + NFS)         | 1    | 2    | 2GB       | 2GB |
   | K8s-svc (DNS + NFS + Minio) | 1    | 4    | 4GB       | 8GB |

#### Kubernetes Cluster VMs

The **Kubernetes Cluster** will require a minimum of 3 VMs for the control planes. Generally in a lab environment where computing resources is limited, you can skip provisioning the worker nodes. However, if you want to learn Kubernetes concepts or deploy SIT/Proof-of-Concept environments where workload isolation is required for applications, then you can decide the number or worker nodes you need to deploy.

   The recommendations for the **Kubernetes** VMs are as follows:

   | Lab Type                    | Control Plane | vCPU | Min. RAM  | RAM  | Worker Node | vCPU | Min. RAM  | RAM  |
   | --------------------------- | :-----------: | :--: | :-------: | :--: | :---------: | :--: | :-------: | :--: |
   | Barebones Cluster           | 3             | 4    | 4GB       | 6GB  | Optional    | 4    | 4GB       | 4GB  |
   | Basic (NFS)                 | 3             | 4    | 8GB       | 12GB | Optional    | 4    | 4GB       | 4GB  |
   | Basic (Longhorn/Rook-ceph)  | 3             | 6    | 12GB      | 16GB | Optional    | 4    | 4GB       | 4GB  |
   | DevOps/DevSecOps            | 3             | 6    | 16GB      | 24GB | Optional    | 4    | 4GB       | 4GB  |
   | Observablity                | 3             | 6    | 16GB      | 24GB | Optional    | 4    | 4GB       | 4GB  |
   | Test/SIT Environment        | 3             | 8    | 24GB      | 32GB | 2 (or more) | 4    | 4GB       | 8GB  |

   > [!WARNING]
   > Both Longhorn and Rook-Ceph CSIs can consume a lot of resources but offers a good learning path and discipline for managing clustered storage. You may need to utilize more than 1 Hyper-V hosts with more RAM if you have insufficient computing resources.

#### Disk Size

The recommendations for **disk size** is the VM default (**127GB**) but when using CSI such as Longhorn or Rook-ceph, it is recommended to set the disk size to **256GB** to support the request limits.

## Setting Up Your Environment
### 1. Install Ansible

Open a terminal in the Ubuntu OS of your WSL and execute the following command to install ansible.
```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```
> [!NOTE]
> You can refer to the detail documentation [here](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html#installing-ansible-on-ubuntu).

### 2. Configure Windows Remote Management (WinRM) on your Windows host.

Open an Ubuntu terminal in your WSL and run the following:
```bash
sudo apt install python3-pip
pip install ansible pywinrm kubernetes jsonpatch
```

Create a Windows user with Administrator (or proper) access rights for ansible in the Windows Hyper-V host. Open a Powershell command prompt with Administrator rights in your Windows host and run the following:
```powershell
$username = "ansible"
$password = ConvertTo-SecureString "p@ssw0rd" -AsPlainText -Force

New-LocalUser -Name $username -Password $password -FullName $username -Description "Ansible Controller Host Account" -AccountNeverExpires -PasswordNeverExpires -UserMayNotChangePassword

Add-LocalGroupMember -Group Administrators -Member $username
```

Run the configuration script provided by ansible to configure your WinRM in the same powershell terminal:
```powershell
$setupscript = "https://raw.githubusercontent.com/ansible/ansible-documentation/ae8772176a5c645655c91328e93196bcf741732d/examples/scripts/ConfigureRemotingForAnsible.ps1"
Invoke-WebRequest $setupscript -OutFile winrm-setup.ps1
.\winrm-setup.ps1
```
> [!NOTE]
> You can refer to the detail documentation [here](https://docs.ansible.com/ansible/latest/os_guide/windows_setup.html).

> [!IMPORTANT]
> Setting up WinRM is usually the hardest part of the pre-requisites. Make sure you have it configured correctly before you attempt to run the playbooks.

### 3. Configure WinRM Settings in Playbooks

Clone this repository into a directory of your choice in the Ubuntu OS inside the WSL. You may need to configure the necessary access rights for the directory.

```bash
git clone https://github.com/serenagrl/ansible-kubernetes.git
```

Set the neccesary credentials in the `\inventories\winrm.yaml` file:
```yaml
# This should be your local machine or an Ansible host. Winrm must be setup correctly for this to work.
# You can have more than 1 winrm host to distribute the kubernetes VMs; remember to set the winrm_host.
winrm:
  hosts:
    local_machine:
      ansible_host: 192.168.0.118
      # You can use username and password if you do not want to use cert.
      ansible_user: ansible            # Create a user in your Windows
      ansible_password: p@ssw0rd       # Provide the password for the user
      ansible_become_method: runas
      ansible_become_user: ansible     # Must set this to the user you created
      ansible_connection: winrm
      ansible_port: 5986
      ansible_winrm_transport: ntlm
      ansible_winrm_server_cert_validation: ignore
```

> [!NOTE]
> Detail documentation [here](https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html).

### 4. Optional: Configure Playbooks to Use WinRM With Certificates

Although not compulsory, it is recommended to use certificates to connect to WinRM from your playbooks. Follow these [steps](docs/configure-winrm-certs.md), if you wish to enable WinRM using certificates.

## Running the Playbooks with Semaphore UI

If you are familiar with [Semaphore UI](https://github.com/semaphoreui/semaphore), you can leverage on it to provide a user interface on-top of the playbooks in this repository. You can run the `setup-semaphore.yaml` playbook to create and configure a separate VM for Semaphore UI and then run the `setup-semaphore-project.yaml` playbook to automate the creation of a project in the semaphore server based on the cloned version of this repository on your filesystem.

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

## Add-ons stack

<table>
<tr>
  <th><b>Category</b></th>
  <th><b>Add-On Components</b></th>
</tr>
<tr>
  <td rowspan=3>Service Proxy</td>
  <td>Metallb</td>
</tr>
<tr>
  <td>Nginx</td>
</tr>
<tr>
  <td>Contour</td>
</tr>
<tr>
  <td rowspan=3>Container Storage Interface (CSI)</td>
  <td>Longhorn</td>
</tr>
<tr>
  <td>NFS</td>
</tr>
<tr>
  <td>Rook-Ceph</td>
</tr>
<tr>
  <td rowspan=4>Security &amp; Compliance</td>
  <td>Cert Manager</td>
</tr>
 <tr><td>Keycloak</td>
</tr>
<tr>
  <td>Trivy</td>
</tr>
<tr>
  <td>Goldilocks</td>
</tr>
<tr>
  <td rowspan=2>Kubernetes-sig</td>
  <td>Kubernetes Dashboard</td>
</tr>
<tr>
  <td>Metrics Server</td>
</tr>
<tr>
  <td rowspan=3>Monitoring</td>
  <td>Prometheus</td>
</tr>
<tr>
  <td>Grafana</td>
</tr>
<tr>
  <td>AlertManager</td>
</tr>
<tr>
  <td rowspan=3>Logging</td>
  <td>ElasticOperator
  <ul>
    <li>Elasticsearch</li>
    <li>Kibana</li>
  </ul>
  </td>
</tr>
<tr>
  <td>FluentD</td>
</tr>
<tr>
  <td> Grafana Loki (Promtail)</td>
</tr>
<tr>
  <td rowspan=2>Tracing</td>
  <td>Jaeger</td>
</tr>
<tr>
  <td>OpenTelemetry</td>
</tr>
<tr>
  <td>Service Mesh</td>
  <td>Istio</td>
</tr>
<tr>
  <td rowspan=8>DevOps</td>
  <td>ArgoCD</td>
</tr>
<tr>
  <td>Gitlab</td>
</tr>
<tr>
  <td>Gitlab Runner</td>
</tr>
<tr>
  <td>Harbor</td>
</tr>
<tr>
  <td>Kyverno</td>
</tr>
<tr>
  <td>Minio</td>
</tr>
<tr>
  <td>Sonarqube</td>
</tr>
<tr>
  <td>Tekton</td>
</tr>
<tr>
  <td rowspan=2>Messaging</td>
  <td>RabbitMQ
    <ul>
      <li>Operator</li>
      <li>Cluster</li>
      <li>Messaging Topology Operator</li>
    </ul>
  </td>
</tr>
<tr>
  <td>Strimzi Kafka
    <ul>
      <li>Operator</li>
      <li>Kafka Bridge</li>
      <li>Kafka Connect</li>
      <li>Kafka UI</li>
    </ul>
  </td>
</tr>
<tr>
  <td>Caching</td>
  <td>Redis</td>
</tr>
<tr>
  <td>Database</td>
  <td>Microsoft SQL Server</td>
</tr>
<tr>
  <td>Serverless</td>
  <td>KNative
    <ul>
      <li>Eventing</li>
      <li>Serving</li>
    </ul>
  </td>
</tr>
<tr>
  <td>Operations</td>
  <td>Velero</td>
</tr>
<tr>
  <td>Automation</td>
  <td>AWX</td>
</tr>
</table>

## Roles Guide

The roles folders are structured in the following manner:

| Folder         | Description |
| -------------- |----         |
| defaults       | Default values that you should not change for the role. |
| tasks          | The tasks that performs the installation and configuration work. |
| vars           | Variable values that you can change to customize the role. |
| templates      | The jinja2 templates that you can modify to add extra configurations. |
| meta           | Role dependencies that you should not touch. |

To install the specific add-on, specify them in the `add_ons` collection before running `setup-add-ons.yaml`.

## Disclaimer
Everything provided here is 'as-is' and comes with no warranty or support.