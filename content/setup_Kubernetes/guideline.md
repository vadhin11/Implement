# Kubernetes HA Cluster Initialization Guideline with kubeadm

This guideline builds a Kubernetes HA cluster from zero to finish using `kubeadm`, `containerd`, `HAProxy`, `Keepalived`, and `Calico`.

It is cleaned from the original command history. Unnecessary commands such as repeated checks, typo commands, failed attempts, `history`, `watch`, and test deployment cleanup are removed.

## 1. Lab Topology

| Role | Hostname | IP Address |
|---|---|---|
| API VIP | `k8s-api.p2ok.site` | `172.16.2.10` |
| Control Plane 1 | `ubuntu11.p2ok.site` | `172.16.2.11` |
| Control Plane 2 | `ubuntu12.p2ok.site` | `172.16.2.12` |
| Control Plane 3 | `ubuntu13.p2ok.site` | `172.16.2.13` |
| Worker 1 | `ubuntu14.p2ok.site` | `172.16.2.14` |
| Worker 2 | `ubuntu15.p2ok.site` | `172.16.2.15` |
| Worker 3 | `ubuntu16.p2ok.site` | `172.16.2.16` |
| DNS Server | `dns.p2ok.site` | `172.16.2.60` |

Network assumptions:

```text
Interface:    ens33
Gateway:      172.16.2.254
DNS Server:   172.16.2.60
Pod CIDR:     192.168.0.0/16
Service CIDR: 10.96.0.0/12
K8s version:  v1.36.1
CNI:          Calico v3.32.0
```

## 2. DNS Preparation

Make sure DNS resolves the Kubernetes API VIP and all node names.

Example forward records:

```dns
k8s-api    IN A 172.16.2.10
ubuntu11   IN A 172.16.2.11
ubuntu12   IN A 172.16.2.12
ubuntu13   IN A 172.16.2.13
ubuntu14   IN A 172.16.2.14
ubuntu15   IN A 172.16.2.15
ubuntu16   IN A 172.16.2.16
dns        IN A 172.16.2.60
```

Example reverse records:

```dns
10 IN PTR k8s-api.p2ok.site.
11 IN PTR ubuntu11.p2ok.site.
12 IN PTR ubuntu12.p2ok.site.
13 IN PTR ubuntu13.p2ok.site.
14 IN PTR ubuntu14.p2ok.site.
15 IN PTR ubuntu15.p2ok.site.
16 IN PTR ubuntu16.p2ok.site.
60 IN PTR dns.p2ok.site.
```

Validate DNS from every node:

```bash
getent hosts k8s-api.p2ok.site
dig @172.16.2.60 k8s-api.p2ok.site +short
dig @172.16.2.60 -x 172.16.2.10 +short
```

Expected:

```text
172.16.2.10
k8s-api.p2ok.site.
```

## 3. Prepare All Nodes

Run this section on all control-plane and worker nodes.

### 3.1 Set Hostname

Run the correct command on each node.

```bash
hostnamectl set-hostname ubuntu11.p2ok.site
```

For other nodes, replace the hostname:

```bash
hostnamectl set-hostname ubuntu12.p2ok.site
hostnamectl set-hostname ubuntu13.p2ok.site
hostnamectl set-hostname ubuntu14.p2ok.site
hostnamectl set-hostname ubuntu15.p2ok.site
hostnamectl set-hostname ubuntu16.p2ok.site
```

Check:

```bash
hostnamectl
hostname -f
```

### 3.2 Configure Timezone

```bash
timedatectl set-timezone Asia/Bangkok
timedatectl
```

### 3.3 Install Basic Tools

```bash
apt update
apt install -y vim nano curl wget gnupg gpg ca-certificates apt-transport-https dnsutils iputils-ping netcat-openbsd arping htop
```

### 3.4 Disable Swap

```bash
swapoff -a
```

Edit `/etc/fstab` and comment out any swap entry.

```bash
vi /etc/fstab
```

Verify:

```bash
swapon --show
```

Expected: no output.

### 3.5 Enable Required Kernel Parameters

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sysctl --system
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

## 4. Install containerd Runtime

Run this section on all nodes.

### 4.1 Install runc

```bash
apt update
apt install -y runc
```

Verify:

```bash
which runc
runc --version
```

Expected example:

