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

## Preamble 

1. Set up SSH keypair for the purpose of this deployment using `ssh-keygen`; when prompted enter a password 

> Optionally choose a more meaningful name for this particular key, eg `id_training_rsa`; use this name in place of `id_rsa` going forward

    - It will generate the private/public keypair in `~/.ssh` as `id_rsa && id_rsa.pub`

## Managed nodes - initial

1. Provision a set of VMs as managed nodes - Noting for each the node number and FQDN

> **STFC Cloud:** 
>     - Use type `l3.micro`
>     - You can upload the public key of the dedicated keypair you generated for this purpose at this stage
>     - If you create multiple hosts in one step, the `count` setting will append `-N` to each hostname
 
2. Ensure the `~/.ssh/authorized_keys` includes head node public key on each node if not provisioned as above

## Head node

1. Provision a VM from your openstack environment which will be used as a head node

> **STFC Cloud:** Use type `l3.nano`

2. Log into host as standard user. 

3. Install ansible and git

```sudo dnf install ansible git -y```

4. Make sure that the keypair generated above is present in `~/.ssh/`

## Set up haproxy node

1. Provision a VM from your openstack environment which will be used as a haproxy node

> **STFC Cloud:** Use type `l3.nano`

2. Generate a certificate for your HAProxy instance

- For a self-signed cert **for testing**, you can use the following for example
    
```
sudo mkdir -p /opt/pocketsoc/certs/
cd /opt/pocketsoc/certs/
sudo openssl req -x509 -newkey rsa:4096 -keyout key.pem -out certificate.pem -days 365 -nodes
sudo cat key.pem >> certificate.pem # This may need to be done as root
```

> **Example options**
> UK
> []
> Abingdon
> Security School 2025
> []
> haproxy
> []

- Cert will be available at `/opt/pocketsoc/certs/certificate.pem`

> Test subject using `openssl x509 -in /opt/pocketsoc/certs/certificate.pem -noout -subject`
    
3. Install HAProxy

```
sudo dnf install haproxy -y
```

4. Build your `haproxy.cfg` config file. In the following example, we're using an example structure of 4XX0Y for the training node ports

- XX: training node number
- Y: service port

4A. Common initial sections:

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
```

4B. For each training node, need to configure ports with format `N001-N003` . First configure bind ports (where for example we have 40 training nodes). We're using the certificate location from the example above, update as appropriate.

```
frontend https_frontend
    bind *:443 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:1001-1003 ssl crt /opt/pocketsoc/certs/certificate.pem
...
    bind *:40001-40003 ssl crt /opt/pocketsoc/certs/certificate.pem
```

4C. Config acls - need portainer, opensearch and misp for each node (again example of 40 nodes):

```
    acl portainer1 dst_port 40101
...
    acl portainer40 dst_port 44001
    
    acl opensearch1 dst_port 40102
...
    acl opensearch40 dst_port 44002
    
    acl misp1 dst_port 40103
...
    acl misp40 dst_port 44003
```

4D. Set up `use_backend` rules

```
    use_backend portainer1 if portainer1
...
    use_backend portainer40 if portainer40

    use_backend opensearch1 if opensearch1
...
    use_backend opensearch40 if opensearch40

    use_backend misp1 if misp1
...
    use_backend misp40 if misp40
```

4E. Set up backends (here you need the fqdns of the training nodes). **NOTE** the port number for the MISP backends **must** match the individual ports as defined above

```
default_backend default

backend portainer1
    server portainer_node1 <NODE1_FQDN>:9443 check ssl verify none

...

backend portainer40
    server portainer_node40 <NODE40_FQDN>:9443 check ssl verify none
    
backend opensearch1
    server opensearch_node1 <NODE1_FQDN>:5601 check ssl verify none

...

backend opensearch40
    server opensearch_node40 <NODE40_FQDN>:5601 check ssl verify none
    
backend misp1
    server misp_node1 <NODE1_FQDN>:40103 check ssl verify none
    
...

backend misp40
    server misp_node40 <NODE1_FQDN>:44003 check ssl verify none 
