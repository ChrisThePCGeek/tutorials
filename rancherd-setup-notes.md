# Setting up an HA Rancher/RKE2 cluster using rancherD**

additional config for rancherD https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-linux/rancherd-configuration/

*Note: need fixed registration address aka external load balancer for HA*

Note: for a single node setup you don't need the load balancer or the pre-configured .yaml file with token and tls-san options. Can skip down to run the installer

**Config is below for a sample nginx.conf which can be run in docker or another vm not part of the new cluster**

## starting up the first node:

Create the path and RancherD config file at /etc/rancher/rke2/config.yaml

 mkdir -p /etc/rancher/rke2
 nano /etc/rancher/rke2/config.yaml

Input this into the config and change the values below

    token: my-shared-secret
    tls-san:
      - my-fixed-registration-address.com
      - another-kubernetes-domain.com

run the installer (start here for subsequent nodes from steps below)

    curl -sfL https://get.rancher.io | sh -

once finished verify rancherd is installed

    rancherd --help

start the service and enable it

    systemctl enable rancherd-server.service
    systemctl start rancherd-server.service

check process of cluster coming online

    journalctl -eu rancherd-server -f

copy kube config to local home directory

    mkdir -p ~/.kube
    cp /etc/rancher/rke2/rke2.yaml ~/.kube/config

run this to add the kubectl binary path to the path env variable in ~/.bashrc

    echo "PATH=\$PATH:/var/lib/rancher/rke2/bin" >> ~/.bashrc

to avoid needing to log out/in export the path now.

    export PATH=$PATH:/var/lib/rancher/rke2/bin

now you can run the kubectl commands. To check the status of the deployments use these two commands. Once rancher says 1/1 on the 2nd cmd you can continue

    kubectl get daemonset rancher -n cattle-system
    kubectl get pod -n cattle-system

**stop here for subsequent nodes, continue only if this is first node**

*only do this on the first node, this will give you the URL and initial admin password to enter the rancher web UI*

    rancherd reset-admin


## subsequent nodes

Create the path and RancherD config file at /etc/rancher/rke2/config.yaml

    mkdir -p /etc/rancher/rke2
    nano /etc/rancher/rke2/config.yaml

Input this into the config and change the values below to match the first node's token and tls-san parameters, also add the URL/IP for the LB


    server: https://my-fixed-registration-address.com:9345
    token: my-shared-secret
    tls-san:
      - my-fixed-registration-address.com
      - another-kubernetes-domain.com

*Return back to above for the steps to get the 2nd and 3rd node up and running*


## Nginx LoadBalancer config for HA

    load_module /usr/lib/nginx/modules/ngx_stream_module.so;
    events {}

    stream {
    upstream k3s_servers {
        server 172.16.33.10:6443;
        server 172.16.33.11:6443;
        server 172.16.33.12:6443;
    }

    upstream rancher_servers {
        server 172.16.33.10:8443;
        server 172.16.33.11:8443;
        server 172.16.33.12:8443;
    }

    upstream node_masters {
        server 172.16.33.10:9345;
        server 172.16.33.11:9345;
        server 172.16.33.12:9345;
    }

    server {
        listen 6443;
        proxy_pass k3s_servers;
    }

    server {
        listen 8443;
        proxy_pass rancher_servers;
    }

    server {
        listen 9345;
        proxy_pass node_masters;
    }
    
    }
