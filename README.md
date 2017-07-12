### What is this
This bunch of playbooks will install a Docker Datacenter environment on top of vCenter on RHEL 7.3 VMs. It will install:

- 3 UCP nodes
- 3 DTR nodes
- 3 worker nodes
- 3 load balancer (one for each of the above trio)

The playbooks are meant to support a different number of nodes (ie. 5 DTR nodes rather than 3) but so far it's only been tested with 3 every time.

The following instructions assume some basic knowledge on vCenter/Linux.

Also, please keep in mind that you need a Docker license (you can request a 1-month trial here: https://store.docker.com/editions/enterprise/docker-ee-server-rhel)

### How to use it
You need to configure two things first:

- A VM template
- An Ansible host

#### Configuring the VM Template
Create a VM with RHEL 7.3 server, with minimal configuration and just one NIC. It requires just one disk (the playbooks will add a second one on the required nodes). Once the OS is installed, reboot and configure a repository. You can do this in two ways:
- Connecting to the RHN network
- Creating a local repo from DVD (https://access.redhat.com/solutions/1355683)
- Using an external repo
Leave the VM on until you've configured the Ansible node.

### Configuring the ansible node
I'm assuming you're using a RHEL/CentOS server here but it can be done in other OSes too. Just 'translate' the steps below when required (ie. apt instead of yum and so on).
- Install ansible : http://docs.ansible.com/ansible/intro_installation.html
- Populate the /etc/hosts file so you can resolve the Docker datacenter nodes
- Run `yum install python-pyvmomi python-netaddr`. This is required for some of the playbooks
- Run `pip install cryptography`. This is required for ansible vault. You might need to install some dependencies first, including pip: `yum install python-pip gcc python-devel openssl-devel`
- Run `pip install pysphere`. Required to work with vCenter/vSphere
- Copy your id into the template (`ssh-copy-id <template_IP>`)
- Clone this repository using `git clone https://github.com/ophintor/ansible-docker-ucp.git`
- Edit vm_hosts with the right names and ip addresses to match your environment (do not modify the group names - ie. dtr_lb, etc.). The group names are ~~hardcoded~~ used inside the playbooks
- Edit group_vars/vars with your own environment details
- Create an vault to store your sensitive details as passwords, as follows:
- Create (or overwrite the `group_vars/vault` file with the below:
```
---
vcenter_pass: 'yourpassword'
vm_pw: 'yourpassword'
```
- Encrypt it by running `ansible-vault encrypt vault group_vars/vault`; enter your password when prompted
- Create a file called .vault_pass containing the password and change permissions to 0600
- Optionally remove it from your git list using .gitignore

### Complete the template creation
Just clean up:
- Run `rm -f /etc/ssh/ssh_host_*`
- Run `history -c`
- Shutdown
- In vSphere, right click on your VM and convert to template.

### Run the playbooks
From inside the cloned folder, run the playbook as follows:
`ansible-playbook -i vm_hosts site.yml --vault-password-file .vault_pass`
It should take about 25 minutes to run.

### Known issues
- Your FQDN should resolve using DNS or else they will fail when trying to set up your DTR. A solution to this is either using your own publicly accessible domain or this: https://github.com/bobfraser1/alpine-router
- If you comment all the env variables you will get plenty of warnings. To avoid that I have added a dummy variable that should be left there. I need to investigate a more elegant way to workaround this.

### TO-DO
- The playbooks will install a vsphere plugin so you can create volumes directly in your datastores, but for this you need to install the latest release of vDVS driver VIB. This has been done manually so far. More info: https://blogs.vmware.com/virtualblocks/2017/03/29/vsphere-docker-volume-service-now-docker-certified
- Test 5 nodes rather than 3
