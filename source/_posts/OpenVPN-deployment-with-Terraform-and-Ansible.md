---
title: OpenVPN deployment with Terraform and Ansible
date: 2020-09-24 20:00:00
tags: [automation, ansible, terraform]
---

As the number of machines and services that you manage increases, manually creating and configuring your infrastructure is not scalable. As such, provisioning and configuration management tools, such as [Terraform](https://www.terraform.io) and [Ansible](https://www.ansible.com) are extremely useful. I'm going to show how to automate the deployment of [OpenVPN](https://openvpn.net) on [Azure](https://azure.microsoft.com).

## OpenVPN

Occasionally, I want to mask my IP address to unlock geo-restricted content. Instead of paying a monthly fee to a VPN provider, a viable alternative is to create a temporary virtual machine on a cloud provider, install OpenVPN, and then destroy it, which is cheap. With automation, this process is very fast. If you want to learn more about OpenVPN and Public Key Infrastructure (PKI), check this [book](https://www.packtpub.com/product/mastering-openvpn/9781783553136). For production use, there are additional security steps that should be taken, especially on the PKI side.

You can follow these instructions to deploy your own OpenVPN server:

1. Clone the [GitHub repository](https://github.com/ruial/openvpn-demo)
2. Generate an SSH key (*ssh-keygen*) or set an admin password for the server
3. Create infrastructure using Terraform (check README)
4. Edit the inventory.ini or your */etc/hosts* to include the server IP
5. Run the Ansible playbook (check README)
6. Start OpenVPN client (*openvpn --config playbooks/out/client-host.conf*)
7. Destroy infrastructure when done (check README)

## Demo project structure

```
├── playbooks
│   ├── ansible.cfg         - ansible configuration file
│   ├── handlers            - handlers for events
│   ├── inventory.ini       - inventory of servers
│   ├── main.yml            - main playbook (the entrypoint)
│   ├── out                 - output folder for client config
│   ├── tasks
│   │   ├── easyrsa.yml     - easyrsa config tasks
│   │   └── openvpn.yml     - openvpn config tasks
│   └── templates
│       ├── client.conf.j2  - openvpn client config template
│       └── server.conf.j2  - openvpn server config template
├── provision
│   ├── instance.tf         - virtual machine and network association
│   ├── main.tf             - provider definition and resource group
│   ├── network.tf          - virtual network, subnet and security group
│   ├── output.tf           - output variables (IP address)
│   ├── terraform.tfstate   - automatically generated tfstate
│   └── vars.tf             - input variables that can be overridden
└── readme.md               - project info and instructions
```

## Terraform

Terraform is the most popular tool to provision resources on public clouds. The infrastructure is defined with a declarative domain-specific-language (DSL) called HashiCorp Configuration Language (HCL). It saves the infrastructure state in a local file, provides locking and allows to plan the changes before applying them. When working with multiple people, [remote state](https://www.terraform.io/docs/state/remote.html) should be used to prevent concurrent runs.

As each cloud provider is different, there are multiple terraform [providers](https://www.terraform.io/docs/providers/azurerm). With Terraform we can keep the entire infrastructure as code and others can review changes, which helps reduce human errors. Initial configurations of the operating system (OS) can be specified, which is helpful. Many companies choose to build their own OS image. Here's a sample of a terraform config file:

```python instance.tf
resource "azurerm_virtual_machine" "demo-instance" {
  name                  = "${var.prefix}-vm"
  location              = var.location
  resource_group_name   = azurerm_resource_group.demo.name
  network_interface_ids = [azurerm_network_interface.demo-instance.id]
  vm_size               = "Standard_B1ls"

  delete_os_disk_on_termination = true
  delete_data_disks_on_termination = true

  storage_image_reference {
    publisher = "OpenLogic"
    offer     = "CentOS"
    sku       = "7.7"
    version   = "latest"
  }
  storage_os_disk {
    name              = "${var.prefix}-osdisk1"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
  }
  os_profile {
    computer_name  = "demo-instance"
    admin_username = "demo"
    #admin_password = "..."
  }
  os_profile_linux_config {
    disable_password_authentication = true
    ssh_keys {
      key_data = file("~/.ssh/id_rsa.pub")
      path     = "/home/demo/.ssh/authorized_keys"
    }
  }
}
...
```

## Ansible

Ansible is an agentless configuration management tool, which uses SSH to push and execute playbooks. Playbooks contain sequences of tasks and should be idempotent. For reusable code check [Ansible Roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html). I've developed a playbook to install OpenVPN on CentOS:

```yaml main.yml
---
- hosts: openvpn
  become: yes

  tasks:
  - name: install packages
    package:
      name: "{{ item }}"
      state: present
    loop:
      - epel-release
      - firewalld
      - easy-rsa
      - openvpn

  - name: easy-rsa
    include_tasks: tasks/easyrsa.yml

  - name: openvpn
    include_tasks: tasks/openvpn.yml

  handlers:
    - include: handlers/main.yml
```

I didn't include all the tasks here because they would take too much space. At first, some required packages are installed and then *easy-rsa* is used to generate server and client certificates. Then, OpenVPN and the firewall are configured. Lastly, any changes to the server config will trigger a service reload and with the client config automatically transferred, you are able to use your VPN.

## Closing thoughts

There is a lot to say about infrastructure and application deployment. Immutable vs mutable infrastructure, push vs pull configuration management, kubernetes and containers vs virtual machines, cloud managed services vs self hosted, but I leave all these for another day.
