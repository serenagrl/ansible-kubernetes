# Configure WinRM to use Certificates

### 1. Create User Certificate

Create a `certs` folder inside the `ansible-kubernetes` project repository which you have cloned earlier i.e. `/root/ansible-kubernetes/certs`.

Open an Ubuntu terminal in WSL to create the certificates in the folder:
```bash
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

### 2. Register User Certificate

Register the certificate to the Windows certificate store and configure WinRM to use it for the `ansible` user. You can manually add the certificates using the Certificate Manager UI (`certmgr.msc`) or execute the following script in a Powershell command prompt with Administrator access:
```powershell
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

### 3. Configure WinRM

Configure WinRM to use the certificate for the ansible user.

> [!CAUTION]
> Please use your own secure password instead of the default password in the sample script.

```powershell
$username = "ansible"
$password = ConvertTo-SecureString -String "p@ssw0rd" -AsPlainText -Force
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

### 4. Configure WinRM Ansible Host File
Configure the WinRM settings in the [`inventories/winrm.yaml`](../inventories/winrm.yaml) file for the playbooks.
```yaml
# This should be your local machine or an Ansible host. Winrm must be setup correctly for this to work.
# You can have more than 1 winrm host to distribute the kubernetes VMs; remember to set the winrm_host.
winrm:
  hosts:
    local_machine:
      ansible_host: 192.168.0.118
      ansible_winrm_cert_pem: ./certs/cert.pem
      ansible_winrm_cert_key_pem: ./certs/cert_key.pem
      ansible_become_method: runas
      ansible_become_user: ansible       # Must set this to the user you created
      ansible_connection: winrm
      ansible_port: 5986
      ansible_winrm_transport: certificate
      ansible_winrm_server_cert_validation: ignore
```