```

4F. Include default deny

```
backend default
    http-request deny
```

4G. A complete version of this config will be included at the end for 40 training nodes.

5. Copy this configuration to `/etc/haproxy/haproxy.cfg`

6. Ensure that the appropriate ports are available

7. Start `haproxy`: `sudo systemctl enable haproxy && sudo systemctl start haproxy`

## Deploy

1. On head node, in normal user area clone the [Ansible repository](https://github.com/drmcrooks/pocketsoc-ng-2025-ansible)  
```git clone https://github.com/drmcrooks/pocketsoc-ng-2025-ansible ```

2. `cd pocketsoc-ng-ansible`

3. Create or edit `inventory.ini` file and include your hosts/nodes.

```
[trainingvms]
node-1 ansible_host=<node1_ip> misp_port=40103
node-2 ansible_host=<node2_ip> misp_port=40203
node-3 ansible_host=<node3_ip> misp_port=40303
...
node-4 ansible_host=<node4_ip> misp_port=44003
```

4. Create or edit `pocketsoc-ng_var.env` and include global variables. 

```
---
# Variables
salt: "${RANDOM}"
admin_password_command: "openssl rand -base64 32"
haproxy_fqdn: "${HAPROXY_FQDN}"
```

5. Start the ssh-agent using dedicated training keypair

```
eval "$(ssh-agent -s)"  &&ssh-add ~/.ssh/id_training_rsa
```

> **OPTIONAL** Create an alias for setting up ssh agent like this

```
alias ssha='eval $(ssh-agent -s) && ssh-add ~/.ssh/id_rsa'
```
6. Make sure that the SSH fingerprints of the training nodes are stored.

```
ssh-keyscan -H $node1_ip >> ~/.ssh/known_hosts
ssh-keyscan -H $node2_ip >> ~/.ssh/known_hosts
ssh-keyscan -H $node3_ip >> ~/.ssh/known_hosts
```

7. Ping to see the connection with the nodes.
```
ansible -i inventory.ini trainingvms -m ping
```

8. Deploy PocketSOC using ansible playbook: 
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

# APPENDIX A

