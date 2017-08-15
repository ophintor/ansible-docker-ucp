# Introduction

The present document describes how to automate the provisioning of a Docker Enterprise Edition environment by using a set of Ansible playbooks. It also outlines a set of manual steps to harden, secure and audit the overall status of the system.

## About Docker Enterprise Edition

Docker Enterprise Edition (EE) is designed for enterprise development and IT teams who build, ship and run business critical applications in production at scale. Docker EE is integrated, certified and supported to provide enterprises with the most secure container platform in the industry to modernize all applications. An application-centric platform, Docker EE is designed accelerate and secure across the entire software supply chain, from development to production running on any infrastructure.

More information about Docker Enterprise Edition can be found here: [https://www.docker.com/enterprise-edition](https://www.docker.com/enterprise-edition)

## About Simplivity

Simplivity is an enterprise-grade hyper-converged platform uniting best-in-class data services with the world&#39;s bestselling server.

Rapid proliferation of applications and the increasing cost of maintaining legacy infrastructure causes significant IT challenges for many organisations. With HPE SimpliVity, you can streamline and enable IT operations at a fraction of the cost of traditional and public cloud solutions by combining your IT infrastructure and advanced data services into a single, integrated solution. HPE SimpliVity is a powerful, simple, and efficient hyperconverged platform that joins best-in-class data services with the world's best-selling server and offers the industry's most complete guarantee.

More information about Simplivity can be found here: [https://www.hpe.com/us/en/integrated-systems/simplivity.html](https://www.hpe.com/us/en/integrated-systems/simplivity.html)

## Assumptions

The present document assumes a minimum understanding in concepts like virtualization, containerization and some knowledge around Linux and VMWare technologies.

## Required Versions

The following versions or higher are required to use the playbooks described in later sections.

- Ansible 2.2
- Docker EE 17.03

# Steps to provision the environment

This section will describe in detail how to provision the environment described in the previous section.

## Creation of a VM template

The very first step to our automated solution will be the creation of a VM Template that will be the base of all your nodes. In order to create a VM Template we will first create a Virtual Machine where the OS will be installed and then the Virtual Machine will be converted to a VM Template. Since the goal of the automation is get rid of as many repetitive tasks as possible, the VM Template will be created as lean as possible, so any additional software installs and/or system configuration will be done later by Ansible.

Of course, the creation of the template could have been automated too, but since this is a one-off task, it's been decided to do it manually since it takes less time than building another playbook and thinking about the possible options and dependencies.

The steps to create a VM template are described below.

1. Log in to vCenter and create a new Virtual Machine. 
1. Provide a name for your template.
1. Choose the location (host/cluster) where you wish to store your template.
1. Choose a datastore where the template files will be stored.
1. Choose the OS, in this case Linux, RHEL7 64bit.
1. Pick the network to attach to your template. In this example we&#39;re only using one NIC but depending on how you plan to architect your environment you might want to add more than one.
1. Create a primary disk. The chosen size in this case is 50GB but 20GB should be typically enough.
1. Confirm that the settings are right and press Finish.
1. The next step is to virtually insert the RHEL7 DVD, we do this by looking into the Settings of the newly created VM. Select your ISO file in the Datastore ISO File Device Type and make sure that the &quot;Connect at power on&quot; checkbox is checked.
1. Finally, you can optionally remove the Floppy Disk as this is not required for the VM.
1. Power on the server and open the console to install the OS. You should see something similar to this. Pick your language and hit Continue.
1. Scroll down and click on Installation Destination.
1. Select your installation drive and hit Done.
1. Leave all the rest by default. Click Begin Installation.
1. Select a root password.
1. Press Done and wait for the install to finish. Reboot and login into the system using the VM console.
2. Once we are logged into the system we need to configure a yum repository so we can install any packages required at a later stage. We can do this in three different ways:

**Option 1:** Use Red Hat subscription manager to register your system. This is the easiest way and will give you automatically access to the official Red Hat repositories. It requires having a Red Hat Network account though, so if you don&#39;t have one, you can use either Option 2 or 3. For option run you would do:

```# subscription-manager register --auto-attach```

If you are behind a proxy you will need to run this beforehand:

```# subscription-manager config --server.proxy_hostname=<proxy IP> --server.proxy_port=<proxy port>```

**Option 2:** Use a local repository by copying the DVD contents into Virtual Machine primary disk. The procedure on how to do this is explained here: [https://access.redhat.com/solutions/1355683](https://access.redhat.com/solutions/1355683)

**Option 3:** Use an internal repository. Instead of pulling the packages locally after copying the DVD contents into the local drive, you could have a dedicated node with the Red Hat packages available and configure the repository to pull the packages from there. Your `/etc/yum.repos.d/redhat.repo` could look something like this:

```
[internal-rhel7-repo]
name = Internal RHEL7 repository
baseurl = http://redhat-node.your.domain/your/packages/directory
enabled = 1
gpgcheck = 0
```

For additional information on how to configure a repository please check the Red Hat documentation: [https://access.redhat.com/documentation/en-US/Red\_Hat\_Enterprise\_Linux/7/html/System\_Administrators\_Guide/sec-Configuring\_Yum\_and\_Yum\_Repositories.html](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/System_Administrators_Guide/sec-Configuring_Yum_and_Yum_Repositories.html)

At this stage the only thing left to do is to power off the Virtual Machine and convert it to a VM Template, but before we do that we need to set up our Ansible host, as explained in the next section.

## Creating the Ansible node

In addition to the VM Template, we need another Virtual Machine where Ansible will be installed. This node will act as the driver to automate the provision of the environment and is essential that it is properly installed. The steps are as follows:

1. Create a Virtual Machine and install your preferred OS (in this example, and for the sake of simplicity, RHEL7 will be used). The rest of the instructions assume that, if the user were to use a different OS, he understands the possible differences in syntax for the provided commands.
2. Configure a repository as explained in the previous section and install Ansible. Please note that version 2.2 is the minimum required. The instructions on how to install Ansible are described in its official website: [http://docs.ansible.com/ansible/intro\_installation.html](http://docs.ansible.com/ansible/intro_installation.html)
3. Make a list of all the hostnames and IPs that will be in your system and update your /etc/hosts accordingly. This includes your UCP nodes, DTR nodes, worker nodes, NFS server, logger server and load balancers.
4. Install the following packages. They are a mandatory requirement for the playbooks to function as expected. Update pip if requested:
`# yum install python-pyvmomi python-netaddr python2-jmespath python-pip gcc python-devel openssl-devel git`
`# pip install cryptography`
`# pip install pysphere`

5. Copy your SSH id to the VM Template so, in the future, your Ansible node can SSH without the need of a password to all the Virtual Machines created from the VM Template.
`# ssh-copy-id root@<VM_Template>`
Please note that in both the Ansible node and the VM Template you might need to configure the network so one node can reach the other. Since this is a basic step and could vary on the user&#39;s environment I have purposefully omitted it.

6. Retrieve the latest version of the playbooks using git.
`# git clone https://github.com/ophintor/ansible-docker-ucp.git`

## Finalize the template

Now that the VM Template has the public key of the Ansible node, we're ready to convert this VM to a VM Template. Perform the following steps in the VM Template to finalize its creation:

1. Clean up the template by running the following commands:
`# rm /etc/ssh/ssh_host_*`
`# history -c`
2. Shut down the VM
`# shutdown –h now`

1. Once the Virtual Machine is ready and turned off, we are ready to convert it to a template.

This completes the creation of the VM Template.

## Prepare your Ansible configuration

In the Ansible node, we need now to prepare the configuration to match your own environment, prior to deploying Docker Datacenter and the rest of the nodes. To do so, we will need to edit and modify three different files:

- The inventory: vm_hosts
- The group variables: group_vars/vars
- The encrypted group variable: group_vars/vault

## Editing the inventory

Change to the directory that you previously cloned using git and edit the vm\_hosts file.

The nodes inside the inventory are organized in groups. The groups are defined by brackets and its names are static so they must not be changed. Anything else (hostnames, specifications, IP addresses…) are meant to be amended to match the user needs. The groups are as follows:

- [ucp\_main]: A group containing one single node which will be the main UCP node and swarm leader. Do not add more than one node under this group.
- [ucp]: A group containing all the UCP nodes, including the main UCP node. Typically you should have either 3 or 5 nodes under this group.
- [dtr\_main]: A group containing one single node which will be the first DTR node to be installed. Do not add more than one node under this group.
- [dtr]: A group containing all the DTR nodes, including the main DTR node. Typically you should have either 3 or 5 nodes under this group.
- [worker]: A group containing all the UCP nodes, including the main UCP node. Typically you should have either 3 or 5 nodes under this group.
- [ucp\_lb]: A group containing one single node which will be the load balancer for the UCP nodes. Do not add more than one node under this group.
- [dtr\_lb]: A group containing one single node which will be the load balancer for the DTR nodes. Do not add more than one node under this group.
- [worker\_lb]: A group containing one single node which will be the load balancer for the worker nodes. Do not add more than one node under this group.
- [lbs]: A group containing all the load balances. This group will have 3 nodes, also defined in the three groups above.
- [nfs]: A group containing one single node which will be the NFS node. Do not add more than one node under this group.
- [logger]: A group containing one single node which will be the logger node. Do not add more than one node under this group.
- [local]: A group containing the local Ansible host. It contains an entry that should not be modified.

There are also a few special groups:

- [docker:children]: A group of groups including all the nodes where Docker will be installed.
- [vms:children]: A group of groups including all the Virtual Machines involved apart from the local host.

Finally, you will find some variables defined for each group:

- [ucp:vars]: A set of variables defined for all nodes in the [ucp] group.
- [dtr:vars]: A set of variables defined for all nodes in the [dtr] group.
- [worker:vars]: A set of variables defined for all nodes in the [worker] group.
- [lbs:vars]: A set of variables defined for all nodes in the [lbs] group.
- [nfs:vars]: A set of variables defined for all nodes in the [nfs] group.
- [logger:vars]: A set of variables defined for all nodes in the [logger] group.

If you wish to configure your nodes with different specifications rather than the ones defined by the group, it is possible to declare the same variables at the node level, overriding the group value. For instance, you could have one of your workers with higher specifications by doing:

```
[worker]
worker01 ip_addr='10.0.0.10/16' esxi_host='esxi1.domain.local'
worker02 ip_addr='10.0.0.11/16' esxi_host='esxi1.domain.local'
worker03 ip_addr='10.0.0.12/16' esxi_host='esxi1.domain.local' cpus='16' ram'32768'
[worker:vars]
cpus='4&'
ram=&#39;16384'
disk2_size='200'
node_policy='bronze'
```
In  the example above, the worker03 node would have 4 times more CPU and double RAM than the rest of worker nodes.

The different variables you can use are as described in the table below. They are all mandatory unless if specified otherwise:
| Variable | Scope | Description |
| --- | --- | --- |
| ip\_addr | Node | IP address in CIDR format to be given to a node |
| esxi\_host | Node | ESXi host where the node will be deployed. Please note that if the cluster is configured with DRS, this option will be overriden |
| cpus | Node/Group | Number of CPUs to assign to a VM or a group of VMs |
| RAM | Node/Group | Amount of RAM in MB to assign to a VM or a group of VMs |
| disk2\_usage | Node/Group | Size of the second disk in GB to attach to a VM or a group of VMs. This variable is only mandatory on Docker nodes (UCP, DTR, worker) and NFS node. It is not required for the logger node or the load balancers. |
| node\_policy | Node/Group | Simplivity backup policy to assign to a VM or a group of VMs. The name has to match one of the backup policies defined in the group\_vars/vars file described in the next section |

## Editing the group variables

Once our inventory is ready, the next step is to modify the group variables to match our environment. To do so, we need to edit the file group\_vars/vars under the cloned directory containing the playbooks. The variables here can be defined in any order but for the sake of clarity they have been divided into sections.

### VMware configuration

All VMware-related variables should be here. All of them are mandatory and described in the Table 2 below.

| Variable | Description |
| --- | --- |
| vcenter\_hostname | IP or hostname of the vCenter appliance |
| vcenter\_username | Username to log in to the vCenter appliance. It might include a domain i.e. &#39; [administrator@vsphere.local](mailto:administrator@vsphere.local)&#39; |
| datacenter | Name of the datacenter where the environment will be provisioned |
| vm\_username | Username to log into the VMs. It needs to match the one from the VM Template, so unless you have created an user, you must use &#39;root&#39; |
| vm\_template | Name of the VM Template to be used. Note that this is the name from a vCenter perspective, not the hostname |
| folder\_name | vCenter folder to deploy the VMs. If you do not wish to deploy in a particular folder, the value should be &#39;/&#39; |
| datastores | List of datastores to be used, in list format, i.e. [&#39;Datastore1&#39;,&#39;Datastore2&#39;...]. Please note that from a Simplivity perspective it&#39;s best practice to use just one Datastore. Using more than one will not provide any advantages in terms of reliability and will add additional complexity. |
| disk2 | UNIX name of the second disk for the Docker VMs. Typically &#39;/dev/sdb&#39; |
| disk2\_part | UNIX name of the partition of the second disk for the Docker VMs. Typically &#39;/dev/sdb1&#39; |
| vsphere\_plugin\_version | Version of the vSphere plugin for Docker. The default is &#39;latest&#39; but you could pick a specific version, i.e. &#39;0.12&#39; |

### Simplivity configuration

All Simplivity-related variables should be here. All of them are mandatory and described in the Table 3 below.

| Variable | Description |
| --- | --- |
| simplivity\_username | Username to log in to the Simplivity Omnistack appliances. It might include a domain i.e. &#39; [administrator@vsphere.local](mailto:administrator@vsphere.local)&#39; |
| omnistack\_ovc | List of Omnistack hosts to be used, in list format, i.e. [&#39;omni1.local&#39;,&#39;onmi2.local&#39;...] |
| backup\_policies | List of dictionaries containing the different backup policies to be used along with the scheduling information. Any number of backup policies can be created and they need to match the node\_policy variables defined in the inventory. The format is as follows:backup\_policies: - name: daily&#39;   days: &#39;All&#39;   start\_time: &#39;11:30&#39;   frequency: &#39;1440&#39;   retention: &#39;10080&#39; - name: &#39;hourly&#39;   days: &#39;All&#39;   start\_time: &#39;00:00&#39;   frequency: &#39;60&#39;   retention: &#39;2880&#39; |
| dummy\_vm\_prefix | In order to be able to backup the Docker volumes, a number of &quot;dummy&quot; VMs need to be spin up. This variable will set a recognizable prefix for them |
| docker\_volumes\_policy | Backup policy to use for the Docker Volumes |

### Networking configuration

All network-related variables should be here. All of them are mandatory and described in the Table 4 below.

| Variable | Description |
| --- | --- |
| nic\_name | Name of the device, for RHEL this is typically ens192 and it is recommended to leave it as is |
| gateway | IP address of the gateway to be used |
| dns | List of DNS servers to be used, in list format, i.e. [&#39;8.8.8.8&#39;,&#39;4.4.4.4&#39;...] |
| domain\_name | Domain name for your Virtual Machines |
| ntp\_server | List of NTP servers to be used, in list format, i.e. [&#39;1.2.3.4&#39;,&#39;0.us.pool.net.org&#39;...] |

### Docker configuration

All Docker-related variables should be here. All of them are mandatory and described in the Table 5 below.

| Variable | Description |
| --- | --- |
| docker\_ee\_url | URL to your Docker EE packages |
| rhel\_version | Version of your RHEL OS, i.e: 7.3 |
| dtr\_version | Version of the Docker DTR you wish to install. You can use a numeric version or latest for the most recent one |
| ucp\_version | Version of the Docker UCP you wish to install. You can use a numeric version or latest for the most recent one |
| images\_folder | Directory in the NTP server that will be mounted in the DTR nodes and that will host your Docker images |
| license\_file | Full path to your Docker license file (it should be stored in your Ansible host) |
| ucp\_username | Username of the administrator user for UCP and DTR, typically admin. |

### Monitoring configuration

All Monitoring-related variables should be here. This section only include versions and it is recommended to leave it as is. All of them are specified in the Table 6 below.

| Variable | Description |
| --- | --- |
| cadvisor\_version | You could try a different version but it&#39;s not guaranteed that it will work. To make sure that no issues arise please use &#39;v0.25.0&#39; |
| node\_exporter\_version | You could try a different version but it&#39;s not guaranteed that it will work. To make sure that no issues arise please use &#39;v1.14.0&#39; |
| prometheus\_version | You could try a different version but it&#39;s not guaranteed that it will work. To make sure that no issues arise please use &#39;v1.7.1&#39; |
| grafana\_version | You could try a different version but it&#39;s not guaranteed that it will work. To make sure that no issues arise please use &#39;4.4.3&#39; |

### Environment configuration

All Environment-related variables should be here. All of them are described in the Table 7 below.

| Variable | Description |
| --- | --- |
| env | Dictionary containing all environment variables. It contains four entries described below. Please comment out the proxy related settings if not required:<ul><li>http\_proxy: HTTP proxy URL, i.e. &#39;http://15.184.4.2:8080&#39;. This variable is optional and only necessary if your environment is behind a proxy.</li><li>https\_proxy: HTTP proxy URL, i.e. &#39;http://15.184.4.2:8080&#39;. This variable is optional and only necessary if your environment is behind a proxy.</li><li>no\_proxy: List of hostnames or IPs that don&#39;t require proxy, i.e. &#39;localhost,127.0.0.1,.cloudra.local,10.10.174.&#39;. This variable is optional and only necessary if your environment is behind a proxy.</li><li>foo: Dummy variable that only exists to keep env not empty in case a proxy is not used. When env is empty they playbooks will still work but plenty of warnings will arise due to the env variable being empty</li></ul>|

## Editing the vault

Once our group variables file is ready, the next step is to create a vault file to match our environment. The vault file is essentially the same thing than the group variables but it will contain all sensitive variables and will be encrypted.

To create a vault we&#39;ll create a new file group\_vars/vault and we&#39;ll add the following entries:
```
---
vcenter_password: 'xxx'
vm_password: 'xxx'
simplivity_password: 'xxx'
ucp_password: 'xxx'
```
To encrypt the vault you need to run the following command:
`# ansible-vault encrypt group_vars/vault`

You will be prompted for a password that will decrypt the vault when required.

Edit your vault anytime by running:
`# ansible-vault edit group_vars/vault`

The password you set on creation will be requested.

In order for ansible to be able to read the vault we&#39;ll need to specify a file where the password is stored, for instance in a file called .vault\_pass. Once the file is created, take the following precautions to avoid illegitimate access to this file:

1. Change the permissions so only root can read it
`# chmod 600 .vault_pass`

1. Add the file to your .gitignore file if you&#39;re pushing the set of playbooks to a git repository

# Running the playbooks

So at this point the system is ready to be deployed. Go to the root folder and run the following command:
`# ansible-playbook -i vm_hosts site.yml --vault-password-file .vault_pass`

The playbooks should run for 25-35 minutes depending on your server specifications and in the size of your environment.

## Scaling out your environment

The playbooks are idempotent, which means that they can be run over and over but only the changes that are found from the previous run will be applied. In a second or subsequent run you might see some errors in the output but these are normal and can be safely ignored.

The reasoning behind providing idempotency is that you would be able to scale out your system if you wished to do so. If you had deployed an environment with 3 workers, 3 DTRs and 3 UCPs and you wanted to have 5 of each, you would just need to amend your inventory (vm\_hosts file) and run the above command again.

# Deep dive into the playbooks

This section will go more in detail about how the playbooks work and what functionalities are being provided.

## site.yml

The site.yml playbook is the main playbook and will contain a list of playbooks to be run sequentially. The list of playbooks is described in the following sections in running order:

## playbooks/create\_vms.yml

This playbook will create all the necessary Virtual Machines for the environment from the VM Template defined in the vm\_template variable.

It is composed of the following sequential tasks:

- Create all VMs: Creates all the required VMs from the specified templates in the specified folder. Each VM will be hosted by the specified esxi\_host defined in the inventory, unless if VMWare DRS technology is used, in which case the setting will be ignored and DRS will decide how to spread the load. The Virtual Machines will be powered on upon creation.
- Add secondary disks: Adds a second disk on the VMs having the disk2\_size variable defined (usually the Docker nodes and the NFS server, but not the load balancers or the logger node). The datastore will be chosen at random from the list provided in the group variables.
- Wait for VMs to boot up: Waits for 3 minutes for the VMs to be completely booted up.

The first two tasks make use of the vmware\_guest Ansible module. More information about the module can be found here: [http://docs.ansible.com/ansible/latest/vmware\_guest\_module.html](http://docs.ansible.com/ansible/latest/vmware_guest_module.html)

## playbooks/config\_networking.yml

This playbook will configure the network settings in all Virtual Machines.

It is composed of the following sequential tasks:

- Change hostname: Updates the /etc/hostname file with the hostname and domain name provided in the group variables.
- Update hostname: Makes the new hostname current by using the hostnamectl tool
- Add new connection: Adds a new network connection using the nmcli tool along with the network information provided in the group variables and inventory (IP address, gateway, interface name, etc.)
- Bring connection up: Brings up the newly created connection.
- Enable connection autoconnect: Makes sure the connection is brought up after a reboot.
- Update /etc/hosts: Updates the /etc/hosts file using a template. The template file is called j2 and is located in the templates folder. The template will loop over all the nodes defined in the inventory to have a complete and up-to-date hosts file. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html)
- Update DNS settings: Updates the /etc/resolv.conf file using a template. The template file is called conf.j2 and is located in the templates folder. The template will loop over all the DNS hosts defined in the group variables to have a complete conf file. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html)

Since at the beginning of the playbook the Virtual Machines don&#39;t have yet network connectivity, in most tasks we will make use of the vmware\_vm\_shell Ansible module, that allows us to run shell commands directly on the Virtual Machines. More information about this module can be found here: [http://docs.ansible.com/ansible/latest/vmware\_vm\_shell\_module.html](http://docs.ansible.com/ansible/latest/vmware_vm_shell_module.html)

## playbooks/distribute\_keys.yml

This playbook is optional and will distribute all nodes&#39; public keys around so all nodes can password-less login to one another. Since this could be seen as a security risk (there is technically no need for a worker node user to log into an UCP node, for instance), it is disabled by default but can be uncommented for the site.yml file if required.

It is composed of the following sequential tasks:

- Register key: Stores the default SSH private key (/root/.ssh/id\_rsa)
- Create keypairs: Creates SSH key pairs in the nodes where these didn&#39;t yet exist, using the ssh-keygen tool.
- Fetch all public ssh keys: Registers all public keys from all nodes.
- Deploy keys on all servers: Pushes all keys to all nodes using a nested loop.

## playbooks/install\_haproxy.yml

This playbook will install and configure the haproxy package in the load balancer nodes. haproxy is the chosen tool to implement load balancing between UCP nodes, DTR nodes and worker nodes.

It is composed of the following sequential tasks:

- Open http and https ports: Makes use of the firewalld command to open the required ports tcp/80 and tcp/443
- Reload firewalld configuration: Reloads the firewall to apply the new rules
- Install haproxy: Installs the latest version of haproxy using the Ansible yum module.
- Update haproxy.cfg on Worker Load Balancer: Updates the /etc/haproxy/haproxy.cfg file using a template on the Worker load balancer. The template file is called worker.j2 and is located in the templates folder. The template will loop over all the Worker nodes defined in the inventory to have a complete list of hosts in the backend. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the haproxy service if a change in the file has occurred.
- Update haproxy.cfg on UCP Load Balancer: Updates the /etc/haproxy/haproxy.cfg file using a template on the Worker load balancer. The template file is called ucp.j2 and is located in the templates folder. The template will loop over all the UCP nodes defined in the inventory to have a complete list of hosts in the backend. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the haproxy service if a change in the file has occurred.
- Update haproxy.cfg on DTR Load Balancer: Updates the /etc/haproxy/haproxy.cfg file using a template on the Worker load balancer. The template file is called dtr.j2 and is located in the templates folder. The template will loop over all the DTR nodes defined in the inventory to have a complete list of hosts in the backend. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the haproxy service if a change in the file has occurred.

This playbook also contains a handler, which is a task that will be run when notified by one or more of the specified tasks above. The goal of having a handler is typically to restart a service only when a configuration file has been changed.

- Enable and restart haproxy service: Makes sure that the service is enabled and restarted.

## playbooks/install\_ntp.yml

This playbook will install and configure the ntp package in all Virtual Machines in order to have a synchronized clock all across the environment. It will use the server or servers specified in the ntp\_servers variable in the group variables file.

It is composed of the following sequential tasks:

- Install ntp: Updates the /etc/haproxy/haproxy.cfg file using a template on the Worker load
- Update ntp.conf: Updates the /etc/ntp.conf file using a template. The template file is called conf.j2 and is located in the templates folder. The template will loop over all the provided NTP servers defined in the ntp\_servers variable. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the ntp service if a change in the file has occurred.

This playbook also contains a handler, which is a task that will be run when notified by one or more of the specified tasks above. The goal of having a handler is typically to restart a service only when a configuration file has been changed.

- Enable and restart ntp service: Makes sure that the service is enabled and restarted.

## playbooks/install\_docker.yml

This playbook will install Docker along with all its dependencies.

It is composed of the following sequential tasks:

- Install dependencies: Uses yum to install all the packages that are required by Docker.
- Set Docker url: Updates the /etc/yum/vars/dockerurl file with the contents of the docker\_ee\_url variable.
- Set Docker version: Updates the /etc/yum/vars/dockerosversion file with the contents of the rhel\_version variable.
- Add Docker repository: Runs yum-config-manager in order to add the yum repository specified in the docker\_ee\_url variable.
- Install Docker: Uses yum to install the latest version of Docker Enterprise Edition.

## playbooks/install\_rsyslog.yml

This playbook will install and configure rsyslog in the logger node and in all Docker nodes. The logger node will be configured to receive all syslogs on port 514 and the Docker nodes will be configured to send all logs (including container logs) to the logger node.

It is composed of the following sequential tasks:

- Open required ports for rsyslog: Makes use of the firewalld command to open the required ports tcp/514 and ucp/514 on the logger node
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Install rsyslog: Install the latest version of rsyslog on all docker and logger nodes
- Configure logger server: Updates the /etc/rsyslog.conf configuration file on the logger node with an updated file that allows to receive logs on port 514 using both UCP and TCP protocols. To achieve this, an amended conf file, stored under the files folder, is copied over to the logger server. This task will notify the handler defined below in order to restart the rsyslog service if a change in the file has occurred.
- Allow docker nodes to send logs: Updates the /etc/rsyslog.conf file using a template to allow sending logs to the logger node. The template file is called conf.j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the rsyslog service if a change in the file has occurred.

This playbook also contains a handler, which is a task that will be run when notified by one or more of the specified tasks above. The goal of having a handler is typically to restart a service only when a configuration file has been changed.

- Restart Rsyslog: Makes sure that the rsyslog service is enabled and restarted.

## playbooks/config\_docker\_lvs.yml

This playbook will perform a set of operations on the Docker nodes in order to create a partition on the second disk and carry out the LVM configuration, required for a sound Docker installation.

It is composed of the following sequential tasks:

- Create partition on second disk: Uses the Ansible parted module to create a LVM partition on the second disk. The partition will have a GPT label and will use the 100% of the drive. The directive ignore\_errors present in this task (and in some of the following ones) exists in order to allow for the playbook to be run more than once, for instance if we&#39;re running yml again to scale out our environment. Since the partition will be already created in this case scenario, the task will fail but will be safely ignored. Unfortunately, Ansible does not provide, as of today, with a more elegant method to allow for this module to be idempotent when using GPT labels.
- Create Docker VG: Creates a Volume Group called docker in the newly created partition.
- Create thinpool LV: Creates a Logical Volume called thinpool in the docker Volume Group.
- Create thinpoolmeta LV: Creates a Logical Volume called thinpoolmeta in the docker Volume Group.
- Convert LVs to thinpool and storage for metadata: Uses the lvconvert command to convert the Logical Volumes to a thin pool and storage location for metadata for the thin pool
- Config thinpool profile: Creates the /etc/lvm/profile/docker-thinpool.profile configuration file. To achieve this, an preconfigured docker-thinpool.profile file, stored under the files folder, is copied over to the Docker nodes
- Apply the LVM profile: Uses the command lvchange to apply the changes of the profile file that was just copied
- Enable monitoring for LVs: Uses the command lvs to enable monitoring on the Logical Volumes
- Create /etc/docker directory: Creates an /etc/docker folder, that will be used to push the json file in the next task
- Config Docker daemon: Creates the /etc/docker/daemon.json file, containing the configuration required to use the devicemapper driver. The file also contains some configuration options for the rsyslog logging, allowing the container logs to be centralized. The template file is called json.j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html)
- Enable and restart docker service: Makes sure that the docker service is enabled and restarted

## playbooks/docker\_post\_config.yml

This playbook will run a variety of tasks to complete the installation of the Docker environment.

It is composed of the following sequential tasks:

- Create Docker service directory: Creates a folder /etc/systemd/system/docker.service.d on all Docker nodes
- Add proxy details: Updates the /etc/systemd/system/docker.service.d/http-proxy.conf file using a template to configure the proxy settings when these have been defined in the env variable. The template file is called http-proxy.conf.j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the docker service if a change in the file has occurred.
- Add insecure registry: Modifies the file /usr/lib/systemd/system/docker.service on all Docker nodes, modifying the line containing the directive ExecStart to allow using an insecure DTR.

At this point we have a meta task which goal is to run the handlers if required so the changes above are taken in account. More information about meta tasks can be found here: [http://docs.ansible.com/ansible/latest/meta\_module.html](http://docs.ansible.com/ansible/latest/meta_module.html)

- Check if vsphere plugin is installed: Queries the list of Docker plugins to find out if the vSphere plugin has already been installed. The task will record the output, which will be used as a conditional for the next step
- Install vsphere plugin: Installs the vSphere plugin if it&#39;s not already installed

This playbook also contains a handler, which is a task that will be run when notified by one or more of the specified tasks above. The goal of having a handler is typically to restart a service only when a configuration file has been changed.

- Restart Docker: Makes sure that the docker service is restarted.

## playbooks/install\_nfs\_server.yml

This playbook will install and configure an NFS server on the NFS node.

It is composed of the following sequential tasks:

- Open required ports: Makes use of the firewalld command to open the required ports for the NFS server
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Create partition on second disk: Uses the Ansible parted module to create a partition on the second disk. Please note that since the GPT label is not used here, the module is by default idempotent and the tasks can be run multiple times, regardless of whether the partition exists or not, without failing. Hence we don&#39;t need the directive ignore\_errors in this occasion.
- Create filesystem: Creates an xfs filesystem in the partition created above.
- Create images folder: Creates a directory in the filesystem above. The directory will be named after the variable images\_folder and will host the images stored in the DTR nodes
- Mount filesystem: Uses the Ansible mount module to mount the directory created above
- Install NFS server: Uses the Ansible yum module to install the latest versions of the packages nfs-utils and rpcbind.
- Enable and start NFS services on server: Makes sure that the following services are enabled and started: rpcbind, nfs-server, nfs-lock, nfs-idmap.
- Modify exports file on NFS server: Updates the /etc/exports file using a template with all the mount points to be exported by the NFS service. The template file is called j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html).
- Refresh exportfs: Runs the exportfs command to make the shared folder available

## playbooks/install\_nfs\_clients.yml

This playbook will install the required packages on the DTR nodes to be able to mount an NFS share.

It is composed of one single task:

- Install NFS client: Uses the Ansible yum module to install the latest version of the package nfs-utils.

## playbooks/install\_ucp\_nodes.yml

This playbook will install and configure the Docker UCP nodes defined in the inventory.

It is composed of the following sequential tasks:

- Open required ports for UCP: Makes use of the firewalld command to open the required ports for UCP
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Check if node already belongs to the swarm: Uses the docker info command to verify if the node is already part of the swarm and records the output to be used in later tasks. This is to prevent trying to install UCP on nodes that are already part of the swarm
- Copy the license: Uses the Ansible copy module to push the license file into the first UCP node, prior to starting the installation
- Install swarm leader and first UCP node: Installs UCP in the first node. The command will specify the UCP version to install, the host IP address, the user and password for the administrator, the license and all the san information. More details on installing UCP can be found here: [https://docs.docker.com/datacenter/ucp/2.1/guides/admin/install/](https://docs.docker.com/datacenter/ucp/2.1/guides/admin/install/)
- Get swarm manager token: Generates and registers the command to add a manager node to the swarm
- Get swarm worker token: Generates and registers the command to add a worker node to the swarm
- Save worker token: Creates (or rewrites) a file /tmp/worker\_token and copies the worker token generated earlier into the file. This file will be used later to add the worker and DTR nodes to the swarm
- Add additional UCP nodes to the swarm: Makes use of registered manager token two tasks above to join the rest of the UCP nodes to the swarm

## playbooks/install\_dtr\_nodes.yml

This playbook will install and configure the Docker DTR nodes defined in the inventory.

It is composed of the following sequential tasks:

- Open required ports for DTR: Makes use of the firewalld command to open the required ports for the DTR
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Get worker token: Retrieves the file created in the previous playbook (/tmp/worker\_token) and registers the contents in a variable
- Check if node already belongs to the swarm: Uses the docker info command to verify if the node is already part of the swarm and records the output to be used in later tasks. This is to prevent trying to install DTR on nodes that are already part of the swarm
- Add DTR nodes to the swarm: Uses the previously registered command to join the swarm
- Install first DTR node: Installs DTR in the first node. The command will specify the DTR version to install, the NFS images folder location, the fully qualified hostname of the DTR node, the address of the DTR load balancer, the URL of the main UCP node and the username and password to access UCP. More details on installing the DTR can be found here: [https://docs.docker.com/datacenter/dtr/2.2/guides/admin/install/](https://docs.docker.com/datacenter/dtr/2.2/guides/admin/install/)
- Get replica ID: Uses the docker ps command to extract and register the replica ID. This is used to add additional DTR nodes to the first one
- Add DTR nodes: Adds additional nodes to the DTR cluster. The command will specify the DTR version to install, the fully qualified hostname of the DTR node, the URL of the main UCP node, the username and password to access UCP and the replica ID captured in the previous step.
- Enable image scanning: Makes use of the DTR REST API to enable the Image Scanning feature. This feature is only available when using a Docker EE Advanced license. More information about the Image Scanning feature can be found here: [https://docs.docker.com/datacenter/dtr/2.2/guides/user/manage-images/scan-images-for-vulnerabilities/#change-the-scanning-mode](https://docs.docker.com/datacenter/dtr/2.2/guides/user/manage-images/scan-images-for-vulnerabilities/#change-the-scanning-mode). A reference for the REST API can be found here: [https://docs.docker.com/datacenter/dtr/2.2/reference/api/](https://docs.docker.com/datacenter/dtr/2.2/reference/api/). Please note that the REST API is still under development and some of the described methods might not reflect the reality of the current available API.

## playbooks/install\_worker\_nodes.yml

This playbook will install and configure the Docker Worker nodes defined in the inventory.

It is composed of the following sequential tasks:

- Open required ports for Workers: Makes use of the firewalld command to open the required ports for the Worker nodes
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Get worker token: Retrieves the file created in the previous playbook (/tmp/worker\_token) and registers the contents in a variable
- Check if node already belongs to the swarm: Uses the docker info command to verify if the node is already part of the swarm and records the output to be used in later tasks. This is to prevent trying to install Workers on nodes that are already part of the swarm
- Add Worker nodes to the swarm: Uses the previously registered command to join the swarm

## playbooks/config\_monitoring.yml

This playbook will configure a monitoring system for the Docker environment by making use of Grafana, Prometheus, cAdvisor and node-exporter Docker containers.

It is composed of the following sequential tasks:

- Open required ports for Grafana and Prometheus: Makes use of the firewalld command to open the required ports for the Grafana and Prometheus
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Copy monitoring files to UCP: Uses the Ansible copy module to push the monitoring folder into the first UCP node. This folder contains all the required files to install the monitoring system.
- Copy docker compose file: Creates the docker-compose.yml file using a template. This file will be used at a later stage by the docker stack deploy command to build the monitoring stack. The template file is called docker-compose.yml.j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html).
- Deploy monitoring stack: Runs the docker stack deploy command to build the monitoring stack.
- Wait for Prometheus to be available: Waits for Prometheus port 9090 to be listening so the playbook can proceed with the configuration steps
- Wait for Grafana to be available: Waits for Grafana port 3000 to be listening so the playbook can proceed with the configuration steps
- Check if datasource already available: Makes use of the Grafana REST API to find out if a Prometheus data source is already there from a previous run. The task will register the output to decide whether to run the next step
- Create Grafana Data Source: Makes use of the Grafana REST API to create the Prometheus data source, using the provided json file. This task is conditional to the previous step output
- Import Dashboard: Makes use of the Grafana REST API to create a monitoring Dashboard, based on the provided JSON file Docker-Swarm-Dashboard.json

At this stage we can connect to the UCP nodes on port 3000 and we will see the Grafana dashboard. The username and password are defaulted to admin/admin. When we log in we can pick up the Dashboard that was imported by the playbooks and observe the ongoing monitoring.

## playbooks/config\_dummy\_vms\_for\_docker\_volumes\_backup.yml

This playbook will make sure that we are able to backup Docker volumes that have been created using the vSphere plugin in Simplivity. There is not a straight forward way to do this, so we need to use a workaround. Since all Docker volumes are going to be stored in the dockvols folder in the datastore(s), we need to create a &#39;dummy&#39; VM per datastore. The vmx, vmsd and vmkd files from this VMs will have to be inside the dockvols folder, so when these VMs are backed up, the volumes are backed up as well. Obviously these VMs don&#39;t need to take any resources and we can keep them powered off.

It is composed of the following sequential tasks:

- Create Dummy VMs: Creates a &#39;dummy&#39; VM per datastore using extremely low specifications. The name of the VMs will be prefixed with the variable dummy\_vm\_prefix and suffixed with the datastore name where it lives.
- Generate powercli script: Generates a powerCLI script using a template that will take in account all defined datastores and that will be copied to the main UCP node in the location defined by the powercli\_script variable. The template file is called powercli\_script.j2 and is located in the templates folder. The script will perform the following tasks:
  - For each &#39;dummy&#39; VM, the files \*.vmx, \*.vmsd and \*.vmdk will be copied over to the dockvols folder in its current datastore
  - Using the \*.vmx files, we&#39;ll registed new &#39;dummy&#39; VMs that now live inside the dockvols folder. The name of these VMs will be composed of the dummy\_vm\_prefix, the string &quot;-in-dockvols-&quot; and the datastore name as the suffix.
  - The old dummy VMs will be removed
- Run powercli script on temporary docker container: Runs the generated script on a powerCLI container to perform the tasks described above. The container provides a powerCLI core environment and is provided by VMWare. More information about powerCLI core can be found here: [https://labs.vmware.com/flings/powercli-core](https://labs.vmware.com/flings/powercli-core). A good article on how to use the container can be found here: [http://www.virtuallyghetto.com/2016/10/5-different-ways-to-run-powercli-script-using-powercli-core-docker-container.html](http://www.virtuallyghetto.com/2016/10/5-different-ways-to-run-powercli-script-using-powercli-core-docker-container.html)
- Delete powercli script from docker host: Deletes the generated script from the UCP host since it contains sensitive information, like the vCenter credentials, in plain text.

## playbooks/config\_simplivity\_backups.yml

This playbook will configure the defined backup policies in the group variables file in Simplivity and will include all Docker nodes plus the &#39;dummy&#39; VMs created before, so the existing Docker volumes are also taken in account. The playbook will mainly use the Simplivite REST API to perform these tasks. A reference to the REST API can be found here: [https://api.simplivity.com/rest-api\_getting-started\_overview/rest-api\_getting-started\_overview\_rest-api-overview.html](https://api.simplivity.com/rest-api_getting-started_overview/rest-api_getting-started_overview_rest-api-overview.html)

It is composed of the following sequential tasks:

- Get Simplivity token: Makes a REST call against the Simplivity API to authenticate and retrieve a token that will be used in the following tasks. More information about authenticating against the Simplivity API can be found here: [https://api.simplivity.com/rest-api\_getting-started\_getting-started/rest-api\_getting-started\_getting-started\_request-oauth-2-token.html](https://api.simplivity.com/rest-api_getting-started_getting-started/rest-api_getting-started_getting-started_request-oauth-2-token.html)
- Retrieve current backup policies: Makes a REST call against the Simplivity API to retrieve all the current backup policies. The result will be stored in a variable current\_policies.
- Extract existing policies names: Uses the Ansible directive set\_fact to create a variable current\_policies\_names that will store a list of the names of all the currently available backup policies, based on the result of a JSON query against the variable current\_policies, registered above. This task provides a subset of the output from the previous task, which will return not just the policies names, but all the related information to each policy.
- Extract policies names to be added: Uses the Ansible directive set\_fact to create a variable backup\_policies\_names that will store a list of the names of the backup policies defined by the user, based on the result of a JSON query against the variable backup\_policies, defined in the group variables.
- Set list of nonexistent policies to be added: Uses the Ansible directive set\_fact to create a variable new\_policies\_names that will store a list of the names of the backup policies to be created, based on the result of the difference between the existing policies and the newly defined ones.
- Create backup policies: Makes a REST call against the Simplivity API to create the backup policies listed in the new\_policies\_names variable.
- Get policy IDs: Makes a REST call against the Simplivity API to retrieve all the current backup policies. The result will be stored in a variable policy\_ids.
- Create backup rules: Makes a REST call against the Simplivity API to create the defined backup rules in the global variable backup\_policies. In order to avoid duplicated rules, the task will only run when the new\_policies\_names is not empty, meaning that the backup policies defined by the user had not been created yet before running this playbook.
- Get VMs information: Makes a REST call against the Simplivity API to retrieve all the VMs information.
- Assign backup policies to VMs: Makes a REST call against the Simplivity API to assign the previously created VMs to a backup policy in Simplivity. Each node or node group in the inventory has a node\_policy variable that will define which policy they should be on.
- Set dummy VM names in one string: Uses the Ansible directive set\_fact to create a variable dummy\_vms\_string that will store a string with the the names of the dummy VMs, separated by commas
- Convert to list: Uses the Ansible directive set\_fact to create a list dummy\_vms that will be a list based on the string dummy\_vms\_string from the previous task
- Assign backup policies to Docker volumes: Makes a REST call against the Simplivity API to assign the previously created dummy VMs to a backup policy in Simplivity. This will make sure that all the Docker volumes will be also backed up on a regular basis

# Accessing the UCP UI
Once the playbooks have run and completed successfully, the Docker UCP UI should be available by browsing to the UCP load balancer or any of the nodes via HTTPS. The authentication screen will appear. Enter your credentials and the dashboard will be displayed. You should see all the nodes information in your Docker environment by clicking on Nodes. By looking into the services you should see the monitoring services that were installed during the playbooks execution:

# Accessing the DTR UI
The Docker DTR UI should be available by browsing to the DTR load balancer or any of the nodes via HTTPS. The authentication screen will appear. Enter your UCP credentials and you should see the empty list of repositories. If you navigate to `Settings > Security`, you should see the Image Scanning feature already enabled (note that you need an Advanced license to have access to this feature).

# Security considerations
In addition to having all logs centralized in an unique place and the image scanning feature enabled, there are another few guidelines that should be followed in order to keep your Docker environment as secure as possible.

## Securing the daemon socket
The default is to create a non-networked socket to communicate with the daemon, but you could alternatively communicate using TLS. This will require you to create a CA and server and client keys with OpenSSL. The whole procedure is explained in detail here: [https://docs.docker.com/engine/security/https/](https://docs.docker.com/engine/security/https/)

## Start with known images
Developers can pick base images from the Docker Hub to start with. Although this allows them a great deal of freedom, some caution is needed. Images from Docker Hub are uncurated and may contain vulnerabilities. Use store images where possible. These are curated images. Use small base images to reduce the surface area.

## Enable Docker Content Trust
Notary/Docker Content Trust is a tool for publishing and managing trusted collections of content. Publishers can digitally sign collections and consumers can verify integrity and origin of content. This ability is built on a straightforward key management and signing interface to create signed collections and configure trusted publishers. More information about Notary can be found here: [https://success.docker.com/Architecture/Docker\_Reference\_Architecture%3A\_Securing\_Docker\_EE\_and\_Security\_Best\_Practices#dtr-notary](https://success.docker.com/Architecture/Docker_Reference_Architecture%3A_Securing_Docker_EE_and_Security_Best_Practices#dtr-notary)

## Prevent tags from being overwritten
By default, users with access to push to a repository, can push the same tag multiple times to the same repository. As an example, a user pushes an image to library/wordpress:latest, and later another user can push the image with exactly the same name but different functionality. This might make it difficult to trace back the image to the build that generated it.

To prevent this from happening you can configure a repository to be immutable. Once you push a tag, DTR won&#39;t anyone else to push another tag with the same name.

More information about immutable tags can be found here: [https://beta.docs.docker.com/datacenter/dtr/2.3/guides/user/manage-images/prevent-tags-from-being-overwritten/](https://beta.docs.docker.com/datacenter/dtr/2.3/guides/user/manage-images/prevent-tags-from-being-overwritten/)

## Use secrets
Use secrets to pass credentials to a container. Never pass as an environment variable which is clear

## Isolate swarm nodes to a specific team
With Docker EE Advanced, you can enable physical isolation of resources by organizing nodes into collections and granting Scheduler access for different users. To control access to nodes, move them to dedicated collections where you can grant access to specific users, teams, and organizations.

More information about this subject can be found here: [https://beta.docs.docker.com/datacenter/ucp/2.2/guides/admin/manage-users/isolate-nodes-between-teams/](https://beta.docs.docker.com/datacenter/ucp/2.2/guides/admin/manage-users/isolate-nodes-between-teams/)

## Docker Bench for Security
The Docker Bench for Security is a script that checks for dozens of common best-practices around deploying Docker containers in production. The tests are all automated, and are inspired by the CIS Docker Community Edition Benchmark v1.1.0.

The Docker Bench for Security should be run on a regular basis to make sure that our system is as secure as we&#39;d expect it to be.

More information about this tool plus the files to run it can be found in its Github repository: [https://github.com/docker/docker-bench-security](https://github.com/docker/docker-bench-security)
