# Ansible-Patch-Managment

## What is Ansible?
Ansible is an open source IT engine that automates application deployment, 
System Patch Management, cloud provisioning and many other IT tools. 
Ansible uses playbooks to describe automation jobs, 
and playbook written using **(YAML)** language.

## How Ansible Works?
Ansible works by connecting to hosts and pushing out small programs, 
called **"Ansible modules"** to them. 
Ansible then executes these modules (over **SSH** by default) and 
removes them when finished.

## Installing Ansible
To begin using Ansible as a means of managing server infrastructure, 
you need to install the Ansible software on the machine that will serve as 
the Ansible control node. We’ll use the default Ubuntu repositories for that.

First, refresh your system’s package index with: <br /> <br />
`$ sudo apt update`

Following this update, you can install the Ansible software with: <br /> <br />
`$ sudo apt install ansible`

Press Y when prompted to confirm installation.
Your Ansible control node now has all of the software required to administer your hosts.

## Setting Up the Inventory File
The inventory file contains information about the hosts you’ll manage with Ansible.
You can include anywhere from one to several hundred servers in your inventory file,
and hosts can be organized into groups and subgroups. 
The inventory file is also often used to set variables that will be valid only for
specific hosts or groups, in order to be used within playbooks and templates.

To edit the contents of your default Ansible inventory, 
open the **/etc/ansible/hosts** file using your text editor of choice, 
on your Ansible control node: <br /><br />

`$ sudo nano /etc/ansible/hosts`

The default inventory file provided by the Ansible installation contains a few examples
that you can use as references for setting up your inventory. 
The following example defines a group named **[redhat_group]** with **RHEL** server in it,
and another group named **[ubuntu_group]** with ubuntu server in it. 
Each server identified by a custom alias: **server1**, and **server2**. <br /><br />

`/etc/ansible/hosts` <br /><br /><br />
`[redhat_group]`<br />
`Server1 ansible_host=18.222.16.81`<br /><br />
`[ubuntu_group]`<br />
`#server2 ansible_host=3.133.12.16`<br /><br />
`[all:vars]`<br />
`ansible_python_interpreter=/usr/bin/python3`<br />

The **all:vars** subgroup sets the ansible_python_interpreter host parameter that will 
be valid for all hosts included in this inventory. 
This parameter makes sure the remote server uses 
the /usr/bin/python3 (Python 3) executable instead of /usr/bin/python (Python 2.7).

## Testing Connection
After setting up the inventory file to include your servers, 
it’s time to check if Ansible can connect to these servers and run commands via SSH.

From Ansible control node, run: <br />

`$ ansible all -m ping -u username`

This command will use Ansible’s built-in ping module to run a connectivity test on 
all nodes from the default inventory, connecting as ansible. 
The ping module will test:
-	if hosts are accessible.
-	if you have valid SSH credentials.
-	if hosts can run Ansible modules using Python.

You should get output like this:

`Output` <br /><br />
`server1 | SUCCESS => {`<br />
    `"changed": false, 
    "ping": "pong"
}`<br /><br />
`server2 | SUCCESS => {`<br />
    `"changed": false, 
    "ping": "pong"
}`

## Running Ad-Hoc Commands
After confirming that your Ansible control node can communicate with your hosts,
you can start running ad-hoc commands and playbooks on your servers.
Any command that you would normally execute on a remote server over SSH can be run 
with Ansible on the servers specified in your inventory file. 
As an example, you can check disk usage on all servers with:

`$ ansible all -a "df -h" -u ansible`

![Ad-Hoc Command Output](/images/kb1.png)

## Creating and Running Ansible Playbooks