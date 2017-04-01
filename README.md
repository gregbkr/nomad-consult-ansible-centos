# Deploy Nomad & Consult via ansible, on Centos nodes #

## What you will get? ###

* Setup a 3 nodes (1 server, 2 clients) hashicorp nomad container orchestrator
* Run all on your prefered cloud provider via ansible


## Architecture of the stack

![nomad.PNG](https://github.com/gregbkr/nomad-consult-ansible-centos/raw/master/nomad.PNG)

- **Master/client nomad host**: master will take care of electing cluster leader, planning and rescheduling nomad jobs. While client will report node status to the master and look for jobs to run, and then run container
- **Consul**: service discovery. Nomad will record nodes, services in this database. Consul is replicated and  present on all nodes so nomad agents always have access to this data locally
- **dnsmasq**: a kind of proxy for dns resolution, route *.consul query to consul, and other query to localhost then internet
- **Ansible**: recipe will provide easy deployment of the solution

## Prerequisit

* Ansible
* A cloud infra providing centos hosts

## Deploy nomad cluster

Clone this repo

    git clone https://github.com/gregbkr/nomad-consult-ansible-centos && cd nomad

Please deploy 3 standard Centos hosts (nomad-server1, nomad-client1, nomad-client2) on your prefered cloud provider.

Edit ansible inventory with your ips and ssh access key:

    nano inventory

**Deploy consul, docker, dnsmasq & nomad**

Run few times until you got no more errors)

    ansible-playbook playbook.yml

**Checks**

Consul: you can view consul UI on http://a_node_ip:8500/
Here are registered nomad clients and consul itself.

Dnsmask will help redirect dns queries to *.consul
Ssh connect to a node and try some dsn query:

    ping node1.node.consul.
    ping node2.node.consul.
    ping consul.service.consul.	

## Run nomad jobs

We will deploy here a very simple application:
- One nginx which redirect 80 --> flask app on 5000
- Flask app, which records in a redis database a number of view
- A redis database, on port 6379

Each of these components exposes a service, registered and avalaible for query in consul.

Copy nomad jobs to a client node and connect to it:

    scp -r -i ~/.ssh/id_rsa_sbexx jobs/* root@client_node_ip:/root/
    ssh -i ~/.ssh/id_rsa_sbexx jobs root@client_node_ip

    nomad run redis.nomad
    nomad run flask.nomad
    nomad run nginx.nomad

**Test services**

Redis

    echo 'PING' | nc global-redis-check.service.consul 6379

flask
  
    curl global-flask-check.service.consul:5000
 
Nginx

    curl global-nginx-check.service.consul
 

## Troubleshooting

check nomad status

    nomad status
    nomad server-members
    nomad status redis
    nomad alloc-status <alloc-id>

Delete nomad job

    nomad stop redis