Sample haproxy config for 40 training nodes

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
    bind *:443 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40101-40103 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40201-40203 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40301-40303 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40401-40403 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40501-40503 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40601-40603 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40701-40703 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40801-40803 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:40901-40903 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41001-41003 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41101-41103 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41201-41203 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41301-41303 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41401-41403 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41501-41503 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41601-41603 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41701-41703 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41801-41803 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:41901-41903 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42001-42003 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42101-42103 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42201-42203 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42301-42303 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42401-42403 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42501-42503 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42601-42603 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42701-42703 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42801-42803 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:42901-42903 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43001-43003 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43101-43103 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43201-43203 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43301-43303 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43401-43403 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43501-43503 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43601-43603 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43701-43703 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43801-43803 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:43901-43903 ssl crt /opt/pocketsoc/certs/certificate.pem
    bind *:44001-44003 ssl crt /opt/pocketsoc/certs/certificate.pem
    
    acl portainer1 dst_port 40101
    acl portainer2 dst_port 40201
    acl portainer3 dst_port 40301
    acl portainer4 dst_port 40401
    acl portainer5 dst_port 40501
    acl portainer6 dst_port 40601
    acl portainer7 dst_port 40701
    acl portainer8 dst_port 40801
    acl portainer9 dst_port 40901
    acl portainer10 dst_port 41001
    acl portainer11 dst_port 41101
    acl portainer12 dst_port 41201
    acl portainer13 dst_port 41301
    acl portainer14 dst_port 41401
    acl portainer15 dst_port 41501
    acl portainer16 dst_port 41601
    acl portainer17 dst_port 41701
    acl portainer18 dst_port 41801
    acl portainer19 dst_port 41901
    acl portainer20 dst_port 42001
    acl portainer21 dst_port 42101
    acl portainer22 dst_port 42201
    acl portainer23 dst_port 42301
    acl portainer24 dst_port 42401
    acl portainer25 dst_port 42501
    acl portainer26 dst_port 42601
    acl portainer27 dst_port 42701
    acl portainer28 dst_port 42801
    acl portainer29 dst_port 42901
    acl portainer30 dst_port 43001
    acl portainer31 dst_port 43101
    acl portainer32 dst_port 43201
    acl portainer33 dst_port 43301
    acl portainer34 dst_port 43401
    acl portainer35 dst_port 43501
    acl portainer36 dst_port 43601
    acl portainer37 dst_port 43701
    acl portainer38 dst_port 43801
    acl portainer39 dst_port 43901
    acl portainer40 dst_port 44001
    
    acl opensearch1 dst_port 40102
    acl opensearch2 dst_port 40202
    acl opensearch3 dst_port 40302
    acl opensearch4 dst_port 40402
    acl opensearch5 dst_port 40502
    acl opensearch6 dst_port 40602
    acl opensearch7 dst_port 40702
    acl opensearch8 dst_port 40802
    acl opensearch9 dst_port 40902
    acl opensearch10 dst_port 41002
    acl opensearch11 dst_port 41102
    acl opensearch12 dst_port 41202
    acl opensearch13 dst_port 41302
    acl opensearch14 dst_port 41402
    acl opensearch15 dst_port 41502
    acl opensearch16 dst_port 41602
    acl opensearch17 dst_port 41702
    acl opensearch18 dst_port 41802
    acl opensearch19 dst_port 41902
    acl opensearch20 dst_port 42002
    acl opensearch21 dst_port 42102
    acl opensearch22 dst_port 42202
    acl opensearch23 dst_port 42302
    acl opensearch24 dst_port 42402
    acl opensearch25 dst_port 42502
    acl opensearch26 dst_port 42602
    acl opensearch27 dst_port 42702
    acl opensearch28 dst_port 42802
    acl opensearch29 dst_port 42902
    acl opensearch30 dst_port 43002
    acl opensearch31 dst_port 43102
    acl opensearch32 dst_port 43202
    acl opensearch33 dst_port 43302
    acl opensearch34 dst_port 43402
    acl opensearch35 dst_port 43502
    acl opensearch36 dst_port 43602
    acl opensearch37 dst_port 43702
    acl opensearch38 dst_port 43802
    acl opensearch39 dst_port 43902
    acl opensearch40 dst_port 44002

    acl misp1 dst_port 40103
    acl misp2 dst_port 40203
    acl misp3 dst_port 40303
    acl misp4 dst_port 40403
    acl misp5 dst_port 40503
    acl misp6 dst_port 40603
    acl misp7 dst_port 40703
    acl misp8 dst_port 40803
    acl misp9 dst_port 40903
    acl misp10 dst_port 41003
    acl misp11 dst_port 41103
    acl misp12 dst_port 41203
    acl misp13 dst_port 41303
    acl misp14 dst_port 41403
    acl misp15 dst_port 41503
    acl misp16 dst_port 41603
    acl misp17 dst_port 41703
    acl misp18 dst_port 41803
    acl misp19 dst_port 41903
    acl misp20 dst_port 42003
    acl misp21 dst_port 42103
    acl misp22 dst_port 42203
    acl misp23 dst_port 42303
    acl misp24 dst_port 42403
    acl misp25 dst_port 42503
    acl misp26 dst_port 42603
    acl misp27 dst_port 42703
    acl misp28 dst_port 42803
    acl misp29 dst_port 42903
    acl misp30 dst_port 43003
    acl misp31 dst_port 43103
    acl misp32 dst_port 43203
    acl misp33 dst_port 43303
    acl misp34 dst_port 43403
    acl misp35 dst_port 43503
    acl misp36 dst_port 43603
    acl misp37 dst_port 43703
    acl misp38 dst_port 43803
    acl misp39 dst_port 43903
    acl misp40 dst_port 44003

    use_backend portainer1 if portainer1
    use_backend portainer2 if portainer2
    use_backend portainer3 if portainer3
    use_backend portainer4 if portainer4
    use_backend portainer5 if portainer5
    use_backend portainer6 if portainer6
    use_backend portainer7 if portainer7
    use_backend portainer8 if portainer8
    use_backend portainer9 if portainer9
    use_backend portainer10 if portainer10
    use_backend portainer11 if portainer11
    use_backend portainer12 if portainer12
    use_backend portainer13 if portainer13
    use_backend portainer14 if portainer14
    use_backend portainer15 if portainer15
    use_backend portainer16 if portainer16
    use_backend portainer17 if portainer17
    use_backend portainer18 if portainer18
    use_backend portainer19 if portainer19
    use_backend portainer20 if portainer20
    use_backend portainer21 if portainer21
    use_backend portainer22 if portainer22
    use_backend portainer23 if portainer23
    use_backend portainer24 if portainer24
    use_backend portainer25 if portainer25
    use_backend portainer26 if portainer26
    use_backend portainer27 if portainer27
    use_backend portainer28 if portainer28
    use_backend portainer29 if portainer29
    use_backend portainer30 if portainer30
    use_backend portainer31 if portainer31
    use_backend portainer32 if portainer32
    use_backend portainer33 if portainer33
    use_backend portainer34 if portainer34
    use_backend portainer35 if portainer35
    use_backend portainer36 if portainer36
    use_backend portainer37 if portainer37
    use_backend portainer38 if portainer38
    use_backend portainer39 if portainer39
    use_backend portainer40 if portainer40

    use_backend opensearch1 if opensearch1
    use_backend opensearch2 if opensearch2
    use_backend opensearch3 if opensearch3
    use_backend opensearch4 if opensearch4
    use_backend opensearch5 if opensearch5
    use_backend opensearch6 if opensearch6
    use_backend opensearch7 if opensearch7
    use_backend opensearch8 if opensearch8
    use_backend opensearch9 if opensearch9
    use_backend opensearch10 if opensearch10
    use_backend opensearch11 if opensearch11
    use_backend opensearch12 if opensearch12
    use_backend opensearch13 if opensearch13
    use_backend opensearch14 if opensearch14
    use_backend opensearch15 if opensearch15
    use_backend opensearch16 if opensearch16
    use_backend opensearch17 if opensearch17
    use_backend opensearch18 if opensearch18
    use_backend opensearch19 if opensearch19
    use_backend opensearch20 if opensearch20
    use_backend opensearch21 if opensearch21
    use_backend opensearch22 if opensearch22
    use_backend opensearch23 if opensearch23
    use_backend opensearch24 if opensearch24
    use_backend opensearch25 if opensearch25
    use_backend opensearch26 if opensearch26
    use_backend opensearch27 if opensearch27
    use_backend opensearch28 if opensearch28
    use_backend opensearch29 if opensearch29
    use_backend opensearch30 if opensearch30
    use_backend opensearch31 if opensearch31
    use_backend opensearch32 if opensearch32
    use_backend opensearch33 if opensearch33
    use_backend opensearch34 if opensearch34
    use_backend opensearch35 if opensearch35
    use_backend opensearch36 if opensearch36
    use_backend opensearch37 if opensearch37
    use_backend opensearch38 if opensearch38
    use_backend opensearch39 if opensearch39
    use_backend opensearch40 if opensearch40

    use_backend misp1 if misp1
    use_backend misp2 if misp2
    use_backend misp3 if misp3
    use_backend misp4 if misp4
    use_backend misp5 if misp5
    use_backend misp6 if misp6
    use_backend misp7 if misp7
    use_backend misp8 if misp8
    use_backend misp9 if misp9
    use_backend misp10 if misp10
    use_backend misp11 if misp11
    use_backend misp12 if misp12
    use_backend misp13 if misp13
    use_backend misp14 if misp14
    use_backend misp15 if misp15
    use_backend misp16 if misp16
    use_backend misp17 if misp17
    use_backend misp18 if misp18
    use_backend misp19 if misp19
    use_backend misp20 if misp20
    use_backend misp21 if misp21
    use_backend misp22 if misp22
    use_backend misp23 if misp23
    use_backend misp24 if misp24
    use_backend misp25 if misp25
    use_backend misp26 if misp26
    use_backend misp27 if misp27
    use_backend misp28 if misp28
    use_backend misp29 if misp29
    use_backend misp30 if misp30
    use_backend misp31 if misp31
    use_backend misp32 if misp32
    use_backend misp33 if misp33
    use_backend misp34 if misp34
    use_backend misp35 if misp35
    use_backend misp36 if misp36
    use_backend misp37 if misp37
    use_backend misp38 if misp38
    use_backend misp39 if misp39
    use_backend misp40 if misp40

