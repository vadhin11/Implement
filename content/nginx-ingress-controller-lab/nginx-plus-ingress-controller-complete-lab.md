# NGINX Plus Ingress Controller LAB Guideline

**Document purpose:** This lab guideline is a practical reference for installing and validating **F5 NGINX Ingress Controller with NGINX Plus** on one Kubernetes cluster. It uses task/job/situation language and does not refer to a commercial scope document.

**Lab tasks covered:**

- Install and configure **F5 NGINX Ingress Controller with NGINX Plus** on **1 Kubernetes cluster**.
- Run the controller with **2 replicas**.
- Use an **NGINX Plus trial license JWT**.
- Configure routing for **1 sample application**.
- Demonstrate advanced configuration by using:
  - Ingress annotations
  - NGINX Ingress Controller ConfigMap
- Enable and demonstrate CRDs:
  - `VirtualServer`
  - `VirtualServerRoute`
  - `Policy`
- Configure SSL/TLS termination at the Ingress layer.
- Demonstrate advanced routing situations:
  - Path-based routing
  - Traffic splitting
  - Access control
  - Rate limiting
- Demonstrate a WAF situation using **F5 WAF for NGINX** if your trial entitlement includes WAF access.
- Integrate logging using Kubernetes logs and an optional open-source logging stack.

> **Important license note:** NGINX Plus Ingress Controller requires an active NGINX Plus subscription or trial JWT. F5 WAF for NGINX requires additional WAF entitlement, WAF image access, and WAF policy bundle preparation. If the trial license includes only NGINX Plus and not F5 WAF for NGINX, complete the NGINX Plus Ingress Controller tasks and skip the WAF task.

---

## 1. Lab Architecture

```text
Client / curl
    |
    | Host: webapp.lab.local
    |
NodeIP:NodePort
    |
NGINX Plus Ingress Controller, 2 replicas
    |
Ingress / VirtualServer / VirtualServerRoute / Policy
    |
Sample application service
    |
Sample application pods
```

### Kubernetes baseline assumption

This guide assumes the Kubernetes cluster is already ready and worker nodes are joined.

Example baseline:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get svc -A -o wide
```

Expected:

```text
Control-plane nodes: Ready
Worker nodes: Ready
CoreDNS: Running
CNI: Running
kube-proxy: Running
```

---

## 2. Lab Variables

Run these commands on the admin node, for example `ubuntu11`.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf

export NIC_NAMESPACE=nginx-ingress
export APP_NAMESPACE=app-lab
export INGRESS_CLASS=nginx
export LAB_HOST=webapp.lab.local

mkdir -p ~/nginx-plus-ingress-lab
cd ~/nginx-plus-ingress-lab
```

---

## 3. Pre-check Task

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get ingressclass || true
kubectl get ingress -A || true
kubectl get crd | egrep 'virtualservers|virtualserverroutes|policies.k8s.nginx.org' || true
```

Install useful packages:

```bash
apt update
apt install -y curl vim jq openssl apache2-utils git
```

Install Helm if it is not already installed:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 3.1 Optional Fix: Bash and kubectl Tab Completion

Use this task if pressing the **Tab** key while typing `kubectl` commands shows an error such as:

```text
bash: _get_comp_words_by_ref: command not found
```

This happens when the `bash-completion` package or Bash completion script is not loaded before `kubectl completion bash`.

Install Bash completion:

```bash
apt update
apt install -y bash-completion
```

Edit `.bashrc`:

```bash
vi ~/.bashrc
```

Add this near the bottom of the file, before or replacing the existing `source <(kubectl completion bash)` lines:

```bash
if [ -f /usr/share/bash-completion/bash_completion ]; then
    . /usr/share/bash-completion/bash_completion
fi

source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
```

Reload the shell profile:

```bash
source ~/.bashrc
```

Test completion:

```bash
kubectl get po<Tab>
kubectl create secret generic nplus-license --from-file=license.jwt=<Tab>
```

If the shell still shows the same error, open a new SSH session and test again.

---

## 4. NGINX Plus License and Registry Preparation

### 4.1 Place the trial JWT file on the admin node

Copy your NGINX Plus trial JWT file to:

```bash
/root/nginx-plus-ingress-lab/license.jwt
```

Verify the file exists:

```bash
ls -l license.jwt
```

Do not print the JWT content to the terminal.

### 4.2 Create the controller namespace

```bash
kubectl create namespace $NIC_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
```

### 4.3 Create the NGINX Plus license secret

```bash
kubectl create secret generic nplus-license \
  --from-file=license.jwt=license.jwt \
  --type=nginx.com/license \
  -n $NIC_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl get secret nplus-license -n $NIC_NAMESPACE
