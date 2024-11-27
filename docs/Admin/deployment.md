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

# Set up head node

Set up a VM from Cloud which will be used as a head node

Install ansible using pip as per Ansible Documentation 
```
python3 -m pip -V
```
If pip is available you will see something like this 

```
pip 21.0.1 from /usr/lib/python3.9/site-packages/pip (python 3.9)
```
If you see `No module named pip` you will need to run
```
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py --user
```
Use pip to install ansible and verify
```
python3 -m pip install --user ansible

python3 -m pip show ansible OR ansible --version
```

Generate ssh-key using `ssh-keygen`

It will generate the private/public in `~/.ssh` as `id_rsa && id_rsa.pub`

Copy the `id_rsa.pub` key on to each managed nodes to `~/.ssh/authorized_keys`

```ssh-copy-id -i ~/.ssh/id_rsa.pub <IP>```


# Set up pockektsoc nodes

- Provision a set of VMs as managed nodes - Noting for each the node number and FQDN    
- Ensure the `~/.ssh/authorized_keys` includes head node public key on each node

# Deploy

On Head node, clone the [Ansible repository](https://github.com/azahmd/pocketsoc-ng-ansible)  
```bash git clone https://github.com/azahmd/pocketsoc-ng-ansible ```

Create or edit `inventory.ini` file and include your hosts/nodes.
```
[trainingvms]
node-1 ansible_host=<node1_ip> misp_port=<some_port>
node-2 ansible_host=<node2_ip> misp_port=<some_port>
node-3 ansible_host=<node3_ip> misp_port=<some_port>
node-4 ansible_host=<node4_ip> misp_port=<some_port>
node-5 ansible_host=<node5_ip> misp_port=<some_port>

[trainingvms:vars]
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
ansible -i inventory.ini trainingvms -m ping
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
# Set up haproxy node

- Provision a separate VM
- Generate a certificate for `haproxy-fqdn`
- Include this configuration to `haproxy.cfg` in `/etc/haproxy`
- Open the required ports
- To restart or check status after updating haproxy.cfg  
`systemctl status haproxy` and `systemctl restart haproxy`
```
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

frontend http_frontend
    bind *:80
    redirect scheme https code 301

frontend https_frontend
    bind *:443 ssl crt /etc/ssl/private/haproxy.pem
    bind *:1001-1040 ssl crt /etc/ssl/private/haproxy.pem
    bind *:2001-2040 ssl crt /etc/ssl/private/haproxy.pem

    acl portainer dst_port 1001
    acl portainer2 dst_port 2001

    acl opensearch dst_port 1002
    acl opensearch2 dst_port 2002

    acl misp dst_port 1003
    acl misp2 dst_port 2003

    use_backend portainer if portainer
    use_backend portainer2 if portainer2

    use_backend opensearch if opensearch
    use_backend opensearch2 if opensearch2

    use_backend misp if misp
    use_backend misp2 if misp2

default_backend default

backend portainer
    server portainer_node1 <NODE1_FQDN>:9443 check ssl verify none

backend portainer2
    server portainer_node2 <NODE2_FQDN>:9443 check ssl verify none

backend opensearch
    server opensearch_node1 <NODE1_FQDN>:5601 check ssl verify none

backend opensearch2
    server opensearch_node2 <NODE2_FQDN>:5601 check ssl verify none

backend misp
    server misp_node1 <NODE1_FQDN>:1003 check ssl verify none

backend misp2
    server misp_node2 <NODE2_FQDN>:2003 check ssl verify none

backend default
    http-request deny

```
