# Platforming Blockchain: Ethereum (Private Network) -> Kubernetes

This Ethereum private network is a highly constrained, miniature replica of the public Ethereum network. Private Blockchain networks such as this are
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
Create two or more Ethereum accounts. The Ethereum Genesis file defined in the following ConfigMap instructs the new Blockchain to pre-fund these accounts (in the first block) with a specified amount of Ether (Ethereum cryptocurrency) available for use within the private network.

Example:
```
Password: Tesal0niki! 


$ geth account new
INFO [11-26|16:15:21.792] Maximum peer count                       ETH=50 LES=0 total=50
INFO [11-26|16:15:21.793] Smartcard socket not found, disabling    err="stat /run/pcscd/pcscd.comm: no such file or directory"
Your new account is locked with a password. Please give a password. Do not forget this password.
Password: 
Repeat password: 

Your new key was generated

Public address of the key:   0xC6C8dAB6dF7Af408D7bB7E7d12A7e9A71aD6C465
Path of the secret key file: /home/davar/.ethereum/keystore/UTC--2020-11-26T14-16-16.634178075Z--c6c8dab6df7af408d7bb7e7d12a7e9a71ad6c465

- You can share your public address with anyone. Others need it to interact with you.
- You must NEVER share the secret key with anyone! The key controls access to your funds!
- You must BACKUP your key file! Without the key, it's impossible to access account funds!
- You must REMEMBER your password! Without the password, it's impossible to decrypt the key!

$ geth account new
INFO [11-26|16:16:52.229] Maximum peer count                       ETH=50 LES=0 total=50
INFO [11-26|16:16:52.229] Smartcard socket not found, disabling    err="stat /run/pcscd/pcscd.comm: no such file or directory"
Your new account is locked with a password. Please give a password. Do not forget this password.
Password: 
Repeat password: 

Your new key was generated

Public address of the key:   0x435D0A3c9C0782B31b822Ab12C627e0938f4dfd6
Path of the secret key file: /home/davar/.ethereum/keystore/UTC--2020-11-26T14-17-04.096273966Z--435d0a3c9c0782b31b822ab12c627e0938f4dfd6

- You can share your public address with anyone. Others need it to interact with you.
- You must NEVER share the secret key with anyone! The key controls access to your funds!
- You must BACKUP your key file! Without the key, it's impossible to access account funds!
- You must REMEMBER your password! Without the password, it's impossible to decrypt the key!
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

## Observing and Exercises

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
### JupyterLab 

Jupyter Notebooks are a browser-based (or web-based) IDE (integrated development environments)

Build custom JupyterLab docker image
```
$ cd ./utils/JupyterLab
$ docker build -t jupyterlab-eth .
$ docker tag jupyterlab-eth:latest davarski/jupyterlab-eth:latest
$ docker login 
$ docker push davarski/jupyterlab-eth:latest
```
Run Jupyter Notebook

```
$ sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/k3s-config-jupyter
$ sed -i "s/127.0.0.1/192.168.0.101/" k3s-config-jupyter
$ docker run --rm --name jl -p 8888:8888 \
   -v "$(pwd)":"/home/jovyan/work" \
   -v "$HOME/.kube/k3s-config-jupyter":"/home/jovyan/.kube/config" \
   --user root \
   -e GRANT_SUDO=yes \
   -e JUPYTER_ENABLE_LAB=yes -e RESTARTABLE=yes \
   davarski/jupyterlab-eth:latest
```
Example:
```
$ docker run --rm --name jl -p 8888:8888 \
>    -v "$(pwd)":"/home/jovyan/work" \
>    -v "$HOME/.kube/k3s-config-jupyter":"/home/jovyan/.kube/config" \
>    --user root \
>    -e GRANT_SUDO=yes \
>    -e JUPYTER_ENABLE_LAB=yes -e RESTARTABLE=yes \
>    davarski/jupyterlab-eth:latest

