# Platforming Blockchain: Ethereum (Private Network) -> Kubernetes

This Ethereum private network network is a highly constrained, miniature replica of the public Ethereum network. Private Blockchain networks such as this are
useful for building and deploying experimental nodes, smart contract development, and connecting any aspect of the Blockchain operations into
the more extensive data and application platform development (PaaS/SaaS).

<img src="https://github.com/adavarski/k8s-Blockchain-Ethereum-playground/blob/main/pictures/Blockchain_private_Ethereum_network.png" width="800">

## Prerequisite

### DNS setup

Example: Setup local DNS server

```
$ sudo apt-get install bind9 bind9utils bind9-doc dnsutils
root@carbon:/etc/bind# cat named.conf.options
options {
        directory "/var/cache/bind";
        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };
        listen-on port 53 { any; };
        allow-query { any; };
        forwarders { 8.8.8.8; };
        recursion yes;
        };
root@carbon:/etc/bind# cat named.conf.local 
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone    "davar.com"   {
        type master;
        file    "/etc/bind/forward.davar.com";
 };

zone   "0.168.192.in-addr.arpa"        {
       type master;
       file    "/etc/bind/reverse.davar.com";
 };
root@carbon:/etc/bind# cat reverse.davar.com 
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     davar.com. root.davar.com. (
                             21         ; Serial
                         604820         ; Refresh
                          864500        ; Retry
                        2419270         ; Expire
                         604880 )       ; Negative Cache TTL

;Your Name Server Info
@       IN      NS      primary.davar.com.
primary IN      A       192.168.0.101

;Reverse Lookup for Your DNS Server
101      IN      PTR     primary.davar.com.

;PTR Record IP address to HostName
101      IN      PTR     gitlab.dev.davar.com.
101      IN      PTR     reg.gitlab.dev.davar.com.
101      IN      PTR     dev-k3s.davar.com.
root@carbon:/etc/bind# cat forward.davar.com 
;
; BIND data file for local loopback interface
;
$TTL    604800

@       IN      SOA     primary.davar.com. root.primary.davar.com. (
                              6         ; Serial
                         604820         ; Refresh
                          86600         ; Retry
                        2419600         ; Expire
                         604600 )       ; Negative Cache TTL

;Name Server Information
@       IN      NS      primary.davar.com.

;IP address of Your Domain Name Server(DNS)
primary IN       A      192.168.0.101

;Mail Server MX (Mail exchanger) Record
davar.local. IN  MX  10  mail.davar.com.

;A Record for Host names
gitlab.dev     IN       A       192.168.0.101
reg.gitlab.dev IN       A       192.168.0.101
dev-k3s        IN       A       192.168.0.101
stats.eth      IN       A       192.168.0.101

;CNAME Record
www     IN      CNAME    www.davar.com.

$ sudo systemctl restart bind9
$ sudo systemctl enable bind9

root@carbon:/etc/bind# ping -c1 gitlab.dev.davar.com
PING gitlab.dev.davar.com (192.168.0.101) 56(84) bytes of data.
64 bytes from primary.davar.com (192.168.0.101): icmp_seq=1 ttl=64 time=0.030 ms

--- gitlab.dev.davar.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.030/0.030/0.030/0.000 ms
root@carbon:/etc/bind# nslookup gitlab.dev.davar.com
Server:		192.168.0.101
Address:	192.168.0.101#53

Name:	gitlab.dev.davar.com
Address: 192.168.0.101

$ sudo apt install resolvconf
$ cat /etc/resolvconf/resolv.conf.d/head|grep nameserver
# run "systemd-resolve --status" to see details about the actual nameservers.
nameserver 192.168.0.101
$ sudo systemctl start resolvconf.service
$ sudo systemctl enable resolvconf.service


```

### Install k3s

k3s is "Easy to install. A binary of less than 40 MB. Only 512 MB of RAM required to run." this allows us to utilized Kubernetes for managing the Gitlab application container on a single node while limited the footprint of Kubernetes itself. 

```bash
$ export K3S_CLUSTER_SECRET=$(head -c48 /dev/urandom | base64)
# copy the echoed secret
$ echo $K3S_CLUSTER_SECRET
$ curl -sfL https://get.k3s.io | sh -
```
### Remote Access with `kubectl`

From your local workstation you should be able to issue a curl command to Kubernetes:

```bash
curl --insecure https://SERVER_IP:6443/
```

The new k3s cluster should return a **401 Unauthorized** response with the following payload:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

