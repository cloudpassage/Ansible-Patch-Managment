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

Three Ansible playbooks were created in this repository described as below:

**1.** first playbook named **“CPHALO_Update_Service-Running_Playbook.yml”** 
executes the following tasks:
- ensure cphalo is at the latest version for RHEL host
- ensure cphalo is at the latest version for Ubuntu host
- ensure cphalo service is running on both RHEL and Ubuntu hosts

```
---
  - name: CPHALO_Update_service-Running_Playbook
    hosts: all
    become: yes
    become_user: "<username>"
    become_method: sudo
    tasks:
      - name: ensure cphalo is at the latest version for RHEL host
        when: "'redhat_group' in group_names"
        yum:
          name: cphalo
          state: latest
      - name: ensure cphalo is at the latest version for Ubuntu host
        when: "'ubuntu_group' in group_names"
        apt:
          update_cache: yes
          name: cphalo
          state: latest
      - name: ensure cphalo service is running on both RHEL and Ubuntu hosts
        service:
          name: cphalod
          state: started
```

**2.** The second playbook named **“List_HALO_Group_SVA_Issues_Playbook.yml”** 
executes the following tasks:
- encode **api_key_id** and **api_key_secret** using **base64** encoding method and generate **access token**.
- filter and retrieve only **SVA** issues for the provided **Halo Group**.
- export retrieved **SVA** issues response into the **log file** "sva-issues-response.txt".

```
---
  - name: List_HALO_Group_SVA_Issues_Playbook
    hosts: all
    become: yes
    become_user: "<username>"
    become_method: sudo
    
    vars:
      api_key_id: "<halo_account_api_key_id>"
      api_key_secret: "<halo_account_api_key_secret>"
      api_key_id_secret_str: "{{api_key_id}}:{{api_key_secret}}"
      halo_group_id: "<halo_group_id>"
      issue_type: "sva"
      api_authentication_url: "https://api.cloudpassage.com/oauth/access_token?grant_type=client_credentials"
      api_svaissues_url: "https://api.cloudpassage.com/v3/issues?group_id={{halo_group_id}}&type={{issue_type}}"
    
    tasks:
      - name: encode api_key_id and api_key_secret using base64 encoding method
        shell: echo {{ api_key_id_secret_str | b64encode }}
        register: echo_content
          
      - name: generate access token
        uri:
          url: "{{api_authentication_url}}"
          method: POST
          headers:
            Authorization: "Basic {{echo_content.stdout}}"
        register: access_token_response
          
      - name: list halo group sva issues
        uri:
          url: "{{api_svaissues_url}}"
          method: GET
          headers:
            Authorization: "Bearer {{access_token_response.json.access_token}}"
            Content-Type: "application/json"
        register: sva_issues_response
        
      - name: export retreived sva issues response into the log file
        shell: echo {{ sva_issues_response }} > /home/ec2-user/sva-issues-response.txt
```

**3.** The third Ansible Playbook named **“Patch_HALO-Group_SVA_Issues_Playbook.yml”** 
used to Patch all SVA findings. 
it fixes all the reported SVA findings of specific server 
in a specific HALO group by executing the following tasks:

- encode **api_key_id** and **api_key_secret** using **base64** encoding method amd generate **access token**. 
- retrieve and export all **HALO group** **SVA** issues into a **log** file.
- save **packages names** that throws the **SVA** issues into Ansible **List of items**.
- loop on all the **list items** of the SVA findings and apply **update fix**.
- reboot ansible hosts to apply the **update fix**.

```
---
  - name: Patch_HALO-Group_SVA_Issues_Playbook
    hosts: all
    gather_facts: yes
    become: yes
    become_user: "<username>"
    become_method: sudo
    
    vars:
      api_key_id: "<halo_account_api_key_id>"
      api_key_secret: "<halo_account_api_key_secret>"
      api_key_id_secret_str: "{{api_key_id}}:{{api_key_secret}}"
      halo_group_id: "<halo_group_id>"
      issue_type: "sva"
      api_authentication_url: "https://api.cloudpassage.com/oauth/access_token?grant_type=client_credentials"
      api_halogroupsvaissues_url: "https://api.cloudpassage.com/v3/issues?group_id={{halo_group_id}}&type={{issue_type}}"
    
    tasks:
      - name: encode api_key_id and api_key_secret using base64 encoding method
        shell: echo {{ api_key_id_secret_str | b64encode }}
        register: echo_content
          
      - name: generate access token
        uri:
          url: "{{api_authentication_url}}"
          method: POST
          headers:
            Authorization: "Basic {{echo_content.stdout}}"
        register: access_token_response
          
      - name: list halo group sva issues
        uri:
          url: "{{api_halogroupsvaissues_url}}"
          method: GET
          headers:
            Authorization: "Bearer {{access_token_response.json.access_token}}"
            Content-Type: "application/json"
        register: halo_group_sva_issues_response
        
      - name: export retreived sva issues response into the log file
        shell: echo  {{ item.package_name }} >> /home/ec2-user/issues.txt
        with_items: "{{halo_group_sva_issues_response.json.issues}}"
        
      - name: fix all sva findings for .rpm servers
        when: "'redhat_group' in group_names"
        yum:
          update_cache: yes
          name: "{{ item.package_name }}"
          state: latest
        with_items: "{{halo_group_sva_issues_response.json.issues}}"
          
      - name: fix all sva findings for .deb servers
        when: "'ubuntu_group' in group_names"
        apt:
          update_cache: yes
          name: "{{ item.package_name }}"
          state: latest
        with_items: "{{halo_group_sva_issues_response.json.issues}}"       
          
      - name: restart system to reboot to newest kernel
        reboot:
          reboot_timeout: 3600
```