default_backend default
    
backend portainer1
    server portainer_node1 <NODE1_FQDN>:9443 check ssl verify none

backend portainer2
    server portainer_node2 <NODE2_FQDN>:9443 check ssl verify none

backend portainer3
    server portainer_node3 <NODE3_FQDN>:9443 check ssl verify none

backend portainer4
    server portainer_node4 <NODE4_FQDN>:9443 check ssl verify none

backend portainer5
    server portainer_node5 <NODE5_FQDN>:9443 check ssl verify none

backend portainer6
    server portainer_node6 <NODE6_FQDN>:9443 check ssl verify none

backend portainer7
    server portainer_node7 <NODE7_FQDN>:9443 check ssl verify none

backend portainer8
    server portainer_node8 <NODE8_FQDN>:9443 check ssl verify none

backend portainer9
    server portainer_node9 <NODE9_FQDN>:9443 check ssl verify none

backend portainer10
    server portainer_node10 <NODE10_FQDN>:9443 check ssl verify none

backend portainer11
    server portainer_node11 <NODE11_FQDN>:9443 check ssl verify none

backend portainer12
    server portainer_node12 <NODE12_FQDN>:9443 check ssl verify none

backend portainer13
    server portainer_node13 <NODE13_FQDN>:9443 check ssl verify none