Set username to: jovyan
usermod: no changes
Granting jovyan sudo access and appending /opt/conda/bin to sudo PATH
Executing the command: jupyter lab
[I 21:37:15.811 LabApp] Writing notebook server cookie secret to /home/jovyan/.local/share/jupyter/runtime/notebook_cookie_secret
[I 21:37:16.594 LabApp] Loading IPython parallel extension
[I 21:37:16.614 LabApp] JupyterLab extension loaded from /opt/conda/lib/python3.7/site-packages/jupyterlab
[I 21:37:16.614 LabApp] JupyterLab application directory is /opt/conda/share/jupyter/lab
[W 21:37:16.623 LabApp] JupyterLab server extension not enabled, manually loading...
[I 21:37:16.638 LabApp] JupyterLab extension loaded from /opt/conda/lib/python3.7/site-packages/jupyterlab
[I 21:37:16.638 LabApp] JupyterLab application directory is /opt/conda/share/jupyter/lab
[I 21:37:16.639 LabApp] Serving notebooks from local directory: /home/jovyan
[I 21:37:16.639 LabApp] The Jupyter Notebook is running at:
[I 21:37:16.639 LabApp] http://(e1696ffe20ab or 127.0.0.1):8888/?token=f0c6d63a7ffb4e67d132716e3ed49745e97b3e7fa78db28d
[I 21:37:16.639 LabApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 21:37:16.648 LabApp] 
    
    To access the notebook, open this file in a browser:
        file:///home/jovyan/.local/share/jupyter/runtime/nbserver-17-open.html
    Or copy and paste one of these URLs:
        http://(e1696ffe20ab or 127.0.0.1):8888/?token=f0c6d63a7ffb4e67d132716e3ed49745e97b3e7fa78db28d
```
Open IDE in browser: http://127.0.0.1:8888/?token=f0c6d63a7ffb4e67d132716e3ed49745e97b3e7fa78db28d

Within the Docker container (at localhost:8888), under the section titled Other within the running Jupyter Notebook, chose Terminal. Once the terminal launches, provide the following command to port-forward all services running in the data Namespace on the k8s cluster 
```
sudo kubefwd svc -n data
```
<img src="https://github.com/adavarski/k8s-Blockchain-Ethereum-playground/blob/main/pictures/JupyterLab-kubefwd.png" width="800">

The utility kubefwd connects and port-forwards Pods backing Services on a remote Kubernetes cluster to a matching set of DNS names and ports on the local workstation (in this case a Jupyter Notebook). Once kubefwd is running, connections to services such as http://eth-geth-tx:8545 are possible just as they are from within the remote cluster.

Create a new Python 3 Jupyter Notebook; copy and execute the following code examples within individual cells.
```
pip install web3
```
Import the Python libraries web3, json, and time:
```
import web3, json, time
import pandas as pd
from IPython.display import clear_output
from web3.contract import ConciseContract
from web3 import Web3
from web3.auto.gethdev import w3
```
Connect to the Geth transaction node:
```
rpc_ep = "http://eth-geth-tx:8545"
web3 = Web3(Web3.HTTPProvider(rpc_ep))
if web3.isConnected():
    print(f"Connected: {rpc_ep}")
    print(f"Peers: {web3.net.peerCount}")
    print(f"Chain ID: {web3.net.version}")
    print(f"Last block: {web3.eth.blockNumber}")
else:
    print("Not connected")
```
Example output:
```
Connected: http://eth-geth-tx:8545
Peers: 4
Chain ID: 27587
Last block: 5549
```
Check the eth balance of the accounts pre-funded in the Genesis block
defined earlier:
```
account_1 = "0xFa4087D3688a289c9C92e773a7b46cb9CCf80353"
account_2 = "0x8ab8F3fc6c660d3f0B22490050C843cafd2c0AAC"
a1_bal = web3.eth.getBalance(account_1)
a2_bal = web3.eth.getBalance(account_2)
print(f"Account 1: {web3.fromWei(a1_bal, 'ether')} ether")
print(f"Account 2: {web3.fromWei(a2_bal, 'ether')} ether")
```
Example output:
```
Account 1: 100 ether
Account 2: 200 ether
```
Add the following code to create a transaction, transferring one ether
to account_2:
```
nonce = web3.eth.getTransactionCount(account_1)
print(f"Account 1 nonce: {nonce}")
tx = {
    'nonce': nonce,
    'to': account_2,
    'value': web3.toWei(1, 'ether'),
    'gas': 2000000,
    'gasPrice': web3.toWei('50', 'gwei'),
}
tx
```
Example output:
```
{'nonce': 15,
'to': '0x8ab8F3fc6c660d3f0B22490050C843cafd2c0AAC',
'value': 1000000000000000000,
'gas': 2000000,
'gasPrice': 50000000000}
```
The private key file and password are required to sign the transaction
as account_1. Within the JupyterLab environment, create a text file named
pass1.txt and populate it with the password used to create account_1
earlier in this chapter, the first pre-funded account used in the alloc
section of the genesis.json configuration. Additionally, upload the secret
key file generated from the geth account new command (performed
earlier in this chapter to create the pre-funded Ethereum accounts). Name
the secret key account1.json

Load the private key and password for account_1 and sign the
transaction created earlier:
```
with open('pass1.txt', 'r') as pass_file:
    kf1_pass = pass_file.read().replace('\n', '')
