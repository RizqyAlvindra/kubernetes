Learn Kubernetes Using k3s Lightweight Kubernetes
==================

Please check out the official [K3S Doccumentation](https://docs.k3s.io/) for more detailed instructions

Install the server
------------
Login to the server node and install K3S

```sh
$ ssh root@ip

# Make sure your nodes have unique hostnames, set the hostname if your nodes are all named localhost or something

$ hostnamectl set-hostname master
$ echo master > /etc/hostname

# Install K3S using the install script
$ curl -sfL https://get.k3s.io | sh -
```


Verify the installation

```sh
# Make sure you have the /usr/local/bin directory in your path to directly access the installed binaries

$ export PATH=/usr/local/bin:$PATH
$ k3s kubectl get nodes

$ kubectl get nodes
$ kubectl get pods -A

```

Deploy test app
------------
```sh
$ kubectl apply -f web-app.yml

# Verify
$ kubectl get all
$ kubectl get pods,svc 
```

Customise the server configuration
------------
```sh
# Create the configuration file
$ cat <<EOF > /etc/rancher/k3s/config.yaml
cluster-init: true
EOF
# Restart k3s
$ systemctl restart k3s
$ kubectl get nodes
```

Install NGINX Ingress Controller
------------
Disable Traefik Ingress Controller
```sh
# Edit the configuration file
$ cat /etc/rancher/k3s/config.yaml
cluster-init: true
disable: servicelb
disable: traefik


# Restart k3s
$ systemctl restart k3s
```

Create the nginx ingress manifest

```sh
$ cat <<EOF > /var/lib/rancher/k3s/server/manifests/nginx-ingress.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  repo: https://kubernetes.github.io/ingress-nginx
  chart: ingress-nginx
  targetNamespace: ingress-nginx
  valuesContent: |-
    controller:
      image:
        tag: "v1.8.1"
      service:
        type: LoadBalancer
EOF

#Verify
$ kubectl -n ingress-nginx get pods
```

Configure a Private Registry
------------
Create the registries configuration
```sh
# Create the registries.yaml file
$ cat <<EOF > /etc/rancher/k3s/registries.yaml
mirrors:
  registry.home-k8s.lab:
    endpoint:
      - "https://registry.home-k8s.lab"

configs:
  "registry.home-k8s.lab":
    auth:
      username: admin
      password: Harbor12345
    tls:
      insecure_skip_verify: true
EOF

# Restart k3s
$ systemctl restart k3s
```

Deploy test app to use the private registry

```sh
$ kubectl apply -f web-app2.yml

# Verify
$ kubectl get pods,svc
$ kubectl describe pod web-app2-k25r8
```


Configure default storage
------------
Refer to the local-path-provisioner [docs](https://github.com/rancher/local-path-provisioner/blob/master/README.md#usage) for more configuration options. 

Set the default local storage path
```sh
# Edit the configuration file
$ cat /etc/rancher/k3s/config.yaml
cluster-init: true
disable: servicelb
disable: traefik
default-local-storage-path: /mnt/disk1

# Restart k3s
$ systemctl restart k3s
```

Set additional configurations
```sh
# Edit the local-path-config configmap
$ kubectl -n kube-system edit configmap local-path-config

# Restart the local-path-provisioner deployment
$ kubectl -n kube-system rollout restart deploy local-path-provisioner
```

Test the storage
```sh
# Apply the example pvc and pod manifests
$ kubectl create -f pvc.yaml
$ kubectl create -f pod.yaml

#Verify
$ kubectl get pod
$ kubectl get pv
$ kubectl get pvc
```

Adding a control plane node
------------

Retrieve token from etcd capable server node
```sh
$ cat /var/lib/rancher/k3s/server/token
```

Login to new server node and install K3S
```sh
$ ssh root@node2


$ curl -fL https://get.k3s.io | sh -s - server --token <token> --disable-etcd --server https://node1:6443 
```

Persist the configuration by creating a config.yaml file
```sh
# Create the configuration file
$ cat <<EOF > /etc/rancher/k3s/config.yaml
disable: traefik
default-local-storage-path: /mnt/disk1
disable: etcd
EOF

# Restart k3s
$ systemctl restart k3s
```

Verify that the new node was added to the cluster
```sh
# Run kubectl get nodes on any server node
$ kubectl get nodes
```

Adding an agent node
------------

Retrieve token from etcd capable server node
```sh
$ cat /var/lib/rancher/k3s/server/token
```

Login to new server node and install K3S
```sh
$ ssh root@node3


$ curl -sfL https://get.k3s.io | K3S_URL=https://node1:6443 K3S_TOKEN=node1token sh -
```

Persist the configuration by creating a config.yaml file
```sh
# Create the configuration file
$ cat <<EOF > /etc/rancher/k3s/config.yaml
disable: traefik
default-local-storage-path: /mnt/disk1
EOF

# Restart k3s
$ systemctl restart k3s-agent
```

Verify that the new node was added to the cluster
```sh
# Run kubectl get nodes on any server node
$ kubectl get nodes
```

Uninstalling K3S
------------

Retrieve token from etcd capable server node
```sh
# On server nodes
$ /usr/local/bin/k3s-uninstall.sh

# On agent nodes
$ /usr/local/bin/k3s-agent-uninstall.sh
```