backend portainer14
    server portainer_node14 <NODE14_FQDN>:9443 check ssl verify none

backend portainer15
    server portainer_node15 <NODE15_FQDN>:9443 check ssl verify none

backend portainer16
    server portainer_node16 <NODE16_FQDN>:9443 check ssl verify none

backend portainer17
    server portainer_node17 <NODE17_FQDN>:9443 check ssl verify none

backend portainer18
    server portainer_node18 <NODE18_FQDN>:9443 check ssl verify none

backend portainer19
    server portainer_node19 <NODE19_FQDN>:9443 check ssl verify none

backend portainer20
    server portainer_node20 <NODE20_FQDN>:9443 check ssl verify none

backend portainer21
    server portainer_node21 <NODE21_FQDN>:9443 check ssl verify none

backend portainer22
    server portainer_node22 <NODE22_FQDN>:9443 check ssl verify none

backend portainer23
    server portainer_node23 <NODE23_FQDN>:9443 check ssl verify none

backend portainer24
    server portainer_node24 <NODE24_FQDN>:9443 check ssl verify none

backend portainer25
    server portainer_node25 <NODE25_FQDN>:9443 check ssl verify none

backend portainer26
    server portainer_node26 <NODE26_FQDN>:9443 check ssl verify none

backend portainer27
    server portainer_node27 <NODE27_FQDN>:9443 check ssl verify none

backend portainer28
    server portainer_node28 <NODE28_FQDN>:9443 check ssl verify none

backend portainer29
    server portainer_node29 <NODE29_FQDN>:9443 check ssl verify none

backend portainer30
    server portainer_node30 <NODE30_FQDN>:9443 check ssl verify none

backend portainer31
    server portainer_node31 <NODE31_FQDN>:9443 check ssl verify none

backend portainer32
    server portainer_node32 <NODE32_FQDN>:9443 check ssl verify none

backend portainer33
    server portainer_node33 <NODE33_FQDN>:9443 check ssl verify none

backend portainer34
    server portainer_node34 <NODE34_FQDN>:9443 check ssl verify none

backend portainer35
    server portainer_node35 <NODE35_FQDN>:9443 check ssl verify none

backend portainer36
    server portainer_node36 <NODE36_FQDN>:9443 check ssl verify none

backend portainer37
    server portainer_node37 <NODE37_FQDN>:9443 check ssl verify none

