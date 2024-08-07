# Configure Semaphore UI (Experimental)

After completing the configuration of the playbooks, you can choose to setup Semaphore UI to provide an easy-to-use user interface to execute the playbooks.

> [!TIP]
> It is recommended that you perform as much configuration as possible on the cloned repository in the file-system before setting up Semaphore UI to reduce the need for additional configurations on Semaphore later.

### 1. Pre-requisites

Share the `D:\Installation Files` and `D:\Virtual Machines` folders on the **Windows Hyper-V Host** and grant full-control access to the `ansible` user created in the earlier [section](https://github.com/serenagrl/ansible-kubernetes#hyper-v-host-requirements).

Configure the credentials to mount the share folders in the [`/roles/vm-linux/setup-vm/vars/main.yaml`](../roles/vm-linux/setup-vm/vars/main.yaml) role variable file.
```yaml
vm:
  # The Windows user credentials to use to access the shared folder on the winrm host.
  mount:
    username: Ansible
    password: <password>
```

### 2. Provision the Semaphore VM

Set the Semaphore VM **hostname** and **IP address** in the [`semaphore_server.yaml`](../inventories/semaphore_server.yaml) inventory file located in the [`/inventories`](../inventories) folder.
```yaml
semaphore:
  hosts:
    semaphore_server:
      ansible_hostname: semaphore
      ansible_host: 192.168.0.220
```

Next, set the specifications of the Semaphore VM in the [`semaphore_server.yaml`](../inventories/host_vars/semaphore_server.yaml) host variable file located in the [`/inventories/host_vars`](../inventories/host_vars) folder.
```yaml
vm:
  specs:
    ram_size: 2GB
    min_ram_size: 2GB
    disk_size: 127GB
    v_cpu: 4
```

Finally, run the [`setup-semaphore.yaml`](../setup-semaphore.yaml) playbook to provision the Semaphore VM.
```bash
ansible-playbook setup-semaphore.yaml
```

After the installation has completed, open a browser and browse to the IP address that was specified for the Semaphore VM i.e. _http://192.168.0.220_ to verify that Semaphore has been setup successfully. The default login credentials are as specified in the [`/roles/semaphore/vm/vars/main.yaml`](../roles/semaphore/vm/vars/main.yaml) variable file.

### 3. Create the Ansible Kubernetes Semaphore Project

The [`setup-semaphore-project.yaml`](../setup-semaphore-project.yaml) playbook can be used to automatically create a Semaphore project based on the cloned version of this repository in the file-system. It will make a copy of this repository which you have cloned (and configured) in the Ubuntu WSL to the Semaphore VM.

The location of the repository on the Semaphore server can be configured in the [`/roles/semaphore/register/vars/main.yaml`](../roles/semaphore/register/vars/main.yaml) variable file.
```yaml
semaphore:
  project:
    # The root folder for the playbooks.
    local_repo_dir: /ansible-playbooks/ansible-kubernetes
```

> [!CAUTION]
> The automation process is currently not compatible with _ansible-vault_ values and will terminate with errors. If you want to use _ansible-vault_ values for sensitive information, please set them in the repository on the Semaphore server only **after you have created the Semaphore project**.

You can customize the installation of the Semaphore project by passing in the values referred in the [`/roles/semaphore/register/vars/main.yaml`](../roles/semaphore/register/vars/main.yaml) roles variable file to the [`setup-semaphore-project.yaml`](../setup-semaphore-project.yaml) playbook as shown in the below example:

```yaml
- name: Creates a new ansible-kubernetes project in Semaphore
  hosts: semaphore_server
  gather_facts: no

  roles:
    - role: semaphore/register
      semaphore:
        project:
          name: "Home Lab"    # Project Name
          lab_name: k8s       # Prefix for all VMs and hostnames
```

Run the following command to create the Semaphore project:
```bash
ansible-playbook setup-semaphore-project.yaml
```

You should be able to see the Semaphore project created in the Semaphore portal after the installation has completed.

### 4. Co-Administrate Semaphore-Created VMs

Using Semaphore will change the deployment architecture as the Semaphore server will be designated as the **Ansible Controller Host** to manage the VMs. The below diagram illustrates the new topology.

<p align="center">
  <img src="images/ansible-kubernetes-semaphore.jpg" alt="Ansible Kubernetes with Semaphore"/>
</p>

Only the Semaphore server can run playbooks to communicate with the VMs it created. You will not be able to run the playbooks on your local Ubuntu WSL to manipulate the Infrastructure Services or Kubernetes Clusters that are hosted on the VMs that were created by the Semaphore server due to missing SSH configuration. Depending on your preference, this can be considered OK as you can use the Semaphore server as a bastion server to manage your VMs.

However, if you want to enable your local Ubuntu WSL to also run playbooks to target the Semaphore-created VMs, you can create a temporary playbook and run the [`semaphore/add-wsl-ssh`](roles/semaphore/add-wsl-ssh) role in the local Ubuntu WSL.

For example, create a `enable-ssh.yaml` playbook with the following content:
```yaml
- hosts: localhost
  become_user: root
  gather_facts: yes

  roles:
    - role: semaphore/add-wsl-ssh
```

Then run the playbook that you have just created:
```bash
ansible-playbook enable-ssh.yaml
```

The Ubuntu WSL should be able run playbooks to target the Semaphore-created VMs after this.