k3s credentials are stored on the server at `/etc/rancher/k3s/k3s.yaml`:

Review the contents of the generated `k8s.yml` file:

```bash
cat /etc/rancher/k3s/k3s.yaml
```
The `k3s.yaml` is a Kubernetes config file used by `kubectl` and contains (1) one cluster, (3) one user and a (2) context that ties them together. `kubectl` uses contexts to determine the cluster you wish to connect to and use for access credentials. The `current-context` section is the name of the context currently selected with the `kubectl config use-context` command.

Before you being configuring [k3s] make sure `kubectl` pointed to the correct cluster: 

```
$ sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/k3s-config
$ export KUBECONFIG=~/.kube/k3s-config
$ kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

```

### Fix k3s CoreDNS for local development

```
$ cd k8s/utils/
$ sudo cp  /var/lib/rancher/k3s/server/manifests/coredns.yaml ./coredns-fixes.yaml
$ vi coredns-fixes.yaml 
$ sudo chown $USER: coredns-fixes.yaml 
$ sudo diff coredns-fixes.yaml /var/lib/rancher/k3s/server/manifests/coredns.yaml 
75,79d74
<     davar.com:53 {
<         errors
<         cache 30
<         forward . 192.168.0.101
<     }
$ kubectl apply -f coredns-fixes.yaml
serviceaccount/coredns unchanged
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
clusterrole.rbac.authorization.k8s.io/system:coredns unchanged
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
clusterrolebinding.rbac.authorization.k8s.io/system:coredns unchanged
configmap/coredns unchanged
deployment.apps/coredns configured
service/kube-dns unchanged
```

### Crate eth namespace: data

```bash
kubectl apply -f ./cluster-davar-eth/000-global/00-namespace.yml
```

### Install Cert Manager / Self-Signed Certificates

Note: Let's Encrypt will be used with Cert Manager for PRODUCTION/PUBLIC when we have internet accessble public IPs and public DNS domain. davar.com is local domain, so we use Self-Signed Certificates, Let's Encrypt is using public DNS names and if you try to use Let's Encrypt for local domain and IPs you will have issue:


```bash
# Kubernetes 1.16+
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml

# Kubernetes <1.16
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager-legacy.yaml
```

Ensure that cert manager is now running:
```bash
kubectl get all -n cert-manager
```

Output:
```plain
$ kubectl get all -n cert-manager
NAME                                          READY   STATUS    RESTARTS   AGE
pod/cert-manager-cainjector-bd5f9c764-z7bh6   1/1     Running   0          13h
pod/cert-manager-webhook-5f57f59fbc-49jk7     1/1     Running   0          13h
pod/cert-manager-5597cff495-rrl52             1/1     Running   0          13h

NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.43.162.66   <none>        9402/TCP   13h
service/cert-manager-webhook   ClusterIP   10.43.202.9    <none>        443/TCP    13h

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager-cainjector   1/1     1            1           13h
deployment.apps/cert-manager-webhook      1/1     1            1           13h
deployment.apps/cert-manager              1/1     1            1           13h

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-cainjector-bd5f9c764   1         1         1       13h
replicaset.apps/cert-manager-webhook-5f57f59fbc     1         1         1       13h
replicaset.apps/cert-manager-5597cff495             1         1         1       13h


```

Add a ClusterIssuer to handle the generation of Certs cluster-wide:

```bash
kubectl apply -f ./cluster-davar-eth/000-global/003-issuer.yaml
kubectl apply -f ./cluster-davar-eth/000-global/005-clusterissuer.yml
```

## Ethereum private network deploy 

### Bootnodes

```
kubectl apply -f ./cluster-davar-eth/100-eth/10-bootnode/10-service.yml
kubectl apply -f ./cluster-davar-eth/100-eth/10-bootnode/30-deployment.yml

```
### Bootnode Registrar

```
kubectl apply -f ./cluster-davar-eth/100-eth/20-bootnode-reg/10-service.yml
kubectl apply -f ./cluster-davar-eth/100-eth/20-bootnode-reg/30-deployment.yml

```
### Ethstats
Note:  Secret WS_SECRET: "uGYQ7lj55FqFxdyIwsv1" is used later in the command-line arguments supplied to Geth nodes.