```text
/usr/sbin/runc
runc version 1.3.4-0ubuntu1~24.04.1
```

### 4.2 Install containerd

If you use the binary release method:

```bash
wget https://github.com/containerd/containerd/releases/download/v2.1.8/containerd-2.1.8-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-2.1.8-linux-amd64.tar.gz
```

Create systemd directory and service file:

```bash
mkdir -p /usr/local/lib/systemd/system
vi /usr/local/lib/systemd/system/containerd.service
```

Use this service file:

```ini
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

Enable containerd:

```bash
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd --no-pager
```

### 4.3 Configure containerd

```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

Edit config:

```bash
vi /etc/containerd/config.toml
```

Make sure these values are correct:

```toml
disabled_plugins = []
SystemdCgroup = true
```

Restart containerd:

```bash
systemctl restart containerd
systemctl status containerd --no-pager
```

Verify CRI plugin:

```bash
ctr plugins ls | grep -i cri
```

Expected to see CRI plugins with status `ok`.

## 5. Install crictl

Run this section on all nodes.

```bash
VERSION="v1.36.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

Create config:

```bash
tee /etc/crictl.yaml >/dev/null <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

Verify:

```bash
crictl version
crictl info | grep -A 20 '"runc"'
```

`RuntimeReady` should be `true`.

`NetworkReady` can be `false` before CNI is installed. That is normal.

## 6. Install kubeadm, kubelet, and kubectl

Run this section on all nodes.

```bash
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg
mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet
```

Verify:

```bash
kubeadm version
kubectl version --client
```

## 7. Install HAProxy and Keepalived on Control-Plane Nodes

Run this section on `ubuntu11`, `ubuntu12`, and `ubuntu13`.

```bash
apt update
apt install -y keepalived haproxy
```

Allow HAProxy to bind the VIP even when the VIP is not local on backup nodes:

```bash
cat > /etc/sysctl.d/99-k8s-ha.conf <<'EOF'
net.ipv4.ip_nonlocal_bind = 1
EOF

sysctl --system
```

## 8. Configure HAProxy

Run this section on `ubuntu11`, `ubuntu12`, and `ubuntu13`.

Edit HAProxy config:

```bash
vi /etc/haproxy/haproxy.cfg
```

Append this configuration:

```cfg
frontend k8s_api_frontend
    bind 172.16.2.10:16443
    mode tcp
    option tcplog
    default_backend k8s_api_backend

backend k8s_api_backend
    mode tcp
    option tcp-check
    balance roundrobin
    server ubuntu11 172.16.2.11:6443 check
    server ubuntu12 172.16.2.12:6443 check
    server ubuntu13 172.16.2.13:6443 check
```

Important:

```text
Do not bind HAProxy to 172.16.2.10:6443 on the same control-plane nodes.
kube-apiserver uses port 6443 locally.
Use 172.16.2.10:16443 for HAProxy frontend.
```

Validate and start HAProxy:

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
systemctl enable --now haproxy
systemctl restart haproxy
systemctl status haproxy --no-pager
```

Verify:

```bash
ss -lntp | egrep '6443|16443'
```

Expected before kubeadm init:

```text
haproxy listens on 172.16.2.10:16443
nothing should be listening on 172.16.2.10:6443
```

## 9. Configure Keepalived

Run this section on each control-plane node with the correct priority.

### 9.1 ubuntu11 Keepalived Config

```bash
vi /etc/keepalived/keepalived.conf
```

```cfg
global_defs {
    router_id K8S_CP_1
}

vrrp_script chk_haproxy {
    script "/usr/bin/systemctl is-active --quiet haproxy"
    interval 2
    weight -20
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 110
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass k8sVIP1
    }

    virtual_ipaddress {
        172.16.2.10/24 dev ens33
    }

    track_script {
        chk_haproxy
    }
}
```

### 9.2 ubuntu12 Keepalived Config

```cfg
global_defs {
    router_id K8S_CP_2
}

vrrp_script chk_haproxy {
    script "/usr/bin/systemctl is-active --quiet haproxy"
    interval 2
    weight -20
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass k8sVIP1
    }

    virtual_ipaddress {
        172.16.2.10/24 dev ens33
    }

    track_script {
        chk_haproxy
    }
}
```

### 9.3 ubuntu13 Keepalived Config

```cfg
global_defs {
    router_id K8S_CP_3
}

