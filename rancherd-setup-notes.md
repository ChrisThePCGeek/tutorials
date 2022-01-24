# Setting up an HA Rancher/RKE2 cluster using rancherD

additional config for rancherD https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-linux/rancherd-configuration/

*Note: need fixed registration address aka external load balancer for HA*

**Note: for a single node setup you don't need the load balancer or the pre-configured .yaml file with token and tls-san options. Can skip down to run the installer**

**Config is below for a sample nginx.conf which can be run in docker or another vm not part of the new cluster**

**For all the commands below (aside from kubectl commands after the cluster is up) assume to run while in a root shell or with sudo**
---

### Starting up the first node:

Create the path and RancherD config file at /etc/rancher/rke2/config.yaml

 mkdir -p /etc/rancher/rke2
 nano /etc/rancher/rke2/config.yaml

Input this into the config and change the values below
    disable:
      - rancherd-ingress
    token: my-shared-secret
    tls-san:
      - my-fixed-registration-address.com
      - another-kubernetes-domain.com

run the installer (start here for subsequent nodes from steps below)

    curl -sfL https://get.rancher.io | sh -

once finished verify rancherd is installed

    rancherd --help

start the service and enable it

    systemctl enable --now rancherd-server.service

check process of cluster coming online

    journalctl -eu rancherd-server -f

copy kube config to local home directory  (either on one the server or locally on your system, cat it out and copy/pasta)

    mkdir -p ~/.kube
    cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
or
    cat /etc/rancher/rke2/rke2.yaml #to copy and paste

run this to add the kubectl binary path to the path env variable in ~/.bashrc

    echo "PATH=\$PATH:/var/lib/rancher/rke2/bin" >> ~/.bashrc

to avoid needing to log out/in export the path now.

    export PATH=$PATH:/var/lib/rancher/rke2/bin

now you can run the kubectl commands. To check the status of the deployments use these two commands. Once rancher says 1/1 on the 2nd cmd you can continue

    kubectl get daemonset rancher -n cattle-system
    kubectl get pod -n cattle-system

**stop here for subsequent nodes, continue only if this is first node**

remove the ingress controller that is built in (for using metalLB and traefik2)
    kubectl delete -f /var/lib/rancher/rke2/server/manifests/rke2-ingress-nginx.yaml

*only do this on the first node, this will give you the URL and initial admin password to enter the rancher web UI*

    rancherd reset-admin



### Subsequent nodes

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

---
### Nginx LoadBalancer config for HA

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

### MetalLB Install

    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml

config.yaml:
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```

    kubectl apply -f config.yaml

### Traefik2
To install traefik2 add the helm repo
    helm repo add traefik https://helm.traefik.io/traefik
    helm repo update

then apply the helm chart with the config file template from below
    helm install traefik traefik/traefik --namespace=kube-system --values=traefik-chart-values.yaml

create dashboard.yaml file from below and then apply to turn on the ingressroute for that
    kubectl apply -f dashboard.yaml

**Note: to access dashboard use the domain you specified in the url followed by “/dashboard/” (trailing slash is required for it to work**

#### traefik-chart-values.yaml
```
deployment:
  enabled: true
  # Number of pods of the deployment
  replicas: 1
  # Additional deployment annotations (e.g. for jaeger-operator sidecar injection)
  annotations: {}
  # Additional pod annotations (e.g. for mesh injection or prometheus scraping)
  podAnnotations: {}
  # Additional containers (e.g. for metric offloading sidecars)
  additionalContainers: []
  # Additional initContainers (e.g. for setting file permission as shown below)
  initContainers:
    # The "volume-permissions" init container is required if you run into permission issues.
    # Related issue: https://github.com/containous/traefik/issues/6972
  # Custom pod DNS policy. Apply if `hostNetwork: true`
  # dnsPolicy: ClusterFirstWithHostNet

# Configure ports
ports:
  traefik:
    port: 9000
    exposedPort: 9000
    protocol: TCP
  web:
    port: 8000
    expose: true
    exposedPort: 80
    protocol: TCP
  websecure:
    port: 8443
    expose: true
    exposedPort: 443
    protocol: TCP
    tls:
      enabled: false
      options: ""
      certResolver: ""
      domains: []
  metrics:
    port: 9100
    expose: false
    exposedPort: 9100
    protocol: TCP

ingressRoute:
  dashboard:
    enabled: false

dashboard:
  enabled: true

rbac:
  enabled: true

service:
  enabled: true
  type: LoadBalancer
  # Additional annotations (e.g. for cloud provider specific config)
  annotations: {}
  # Additional service labels (e.g. for filtering Service by custom labels)
  labels: {}
  # Additional entries here will be added to the service spec. Cannot contains
  # type, selector or ports entries.
  spec:
    # externalTrafficPolicy: Cluster
    loadBalancerIP: "192.168.99.20" # this should be your Metal LB IP
    # clusterIP: "2.3.4.5"
  loadBalancerSourceRanges: []
    # - 192.168.0.1/32
    # - 172.16.0.0/16
  externalIPs: []
    # - 1.2.3.4
```

#### dashboard.yaml
```
# dashboard.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
  namespace: kube-system
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefik.example.com`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
```