```

### 4.4 Create the private registry secret

Use the JWT as the Docker registry username and `none` as the password.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.nginx.com \
  --docker-username="$(cat license.jwt)" \
  --docker-password=none \
  -n $NIC_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl get secret regcred -n $NIC_NAMESPACE
```

Security cleanup:

```bash
history -d $(history 1 | awk '{print $1}') 2>/dev/null || true
```

---

## 5. Install NGINX Plus Ingress Controller with Helm

### 5.1 Add Helm repository

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
```

The current NGINX documentation also supports installing from the OCI chart repository. This guide uses the NGINX stable Helm repository because it is simple for a lab.

### 5.2 Install CRDs

Install CRDs before using `VirtualServer`, `VirtualServerRoute`, and `Policy`.

```bash
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.5.0/deploy/crds.yaml
```

Verify:

```bash
kubectl get crd | egrep 'virtualservers|virtualserverroutes|policies|globalconfigurations|transportservers'
```

Expected examples:

```text
virtualservers.k8s.nginx.org
virtualserverroutes.k8s.nginx.org
policies.k8s.nginx.org
globalconfigurations.k8s.nginx.org
transportservers.k8s.nginx.org
```

### 5.3 Create Helm values for NGINX Plus with 2 replicas

Create `nic-plus-values.yaml`:

```bash
cat > nic-plus-values.yaml <<'YAML'
controller:
  nginxplus: true

  replicaCount: 2

  image:
    repository: private-registry.nginx.com/nginx-ic/nginx-plus-ingress
    tag: "5.5.0"

  serviceAccount:
    imagePullSecretName: regcred

  mgmt:
    licenseTokenSecretName: nplus-license

  ingressClass:
    name: nginx
    create: true
    setAsDefaultIngress: false

  enableCustomResources: true
  enablePreviewPolicies: true

  service:
    type: NodePort
    httpPort:
      nodePort: 30080
    httpsPort:
      nodePort: 30443

  config:
    name: nginx-config
    entries:
      proxy-connect-timeout: "30s"
      proxy-read-timeout: "60s"
      proxy-send-timeout: "60s"
      client-max-body-size: "20m"
      error-log-level: "info"
      log-format: '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$request_time" "$upstream_response_time"'

  nginxStatus:
    enable: true

prometheus:
  create: true
  port: 9113
YAML
```

### 5.4 Install controller

```bash
helm install nic nginx-stable/nginx-ingress \
  -n $NIC_NAMESPACE \
  -f nic-plus-values.yaml
```

Verify:

```bash
kubectl get pods -n $NIC_NAMESPACE -o wide
kubectl get deploy -n $NIC_NAMESPACE
kubectl get svc -n $NIC_NAMESPACE -o wide
kubectl get ingressclass
```

Expected:

```text
2 controller pods Running
Deployment READY 2/2
Service exposes NodePort 30080 and 30443
IngressClass nginx exists
```

If the controller image cannot be pulled:

```bash
kubectl describe pod -n $NIC_NAMESPACE -l app.kubernetes.io/instance=nic
kubectl get events -n $NIC_NAMESPACE --sort-by=.lastTimestamp | tail -n 30
```

Common causes:

```text
Invalid JWT
Expired trial license
No access to private-registry.nginx.com
Wrong image tag
Firewall or proxy blocking registry access
```

---

## 6. Prepare 1 Sample Application

This lab uses one simple web application with two versions. It still counts as one application because both versions represent the same application.

### 6.1 Create namespace

```bash
kubectl create namespace $APP_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
```

### 6.2 Deploy application version 1

```bash
cat > app-v1.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-v1
  namespace: app-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
      version: v1
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
      - name: webapp
        image: hashicorp/http-echo:1.0
        args:
        - "-text=webapp version v1"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-v1
  namespace: app-lab