backend portainer38
    server portainer_node38 <NODE38_FQDN>:9443 check ssl verify none

backend portainer39
    server portainer_node39 <NODE39_FQDN>:9443 check ssl verify none

backend portainer40
    server portainer_node40 <NODE40_FQDN>:9443 check ssl verify none

backend opensearch1
    server opensearch_node1 <NODE1_FQDN>:5601 check ssl verify none
    
backend opensearch2
    server opensearch_node2 <NODE2_FQDN>:5601 check ssl verify none

backend opensearch3
    server opensearch_node3 <NODE3_FQDN>:5601 check ssl verify none

backend opensearch4
    server opensearch_node4 <NODE4_FQDN>:5601 check ssl verify none

backend opensearch5
    server opensearch_node5 <NODE5_FQDN>:5601 check ssl verify none

backend opensearch6
    server opensearch_node6 <NODE6_FQDN>:5601 check ssl verify none

backend opensearch7
    server opensearch_node7 <NODE7_FQDN>:5601 check ssl verify none

backend opensearch8
    server opensearch_node8 <NODE8_FQDN>:5601 check ssl verify none

backend opensearch9
    server opensearch_node9 <NODE9_FQDN>:5601 check ssl verify none

backend opensearch10
    server opensearch_node10 <NODE10_FQDN>:5601 check ssl verify none

backend opensearch11
    server opensearch_node11 <NODE11_FQDN>:5601 check ssl verify none

backend opensearch12
    server opensearch_node12 <NODE12_FQDN>:5601 check ssl verify none

backend opensearch13
    server opensearch_node13 <NODE13_FQDN>:5601 check ssl verify none

backend opensearch14
    server opensearch_node14 <NODE14_FQDN>:5601 check ssl verify none

backend opensearch15
    server opensearch_node15 <NODE15_FQDN>:5601 check ssl verify none

backend opensearch16
    server opensearch_node16 <NODE16_FQDN>:5601 check ssl verify none

backend opensearch17
    server opensearch_node17 <NODE17_FQDN>:5601 check ssl verify none

backend opensearch18
    server opensearch_node18 <NODE18_FQDN>:5601 check ssl verify none

backend opensearch19
    server opensearch_node19 <NODE19_FQDN>:5601 check ssl verify none

backend opensearch20
    server opensearch_node20 <NODE20_FQDN>:5601 check ssl verify none

backend opensearch21
    server opensearch_node21 <NODE21_FQDN>:5601 check ssl verify none

backend opensearch22
    server opensearch_node22 <NODE22_FQDN>:5601 check ssl verify none

backend opensearch23
    server opensearch_node23 <NODE23_FQDN>:5601 check ssl verify none

backend opensearch24
    server opensearch_node24 <NODE24_FQDN>:5601 check ssl verify none

backend opensearch25
    server opensearch_node25 <NODE25_FQDN>:5601 check ssl verify none

backend opensearch26
    server opensearch_node26 <NODE26_FQDN>:5601 check ssl verify none

backend opensearch27
    server opensearch_node27 <NODE27_FQDN>:5601 check ssl verify none

backend opensearch28
    server opensearch_node28 <NODE28_FQDN>:5601 check ssl verify none

backend opensearch29
    server opensearch_node29 <NODE29_FQDN>:5601 check ssl verify none

backend opensearch30
    server opensearch_node30 <NODE30_FQDN>:5601 check ssl verify none

backend opensearch31
    server opensearch_node31 <NODE31_FQDN>:5601 check ssl verify none

backend opensearch32
    server opensearch_node32 <NODE32_FQDN>:5601 check ssl verify none

backend opensearch33
    server opensearch_node33 <NODE33_FQDN>:5601 check ssl verify none

backend opensearch34
    server opensearch_node34 <NODE34_FQDN>:5601 check ssl verify none

backend opensearch35
    server opensearch_node35 <NODE35_FQDN>:5601 check ssl verify none

backend opensearch36
    server opensearch_node36 <NODE36_FQDN>:5601 check ssl verify none

backend opensearch37
    server opensearch_node37 <NODE37_FQDN>:5601 check ssl verify none

