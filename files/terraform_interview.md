# Terraform Interview Preparation
## For DevOps Engineers (1 Year Experience)

---

## MOST ASKED INTERVIEW QUESTIONS WITH CODE

---

### Q1. What is Terraform and how does it work?

Terraform is an Infrastructure as Code (IaC) tool by HashiCorp.
You write code to create infrastructure instead of clicking in the portal.

```
You write main.tf
      ↓
terraform plan   → shows what will be created
      ↓
terraform apply  → actually creates resources
      ↓
terraform destroy → deletes everything
```

---

### Q2. What is terraform init?

```bash
terraform init
```

```
What it does:
- downloads provider plugins (azurerm, aws, google)
- connects to backend (state file location)
- sets up working directory
- must run before plan or apply
```

---

### Q3. Write a basic Azure Resource Group in Terraform

```hcl
# main.tf

# terraform block - tells which provider to use
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"   # provider source
      version = "~> 3.0"              # version constraint
    }
  }
}

# provider block - configures the provider
provider "azurerm" {
  features {}   # required empty block for azurerm
}

# resource block - creates actual resource
resource "azurerm_resource_group" "rg" {
  # azurerm_resource_group = resource type
  # rg = local name (used to reference this resource)

  name     = "my-resource-group"   # name in Azure
  location = "eastus"              # azure region
}
```

---

### Q4. What are variables in Terraform?

Variables make your code reusable. Instead of hardcoding values you use variables.

```hcl
# variables.tf - declare variables here

variable "resource_group_name" {
  description = "Name of the resource group"  # explains what it is
  type        = string                         # data type
  default     = "my-rg"                       # default value if not provided
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "eastus"
}

variable "vm_count" {
  description = "Number of VMs to create"
  type        = number
  default     = 2
}

variable "tags" {
  description = "Tags for resources"
  type        = map(string)         # key value pairs
  default     = {
    environment = "dev"
    team        = "devops"
  }
}
```

```hcl
# main.tf - use variables with var.variable_name

resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name   # using variable
  location = var.location
  tags     = var.tags
}
```

---

### Q5. What is terraform.tfvars?

Instead of setting default values you can pass values from a file.

```hcl
# terraform.tfvars - actual values

resource_group_name = "sachin-rg"
location            = "eastus"
vm_count            = 3
tags = {
  environment = "prod"
  team        = "platform"
}
```

```
terraform apply                          # uses terraform.tfvars automatically
terraform apply -var-file="prod.tfvars"  # use specific file
terraform apply -var="location=westus"  # pass single variable
```

---

### Q6. What are outputs in Terraform?

Outputs show values after terraform apply. Useful for getting IPs, connection strings etc.

```hcl
# outputs.tf

output "resource_group_name" {
  value       = azurerm_resource_group.rg.name   # reference resource attribute
  description = "Name of created resource group"
}

output "resource_group_id" {
  value = azurerm_resource_group.rg.id
}

output "vm_public_ip" {
  value     = azurerm_public_ip.pip.ip_address
  sensitive = true   # hides value in terminal output (for passwords, IPs)
}
```

```bash
terraform output                    # show all outputs
terraform output resource_group_id  # show specific output
```

---

### Q7. What is Terraform state?

```
State file = terraform.tfstate

Terraform remembers what it created in this file.

First apply:
  resource does not exist → create it
  save to state file

Second apply:
  check state file → resource exists
  check Azure → resource exists
  nothing changed in code → do nothing

You changed code:
  check state → resource exists
  code changed → update resource
```

```hcl
# store state in Azure Storage (remote backend)
# so team can share state

terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-rg"       # where storage account is
    storage_account_name = "tfstatesachin"      # storage account name
    container_name       = "tfstate"            # container name
    key                  = "prod.terraform.tfstate"  # state file name
  }
}
```

---

### Q8. Create Azure Storage Account with Terraform

```hcl
# main.tf

resource "azurerm_storage_account" "storage" {
  name                     = "sachinstorage123"     # must be globally unique, lowercase only
  resource_group_name      = azurerm_resource_group.rg.name  # reference from rg resource
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"             # Standard or Premium
  account_replication_type = "LRS"                  # LRS, GRS, ZRS, RAGRS

  tags = {
    environment = "dev"
  }
}

# create container inside storage account
resource "azurerm_storage_container" "container" {
  name                  = "mycontainer"
  storage_account_name  = azurerm_storage_account.storage.name  # reference storage account
  container_access_type = "private"   # private, blob, container
}
```

---

### Q9. Create Azure Virtual Machine with Terraform

