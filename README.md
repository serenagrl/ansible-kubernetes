# Kubernetes Lab on Hyper-V
A collection of Ansible playbooks and roles to create a Kubernetes cluster on Hyper-V, either for testing/learning purposes or SIT/Proof-of-Concept environments. These playbooks and add-ons were tailored to my lab environment and were not intended to be an all-purpose "installer" for Kubernetes. Therefore, please feel free to customize them as you see fit.

## Quick Notes
* Working knowledge of [Ansible](https://www.ansible.com/) is required to navigate around the playbooks.
* Playbooks are tested on **Windows 11** and **Windows Server 2022** Hyper-V hosts.
* Playbooks can provision VMs for **Ubuntu 22.04/24.04** or **Red Hat 9.3/9.4** Linux.
* Playbooks default to setup a DNS, 2 HAProxy load balancers and 3 control-plane VMs.
* The Kubernetes cluster uses **Containerd** as container runtime and **Calico** as CNI.
* Configure everything in the `/inventories` folder and `roles/vm-linux/setup-vm/vars/main.yaml`.

## Hyper-V Host Requirements

A Windows machine or server with sufficient processing power, RAM and disk space (i7 CPU, 128GB RAM, SSD recommended) with:
* Hyper-V feature enabled.
* [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install) installed.
* Ubuntu 22.04 distribution installed in WSL.

Open an Ubuntu terminal in the WSL and edit the `/etc/wsl.conf` file to configure Ubuntu WSL to always login as root user.
```
[boot]
systemd=true

[user]
default = root
```

> [!NOTE]
> You need to reload the terminal for the changes to take effect.

Prepare two folders on the Windows Hyper-V host for the playbooks:
* A folder for storing temporary seed iso images i.e. `"D:\Installation Files"`
* A folder where the Virtual Machines will be created in i.e. `"D:\Virtual Machines"`

> [!CAUTION]
> You can set both to point to the same folder but you may have some residue seed iso images leftover from provisioning errors. In such cases, you may need to manually clean them up.

### Virtual Machine Requirements

The required computing resources for each VM is dependent on the number of add-ons you are planning to install and how much capacity your Windows host can provide. By default, the VMs created are using **Dynamic Memory**. You can change this behavior in **Hyper-V Manager** to prevent the VMs from exceeding the assigned memory limits.

> [!TIP]
> You can pool multiple Windows Hyper-V servers together to host the VMs in order to distribute the load but make sure each Hyper-V host is configured correctly.

> [!CAUTION]
> VM Checkpoints are created to provide recovery in case of installation failures. **These checkpoints consume additional disk space**. Please perform the necessary housekeeping manually to reclaim the disk space or disable checkpoint creation by setting `vm_checkpoint: no` in the `inventories/group_vars/all.yaml` inventory file.

#### Infrastructure Services VMs

There are 3 VMs that are optional to support the Kubernetes Cluster and these are categorized as **Infrastructure Services**. These VMs need to be provisioned and configured first, before setting up the Kubernetes Cluster (if you choose to include them). The Load-Balancers will each have their own VM and a separate VM is dedicated to host DNS, NFS and Minio.

> [!WARNING]
> By default, the Infrastructure Services VM will only be created during the provisioning of DNS. NFS and Minio will not provision their own VMs by default. Configure the `provision_vm` variable for each DNS, NFS and Minio to change this behavior.

> [!CAUTION]
> The Infrastructure Services VMs are purely meant to simulate existing infrastructure in a real-world network topology. While you can use them as starting points to configure a production environment, **do not target them to existing production servers** that are already running. **Doing so will corrupt the existing servers.**

Recommendations for **Infrastructure Services** VMs are as follows:

| Services                    | Unit | vCPU | Min. RAM  | RAM | Disk  |
| --------------------------- | :--: | :--: | :-------: | :-: | :--:  |
| Load-Balancers              | 2    | 2    | 1GB       | 1GB | 127GB |
| K8s-svc (DNS)               | 1    | 2    | 1GB       | 1GB | 127GB |
| K8s-svc (DNS + NFS)         | 1    | 2    | 2GB       | 2GB | 127GB |
| K8s-svc (DNS + NFS + Minio) | 1    | 4    | 4GB       | 8GB | 256GB |

#### Kubernetes Cluster VMs

The **Kubernetes Cluster** requires a minimum of 3 VMs for the control planes. Worker nodes are optional - you can determine the number and size of worker nodes base on your needs.

Recommendations for **Kubernetes Cluster** VMs are as follows:

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

The recommendation for **disk size** is the VM default of **127GB** but when using CSIs such as Longhorn or Rook-ceph, it is recommended to set the disk size to **256GB** to support the pod request limits.

## Setting Up Your Environment

It is recommended that you download and install [Visual Studio Code](https://code.visualstudio.com/) and enable the [Ansible VS Code Extension by Red Hat](https://marketplace.visualstudio.com/items?itemName=redhat.ansible) from the marketplace to work with the playbooks.

### 1. Install Ansible

Open a terminal **(with root access)** in the Ubuntu OS of the WSL and execute the following command to install Ansible:
```bash
apt update
apt install software-properties-common python3-pip
pip install ansible jmespath
```

> [!NOTE]
> Refer [here](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#pip-install) for more details.

> [!TIP]
> For Ubuntu 24.04, add the `--break-system-packages` option to the `pip` commands.

### 2. Configure Windows Remote Management (WinRM) on Windows Host

Open an Ubuntu terminal **(with root access)** in the WSL and run the following to install the pre-requisites:
```bash
pip install pywinrm
```

Create a Windows user with Administrator (or proper) priviledges for ansible in the Windows Hyper-V host. You can use the sample script below in a Powershell command prompt with Administrator rights to create the ansible user but please change the password accordingly.
```powershell
$username = "ansible"
$password = ConvertTo-SecureString "p@ssw0rd" -AsPlainText -Force

New-LocalUser -Name $username -Password $password -FullName $username -Description "Ansible Controller Host Account" -AccountNeverExpires -PasswordNeverExpires -UserMayNotChangePassword

Add-LocalGroupMember -Group Administrators -Member $username
```

Run the configuration script provided by ansible to configure WinRM in the same powershell terminal:
```powershell
$setupscript = "https://raw.githubusercontent.com/ansible/ansible-documentation/ae8772176a5c645655c91328e93196bcf741732d/examples/scripts/ConfigureRemotingForAnsible.ps1"
Invoke-WebRequest $setupscript -OutFile winrm-setup.ps1
.\winrm-setup.ps1
```
> [!NOTE]
> Refer [here](https://docs.ansible.com/ansible/latest/os_guide/windows_setup.html) for detail documentation.

### 3. Configure WinRM Settings in Playbooks

Clone this repository to a directory in the Ubuntu OS inside the WSL. Necessary access rights may need to be configured for the directory.

```bash
git clone https://github.com/serenagrl/ansible-kubernetes.git
```

In the `/inventories/winrm.yaml` file, set the WinRM credentials that you have created earlier:
```yaml
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
> Refer [here](https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html) for detail documentation.

### 4. Optional: Configure Playbooks to Use WinRM With Certificates

Using certificates to connect to WinRM from the playbooks is recommended. Follow these [steps](docs/configure-winrm-certs.md) if you wish to apply this practice.

### 5. Testing WinRM Connectivity

Run the `test-winrm-connection.yaml` playbook to verify the WinRM connection.
```bash
ansible-playbook test-winrm-connection.yaml
```

You should get something similar to the following results if successful:
```bash
PLAY [Test WinRM Connection] *******************************************************************************************

TASK [Sending echo to WinRM host] **************************************************************************************
Wednesday 26 June 2024  17:17:22 +0800 (0:00:00.013)       0:00:00.013 ********
changed: [local_machine]

PLAY RECAP *************************************************************************************************************
local_machine              : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

> [!IMPORTANT]
> Setting up WinRM is usually the hardest part of the pre-requisites. Make sure you have configured it correctly before any attempts to run the playbooks.

## Running the Playbooks with Semaphore UI

You can use the `setup-semaphore.yaml` playbook to deploy and configure an independent VM for [Semaphore UI](https://github.com/semaphoreui/semaphore). You can use the `setup-semaphore-project.yaml` playbook to create a project on the Semaphore UI server based on the cloned version of this repository on its filesystem.

**Additional steps required to support Semaphore UI** - Share the `Installation Files` and `Virtual Machines` folders on the Windows Hyper-V Host and grant full-control access to the `ansible` user created in the earlier section.

## Running the Playbooks via Command Line

### Configuring the Inventories and Settings

> [!IMPORTANT]
> **You should review all inventory files in the `/inventories` folder and make the necessary changes for your lab environment**.

Configure the settings for the Kubernetes Cluster in the `inventories/group_vars/all.yaml` file. You can configure the `lab_name` to allow the playbooks to use it as a prefix when auto generating names for the hosts and VMs in the Infrastructure Services and Kubernetes Cluster. The playbooks will also organize the physical VMs inside a folder based on the `lab_name` for ease of management.

```yaml
# Will be used as the prefix for all the hostnames
lab_name: devops
```

> [!TIP]
> You can change this behaviour manually by hard-coding the hostnames in each of the host inventory files.

Configure the desired VM specifications in the remaining group variable files in the `/inventories/group_vars` folder. Below is example of the VM settings for the `inventories/group_vars/kubernetes_control_planes.yaml` file for the Kubernetes control-planes.
```yaml
# The VM settings for each host.
# Note: Adjust accordingly to the number of add-ons you plan to install. You can manually adjust it in
#       Hyper-V manager later.
vm:
  specs:
    ram_size: 16GB
    min_ram_size: 12GB
    disk_size: 256GB
    v_cpu: 6
```

Configure the IP addresses of the VMs in the inventory files located in the `/inventories` folder. Below is an example of the `inventories/kubernetes_control_planes.yaml` file for the Kubernetes control-planes. **You should review and change all the other host inventory files accordingly**.
```yaml
kubernetes_control_planes:
  hosts:
    control-plane-01:
      ansible_hostname: "{{ lab_name }}-cp1"
      ansible_host: 192.168.0.204

    control-plane-02:
      ansible_hostname: "{{ lab_name }}-cp2"
      ansible_host: 192.168.0.205

    control-plane-03:
      ansible_hostname: "{{ lab_name }}-cp3"
      ansible_host: 192.168.0.206
```

Finally, configure the settings for the VMs in the `roles/vm-linux/setup-vm/vars/main.yaml` file.

Configure the network settings for your VMs:
```yaml
vm:
  network:
    # The ipv4 of the gateway.
    gateway: 192.168.0.1
    cidr: 24
    dns: [ 8.8.8.8, 1.1.1.1 ]

  # The hyper-v switch to use
  switch: LAN
```

Specify the two folders which you have created earlier in the `vm:` section:
```yaml
vm:
  # The location where the seed isos will be placed
  iso_folder: "D:\\Virtual Machines\\Kubernetes"

  # The location where the VMs will be created
  vm_folder: "D:\\Virtual Machines\\Kubernetes"
```

Select which Linux OS to install by specifying it in `vm.os`. The playbooks currently support `ubuntu` (default) or `redhat`.

If `redhat` is selected:
* Please download the **Red Hat Enterprise Linux Guest Image** and specify it in `vm.redhat.kvm_image`.
* Specify the Red Hat Subscription username and password for automatic registration of the VMs to enable the dnf repositories.

```yaml
vm:
  # Valid values for now are:
  # ubuntu - for ubuntu
  # redhat - Red Hat Enterprise Linux
  os: redhat

  # RHEL images cannot be downloaded without signing in. Therefore, you will need to manually
  # download it and place it somewhere.
  redhat:
    # The full path and name to the RHEL KVM image to use.
    kvm_image: "/mnt/c/Installation Files/RedHat/rhel-9.3-x86_64-kvm.qcow2"

    # Your redhat login id
    username: <username>

    # Your redhat login password
    password: <password>
```

> [!NOTE]
> For ubuntu, the `setup-vm` role will attempt to automatically download the cloud image as specified in `vm.ubuntu.img_url` but sometimes, it may intermittently take a bit longer due to the designated mirror or connectivity issues. If such cases occur, you can manually download the cloud image file, rename it to `ubuntu-cloud.img` and copy it into the `temp_dir` location that you have specified i.e. `/var/automate`

### Running the Playbooks

The playbooks can be run with the `ansible-playbook` command. i.e.
```bash
ansible-playbook setup-load-balancers.yaml
```
a. To setup any **(Optional) Infrastructure Services**, run the following playbooks in sequence:

| No. | Name                        | Description                                                 | Remarks |
|  :-:| ----------------------------| ----------------------------------------------------------- | ------- |
|  1. | `setup-dns.yaml`            | Provision and configure a VM for BIND (DNS) server.         | **WARNING! Do not set host to existing DNS server!** <br> Additional configuration - Set `kube.cluster.use_dns_server:` to `yes` or `no` in `inventories/group_vars/all.yaml` accordingly. |
|  2. | `setup-load-balancers.yaml` | Provision and configure two VMs for HAProxy and Keepalived. | When deploying load-balancers:<br><ul><li>Set `register_to_load_balancer: yes` in `inventories/group_vars/kubernetes_cluster.yaml`. </li><li>Set `kube.cluster.address` to the desired **virtual ip address** in `inventories/group_vars/all.yaml`. </li></ul>When not deploying load-balancers:<ul><li>Set `register_to_load_balancer: no` in `inventories/group_vars/kubernetes_cluster.yaml`. </li><li>Set `kube.cluster.address` to point to the **ip address of primary control plane** in `inventories/group_vars/all.yaml`.</li></ul> |
|  3. | `setup-nfs.yaml`            | Install NFS server on infrastructure service VM.            | Run this if you plan to use CSI NFS. Note: You must run this **before** running the csi/nfs role. |
|  4. | `setup-minio.yaml`          | Install Minio on infrastructure service VM.                 | Run this if you plan to test add-ons that requires external S3 storage for backups i.e. **velero** or **csi/longhorn** |

b. To setup the **Kubernetes Cluster**, run the following playbooks in sequence:

| No. | Name                               | Description                                                         | Remarks |
|  :-:| ---------------------------------- | ------------------------------------------------------------------- | ------- |
|  1. | `setup-control-plane-servers.yaml` | Provision VMs for kubernetes control planes. ||
|  2. | `setup-control-plane-primary.yaml` | Create primary control plane in the kubernetes cluster. ||
|  3. | `setup-control-plane-other.yaml`   | Create and join secondary control planes to the kubernetes cluster. ||
|  4. | `setup-worker-node-servers.yaml`   | Provisions VMs for the kubernetes worker nodes. | Run this to create worker nodes. |
|  5. | `setup-worker-nodes.yaml`          | Create and join worker nodes into the kubernetes cluster. | Run this to create worker nodes. |
|  6. | `setup-add-ons.yaml`               | Install add-on components listed in `add_ons`. | See [Installing and Configuring Add-ons](#installing-and-configuring-add-ons). |

## Installing and Configuring Add-ons

The `setup-add-ons.yaml` playbook is used to install and configure the add-on stack for the Kubernetes Cluster. It can be executed multiple times to install different add-ons. Each add-on is modularized into ansible roles and you can specify which one to install by listing them in the `add_ons:` collection.

The roles folders are structured in the following manner:

| Folder         | Description |
| -------------- |----         |
| defaults       | Default values that you should not change for the role. |
| tasks          | The tasks that performs the installation and configuration work. |
| vars           | Variable values that you can change to customize the role. |
| templates      | The jinja2 templates that you can modify to add extra configurations. |
| meta           | Role dependencies that you should not touch. |

Below is an example of installing some basic add-ons with csi/longhorn as the persistent technology:
```yaml
- name: Install Kubernetes add-ons
  hosts: kubernetes_control_planes[0]
  become_user: "{{ kube.user }}"
  gather_facts: yes

  roles:
    # Comment/uncomment the add-ons you want to build your stack. The sequence matters.
    # You can rerun this many times with different add-ons selected.
    - role: kubernetes/add-on
      add_ons:
        - name: cert-manager
        - name: ingress
        - name: dashboard
        - name: monitoring
        - name: metrics-server
        - name: csi/longhorn
```

> [!WARNING]
> * Some roles are deploying sample/example configurations and some may be deploying production environments.
> * Add-ons may conflict with each other (i.e. Kube-Prometheus vs. Metrics Server). The order of installation is important.
> * Version incompatibility can occur i.e. new Kubernetes version may break everything or some add-ons version may not be compatible with each other. Configure the desired versions in the `vars\main.yaml` of the role.
> * If an add-on fails to install, use an older release of the add-on since the playbooks may not be compatible with the latest releases.
> * Be patient if you are running on a slow internet connection. Installation may take some time. Increase the timeout of tasks if more time is required to complete certain tasks (requires ansible knowledge).

### List of Add-ons Available

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

## Disclaimer
Everything provided here is 'as-is' and comes with no warranty or support.