backend opensearch38
    server opensearch_node38 <NODE38_FQDN>:5601 check ssl verify none

backend opensearch39
    server opensearch_node39 <NODE39_FQDN>:5601 check ssl verify none

backend opensearch40
    server opensearch_node40 <NODE40_FQDN>:5601 check ssl verify none

backend misp1
    server misp_node1 <NODE1_FQDN>:40103 check ssl verify none

backend misp2
    server misp_node2 <NODE2_FQDN>:40203 check ssl verify none

backend misp3
    server misp_node3 <NODE3_FQDN>:40303 check ssl verify none

backend misp4
    server misp_node4 <NODE4_FQDN>:40403 check ssl verify none

backend misp5
    server misp_node5 <NODE5_FQDN>:40503 check ssl verify none

backend misp6
    server misp_node6 <NODE6_FQDN>:40603 check ssl verify none

backend misp7
    server misp_node7 <NODE7_FQDN>:40703 check ssl verify none

backend misp8
    server misp_node8 <NODE8_FQDN>:40803 check ssl verify none

backend misp9
    server misp_node9 <NODE9_FQDN>:40903 check ssl verify none

backend misp10
    server misp_node10 <NODE10_FQDN>:41003 check ssl verify none

backend misp11
    server misp_node11 <NODE11_FQDN>:41103 check ssl verify none

backend misp12
    server misp_node12 <NODE12_FQDN>:41203 check ssl verify none

backend misp13
    server misp_node13 <NODE13_FQDN>:41303 check ssl verify none

backend misp14
    server misp_node14 <NODE14_FQDN>:41403 check ssl verify none

backend misp15
    server misp_node15 <NODE15_FQDN>:41503 check ssl verify none

backend misp16
    server misp_node16 <NODE16_FQDN>:41603 check ssl verify none

backend misp17
    server misp_node17 <NODE17_FQDN>:41703 check ssl verify none

backend misp18
    server misp_node18 <NODE18_FQDN>:41803 check ssl verify none

backend misp19
    server misp_node19 <NODE19_FQDN>:41903 check ssl verify none

backend misp20
    server misp_node20 <NODE20_FQDN>:42003 check ssl verify none

backend misp21
    server misp_node21 <NODE21_FQDN>:42103 check ssl verify none

backend misp22
    server misp_node22 <NODE22_FQDN>:42203 check ssl verify none

backend misp23
    server misp_node23 <NODE23_FQDN>:42303 check ssl verify none

backend misp24
    server misp_node24 <NODE24_FQDN>:42403 check ssl verify none

backend misp25
    server misp_node25 <NODE25_FQDN>:42503 check ssl verify none

backend misp26
    server misp_node26 <NODE26_FQDN>:42603 check ssl verify none

backend misp27
    server misp_node27 <NODE27_FQDN>:42703 check ssl verify none

backend misp28
    server misp_node28 <NODE28_FQDN>:42803 check ssl verify none

backend misp29
    server misp_node29 <NODE29_FQDN>:42903 check ssl verify none

backend misp30
    server misp_node30 <NODE30_FQDN>:43003 check ssl verify none

backend misp31
    server misp_node31 <NODE31_FQDN>:43103 check ssl verify none

backend misp32
    server misp_node32 <NODE32_FQDN>:43203 check ssl verify none

backend misp33
    server misp_node33 <NODE33_FQDN>:43303 check ssl verify none

backend misp34
    server misp_node34 <NODE34_FQDN>:43403 check ssl verify none

backend misp35
    server misp_node35 <NODE35_FQDN>:43503 check ssl verify none

backend misp36
    server misp_node36 <NODE36_FQDN>:43603 check ssl verify none

backend misp37
    server misp_node37 <NODE37_FQDN>:43703 check ssl verify none

backend misp38
    server misp_node38 <NODE38_FQDN>:43803 check ssl verify none

backend misp39
    server misp_node39 <NODE39_FQDN>:43903 check ssl verify none

backend misp40
    server misp_node40 <NODE40_FQDN>:44003 check ssl verify none
     
backend default
     http-request deny
```