vrrp_script chk_haproxy {
    script "/usr/bin/systemctl is-active --quiet haproxy"
    interval 2
    weight -20
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 90
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass k8sVIP1
    }

    virtual_ipaddress {
        172.16.2.10/24 dev ens33
    }

    track_script {
        chk_haproxy
    }
}
```

Start Keepalived on all control-plane nodes:

```bash
systemctl enable --now keepalived
systemctl restart keepalived
systemctl status keepalived --no-pager
```

Verify VIP:

```bash
ip addr show ens33 | grep 172.16.2.10
ping -c 3 172.16.2.10
arping -I ens33 172.16.2.10
```

## 10. Preflight Checks Before kubeadm Init

Run this section on `ubuntu11`.

```bash
hostname -f
ip route show
containerd --version
systemctl is-active containerd
ls -l /run/containerd/containerd.sock
ctr plugins ls | grep -i cri
crictl version
crictl info | grep -A 20 '"status"'
swapon --show
getent hosts k8s-api.p2ok.site
dig @172.16.2.60 k8s-api.p2ok.site +short
curl -I https://registry.k8s.io/v2/
```

Expected:

```text
containerd is active
RuntimeReady is true
swap is disabled
k8s-api.p2ok.site resolves to 172.16.2.10
registry.k8s.io is reachable
```

## 11. Create kubeadm Init Config on ubuntu11

Create `/root/kubeadm-config.yaml`:

```bash
cat > kubeadm-config.yaml <<'EOF'
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.16.2.11
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.36.1
controlPlaneEndpoint: k8s-api.p2ok.site:16443
networking:
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
  dnsDomain: cluster.local
imageRepository: registry.k8s.io
EOF
```

Validate:

```bash
kubeadm config validate --config kubeadm-config.yaml
```

Pull required images:

```bash
kubeadm config images pull --config kubeadm-config.yaml
```

Check images:

```bash
crictl images | egrep 'kube-apiserver|kube-controller-manager|kube-scheduler|kube-proxy|coredns|pause|etcd'
```

## 12. Initialize First Control Plane

Run on `ubuntu11`.

Dry run first:

```bash
kubeadm init --config kubeadm-config.yaml --upload-certs --dry-run
```

If dry-run passes, run the real init:

```bash
kubeadm init --config kubeadm-config.yaml --upload-certs
```

Save the output. It contains:

```text
kubeadm join ... --control-plane --certificate-key ...
kubeadm join ... for worker nodes
```

## 13. Configure kubectl on ubuntu11

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```

Optional persistent setting:

```bash
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
```

Check:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

At this stage, nodes may be `NotReady` until CNI is installed.

## 14. Join Additional Control-Plane Nodes

Run the control-plane join command from the `kubeadm init` output on `ubuntu12` and `ubuntu13`.

Example format:

```bash
kubeadm join k8s-api.p2ok.site:16443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY> \
  --cri-socket unix:///run/containerd/containerd.sock
```

If the token expired, generate a new one on `ubuntu11`:

```bash
kubeadm token create --print-join-command
```

If the certificate key expired or was lost, upload certs again on `ubuntu11`:

```bash
kubeadm init phase upload-certs --upload-certs
```

Then append the new `--certificate-key` to the control-plane join command.

After joining, check from `ubuntu11`:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
```

## 15. Install Calico CNI

Run this section on `ubuntu11`.

Use standard Calico first. Do not use the eBPF custom resources for the initial lab.

### 15.1 Install Calico CRDs and Tigera Operator

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/v1_crd_projectcalico_org.yaml
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
```

Check operator:

```bash
kubectl get pods -n tigera-operator -o wide
```

Expected:

```text
tigera-operator   1/1 Running
```

### 15.2 Apply Calico Custom Resources

```bash
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
grep -n "cidr:" custom-resources.yaml
```

Expected:

```text
cidr: 192.168.0.0/16
```

Apply:

```bash
kubectl apply -f custom-resources.yaml
```

Monitor:

```bash
kubectl get pods -n calico-system -o wide
kubectl get tigerastatus
```

Expected:

```text
calico      True   False   False   All objects available
apiserver   True   False   False   All objects available
```

Check all pods:

```bash
kubectl get pods -A -o wide
```

Expected:

