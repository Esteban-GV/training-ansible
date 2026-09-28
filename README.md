# Infrastructure and Application Deployment with Terraform and Ansible on Azure

This project automates the creation of cloud infrastructure (Azure) using **Terraform** and the server configuration using **Ansible**. As a proof of concept, it deploys an Ubuntu virtual machine, installs Docker, and runs a container of the classic Super Mario Bros game.

Terraform describes the desired state and uses the Azure provider to create, query, and delete those resources, while Ansible handles the internal provisioning of the operating system.

## Resources Created in Azure

- A resource group in `canadacentral`.
- A virtual network and a subnet.
- A static public IP address.
- A network interface connected to the subnet and the public IP.
- An Ubuntu 22.04 LTS virtual machine (`Standard_B1s`).
- A network security group (NSG) allowing SSH access (port 22) and HTTP/Custom access (ports 80/8787).

## Requirements

- An active Azure subscription with permissions to create resources.
- [Terraform](https://developer.hashicorp.com/terraform/downloads) 1.1.0 or later.
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli), to sign in from the terminal.

## Main Files

**Infrastructure (Terraform):**
- `main.tf`: provider, resources, username and password variables, and the public IP output.
- `.terraform.lock.hcl`: verified provider versions; keep it in Git.
- `terraform.tfvars.example`: example variable configuration, without a real password.
- `terraform.tfvars`: local values; excluded from Git for security.
- `.terraform/`: plugins downloaded by Terraform; generated locally and excluded from Git.
- `terraform.tfstate`: local state of the managed resources; excluded from Git.

**Configuration (Ansible):**
- `inventory/hosts.ini`: defines the virtual machine's IP and the SSH connection details.
- `playbooks/install_docker.yml`: main playbook that installs Docker and deploys the application container.

## Step-by-Step Execution

### 1. Azure Authentication
Sign in and, if you have more than one subscription, select the one you will use:
```bash
az login
az account set --subscription "SUBSCRIPTION_ID_OR_NAME"
```

### 2. Terraform Configuration
Copy the example file and edit `terraform.tfvars` with your credentials:

```bash
cp terraform.tfvars.example terraform.tfvars
```

```hcl
admin_username = "admin_user"
admin_password = "REPLACE_WITH_A_STRONG_PASSWORD"
```

(Make sure `terraform.tfvars` is excluded in your `.gitignore`.)

### 3. Initialize and Create the Infrastructure (Terraform)
Run the following commands from the project root to download the Azure plugins, verify the resources to be created, and apply the changes:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply -auto-approve
```

When it finishes, Terraform will display the public IP of your new virtual machine. Copy it.

### 4. Configure the Inventory (Ansible)
Open the `inventory/hosts.ini` file and update the IP address with the value returned by Terraform:

```ini
[azure_vm]
YOUR_PUBLIC_IP ansible_user=admin_user
```

### 5. Run the Playbooks (Ansible)
Provision the server, install Docker, and start the container by running the main playbook:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/install_docker.yml
```

### 6. Access the Application
Once Ansible finishes successfully, open your web browser and enter your server's public IP, specifying port 8787:

`http://YOUR_PUBLIC_IP:8787`

### Resource Cleanup (Destruction)
To delete all the infrastructure and avoid unnecessary charges in Azure, run:

```bash
terraform plan -destroy
terraform destroy
```

Wait for Terraform to confirm the complete deletion. Do not manually delete the state files.

### Lessons Learned and Troubleshooting

**Port Synchronization (Azure NSG vs Docker):** The ports mapped in the Ansible playbooks (e.g., `ports: - "8787:8080"`) must be explicitly allowed in the Terraform Network Security Group (NSG). The `destination_port_ranges = ["80", "8787"]` property was used to open multiple ports in a single rule.

**SSH Diagnostics (Connection timed out):** If Ansible fails with a timeout error on port 22, it indicates a network block at the Azure firewall level (the NSG does not allow inbound traffic or is not attached to the network interface), not a password error.

**REMOTE HOST IDENTIFICATION HAS CHANGED Warning:** When destroying and recreating cloud machines (`terraform destroy` and then `apply`), the server generates a different cryptographic fingerprint even if it keeps the same IP. Clear the local record by running `ssh-keygen -R 'YOUR_PUBLIC_IP'` so SSH doesn't block the connection for security reasons.

**Inline Syntax in Terraform:** When defining security rules inside the `azurerm_network_security_group` resource (inline block), Terraform implicitly inherits the context. Declaring arguments such as `resource_group_name` inside that block causes compilation errors (`Unsupported argument`).