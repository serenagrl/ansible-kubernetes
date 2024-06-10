# Ansible Kubernetes Lab on Hyper-V
A collection of Ansible playbooks to provision a bare-metal Kubernetes cluster on Hyper-V for testing/learning purposes. These playbooks and add-ons were tailored to my working environment and were not intended to be an all-purpose "installer" for Kubernetes. Therefore, please feel free to customize them as you see fit.

### Pre-requisites / Setup
1. A Windows machine with sufficient processing power, RAM and disk space (i7 CPU, 128GB RAM, SSD recommended) with [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install) installed. These playbooks have been tested on a Windows host with Ubuntu 22.04 running in WSL.

2. Install Ansible on the Ubuntu OS in your WSL.
   ```
   sudo apt update
   sudo apt install software-properties-common
   sudo add-apt-repository --yes --update ppa:ansible/ansible
   sudo apt install ansible
   ```

   You can refer to the detail documentation [here](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html#installing-ansible-on-ubuntu).

3. Configure Windows Remote Management (WinRM) on your Windows host that has Hyper-V feature enabled. You can run on Windows 11 or Windows Server 2022 with Hyper-V enabled. You may pool multiple Windows Hyper-V servers together to host the virtual machines but each of them will need to be configured with WinRM properly.

   3.1. Open an Ubuntu terminal in your WSL and run the following:
   ```
   sudo apt install python3-pip
   pip install ansible pywinrm kubernetes jsonpatch
   ```

   3.2. Open a Powershell command prompt with Administrator access in your Windows host and run the following:
   ```
   $username = "ansible"
   $password = ConvertTo-SecureString "Ans1bl3h" -AsPlainText -Force

   New-LocalUser -Name $username -Password $password -FullName $username -Description "Ansible Controller Host Account" -AccountNeverExpires -PasswordNeverExpires -UserMayNotChangePassword

   Add-LocalGroupMember -Group Administrators -Member $username

   $setupscript = "https://raw.githubusercontent.com/ansible/ansible-documentation/ae8772176a5c645655c91328e93196bcf741732d/examples/scripts/ConfigureRemotingForAnsible.ps1"
   Invoke-WebRequest $setupscript -OutFile winrm-setup.ps1
   .\winrm-setup.ps1
   ```

   You can refer to the detail documentation [here](https://docs.ansible.com/ansible/latest/os_guide/windows_setup.html).

   **Important!**: Setting up WinRM is usually the hardest part of the pre-requisites. Make sure you have it configured correctly before you attempt to run the playbooks.

4. Create a Windows user with Administrator (or proper) access rights for ansible in the Windows Hyper-V machine. The following is a simple powershell script that you can execute to create the user:

   ```
   $username = "ansible"
   $password = ConvertTo-SecureString "p@ssw0rd" -AsPlainText -Force

   New-LocalUser -Name $username -Password $password -FullName $username -Description "Ansible Controller Host Account" -AccountNeverExpires -PasswordNeverExpires -UserMayNotChangePassword

   Add-LocalGroupMember -Group Administrators -Member $username
   ```
5. Clone this repository into a directory of your choice in the WSL and [configure WinRM access](https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html) in the `\inventories\winrm.yaml` file. You may need to configure the necessary access rights for the directory.

   ```
   git clone https://github.com/serenagrl/ansible-kubernetes.git
   ```

6. Optional: Follow the below steps, if you wish to connect to WinRM using certificates.

   6.1 Create a `certs` folder inside the `ansible-kubernetes` project repository which you have cloned earlier i.e. `/root/ansible-kubernetes/certs`.

   Open an Ubuntu terminal in WSL to create the certificates:
   ```
   USERNAME="ansible"

   cat > openssl.conf << EOL
   distinguished_name = req_distinguished_name
   [req_distinguished_name]
   [v3_req_client]
   extendedKeyUsage = clientAuth
   subjectAltName = otherName:1.3.6.1.4.1.311.20.2.3;UTF8:$USERNAME@localhost
   EOL

   export OPENSSL_CONF=openssl.conf
   openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -out cert.pem -outform PEM -keyout cert_key.pem -subj "/CN=$USERNAME" -extensions v3_req_client
   rm openssl.conf
   ```

   6.2 Register the certificates to the Windows certificate store and configure WinRM to use it for the ansible user.

   You can manually add the certificates using the Certificate Manager UI (`certmgr.msc`) or execute the following script in a Powershell command prompt with Administrator access:
   ```
   $cert = New-Object -TypeName System.Security.Cryptography.X509Certificates.X509Certificate2 "\\wsl.localhost\Ubuntu-22.04\root\ansible-kubernetes\certs\cert.pem"

   $store_name = [System.Security.Cryptography.X509Certificates.StoreName]::Root
   $store_location = [System.Security.Cryptography.X509Certificates.StoreLocation]::LocalMachine
   $store = New-Object -TypeName System.Security.Cryptography.X509Certificates.X509Store -ArgumentList $store_name, $store_location
   $store.Open("MaxAllowed")
   $store.Add($cert)
   $store.Close()

   $cert = New-Object -TypeName System.Security.Cryptography.X509Certificates.X509Certificate2 "\\wsl.localhost\Ubuntu-22.04\root\ansible-kubernetes\certs\cert.pem"

   $store_name = [System.Security.Cryptography.X509Certificates.StoreName]::TrustedPeople
   $store_location = [System.Security.Cryptography.X509Certificates.StoreLocation]::LocalMachine
   $store = New-Object -TypeName System.Security.Cryptography.X509Certificates.X509Store -ArgumentList $store_name, $store_location
   $store.Open("MaxAllowed")
   $store.Add($cert)
   $store.Close()
   ```

   Finally, configure WinRM to use the certificate for the ansible user.
   ```
   $username = "ansible"
   $password = ConvertTo-SecureString -String "Ans1bl3h" -AsPlainText -Force
   $credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $username, $password

   $thumbprint = (Get-ChildItem -Path cert:\LocalMachine\root | Where-Object { $_.Subject -eq "CN=$username" }).Thumbprint

   New-Item -Path WSMan:\localhost\ClientCertificate `
       -Subject "$username@localhost" `
       -URI * `
       -Issuer $thumbprint `
       -Credential $credential `
       -Force

   Set-Item -Path WSMan:\localhost\Service\Auth\Certificate -Value $true
   ```

### Requirements for Kubernetes VMs
1. The specifications for each Kubernetes VM is depending on how much add-ons you wish to install on the Kubernetes Cluster. The following specifications will give you a rough guide on what to set but you are free to adjust them accordingly to your needs:

    Recommended specifications for learning the basics of kubernetes with basic add-ons installed:

    | Hosts           | vCPU | Min. RAM  | Recommended RAM |
    | --------------- |:----:|  :----:   |     :----:      |
    | Control plane 1 | 4    | 8GB       | 12GB             |
    | Control plane 2 | 4    | 8GB       | 12GB             |
    | Control plane 3 | 4    | 8GB       | 12GB             |

    Recommended specifications for a test-lab with majority of the add-ons installed:

    | Hosts           | vCPU | Min. RAM  | Recommended RAM |
    | --------------- |:----:|  :----:   |     :----:      |
    | Control plane 1 | 6    | 12GB       | 16GB            |
    | Control plane 2 | 6    | 12GB       | 16GB            |
    | Control plane 3 | 6    | 12GB       | 16GB            |

    Recommended specifications for an SIT environment with nearly all of the add-ons installed and with worker nodes:

    | Hosts           | vCPU | Min. RAM  | Recommended RAM |
    | --------------- |:----:|  :----:   |     :----:      |
    | Infra Service   | 2    | 1GB       | 2GB             |
    | Load Balancer 1 | 2    | 1GB       | 2GB             |
    | Load Balancer 2 | 2    | 1GB       | 2GB             |
    | Control plane 1 | 6    | 12GB      | 32GB            |
    | Control plane 2 | 6    | 12GB      | 32GB            |
    | Control plane 3 | 6    | 12GB      | 32GB            |
    | Worker Node 1   | 4    | 4GB       | 8GB             |
    | *Worker Node 2  | 4    | 4GB       | 8GB             |

    **Note**: You may need to increase the specifications of the worker nodes depending on what you plan to host on them. The number of worker nodes is depending on your requirements and machine resources.

    **Warning**: Both Longhorn and Rook-Ceph CSIs can consume a lot of resources but offers a good learning path and discipline for managing clustered storage. You may need to utilize more than 1 Hyper-V hosts with more RAM if you ran out of resources.

### About These Playbooks
* The Kubernetes cluster is currently setup to use Calico and Containerd.
* Playbooks are defaulted to setup a DNS server, 2 HAProxy load balancers and 3 control-planes.
* Configure the IP addresses, host names and domain to your environment in the host files located in `\inventories`
* Both DNS and NFS are configured to share a single host to conserve resources. The NFS server is configured to not provision a new server by default.
* Some roles are deploying sample/example configurations only (although some may deploy production environments)
* Add-ons may conflict with each other (i.e. Kube-Prometheus vs. Metrics Server). Adjust the order of installation if needed.
* Version incompatibility may occur i.e. new Kubernetes version may break everything or some add-ons version may not be compatible with each other. Configure the desired versions in the `vars\main.yaml` of the role.
* If an add-on fails to install, try using an older release of the add-on since the playbooks may not be compatible with the latest releases.
* Be patient if you are running on a slow internet connection. Installation may take some time. Increase the timeout of tasks if your environment needs more time to complete certain tasks.
* VM Checkpoints will be created to provide safety just in case your installation breaks. These checkpoints may consume a lot of disk space. You may remove the checkpoints when you feel everything is stable to recover the disk space.

### Playbooks

* The following is a list of playbooks that you can run in sequence:

    | No. | Name | Description |
    |  :-:| ---- | ---- |
    |  1. | `setup-dns.yaml` | Optional. Provisions the VM and configure BIND (DNS) server. |
    |  2. | `setup-load-balancers.yaml` | Optional. Provisions the VMs and configure HAProxy and Keepalived. |
    |  3. | `setup-control-plane-servers.yaml` | Provisions VMs for the kubernetes control planes and installs the pre-requisites. |
    |  4. | `setup-control-plane-primary.yaml` | Creates the primary control plane in the kubernetes cluster. |
    |  5. | `setup-control-plane-other.yaml` | Creates and joins secondary control planes to the kubernetes cluster. |
    |  6. | `setup-worker-node-servers.yaml` | Optional. Provisions VMs for the kubernetes worker nodes and installs the pre-requisites. |
    |  7. | `setup-worker-nodes.yaml` | Optional. Creates/Joins worker nodes into the kubernetes cluster. |
    |  8. | `setup-nfs.yaml` | Installs NFS server on a designated server. Optionally, you can provision a VM for it. |
    |  9. | `setup-add-ons.yaml` | Install add-on components listed in `add_ons`. |

  **Reminder**
  1. If you skipped the creation of the DNS Server, please remember to:
     - Set `kube.cluster.use_dns_server: no` in the `group_vars\all.yaml` file.
       WARNING: Connecting to an existing DNS server and performing the necessary configuration is not supported.
  3. If you skipped the creation of load balancers, please remember to:
     - Set `register_to_load_balancer: no` in the `inventories\group_vars\kubernetes_control_planes.yaml` and `inventories\group_vars\kubernetes_worker_nodes.yaml` files.
     - Set `kube.cluster.address` in the `group_vars\all.yaml` to your cp1's IP address.
  4. Specify the addons to install in the `add_ons` collection in `setup-add-ons.yaml`.
  5. You may rerun the `setup-add-ons.yaml` everytime you change the `add_ons` but keep an eye on the checkpoints.
  6. The installation order is important as some add-ons have dependencies on the others.

* The following is a list of playbooks that merges some of the steps above for convenience:

    | Name | Description |
    | ---- | ---- |
    | `setup-control-planes.yaml` | Creates a kubernetes cluster with both primary and secondary control planes at-one-go. |
    | `setup-kubernetes-servers.yaml` | Provision VMs and install pre-requisites for the control planes and worker nodes at-one-go. |
    | `setup-kubernetes.yaml` | Creates a kubernetes cluster with all control planes and worker nodes at-one-go. |

* The following is a list of playbooks that creates a separate Ansible Semaphore and Controller Host VM (Experimental):

    | Name | Description |
    | ---- | ---- |
    | `setup-semaphore.yaml` | Installs Ansible Semaphore on a designated server. Optionally, you can provision a VM for it. |
    | `setup-semaphore-project.yaml` | Creates a new Semaphore project. |

  **Note**: These playbooks were made to create an environment for absolute beginners as the Semaphore will provide a User Interface on-top of the playbooks in this repository. However, you are required to configure WinRM and create 2 share folders (Installation Files and Virtual Machines) on each Windows Hyper-V host.

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

To install the specific add-on, specify them in the `add_ons` collection before running `setup-add-ons.yaml`.

### Disclaimer
Everything provided here is 'as-is' and with no warranty or support.