spec:
  selector:
    app: webapp
    version: v1
  ports:
  - port: 80
    targetPort: 5678
YAML

kubectl apply -f app-v1.yaml
```

### 6.3 Deploy application version 2

```bash
cat > app-v2.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-v2
  namespace: app-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
      version: v2
  template:
    metadata:
      labels:
        app: webapp
        version: v2
    spec:
      containers:
      - name: webapp
        image: hashicorp/http-echo:1.0
        args:
        - "-text=webapp version v2"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-v2
  namespace: app-lab
spec:
  selector:
    app: webapp
    version: v2
  ports:
  - port: 80
    targetPort: 5678
YAML

kubectl apply -f app-v2.yaml
```

Verify:

```bash
kubectl get deploy,svc,pods -n $APP_NAMESPACE -o wide
```

---

## 7. Get Controller Access Information

```bash
export NIC_IP=$(kubectl get pod -n $NIC_NAMESPACE -l app.kubernetes.io/instance=nic -o jsonpath='{.items[0].status.hostIP}')
export HTTP_PORT=$(kubectl get svc -n $NIC_NAMESPACE nic-nginx-ingress-controller -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
export HTTPS_PORT=$(kubectl get svc -n $NIC_NAMESPACE nic-nginx-ingress-controller -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

echo "NIC_IP=$NIC_IP"
echo "HTTP_PORT=$HTTP_PORT"
echo "HTTPS_PORT=$HTTPS_PORT"
```

Expected:

```text
HTTP_PORT=30080
HTTPS_PORT=30443
```

---

## 8. Task: Standard Ingress Routing with Annotations

This task demonstrates routing for one sample application using standard Kubernetes `Ingress`.

```bash
cat > webapp-ingress.yaml <<'YAML'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: app-lab
  annotations:
    nginx.org/proxy-connect-timeout: "30s"
    nginx.org/proxy-read-timeout: "60s"
    nginx.org/proxy-send-timeout: "60s"
    nginx.org/client-max-body-size: "20m"
spec:
  ingressClassName: nginx
  rules:
  - host: webapp.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-v1
            port:
              number: 80
YAML

kubectl apply -f webapp-ingress.yaml
```

Verify:

```bash
kubectl get ingress -n $APP_NAMESPACE
kubectl describe ingress webapp-ingress -n $APP_NAMESPACE
```

Test:

```bash
curl -i -H "Host: $LAB_HOST" http://$NIC_IP:$HTTP_PORT/
```

Expected response body:

```text
webapp version v1
```

---

## 9. Task: Advanced Configuration with ConfigMap

The controller ConfigMap was created by Helm as `nginx-config`.

Check it:

```bash
kubectl get configmap nginx-config -n $NIC_NAMESPACE -o yaml
```

Update global controller behavior:

```bash
kubectl patch configmap nginx-config -n $NIC_NAMESPACE --type merge -p '{
  "data": {
    "proxy-connect-timeout": "30s",
    "proxy-read-timeout": "60s",
    "proxy-send-timeout": "60s",
    "client-max-body-size": "20m",
    "error-log-level": "info"
  }
}'
```

Restart controller if required:

```bash
kubectl rollout restart deployment nic-nginx-ingress-controller -n $NIC_NAMESPACE
kubectl rollout status deployment nic-nginx-ingress-controller -n $NIC_NAMESPACE
```

Validate:

```bash
curl -i -H "Host: $LAB_HOST" http://$NIC_IP:$HTTP_PORT/
```

---

## 10. Task: SSL/TLS Termination at the Ingress Layer

### 10.1 Create a self-signed lab certificate

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout webapp.lab.local.key \
  -out webapp.lab.local.crt \
  -subj "/CN=webapp.lab.local/O=Lab"
```

Create TLS secret:

```bash
kubectl create secret tls webapp-tls \
  --cert=webapp.lab.local.crt \
  --key=webapp.lab.local.key \
  -n $APP_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 10.2 Update Ingress for TLS

```bash
cat > webapp-ingress-tls.yaml <<'YAML'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: app-lab
  annotations:
    nginx.org/proxy-connect-timeout: "30s"
    nginx.org/proxy-read-timeout: "60s"
    nginx.org/proxy-send-timeout: "60s"
    nginx.org/client-max-body-size: "20m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - webapp.lab.local
    secretName: webapp-tls
  rules:
  - host: webapp.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-v1
            port:
              number: 80
YAML

kubectl apply -f webapp-ingress-tls.yaml
```

Test TLS:

```bash
curl -k -i --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
```

Expected response body:

```text
webapp version v1
```

---

## 11. Situation: VirtualServer and VirtualServerRoute

This situation demonstrates how routing logic can be split into a parent `VirtualServer` and separate child `VirtualServerRoute` resources.

The parent `VirtualServer` owns the hostname and TLS settings. Each child `VirtualServerRoute` owns one route section.

### 11.1 Remove standard Ingress

```bash
kubectl delete ingress webapp-ingress -n $APP_NAMESPACE --ignore-not-found
```

### 11.2 Create VirtualServerRoute for `/v1`

```bash
cat > webapp-v1-vsr.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: VirtualServerRoute
metadata:
  name: webapp-v1-routes
  namespace: app-lab
spec:
  host: webapp.lab.local
  upstreams:
  - name: webapp-v1
    service: webapp-v1
    port: 80
  subroutes:
  - path: /v1
    action:
      pass: webapp-v1
YAML

kubectl apply -f webapp-v1-vsr.yaml
```

### 11.3 Create VirtualServerRoute for `/v2`

```bash
cat > webapp-v2-vsr.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: VirtualServerRoute
metadata:
  name: webapp-v2-routes
  namespace: app-lab
spec:
  host: webapp.lab.local
  upstreams:
  - name: webapp-v2
    service: webapp-v2
    port: 80
  subroutes:
  - path: /v2
    action:
      pass: webapp-v2
YAML

kubectl apply -f webapp-v2-vsr.yaml
```

### 11.4 Create parent VirtualServer

```bash
cat > webapp-vs.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: webapp
  namespace: app-lab
spec:
  host: webapp.lab.local
  ingressClassName: nginx
  tls:
    secret: webapp-tls
  routes:
  - path: /v1
    route: app-lab/webapp-v1-routes
  - path: /v2
    route: app-lab/webapp-v2-routes
YAML

kubectl apply -f webapp-vs.yaml
```

Verify:

```bash
kubectl get vs,vsr -n $APP_NAMESPACE -o wide
kubectl describe vs webapp -n $APP_NAMESPACE
kubectl describe vsr webapp-v1-routes -n $APP_NAMESPACE
kubectl describe vsr webapp-v2-routes -n $APP_NAMESPACE
```

Expected state:

```text
VirtualServer: Valid
VirtualServerRoute webapp-v1-routes: Valid
VirtualServerRoute webapp-v2-routes: Valid
```

Expected event:

```text
Normal AddedOrUpdated Configuration for app-lab/webapp was added or updated
```

Test:

```bash
curl -k --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/v1
curl -k --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/v2
```

Expected:

```text
webapp version v1
webapp version v2
```

---

## 12. Situation: Traffic Splitting

This situation demonstrates canary-style traffic splitting between `v1` and `v2` for one application.

```bash
cat > webapp-vs-split.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: webapp
  namespace: app-lab
spec:
  host: webapp.lab.local
  ingressClassName: nginx
  tls:
    secret: webapp-tls
  upstreams:
  - name: webapp-v1
    service: webapp-v1
    port: 80
  - name: webapp-v2
    service: webapp-v2
    port: 80
  routes:
  - path: /
    splits:
    - weight: 80
      action:
        pass: webapp-v1
    - weight: 20
      action:
        pass: webapp-v2
YAML

kubectl apply -f webapp-vs-split.yaml
```

Test multiple requests:

```bash
for i in {1..20}; do
  curl -k -s --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
done
```

Expected: most responses should be `v1`, and some should be `v2`.

---

## 13. Situation: Access Control with Policy CRD

Create a policy that allows only the lab client CIDR. Replace `172.16.2.0/24` if your client CIDR is different.

```bash
cat > webapp-access-policy.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: allow-lab-network
  namespace: app-lab
spec:
  accessControl:
    allow:
    - 172.16.2.0/24
YAML

kubectl apply -f webapp-access-policy.yaml
```

Attach the policy to the `VirtualServer`:

```bash
cat > webapp-vs-access.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: webapp
  namespace: app-lab
spec:
  host: webapp.lab.local
  ingressClassName: nginx
  tls:
    secret: webapp-tls
  policies:
  - name: allow-lab-network
  upstreams:
  - name: webapp-v1
    service: webapp-v1
    port: 80
  routes:
  - path: /
    action:
      pass: webapp-v1
YAML

kubectl apply -f webapp-vs-access.yaml
```

Test:

```bash
curl -k -i --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
```

Expected from an allowed client:

```text
HTTP/1.1 200 OK
```

---

## 14. Situation: Rate Limiting with Policy CRD

Create a rate-limiting policy:

```bash
cat > webapp-rate-limit-policy.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: webapp-rate-limit
  namespace: app-lab
spec:
  rateLimit:
    rate: 5r/s
    key: ${binary_remote_addr}
    zoneSize: 10M
    rejectCode: 429
YAML

kubectl apply -f webapp-rate-limit-policy.yaml
```

Attach the policy:

```bash
cat > webapp-vs-rate-limit.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: webapp
  namespace: app-lab
spec:
  host: webapp.lab.local
  ingressClassName: nginx
  tls:
    secret: webapp-tls
  policies:
  - name: webapp-rate-limit
  upstreams:
  - name: webapp-v1
    service: webapp-v1
    port: 80
  routes:
  - path: /
    action:
      pass: webapp-v1
YAML

kubectl apply -f webapp-vs-rate-limit.yaml
```

Test quickly:

```bash
for i in {1..20}; do
  curl -k -s -o /dev/null -w "%{http_code}\n" --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
done
```

Expected: some requests may return `429` when the rate limit is exceeded.

---

## 15. Situation: Basic Authentication

Create a password file:

```bash
htpasswd -bc auth admin P@ssw0rd123
```

Create a secret:

```bash
kubectl create secret generic basic-auth \
  --from-file=auth \
  -n $APP_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -
```

Create a Basic Auth policy:

```bash
cat > webapp-basic-auth-policy.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: webapp-basic-auth
  namespace: app-lab
spec:
  basicAuth:
    secret: basic-auth
    realm: "webapp lab"
YAML

kubectl apply -f webapp-basic-auth-policy.yaml
```

Attach the policy:

```bash
cat > webapp-vs-basic-auth.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: webapp
  namespace: app-lab
spec:
  host: webapp.lab.local
  ingressClassName: nginx
  tls:
    secret: webapp-tls
  policies:
  - name: webapp-basic-auth
  upstreams:
  - name: webapp-v1
    service: webapp-v1
    port: 80
  routes:
  - path: /
    action:
      pass: webapp-v1
YAML

kubectl apply -f webapp-vs-basic-auth.yaml
```

Test without credentials:

```bash
curl -k -i --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
```

Expected:

```text
HTTP/1.1 401 Unauthorized
```

Test with credentials:

```bash
curl -k -i -u admin:P@ssw0rd123 --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
```

Expected:

```text
HTTP/1.1 200 OK
```

---

## 16. Situation: F5 WAF for NGINX

### 16.1 WAF readiness note

This task is only applicable if the trial entitlement includes **F5 WAF for NGINX** and you have access to the required WAF images and policy bundle workflow.

F5 WAF for NGINX v5 with NGINX Ingress Controller is enabled through Helm values and adds the required WAF sidecar containers. WAF protection is configured through `Policy` resources that reference compiled WAF policy bundles.

### 16.2 Prepare Helm values for WAF-enabled controller

Create `nic-plus-waf-values.yaml`:

```bash
cat > nic-plus-waf-values.yaml <<'YAML'
controller:
  nginxplus: true

  replicaCount: 2

  image:
    repository: private-registry.nginx.com/nginx-ic/nginx-plus-ingress
    tag: "5.5.0"

  serviceAccount:
    imagePullSecretName: regcred

  mgmt:
    licenseTokenSecretName: nplus-license

  ingressClass:
    name: nginx
    create: true
    setAsDefaultIngress: false

  enableCustomResources: true
  enablePreviewPolicies: true

  appprotect:
    enable: true
    v5: true

  service:
    type: NodePort
    httpPort:
      nodePort: 30080
    httpsPort:
      nodePort: 30443

  config:
    name: nginx-config
    entries:
      error-log-level: "info"

  nginxStatus:
    enable: true

prometheus:
  create: true
  port: 9113
YAML
```

Upgrade the controller:

```bash
helm upgrade nic nginx-stable/nginx-ingress \
  -n $NIC_NAMESPACE \
  -f nic-plus-waf-values.yaml
```

Verify that extra WAF containers are present:

```bash
kubectl get pods -n $NIC_NAMESPACE -o wide
kubectl describe pod -n $NIC_NAMESPACE -l app.kubernetes.io/instance=nic | egrep -i 'waf|app|enforcer|config'
```

### 16.3 Create a syslog receiver for WAF security logs

```bash
cat > waf-syslog.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: waf-syslog
  namespace: app-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: waf-syslog
  template:
    metadata:
      labels:
        app: waf-syslog
    spec:
      containers:
      - name: syslog
        image: balabit/syslog-ng:latest
        ports:
        - containerPort: 514
          protocol: UDP
---
apiVersion: v1
kind: Service
metadata:
  name: waf-syslog
  namespace: app-lab
spec:
  selector:
    app: waf-syslog
  ports:
  - name: syslog
    port: 514
    targetPort: 514
    protocol: UDP
YAML

kubectl apply -f waf-syslog.yaml
```

Verify:

```bash
kubectl get pods,svc -n $APP_NAMESPACE | grep waf-syslog
```

### 16.4 Create WAF policy resource

The following example assumes the compiled WAF bundle already exists and is mounted where the WAF-enabled NGINX Ingress Controller can read it.

Replace these bundle names with the real compiled bundle files from your WAF preparation job:

```text
lab_waf_policy.tgz
lab_waf_log.tgz
```

Create the policy:

```bash
cat > webapp-waf-policy.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: webapp-waf
  namespace: app-lab
spec:
  waf:
    enable: true
    apBundle: "lab_waf_policy.tgz"
    securityLogs:
    - enable: true
      apLogBundle: "lab_waf_log.tgz"
      logDest: "syslog:server=waf-syslog.app-lab.svc.cluster.local:514"
YAML

kubectl apply -f webapp-waf-policy.yaml
```

Attach the WAF policy:

```bash
cat > webapp-vs-waf.yaml <<'YAML'
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: webapp
  namespace: app-lab
spec:
  host: webapp.lab.local
  ingressClassName: nginx
  tls:
    secret: webapp-tls
  policies:
  - name: webapp-waf
  upstreams:
  - name: webapp-v1
    service: webapp-v1
    port: 80
  routes:
  - path: /
    action:
      pass: webapp-v1
YAML

kubectl apply -f webapp-vs-waf.yaml
```

Verify:

```bash
kubectl get policy webapp-waf -n $APP_NAMESPACE -o yaml
kubectl describe vs webapp -n $APP_NAMESPACE
```

Test normal request:

```bash
curl -k -i --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
```

Test malicious-looking request:

```bash
curl -k -i --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP "https://$LAB_HOST:$HTTPS_PORT/?id=1%20UNION%20SELECT%20password%20FROM%20users"
```

Check WAF-related logs:

```bash
kubectl logs -n $NIC_NAMESPACE deploy/nic-nginx-ingress-controller --all-containers --tail=300 | egrep -i 'waf|app_protect|violation|attack|blocked|security'
kubectl logs -n $APP_NAMESPACE deploy/waf-syslog --tail=300
```

If WAF policy bundle files are not mounted, this task will not become valid. Prepare and mount WAF bundles before running this task.

---

## 17. Logging Task

### 17.1 Check NGINX Ingress Controller logs

```bash
kubectl logs -n $NIC_NAMESPACE deploy/nic-nginx-ingress-controller --tail=100
kubectl logs -n $NIC_NAMESPACE deploy/nic-nginx-ingress-controller -f
```

Generate traffic:

```bash
curl -k --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
```

Check logs again:

```bash
kubectl logs -n $NIC_NAMESPACE deploy/nic-nginx-ingress-controller --tail=100
```

### 17.2 Optional open-source logging stack: Loki, Promtail, Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace logging --dry-run=client -o yaml | kubectl apply -f -

helm install loki grafana/loki-stack \
  -n logging \
  --set grafana.enabled=true \
  --set promtail.enabled=true
```

Verify:

```bash
kubectl get pods -n logging -o wide
kubectl get svc -n logging
```

Port-forward Grafana:

```bash
kubectl port-forward -n logging svc/loki-grafana 3000:80 --address 0.0.0.0
```

Get Grafana admin password:

```bash
kubectl get secret -n logging loki-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Open Grafana:

```text
http://<admin-node-ip>:3000
```

Search logs by namespace:

```text
nginx-ingress
app-lab
```

---

## 18. Validation Checklist

### 18.1 Controller validation

```bash
kubectl get pods -n $NIC_NAMESPACE -o wide
kubectl get deploy -n $NIC_NAMESPACE
kubectl get svc -n $NIC_NAMESPACE -o wide
kubectl get ingressclass
```

Expected:

```text
NGINX Plus Ingress Controller pods: 2/2 Running
Service NodePort: HTTP 30080, HTTPS 30443
IngressClass: nginx
```

### 18.2 Application validation

```bash
kubectl get deploy,svc,pods -n $APP_NAMESPACE -o wide
```

Expected:

```text
webapp-v1 pods Running
webapp-v2 pods Running
Services webapp-v1 and webapp-v2 exist
```

### 18.3 TLS validation

```bash
curl -k -I --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/
```

Expected:

```text
HTTP/1.1 200 OK
```

### 18.4 CRD validation

```bash
kubectl get vs,vsr,policy -n $APP_NAMESPACE -o wide
kubectl describe vs webapp -n $APP_NAMESPACE
kubectl describe vsr webapp-v1-routes -n $APP_NAMESPACE
kubectl describe vsr webapp-v2-routes -n $APP_NAMESPACE
```

Expected:

```text
State: Valid
Reason: AddedOrUpdated
```

### 18.5 NGINX Plus and license validation

```bash
kubectl get secret nplus-license -n $NIC_NAMESPACE
kubectl logs -n $NIC_NAMESPACE deploy/nic-nginx-ingress-controller --all-containers --tail=200 | egrep -i 'license|usage|nginx plus|nginx\+|error|warn'
```

Expected: no repeated license error.

### 18.6 WAF validation, if enabled

```bash
kubectl describe pod -n $NIC_NAMESPACE -l app.kubernetes.io/instance=nic | egrep -i 'waf|enforcer|config'
kubectl get policy webapp-waf -n $APP_NAMESPACE -o yaml
kubectl describe vs webapp -n $APP_NAMESPACE
```

Expected:

```text
WAF containers are running
Policy is accepted
VirtualServer is Valid
```

---

## 19. Common Troubleshooting

### 19.1 Image pull error from private registry

```bash
kubectl get events -n $NIC_NAMESPACE --sort-by=.lastTimestamp | tail -n 30
kubectl describe pod -n $NIC_NAMESPACE -l app.kubernetes.io/instance=nic
```

Check:

```text
JWT is valid
regcred exists in nginx-ingress namespace
controller.serviceAccount.imagePullSecretName=regcred
Image repository and tag are correct
Firewall allows private-registry.nginx.com
```

### 19.2 License error

```bash
kubectl get secret nplus-license -n $NIC_NAMESPACE -o yaml
kubectl logs -n $NIC_NAMESPACE deploy/nic-nginx-ingress-controller --all-containers --tail=200 | grep -i license
```

Check:

```text
Secret type is nginx.com/license
Secret contains license.jwt
controller.mgmt.licenseTokenSecretName=nplus-license
```

### 19.3 VirtualServerRoute warning

If `VirtualServer` and `VirtualServerRoute` show `Warning`, check:

```bash
kubectl describe vs webapp -n $APP_NAMESPACE
kubectl describe vsr webapp-v1-routes -n $APP_NAMESPACE
kubectl describe vsr webapp-v2-routes -n $APP_NAMESPACE
```

Common issue:

```text
A child VirtualServerRoute path must be compatible with the parent VirtualServer path.
```

Good design:

```text
Parent route /v1 -> child VSR with subroute /v1
Parent route /v2 -> child VSR with subroute /v2
```

### 19.4 Application returns 404

Check:

```bash
kubectl get ingress,vs,vsr -n $APP_NAMESPACE -o wide
kubectl describe vs webapp -n $APP_NAMESPACE
kubectl logs -n $NIC_NAMESPACE deploy/nic-nginx-ingress-controller --tail=100
```

Common causes:

```text
Host header mismatch
Wrong ingressClassName
VirtualServer warning
Ingress still exists with same host
Wrong NodePort
```

---

## 20. Cleanup Task

Delete application resources:

```bash
kubectl delete namespace $APP_NAMESPACE
```

Delete NGINX Plus Ingress Controller:

```bash
helm uninstall nic -n $NIC_NAMESPACE
kubectl delete namespace $NIC_NAMESPACE
```

Delete optional logging stack:

```bash
helm uninstall loki -n logging
kubectl delete namespace logging
```

Delete CRDs only if you no longer need NGINX Ingress Controller CRDs:

```bash
kubectl delete -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.5.0/deploy/crds.yaml
```

---

## 21. Quick Command Summary

```bash
# Variables
export KUBECONFIG=/etc/kubernetes/admin.conf
export NIC_NAMESPACE=nginx-ingress
export APP_NAMESPACE=app-lab
export INGRESS_CLASS=nginx
export LAB_HOST=webapp.lab.local

# Namespace
kubectl create namespace $NIC_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# NGINX Plus license and registry secrets
kubectl create secret generic nplus-license \
  --from-file=license.jwt=license.jwt \
  --type=nginx.com/license \
  -n $NIC_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret docker-registry regcred \
  --docker-server=private-registry.nginx.com \
  --docker-username="$(cat license.jwt)" \
  --docker-password=none \
  -n $NIC_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -

# Helm install
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.5.0/deploy/crds.yaml
helm install nic nginx-stable/nginx-ingress -n $NIC_NAMESPACE -f nic-plus-values.yaml

# Verify controller
kubectl get pods -n $NIC_NAMESPACE -o wide
kubectl get svc -n $NIC_NAMESPACE -o wide

# Sample app
kubectl create namespace $APP_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f app-v1.yaml
kubectl apply -f app-v2.yaml

# Access variables
export NIC_IP=$(kubectl get pod -n $NIC_NAMESPACE -l app.kubernetes.io/instance=nic -o jsonpath='{.items[0].status.hostIP}')
export HTTP_PORT=$(kubectl get svc -n $NIC_NAMESPACE nic-nginx-ingress-controller -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
export HTTPS_PORT=$(kubectl get svc -n $NIC_NAMESPACE nic-nginx-ingress-controller -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

# Test Ingress / TLS / CRDs
kubectl apply -f webapp-ingress-tls.yaml
curl -k --resolve $LAB_HOST:$HTTPS_PORT:$NIC_IP https://$LAB_HOST:$HTTPS_PORT/

kubectl apply -f webapp-v1-vsr.yaml
kubectl apply -f webapp-v2-vsr.yaml
kubectl apply -f webapp-vs.yaml
kubectl get vs,vsr -n $APP_NAMESPACE -o wide
```

---

## 22. Official References

- NGINX Ingress Controller installation overview: <https://docs.nginx.com/nginx-ingress-controller/installation/>
- Helm install with NGINX Plus: <https://docs.nginx.com/nginx-ingress-controller/install/helm/plus/>
- Create NGINX Plus license Secret: <https://docs.nginx.com/nginx-ingress-controller/install/license-secret/>
- Download NGINX Ingress Controller from F5 registry: <https://docs.nginx.com/nginx-ingress-controller/install/images/registry-download/>
- VirtualServer and VirtualServerRoute: <https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/>
- Policy resources: <https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/>
- F5 WAF for NGINX v5 installation with NGINX Ingress Controller: <https://docs.nginx.com/nginx-ingress-controller/integrations/app-protect-waf-v5/installation/>
- F5 WAF for NGINX v5 configuration with NGINX Ingress Controller: <https://docs.nginx.com/nginx-ingress-controller/integrations/app-protect-waf-v5/configuration/>