with open("account1.json") as kf1_file:
    enc_key = kf1_file.read();
p_1 = w3.eth.account.decrypt(enc_key, kf1_pass)
signed_tx = web3.eth.account.signTransaction(tx, p_1)
signed_tx
```
Example output:
```
AttributeDict({'rawTransaction': HexBytes('0xf86d0f850ba43b74
00831e8480948ab8f3fc6c660d3f0b22490050c843cafd2c0aac880de0b6b3
a7640000801ca0917ae987a8c808cf01221dad4571fd0b1b8f5429d13c469
c72bc13647e9c1744a068507c8542ccdebb96e534d13a140ddcbdaedbfa3b
a82dcbf86d4b196cc41b1f'),
'hash': HexBytes('0x9de62dc620274e2c9dba2194d90c245a933af8468
ace5f2d38e802da09c06769'),
'r': 65802530150742945852115878650256413649726940478651153584
824595116007827969860,
's': 47182743427096773798449059805443774712403275692049277894
020390344384483433247,
'v': 28}
```

Send the signed transaction to the transaction node and retrieve the
resulting hash. This hash is the unique identifier for the transaction on the
Ethereum Blockchain:
```
signed_tx = signed_tx.rawTransaction
tx_hash = web3.eth.sendRawTransaction(signed_tx)
web3.toHex(tx_hash)
```
Example output:
```
'0x9de62dc620274e2c9dba2194d90c245a933af8468ace5f2d38e802d
a09c06769'
```

After a node receives the transaction, it propagates to all nodes for
validation and inclusion into the pending transaction pool, ready to be
mined with the next block. The following code queries the connected
transaction node every second until the transaction returns with a block
number:
```
%%time
blockNumber = None
check = 0
while type(blockNumber) is not int:
    check += 1
    tx = web3.eth.getTransaction(tx_hash)
    blockNumber = tx.blockNumber
    clear_output(wait=True)
    print(f"Check #{check}\n")
    if type(blockNumber) is not int:
        time.sleep(1)
tx
```
Example output:
```
Check #451
CPU times: user 129 ms, sys: 904 μs, total: 130 ms
Wall time: 11.1 s
AttributeDict({'blockHash': HexBytes('0x676a24aa8117b51958031a2
863b17f91ed3356276036a9de7c596124a6234986'),
'blockNumber': 8050,
'from': '0xFa4087D3688a289c9C92e773a7b46cb9CCf80353',
'gas': 2000000,
'gasPrice': 50000000000,
'hash': HexBytes('0xa3f02c685ff05b13b164afcbe11d2aa83d2dab3ff9
72ee7008cc931282587cee'),
'input': '0x',
'nonce': 16,
'to': '0x8ab8F3fc6c660d3f0B22490050C843cafd2c0AAC',
'transactionIndex': 0,
'value': 1000000000000000000,
'v': 28,
'r': HexBytes('0x89d052927901e8a7a727ebfb7709d4f9b99362c0f0001
f62f37300ed17cb7414'),
's': HexBytes('0x3ea3b4f5f8e4c10e4f30cc5b8a7ff0a833d8714f20744
c289dee86006af420c8')})
```
<img src="https://github.com/adavarski/k8s-Blockchain-Ethereum-playground/blob/main/pictures/JupyterLab-python-exercises.png" width="800">


The transaction is now complete and its record immutably stored
on the private blockchain. The network attempts to create a new block
every 10 to 15 seconds by adjusting the required difficulty; however,
this resource-constrained network with only three miners may fluctuate
considerably. 

The final code block in the exercise queries the transaction
node for the last 100 blocks and plots the time delta between block
timestamps:
```
df = pd.DataFrame(columns=['timestamp'])
for i in range (0,100):
    block = web3.eth.getBlock(tx.blockNumber - i)
    df.loc[i] = [block.timestamp]
df['delta'] = df.timestamp.diff().shift(-1) * -1
df.reset_index().plot(x='index', y="delta", figsize=(12,5))
```
example output from the block timestamp delta plot:

<img src="https://github.com/adavarski/k8s-Blockchain-Ethereum-playground/blob/main/pictures/JupyterLab-python-plot.png" width="800">

## Clean environment

```
kubectl delete -f ./cluster-davar-eth/000-global/00-namespace.yml
```
Note: all resources/objects into data namespace will be auto-removed by k8s.

### Note1: Add k3s worker with NVIDIA Runtime (bare-metal)

```

# Install Ubuntu 18.04 on some bare-metal server/workstation with an NVIDIA GeForce GPU (for example Gaming NVIDIA GeForce 1070 GPU)   .
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
