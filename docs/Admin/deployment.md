# Deployment Overview

The PocketSOC architecture includes several key components:

- **Portainer**: Provides a web-based UI for managing docker containers.
- **MISP**: Threat Intelligence platform that allows collecting and sharing threat information.
- **Zeek**: Network traffic analysis tool that monitors and logs network activity.
- **Logstash**: Data processing pipeline used for Log parsing and forwarding.
- **OpenSearch**: Search and analytics engine used for storing, searching, and analysing logs.
- **Docker & Docker Compose**: Orchestrating the containers that run these services.

PocketSOC is designed to be easily deployable using Docker and Ansible playbooks. 

# Installation

This section will cover the steps to install PocketSOC on your system.

## Prerequisites

- Python3 & pip
- Ansible

Set up a VM from Cloud which will be used as a head node.

Set up X additional VMs as managed nodes.

On head node install ansible using pip as per Ansible Documentation. 
```
python3 -m pip -V
```
If pip is available you will see something like this 

```
pip 21.0.1 from /usr/lib/python3.9/site-packages/pip (python 3.9)
```
If you see `No module named pip` you will need to run.
```
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py --user
```
Use pip to install ansible and verify
```
python3 -m pip install --user ansible

python3 -m pip show ansible OR ansible --version
```

On head node generate SSH-key

Generate ssh-key using `ssh-keygen`

It will generate the private/public in `~/.ssh` as `id_rsa && id_rsa.pub`

Copy the `id_rsa.pub` key on to each managed nodes to `~/.ssh/authorized_keys`

```ssh-copy-id -i ~/.ssh/id_rsa.pub <IP>```


## Deploy

On Head node, clone the [Ansible repository](https://github.com/azahmd/pocketsoc-ng-ansible)
```bash git clone https://github.com/azahmd/pocketsoc-ng-ansible ```

Create or edit `inventory.ini` file and include your hosts/nodes.
```
[vms]
node-1_ip
node-2_ip

[vms:vars]
ansible_user=root
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Start the ssh-agent

```
eval "$(ssh-agent -s)" 

ssh-add ~/.ssh/ansible_id_rsa

```

If you don't want to run these long commands each time you want to run the ansible. 
Create an alias for it.

```
alias ssha='eval $(ssh-agent -s) && ssh-add ~/.ssh/id_rsa'
```
Ping to see the connection with the nodes.
```
ansible -i inventory.ini vms -m ping
```

Run this command 
```
ansible-playbook -i inventory.ini pocketsoc-ng.yml --extra-vars "@pocketsoc-ng_var.env"
```
To stop 
```
ansible-playbook -i inventory.ini pocketsoc-ng-stop.yml --extra-vars "@pocketsoc-ng_var.env"
```

If you want to have a fresh installation you need to run `./cleanup.sh` on the managed hosts for a clean start.

`./cleanup.sh` includes

``` bash
sudo docker stop $(sudo docker ps -a -q)
docker stop portainer
docker container prune
docker volume rm portainer_data
```