```text
calico-node-*                 1/1 Running
calico-typha-*                1/1 Running
calico-kube-controllers-*     1/1 Running
coredns-*                     1/1 Running
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Expected:

```text
ubuntu11.p2ok.site   Ready   control-plane
ubuntu12.p2ok.site   Ready   control-plane
ubuntu13.p2ok.site   Ready   control-plane
```

## 16. Join Worker Nodes

Run the worker join command from `kubeadm init` output on worker nodes `ubuntu14`, `ubuntu15`, and `ubuntu16`.

Example format:

```bash
kubeadm join k8s-api.p2ok.site:16443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket unix:///run/containerd/containerd.sock
```

If token expired, generate a new worker join command on `ubuntu11`:

```bash
kubeadm token create --print-join-command
```

Then add the CRI socket:

```bash
--cri-socket unix:///run/containerd/containerd.sock
```

Check from `ubuntu11`:

```bash
kubectl get nodes -o wide
```

Expected:

```text
ubuntu11.p2ok.site   Ready   control-plane
ubuntu12.p2ok.site   Ready   control-plane
ubuntu13.p2ok.site   Ready   control-plane
ubuntu14.p2ok.site   Ready   <none>
ubuntu15.p2ok.site   Ready   <none>
ubuntu16.p2ok.site   Ready   <none>
```

## 17. Validate Cluster Health

Run from `ubuntu11`.

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get tigerastatus
kubectl get --raw='/readyz?verbose'
kubectl get ippool -o wide
kubectl get pods -A -o wide | grep 192.168
```

Expected IP pool:

```text
192.168.0.0/16
```

## 18. Test Workload After Worker Nodes Are Joined

Do this after worker nodes are Ready.

```bash
kubectl create deployment nginx --image=nginx
kubectl get pods -o wide
```

The nginx pod should schedule on a worker node, not on control-plane nodes.

Expose it:

```bash
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx
```

Test with the NodePort:

```bash
curl http://172.16.2.14:<NODE_PORT>
curl http://172.16.2.15:<NODE_PORT>
curl http://172.16.2.16:<NODE_PORT>
```

Clean up the test:

```bash
kubectl delete deployment nginx
kubectl delete svc nginx
```

If the service does not exist, this message is normal:

```text
Error from server (NotFound): services "nginx" not found
```

## 19. Useful Troubleshooting

### 19.1 Port 6443 Is in Use

Check:

```bash
ss -lntp | grep 6443
```

If HAProxy is using `172.16.2.10:6443`, change HAProxy frontend to `172.16.2.10:16443`.

Do not ignore this preflight error.

### 19.2 runc Is Missing

Symptom:

```text
exec: "runc": executable file not found in $PATH
```

Fix:

```bash
apt update
apt install -y runc
systemctl restart containerd kubelet
```

### 19.3 Calico ImagePullBackOff with Docker Hub 429

Symptom:

```text
429 Too Many Requests
You have reached your unauthenticated pull rate limit
```

Options:

```text
1. Wait for Docker Hub rate limit reset.
2. Use Docker Hub login / imagePullSecret.
3. Pre-pull images on each node using authenticated registry access.
```

### 19.4 CoreDNS Pending

This is normal before CNI is ready.

After Calico is running, CoreDNS should become Running.

### 19.5 Pod Pending on Control-Plane-Only Cluster

Control-plane nodes have this taint by default:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

Check:

```bash
kubectl describe node ubuntu11.p2ok.site | grep -i taint
```

This is why normal workloads should be tested after worker nodes are joined.

## 20. Optional Reset Workflow

Use this only when you need to rebuild a node.

```bash
kubeadm reset -f --cri-socket unix:///run/containerd/containerd.sock
rm -rf /etc/kubernetes
rm -rf /var/lib/etcd
rm -rf /etc/cni/net.d/*
rm -rf $HOME/.kube
systemctl restart containerd kubelet
```

For this lab, `etcdctl del "" --prefix` is not required because kubeadm uses local stacked etcd, not external etcd.

## 21. Final Expected Result

```text
3 control-plane nodes Ready
3 worker nodes Ready
Calico Running
CoreDNS Running
HAProxy listening on VIP port 16443
Keepalived managing VIP 172.16.2.10
kubeadm endpoint: k8s-api.p2ok.site:16443
Pod CIDR: 192.168.0.0/16
Service CIDR: 10.96.0.0/12
```