```
kubectl apply -f ./cluster-davar-eth/100-eth/30-ethstats/10-service.yml
kubectl apply -f ./cluster-davar-eth/100-eth/30-ethstats/15-secret.yml
kubectl apply -f ./cluster-davar-eth/100-eth/30-ethstats/30-deployment.yml
kubectl apply -f ./cluster-davar-eth/100-eth/30-ethstats/50-ingress.yml

```
Visit https://stats.eth.davar.com in a web browser. There should be no data until Geth nodes begin reporting as configured in
the following sections.

### Geth Miners

```
kubectl apply -f ./cluster-davar-eth/100-eth/40-miner/15-secret.yml
```
Install Geth on a local workstation (ref: https://geth.ethereum.org/downloads/). For Ubuntu:

```
The easiest way to install go-ethereum on Ubuntu-based distributions is with the built-in launchpad PPAs (Personal Package Archives). Use provided a single PPA repository that contains both our stable and development releases for Ubuntu versions trusty, xenial, zesty and artful.

To enable our launchpad repository run:

sudo add-apt-repository -y ppa:ethereum/ethereum
Then install the stable version of go-ethereum:

sudo apt-get update
sudo apt-get install ethereum
Or the develop version via:

sudo apt-get update
sudo apt-get install ethereum-unstable
The abigen, bootnode, clef, evm, geth, puppeth, rlpdump, and wnode commands are then available on your system in /usr/bin/.

Find the different options and commands available with geth --help
```
After creating multiple accounts with the geth account new command, copy and save the “Public address of the key:” from the output.
Next, edit [Geth ConfigMap](cluster-davar-eth/100-eth/40-miner/20-configmap.yml) for Geth. Update the alloc section of the genesis.json with the
newly created accounts. The genesis.json file defined within the ConfigMap configures the first block of an Ethereum Blockchain. Any node wishing
to join the private network must first initialize against this Ethereum Genesis file. Both miner and transaction nodes described in the following
are configured to mount the genesis.json file defined as a key in the ConfigMap.

```
kubectl apply -f ./cluster-davar-eth/100-eth/40-miner/20-configmap.yml
kubectl apply -f ./cluster-davar-eth/100-eth/40-miner/30-deployment.yml

```
### Geth Transaction Nodes

```
kubectl apply -f ./cluster-davar-eth/100-eth/50-tx/10-service.yml
kubectl apply -f ./cluster-davar-eth/100-eth/50-tx/30-deployment.yml

```

At this stage, there should now be five nodes reporting into the Ethstats Dashboard configured earlier, consisting of three miners and two transaction nodes:

<img src="https://github.com/adavarski/k8s-Blockchain-Ethereum-playground/blob/main/pictures/Blockchain-Ethereum-stats-private-Ethereum-nodes-reporting.png" width="800">

## Observing 

### Geth Attach
Geth offers an interactive console (https://github.com/ethereum/go-ethereum/wiki/JavaScript-Console) for interacting with its API. One of the
easiest ways to experiment with the API involves using geth to attach to another local instance of geth. The following example executes geth on
one of the three miner nodes and interacts with the running miner (Example geth console output):
```
$ kubectl get all -n data
NAME                                       READY   STATUS    RESTARTS   AGE
pod/eth-bootnode-85847546f6-gmbwl          2/2     Running   0          34m
pod/eth-bootnode-85847546f6-zsjrr          2/2     Running   0          34m
pod/eth-bootnode-registrar-b458ccb-mgfc2   1/1     Running   0          31m
pod/eth-ethstats-5f7fdbd57-l6h64           1/1     Running   0          27m
pod/eth-geth-miner-6b998b7565-m9vwm        1/1     Running   0          2m49s
pod/eth-geth-miner-6b998b7565-2hwxf        1/1     Running   0          2m49s
pod/eth-geth-miner-6b998b7565-zpxjk        1/1     Running   0          2m49s
pod/eth-geth-tx-f8c7db78c-p2n8c            1/1     Running   0          52s
pod/eth-geth-tx-f8c7db78c-dfpdv            1/1     Running   0          52s

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
service/eth-bootnode             ClusterIP   None            <none>        30301/UDP,8080/TCP   35m
service/eth-bootnode-registrar   ClusterIP   10.43.134.135   <none>        80/TCP               31m
service/eth-ethstats             ClusterIP   10.43.60.27     <none>        8080/TCP             29m
service/eth-geth-tx              ClusterIP   10.43.156.10    <none>        8545/TCP,8546/TCP    75s

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/eth-bootnode             2/2     2            2           34m
deployment.apps/eth-bootnode-registrar   1/1     1            1           31m
deployment.apps/eth-ethstats             1/1     1            1           27m
deployment.apps/eth-geth-miner           3/3     3            3           2m49s
deployment.apps/eth-geth-tx              2/2     2            2           52s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/eth-bootnode-85847546f6          2         2         2       34m
replicaset.apps/eth-bootnode-registrar-b458ccb   1         1         1       31m
replicaset.apps/eth-ethstats-5f7fdbd57           1         1         1       27m
replicaset.apps/eth-geth-miner-6b998b7565        3         3         3       2m49s
replicaset.apps/eth-geth-tx-f8c7db78c            2         2         2       52s

At this stage, there should now be five nodes reporting into the Ethstats
Dashboard configured earlier (see Figure 10-4), consisting of three miners
and two transaction nodes.

$ kubectl exec -it -n data eth-geth-miner-6b998b7565-m9vwm -- geth attach
Welcome to the Geth JavaScript console!

instance: Geth/v1.9.13-stable-cbc4ac26/linux-amd64/go1.14.2
coinbase: 0x662c50ab9c34945f06df2112fd02b6d5490bd6c0
at block: 0 (Thu Jan 01 1970 00:00:00 GMT+0000 (UTC))
 datadir: /root/.ethereum
 modules: admin:1.0 debug:1.0 eth:1.0 ethash:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

> eth.blockNumber
6
```

Communicate with geth from a local workstation by port-forwarding the eth-geth-tx Service, set up earlier, and attach a local geth to the forwarded Service.
```
$ kubectl port-forward svc/eth-geth-tx 8545 -n data
Forwarding from 127.0.0.1:8545 -> 8545
Forwarding from [::1]:8545 -> 8545
```
Open an additional terminal on the local workstation and attach geth:
```
$ geth attach http://localhost:8545
```
Geth’s interactive JavaScript console is a great way to explore the API. However, Ethereum provides a variety of mature client libraries for
building applications that interact with the Ethereum Blockchain. 

### Python: Geth 

Ethereum’s Web3 Python library example:

```
$ cd ./utils/functions/last-block
$ sudo apt-get install python3-venv python3-dev
$ python3 -m venv venv
$ source ./venv/bin/activate
(venv) $ pip install wheel
(venv) $ pip install hexbytes==0.2.0 web3==5.9.0
(venv) $ pip freeze > requirements.txt

# Test the last-block function on a local workstation by port-forwarding the eth-geth-tx service in one terminal and executing the Python script handler.py in another.Open a separate terminal and port-forward eth-geth-tx:
$ kubectl port-forward svc/eth-geth-tx 8545:8545 -n data
Forwarding from 127.0.0.1:8545 -> 8545
Forwarding from [::1]:8545 -> 8545

# Execute the Python script handler.py from the current (virtual environment enabled) terminal:

(venv) $ export GETH_RPC_ENDPOINT=http://localhost:8545
(venv) $ $ python3 ./handler.py
{'statusCode': 200, 'body': {'difficulty': 131072, 'extraData': '0xd88301090d846765746888676f312e31342e32856c696e7578', 'gasLimit': 133433221, 'gasUsed': 0, 'hash': '0x1977cc65cd4e2404fb292324156ee7f6609fc89e890111d68c1d8c2aeb9a25b2', 'logsBloom': '0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000', 'miner': '0x06E0266a396460eB52B958110f64181b990AE429', 'mixHash': '0x20637c0da5a60fd3214aae900e13a6141ffc00ea1d1a137e416b0d8241e09335', 'nonce': '0x1fac3c05f48db9ba', 'number': 6, 'parentHash': '0x4d979042da5cbacfd2c7dd308d114b8e02942eeae872ca3b4d1aea442031b04c', 'receiptsRoot': '0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421', 'sha3Uncles': '0xf60abffc8c8f729c11dc3d8a228d99764cf85caa42e35b956f25cfafe32ef12f', 'size': 1071, 'stateRoot': '0x9c5675b7ba80c5a781bed22364cdae8c53278117933b52992c714a9664063ec2', 'timestamp': 1606409163, 'totalDifficulty': 787456, 'transactions': [], 'transactionsRoot': '0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421', 'uncles': ['0xc1a694719c7d3b720f3ba31464ecbf70c68c3ecb0dbcd75bd88e5847eb9c85e4']}}


$ deactivate
```
## Clean environment

```
kubectl delete -f ./cluster-davar-eth/000-global/00-namespace.yml
```
Note: all resources/objects into data namespace will be auto-removed by k8s.

### Note1: Add k3s worker (bare-metal) with NVIDIA Runtime

```

# Install Ubuntu 18.04 on some bare-metal server/workstation with an NVIDIA GeForce GPU (Gaming)  .
$ sudo su
$ apt update && apt upgrade -y
$ apt install -y apt-transport-https \
ca-certificates gnupg-agent \
software-properties-common
$ apt install -y linux-headers-$(uname -r)
# Ceph block device kernel module (for Ceph support)
$ modprobe rbd
# NVIDIA GeForce GPU support. Install GPU drivers along with the nvidia-container-runtime plug-in for
containerd, the default container runtime for k3s.
$ apt install ubuntu-drivers-common
$ modprobe ipmi_devintf
$ add-apt-repository -y ppa:graphics-drivers
$ curl -s -L \
   https://nvidia.github.io/nvidia-docker/gpgkey \
   | sudo apt-key add -
$ curl -s -L https://nvidia.github.io/nvidia-container-runtime/
ubuntu18.04/nvidia-container-runtime.list | tee /etc/apt/
sources.list.d/nvidia-container-runtime.list   
$ apt-get install -y nvidia-driver-440
$ apt-get install -y nvidia-container-runtime
$ apt-get install -y nvidia-modprobe nvidia-smi
$ /sbin/modprobe nvidia
$ /sbin/modprobe nvidia-uvm  
$ reboot
$ nvidia-smi
# k3s with NVIDIA Runtime
sudo su -
export K3S_CLUSTER_SECRET="<PASTE VALUE>"
export K3S_URL="https://dev-k3s.davar.com:6443"
export INSTALL_K3S_SKIP_START=true
$ curl -sfL https://get.k3s.io | \
sh -s - agent 
$ mkdir -p /var/lib/rancher/k3s/agent/etc/containerd/
$ cat <<"EOF" > \
/var/lib/rancher/k3s/agent/etc/containerd/config.toml
[plugins.opt]
  path = "/var/lib/rancher/k3s/agent/containerd"
[plugins.cri]
  stream_server_address = "127.0.0.1"
  stream_server_port = "10010"
  sandbox_image = "docker.io/rancher/pause:3.1"
[plugins.cri.containerd.runtimes.runc]
  runtime_type = "io.containerd.runtime.v1.linux"
[plugins.linux]
  runtime = "nvidia-container-runtime"
EOF
$ systemctl start k3s
$ kubectl label node gpu-metal kubernetes.io/role=gpu
# Edit k8s Geth miners yaml ( ./cluster-davar-eth/100-eth/40-miner/30-deployment.yml) to run Pods on gpu-metal using label:gpu

```

## Technology Reference
- [Ethereum]
- [Geth]
- [Ethstats]

## Listings

- [Listing : Bootnode Service](cluster-davar-eth/100-eth/10-bootnode/10-service.yml)
- [Listing : Bootnode Deployment](cluster-davar-eth/100-eth/10-bootnode/30-deployment.yml)
- [Listing : Bootnode Registrar Service](cluster-davar-eth/100-eth/20-bootnode-reg/10-service.yml)
- [Listing : Bootnode Deployment](cluster-davar-eth/100-eth/20-bootnode-reg/30-deployment.yml)
- [Listing : Ethstats Service](cluster-davar-eth/100-eth/30-ethstats/10-service.yml)
- [Listing : Ethstats Secret](cluster-davar-eth/100-eth/30-ethstats/15-secret.yml)
- [Listing : Ethstats Deployment](cluster-davar-eth/100-eth/30-ethstats/30-deployment.yml)
- [Listing : Ethstats Ingress](cluster-davar-eth/100-eth/30-ethstats/50-ingress.yml)
- [Listing : Geth Secret](cluster-davar-eth/100-eth/40-miner/15-secret.yml)
- [Listing : Geth ConfigMap](cluster-davar-eth/100-eth/40-miner/20-configmap.yml)
- [Listing : Geth Deployment](cluster-davar-eth/100-eth/40-miner/30-deployment.yml)
- [Listing : Geth transaction node Service](cluster-davar-eth/100-eth/50-tx/10-service.yml)
- [Listing : Geth transaction node Deployment](cluster-davar-eth/100-eth/50-tx/30-deployment.yml)
- [Listing : Function for returning details on the last block in the Blockchain](/utils/functions/last-block/handler.py)<!-- @IGNORE PREVIOUS: link -->

[Ethstats]: https://github.com/cubedro/eth-netstats
[Geth]: https://geth.ethereum.org/
[Ethereum]: https://ethereum.org/en/
