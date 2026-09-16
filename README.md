This is based on grycap's Ansible Role (https://github.com/grycap/ansible-role-kubernetes)

Ludus Kubernetes Role
=======================

This ansible role installs a [Kubernetes](https://kubernetes.io/) cluster to [Ludus](https://docs.ludus.cloud/).

## Installation in Ludus

If you want to deploy Kubernetes manifests or Helm charts alongside your range, clone this repository to your Ludus system and run:

```bash
ludus ansible roles add -d ./ludus_k8s
```

Install via Ansible Galaxy (coming soon):

```bash
ludus ansible collection add Sw4mpf0x.ludus_k8s
```

Add and build a Ubuntu 22.04 template:

```bash
ludus source add ludus-source-bsl --templates ubuntu-22.04-x64-server-template
ludus templates build
```

## Ludus Usage

ludus_k8s supports automating the following in your range yaml file:
 - automating Kubernetes misconfigurations
 - BadPods deployment
 - Helm via oci, repo, or local charts
 - deploying your Kubernetes manifests

To deploy in Ludus, point it to your range yaml file and deploy:

```bash
ludus range config set -f k8s-range.yml
ludus range deploy
```

At a minimum, you will need to deploy two nodes, a front (control plane) and worker, when deploying a cluster with kubeadm (default behavior). Helm and kubectl are instaled on the front node, so any manifests or helm charts need to be specified on it in the range.yaml spec, not a worker node. Front pods by default will not deploy pods outside of the control plane, so they will wait until a worker node is availble and then deploy there. 

### Basic Kubernetes Cluster

A basic two node (control plane and worker) cluster with a couple of namespaces and a secret populated. Modifications, as with other ludus rangers, are made via `role_vars`. See examples below for different role variables that can be used to populate the cluster.

```yaml
ludus:
  - vm_name: "{{ range_id }}-k8s-control-plane"
    hostname: "{{ range_id }}-control-plane"
    template: ubuntu-22.04-x64-server-template
    vlan: 20
    ip_last_octet: 1
    ram_gb: 8
    cpus: 4
    linux: true
    testing:
      snapshot: false
      block_internet: false
    roles:
      - ludus-k8s    

  - vm_name: "{{ range_id }}-k8s-node-1"
    hostname: "{{ range_id }}-node-1"
    template: ubuntu-22.04-x64-server-template
    vlan: 20
    ip_last_octet: 20
    ram_gb: 8
    cpus: 4
    linux: true
    testing:
      snapshot: false
      block_internet: false
    roles:
      - ludus-k8s
    role_vars:
      kube_type_of_node: 'wn' # defaults to a 'front' node
      kube_server: '10.2.20.1'
```

### Primary Role Functionality/Variables

#### Populate Namespaces and Secrets

```yaml
  role_vars:
    kube_namespaces:
      - "yourspace"
      - "myspace"
    kube_secrets:
      - name: my-app-env
        namespace: myspace
        type: Opaque
        stringData:
        DB_USER: "dbuser"
        DB_PASS: "S3cretP@ss"
```

#### Deploy BadPods

BadPods is a collection of pod manifests with overly permissive configurations. They were created by BishopFox and can be found [here](https://github.com/BishopFox/badPods). The URL to each BadPod is indexed in the role and can be deployed a specified namespace with the following role variables. In this example, the `everything_allowed_exec` and `hostnetwork_exec` pods are deployed to the badpods, which will be created if it doesn't exist. 

```yaml
  role_vars:
    badpods_namespace: badpods
    badpods:
      - everything_allowed_exec
    # - priv_and_hostpid_exec
    # - priv_exec
    # - hostpath_exec
    # - hostpid_exec
      - hostnetwork_exec
    # - hostipc_exec
    # - nothing_allowed_exec
```

#### Disable kubelet authentication and authorization

If you want to turn off authz/authn on your kubelet APIs, use the following:

```yaml
  role_vars:
    kubelet_anonymous_access: true
```

#### Install Kubernetes Goat

To install [Kubernetes Goat](https://github.com/madhuakula/kubernetes-goat):

```yaml
  role_vars:
    install_kubernetes_goat: true
```

This clones the Kubernetes Goat repository to `/tmp/kubernetes-goat` and runs the install script. To get the portal setup:
- ssh to the front node
- run `sudo chmod +x /tmp/kubernetes-goat/access-kubernetes-goat.sh`
- run `sudo /tmp/kubernetes-goat/access-kubernetes-goat.sh`
- access the portal at `http://<front-node-ip>:1234/`

#### Deploy via kubectl

Deploy a single or folder full of kubernetes manifests with the `kubectl_apply_path` variable. These must be places in the `files` folder of this repository, so you will need to clone it and use `ludus ansible roles add -d ./ludus-k8s` to install the role in Ludus. You will need to re-add it anytime changes are made to the role, including modifications to the `files` folder.

After your range is running, you can use `kubectl` on the front node to interact with the cluster. Note that you need to run `kubectl` with `sudo`.

The following will use the `files/pods.yaml` file:

```yaml
  role_vars:
    kubectl_apply_path: "pods.yaml"
```

For a folder, simply specify the folder name. Here is an example for a folder at `files/manifests`

```yaml
  role_vars:
    kubectl_apply_path: "manifests"
```

#### Deploy via Helm

Helm charts can be deployed as an OCI address, local folder, or by specifying a public repo and name. Charts can be deployed to a specific namespace with the `helm_namespace` variable and the release name can be set with `helm_release_name`.

After your range is running, you can use `helm` on the front node to interact with the cluster. Note that you need to run `kubectl`, and as a result `helm`, with `sudo`.

Public repo and name:

```yaml
  role_vars:
    helm_repo_url: "https://charts.bitnami.com/bitnami"
    helm_repo_name: "my-nginx"   # arbitrary
    helm_chart_name: "nginx"
```

OCI address that installs [podinfo](https://github.com/stefanprodan/podinfo):

```yaml
  role_vars:
    helm_chart_oci_ref: "oci://ghcr.io/stefanprodan/modules/podinfo"
  #  oci_username: ""   # optional
  #  oci_password: ""  # optional
```

Local Folder at `files/mycharts`:

```yaml
  role_vars:
    helm_chart_local_path: "mycharts"
```

## Other Role Variables

The variables that can be passed to this role and a brief description about them are as follows.

```yaml
  # Version to install or latest (1.24 or higher)
  kube_version: 1.32
  # Type of node front (control plane) or wn (worker node)
  kube_type_of_node: front
  # IP address or name of the Kube front node
  kube_server: "{{ ansible_default_ipv4.address }}"
  # Security Token to join nodes to the cluster
  kube_token: "kube01.{{ lookup('password', '/tmp/tokenpass chars=ascii_lowercase,digits length=16') }}"
  # Token TTL duration (0 do not expire)
  kube_token_ttl: 0
  # POD network cidr
  kube_pod_network_cidr: 10.244.0.0/16
  # Type of network to install: currently supported: flannel, kube-router, calico, weave
  kube_network: calico
  # Kubelet extra args
  kubelet_extra_args: ''
  # Kube API server options
  kube_apiserver_options: []
  # Helm version
  kube_install_helm_version: "v3.8.2"
  # Deploy the Dashboard
  kube_deploy_dashboard: false
  # value to pass to the kubeadm init --apiserver-advertise-address option
  kube_api_server: 0.0.0.0
  # A set of git repos and paths to be applied in the cluster. Following this format:
  # kube_apply_repos: [{repo: "https://github.com/kubernetes-incubator/metrics-server", version: "master", path: "deploy/1.8+/"}]
  kube_apply_repos: []
  # Flag to set Metrics-Server to be installed
  kube_install_metrics: false
  # Metrics-Server Helm chart version to install
  kube_metrics_chart_version: "3.12.2"
  # Flag to set the nginx ingress controller to be installed
  kube_install_ingress: false
  # Nginx ingress controller Helm chart version to install
  kube_ingress_chart_version: "4.12.1"
  # Flag to set the kubeapps UI to be installed
  kube_install_kubeapps: false
  # KubeApps chart version to install (or latest)
  kube_kubeapps_chart_version: "7.3.2"
  # Flag to set nfs-client-provisioner to be installed
  kube_install_nfs_client: false
  # NFS path used by nfs-client-provisioner
  kube_nfs_path: /pv
  # NFS server used by nfs-client-provisioner
  kube_nfs_server: kubeserver.localdomain
  # Set reclaimPolicy of NFS StorageClass Delete or Retain
  kube_nfs_reclaim_policy: Delete
  # NFS client Helm chart version to install
  kube_nfs_chart_version: "4.0.18"
  # Extra options for the flannel plugin
  kube_flanneld_extra_args: [] 
  # Enable to install and manage Certificates with Cert-manager
  kube_cert_manager: false
  # Public IP to use by the cert-manager (not needed if kube_public_dns_name is set)
  kube_cert_public_ip: "{{ ansible_default_ipv4.address }}"
  # Public DNS name to use in the dashboard tls certificate
  kube_public_dns_name: ""
  # Email to be used in the Let's Encrypt issuer
  kube_cert_user_email: jhondoe@server.com
  # Override default docker version
  kube_docker_version: ""
  # Options to add in the docker.json file
  kube_docker_options: {}
  # Install docker with pip
  kube_install_docker_pip
  # Command flags to use for launching k3s in the systemd service
  kube_k3_exec: ""
  # How to install K8s: kubeadm or k3s
  kube_install_method: kubeadm
  # Servers to install and configure ntp. If [] ntp will not be configured
  kube_ntp_servers: [ntp.upv.es, ntp.uv.es]
```