```hcl
# main.tf - full VM setup

# resource group
resource "azurerm_resource_group" "rg" {
  name     = "sachin-vm-rg"
  location = "eastus"
}

# virtual network
resource "azurerm_virtual_network" "vnet" {
  name                = "sachin-vnet"
  address_space       = ["10.0.0.0/16"]   # IP range for vnet
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# subnet inside vnet
resource "azurerm_subnet" "subnet" {
  name                 = "sachin-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]  # subset of vnet range
}

# public IP for VM
resource "azurerm_public_ip" "pip" {
  name                = "sachin-pip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"   # Static = fixed IP, Dynamic = changes on restart
}

# network interface - connects VM to network
resource "azurerm_network_interface" "nic" {
  name                = "sachin-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id
  }
}

# virtual machine
resource "azurerm_linux_virtual_machine" "vm" {
  name                = "sachin-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"   # VM size
  admin_username      = "adminuser"

  # attach nic to vm
  network_interface_ids = [azurerm_network_interface.nic.id]

  # SSH key for login
  admin_ssh_key {
    username   = "adminuser"
    public_key = file("~/.ssh/id_rsa.pub")   # reads file from local machine
  }

  # OS disk
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  # OS image
  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }
}
```

---

### Q10. What is depends_on in Terraform?

```hcl
# normally terraform figures out dependencies automatically
# but sometimes you need to force an order

resource "azurerm_resource_group" "rg" {
  name     = "sachin-rg"
  location = "eastus"
}

resource "azurerm_storage_account" "storage" {
  name                     = "sachinstorage"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  depends_on = [azurerm_resource_group.rg]
  # wait for rg to be created before creating storage account
  # not needed here because we reference rg.name (terraform auto detects)
  # but useful when there is no direct reference
}
```

---

### Q11. What is count and for_each in Terraform?

```hcl
# count - create multiple resources

resource "azurerm_resource_group" "rg" {
  count    = 3                              # creates 3 resource groups
  name     = "sachin-rg-${count.index}"    # sachin-rg-0, sachin-rg-1, sachin-rg-2
  location = "eastus"
}

# access specific resource
output "first_rg" {
  value = azurerm_resource_group.rg[0].name   # index starts at 0
}
```

```hcl
# for_each - create resources from a map or set

variable "resource_groups" {
  default = {
    dev  = "eastus"
    prod = "westus"
    test = "centralus"
  }
}

resource "azurerm_resource_group" "rg" {
  for_each = var.resource_groups        # loops over map
  name     = "sachin-${each.key}-rg"   # each.key = dev, prod, test
  location = each.value                 # each.value = eastus, westus, centralus
}
```

---

### Q12. What is terraform taint and terraform import?

```bash
# terraform taint - mark resource for recreation
# next apply will destroy and recreate this resource

terraform taint azurerm_virtual_machine.vm
terraform apply  # destroys and recreates vm


# terraform import - import existing resource into state
# useful when resource was created manually and you want terraform to manage it

terraform import azurerm_resource_group.rg /subscriptions/xxx/resourceGroups/sachin-rg
# now terraform knows about this resource
# you still need to write the config in main.tf manually
```

---

### Q13. What is a terraform module?

```
Module = reusable terraform code
Like a function in programming

Instead of writing same code again and again
you put it in a module and call it
```

```hcl
# modules/resource_group/main.tf

variable "name" {}
variable "location" {}

resource "azurerm_resource_group" "rg" {
  name     = var.name
  location = var.location
}

output "id" {
  value = azurerm_resource_group.rg.id
}
```

```hcl
# root main.tf - calling the module

module "dev_rg" {
  source   = "./modules/resource_group"   # path to module
  name     = "dev-rg"
  location = "eastus"
}

module "prod_rg" {
  source   = "./modules/resource_group"   # same module, different values
  name     = "prod-rg"
  location = "westus"
}

# access module output
output "dev_rg_id" {
  value = module.dev_rg.id
}
```

---

### Q14. What are terraform workspaces?

```bash
# workspaces = multiple states for same code
# useful for dev, staging, prod environments

terraform workspace list       # show all workspaces
terraform workspace new dev    # create dev workspace
terraform workspace new prod   # create prod workspace
terraform workspace select dev # switch to dev
terraform workspace show       # show current workspace
```

```hcl
# use workspace name in resource names
resource "azurerm_resource_group" "rg" {
  name     = "sachin-${terraform.workspace}-rg"
  # dev workspace  → sachin-dev-rg
  # prod workspace → sachin-prod-rg
  location = "eastus"
}
```

---

### Q15. Most common Terraform commands

```bash
terraform init       # initialize working directory
terraform validate   # check code for syntax errors
terraform fmt        # format code properly
terraform plan       # show what will change
terraform apply      # apply changes
terraform destroy    # destroy all resources
terraform show       # show current state
terraform state list # list all resources in state
terraform output     # show output values
terraform refresh    # sync state with real infrastructure
```

---

### Common Interview Mistakes to Avoid

```
1. Forgetting terraform init before plan
2. Not using remote backend for team projects
3. Committing terraform.tfstate to git (security risk)
4. Hardcoding credentials in tf files
5. Not using variables for reusability
6. Not using depends_on when needed
7. Forgetting to run terraform plan before apply
```

---

### .gitignore for Terraform projects

```
# .gitignore

.terraform/           # downloaded providers (large, not needed in git)
terraform.tfstate     # state file (contains sensitive data)
terraform.tfstate.backup
*.tfvars              # variable files (may contain secrets)
.terraform.lock.hcl   # can be committed but optional
```
