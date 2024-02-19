![Untitled](https://github.com/karthi770/Azure_Terraform_Dev_env/assets/102706119/b61236f3-a5c9-4603-b4a4-804b56c49300)

Installing the Azure CLI from the following documentation:

![image](https://github.com/karthi770/Jira_GitHub_intergration_Python/assets/102706119/2993a0ab-3d09-43d9-a100-751c30adefbd)

Follow the steps shown below:
![image](https://github.com/karthi770/Jira_GitHub_intergration_Python/assets/102706119/3844bd90-b3f0-4b8c-ae0c-89ae30b2bb2e)

![image](https://github.com/karthi770/Jira_GitHub_intergration_Python/assets/102706119/03e016f6-c5d3-487d-98c9-ed8402814876)

You can verify if the account has been logged in by using this command `az account show`
![image](https://github.com/karthi770/Jira_GitHub_intergration_Python/assets/102706119/f10d289c-d9ee-43a2-9405-361ca9f49f38)

```python
#main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=3.0.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "new-rg" {
  name     = "dev-resources"
  location = "East Us"
  tags = {
    environment = "dev"
  }
}

```

>[!note]
>Use `terraform state list` that will list the available resources and then type `terraform state show <resource>`
### Subnet creation

```python
resource "azurerm_subnet" "mtc-subnet" {
  name                 = "new-subnet"
  resource_group_name  = azurerm_resource_group.new-rg.name
  virtual_network_name = azurerm_virtual_network.new-vn.name
  address_prefixes     = ["10.123.1.0/24"]
}
```
### Security group

The rule for the security group can be either inline or can be deployed as a separate resource. In our case we are trying to create a separate resource for the security group rule.

```python
resource "azurerm_network_security_group" "new-sg" {
    name = "new-security"
    location = azurerm_resource_group.new-rg.location
    resource_group_name = azurerm_resource_group.new-rg.name

    tags = {
        environment = "dev"
    }
}
```

```python
resource "azurerm_network_security_rule" "new-sr" {
  name                        = "new-security-rule"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "*"
  source_port_range           = "*"
  destination_port_range      = "*"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.new-rg.name
  network_security_group_name = azurerm_network_security_group.new-sg.name
}
```

```python
resource "azurerm_subnet_network_security_group_association" "new-sga" {
  subnet_id                 = azurerm_subnet.new-subnet.id
  network_security_group_id = azurerm_network_security_group.new-sg.id
}
```

### IP address configuration
```python
resource "azurerm_public_ip" "new-ip" {
  name                = "new-ip address"
  resource_group_name = azurerm_resource_group.new-rg.name
  location            = azurerm_resource_group.new-rg.location
  allocation_method   = "Dynamic"

  tags = {
    environment = "dev"
  }
}
```
>[!info]
>Even after we create `azurerm_public_ip` the ip address won’t be displayed, one of the reasons is the allocation method is Dynamic and we need to create the other resources so that we can get access to the IP address.

```python
resource "azurerm_network_interface" "new-nic" {
  name                = "new-network-interface"
  location            = azurerm_resource_group.new-rg.location
  resource_group_name = azurerm_resource_group.new-rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.new-subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.new-ip.id
  }
  tags = {
    environment = "dev"
  }
}
```

### SSH connection

- Create a Keygen to ssh into the VM - `ssh-keygen -t rsa`
- Give the location where it needs to be saved.
- While creating the Virtual machine we need to mention the path of the keygen.
### Create a Virtual Machine

```python
resource "azurerm_linux_virtual_machine" "new-vm" {
  name                = "new-virtual-machine"
  resource_group_name = azurerm_resource_group.new-rg.name
  location            = azurerm_resource_group.new-rg.location
  size                = "Standard_F2"
  admin_username      = "adminuser"
  network_interface_ids = [
    azurerm_network_interface.new-nic.id,
  ]

  admin_ssh_key {
    username   = "adminuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }
}
```

>[!info]
>To view the public IP address, run the code `terraform state list` which will give you the list of resources that has been created. Now select the VM resource and run the command `terraform state show azurerm_linux_virtual_machine.new-vm` - 52.168.92.31

>[!note]
> In order to ssh into the Virtual machine we need to run the following command
> `ssh -i ~/.ssh/id_rsa adminuser@52.168.92.31`
> 

### Custom data

>[!info]
>This Custom data is like the user meta data that we used to give when we create the EC2 instance. This is typically a bash script that runs only once, when we initialize the EC2 instance. 

```bash
#this is a template file
#//customdata.tpl 

#!/bin/bash
sudo apt-get update -y &&
sudo apt-get install -y \
apt-transport-hhtps \
ca-certificates \
curl \
gnupg-agent \
software-properties-common &&
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - &&
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_realease -cs) stable" &&
sudo apt-get update -y &&
sudo sudo apt-get install docker-ce docker-ce-cli containered.io -y &&
sudo usermod -aG docker ubuntu
```

```python
resource "azurerm_linux_virtual_machine" "new-vm" {
  name                = "new-virtual-machine"
  resource_group_name = azurerm_resource_group.new-rg.name
  location            = azurerm_resource_group.new-rg.location
  size                = "Standard_F2"
  admin_username      = "adminuser"
  network_interface_ids = [
    azurerm_network_interface.new-nic.id,
  ]

  custom_data = filebase64("customdata.tpl")

  admin_ssh_key {
    username   = "adminuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }
 # code continues ...
```
>[!seealso]
>This is updated line of code while executing the VM again.
>Custom data is added with filebase64 and the path is not required to specify since the file is in the same directory.

### SSH configuration

```bash
#linux-ssh-script.tpl

cat << EOF >> ~/.ssh/config

Host ${hostname}
    HostName ${hostname}
    User ${user}
    IdentityFile ${identityfile}
EOF
```

### Provisioners 

```python

resource "azurerm_linux_virtual_machine" "new-vm" {
  name                = "new-virtual-machine"
  resource_group_name = azurerm_resource_group.new-rg.name
  location            = azurerm_resource_group.new-rg.location
  size                = "Standard_F2"
  admin_username      = "adminuser"
  network_interface_ids = [
    azurerm_network_interface.new-nic.id,
  ]
  
	 #code inbetween ...
	 
  custom_data = filebase64("customdata.tpl")
  provisioner "local-exec" {
    command = templatefile("linux-ssh-script.tpl", {
      hostname = self.public_ip_address,
      user = "adminuser",
      identityfile = "~/.ssh/id_rsa"
    })
    interpreter = [ "bash","-c" ]
  }
```
>[!important]
>What ever changes we make using a provisioner is not picked up my the terraform state file. When we run `terraform plan` we can see that “No changes” message is shown.
>In order to make the state file to notice the change, we have to destroy the instance and recreate.
> To do that use terraform apply replace flag and then run the command.
>`terraform apply -replace azurerm_linux_virtual_machine.new-vm `

>[!note]
>After the VM is initiated, click on the command pallet → enter remote ssh → select the host IP and give select the host IP address.

### Data Source

```python
data "azurerm_public_ip" "mtc-ip-data"{
  name = azurerm_public_ip.new-ip.name
  resource_group_name = azurerm_resource_group.new-rg.name
}
```
>[!info]
>The data source will fetch the values that we specify from the Azure console and display that in the statefile.
>Run the command `terraform refresh` and `terraform apply`

### Output

```python
output "public_ip_address" {
  value = "${azurerm_linux_virtual_machine.new-vm.name}: ${data.azurerm_public_ip.mtc-ip-data.ip_address}"
}
```

>[!info]
>We are now trying to use the data source in the output block.
>Now run the commands `terraform apply -refresh-only` and `terraform output`

### Variables

```python
  custom_data = filebase64("customdata.tpl")
  provisioner "local-exec" {
    command = templatefile("${var.host_os}-script.tpl", {
      hostname = self.public_ip_address,
      user = "adminuser",
      identityfile = "~/.ssh/id_rsa"
    })
    interpreter = [ "bash","-c" ]
  }
```

>[!note]
> Create a `variable.tf` file  and add the variable block
> 

```python
variable "host_os" {
  type = string
  default = "windows"
}
```
>[!important] 
>We need to now enter the value with `var.host_os`
>For example, if the `host name = “windows”`, after declaring the variable, it should be,
>`host name = var.host_os`

>[!info]
>The command `terraform console` will help us find values within the state file.
>For example if we type `terraform console var.host_os` we get result windows.

>[!note]
> In continuation to the last example instead of writing `default = "windows"`, this value can be stored in `terraform.tfvars` file
> In `terraform.tfvars` type  `host_os = “windows”`

>[!tip]
>Look more into `terraform console -var-file =”terraform.tfvars”`

### Conditionals

```python
condition ? true_val : false_val
```

>[!info]
>This is a ternary operator, first the condition must be specified and if its true the true value will be printed else the false value will be printed.

>[!example]
>`var.host_os == "linux" ? "bash" : "poweshell"`
>In the above code the if the var.host_os == linux the output will be linux. We shall give this condition to the interpreter that has both the ssh config files of windows and linux.

```python
interpreter = var.host_os == "windows" ? ["Powershell","-Command"] : ["bash", "-c"]
```



