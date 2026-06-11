# NGINX Open Source Ingress Controller LAB Guideline

**Document purpose:** This lab guideline is a practical reference for installing and validating an **NGINX Open Source Ingress Controller** on one Kubernetes cluster. It is adjusted for lab.

**Lab scope requested:**

- Install and configure **NGINX Open Source Ingress Controller** on **1 Kubernetes cluster**.
- Run the controller with **2 replicas**.
- Configure routing for **1 sample application**.
- Demonstrate advanced configuration by using:
  - Annotations
  - ConfigMap
- Enable and demonstrate CRDs:
  - VirtualServer
  - VirtualServerRoute
- Configure SSL/TLS termination at the Ingress layer.
- Implement a **sample WAF scenario for 1 sample application**.
- Integrate logging using open-source tooling.

> Important WAF note: F5 WAF for NGINX / NGINX App Protect WAF requires NGINX Plus. This guideline uses NGINX Open Source for the main Ingress Controller lab. For the WAF portion, it provides a lab-only open-source WAF option using ModSecurity + OWASP CRS with the community ingress-nginx controller. This is not the same as F5 WAF for NGINX.

---

## 1. Lab Architecture

```text
Client / curl
    |
    | Host: webapp.lab.local
    |
NodeIP:NodePort
    |
NGINX Open Source Ingress Controller, 2 replicas
    |
Ingress / VirtualServer / VirtualServerRoute
    |
Sample application service
    |
Sample application pods
```

### Current Kubernetes lab assumptions

This guide assumes the Kubernetes cluster is already ready and has worker nodes joined. Your current cluster output shows Calico, CoreDNS, kube-proxy, and worker-node Calico pods running successfully.

Expected baseline:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get svc -A -o wide
```

---

## 2. Pre-check

Run from the admin node, for example `ubuntu11`:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get ingressclass
kubectl get ingress -A
kubectl get crd | egrep 'virtualserver|virtualserverroute|policies.k8s.nginx.org' || true
```

Install useful tools:

```bash
apt update
apt install -y curl vim jq openssl apache2-utils git
```

Install Helm if it is not installed:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Create a working directory:

```bash
mkdir -p ~/nginx-oss-ingress-lab
cd ~/nginx-oss-ingress-lab
```

---

## 3. Install NGINX Open Source Ingress Controller

This lab uses the **F5 NGINX Ingress Controller Open Source** Helm chart.

### 3.1 Add Helm repository

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm search repo nginx-stable/nginx-ingress
```

### 3.2 Create namespace

```bash
kubectl create namespace nginx-ingress
```

If the namespace already exists, continue.

### 3.3 Install core CRDs

Install CRDs before using `VirtualServer` and `VirtualServerRoute`:

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

### 3.4 Create Helm values for 2 replicas

Create `nic-oss-values.yaml`:

```bash
cat > nic-oss-values.yaml <<'YAML'
controller:
  nginxplus: false

  replicaCount: 2

  ingressClass:
    name: nginx
    create: true
    setAsDefaultIngress: false

  enableCustomResources: true

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

### 3.5 Install controller

```bash
helm install nic nginx-stable/nginx-ingress \
  -n nginx-ingress \
  -f nic-oss-values.yaml
```

Verify:

```bash
kubectl get pods -n nginx-ingress -o wide
kubectl get deploy -n nginx-ingress
kubectl get svc -n nginx-ingress -o wide
kubectl get ingressclass
```

Expected:

```text
2 controller pods Running
Deployment READY 2/2
Service exposes NodePort 30080 and 30443
IngressClass nginx exists
```

If only one controller pod is running, check worker-node capacity and scheduling:

```bash
kubectl describe deploy -n nginx-ingress nic-nginx-ingress-controller
kubectl get events -n nginx-ingress --sort-by=.lastTimestamp
```

---

## 4. Prepare 1 Sample Application

This lab uses one simple web application with two versions. It helps demonstrate routing, traffic splitting, and VirtualServerRoute.

### 4.1 Create namespace

```bash
kubectl create namespace app-lab
```

### 4.2 Deploy application version 1

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

### 4.3 Deploy application version 2

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
kubectl get deploy,svc,pods -n app-lab -o wide
```

---

## 5. Get NGINX Ingress Controller Access Information

```bash
export NIC_IP=$(kubectl get pod -n nginx-ingress -l app.kubernetes.io/instance=nic -o jsonpath='{.items[0].status.hostIP}')
export HTTP_PORT=$(kubectl get svc -n nginx-ingress nic-nginx-ingress-controller -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
export HTTPS_PORT=$(kubectl get svc -n nginx-ingress nic-nginx-ingress-controller -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

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

## 6. Application Routing Configuration, Standard Ingress

This section demonstrates routing for one sample application using standard Kubernetes `Ingress`.

### 6.1 Create Ingress with annotations

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
kubectl get ingress -n app-lab
kubectl describe ingress webapp-ingress -n app-lab
```

Test:

```bash
curl -i -H "Host: webapp.lab.local" http://$NIC_IP:$HTTP_PORT/
```

Expected response body:

```text
webapp version v1
```

---

## 7. Advanced Configuration: ConfigMap

The controller ConfigMap was already created by Helm as `nginx-config`.

Check it:

```bash
kubectl get configmap nginx-config -n nginx-ingress -o yaml
```

Update global controller behavior with `kubectl patch`:

```bash
kubectl patch configmap nginx-config -n nginx-ingress --type merge -p '{
  "data": {
    "proxy-connect-timeout": "30s",
    "proxy-read-timeout": "60s",
    "proxy-send-timeout": "60s",
    "client-max-body-size": "20m",
    "error-log-level": "info"
  }
}'
```

Restart the controller if required:

```bash
kubectl rollout restart deployment nic-nginx-ingress-controller -n nginx-ingress
kubectl rollout status deployment nic-nginx-ingress-controller -n nginx-ingress
```

Validate again:

```bash
curl -i -H "Host: webapp.lab.local" http://$NIC_IP:$HTTP_PORT/
```

---

## 8. SSL/TLS Termination at Ingress Layer

### 8.1 Create a lab self-signed certificate

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
  -n app-lab
```

### 8.2 Update Ingress for TLS

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
curl -k -i --resolve webapp.lab.local:$HTTPS_PORT:$NIC_IP https://webapp.lab.local:$HTTPS_PORT/
```

Expected response body:

```text
webapp version v1
```

---

## 9. Enable CRDs: VirtualServer and VirtualServerRoute

This section creates a situation where the application owner wants to split routing logic into a parent `VirtualServer` and a child `VirtualServerRoute`.

### 9.1 Remove standard Ingress

```bash
kubectl delete ingress webapp-ingress -n app-lab
```

### 9.2 Create VirtualServerRoute

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

and

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

### 9.3 Create VirtualServer

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
kubectl get vs,vsr -n app-lab -o wide
kubectl describe vs webapp -n app-lab
kubectl describe vsr webapp-v1-routes -n app-lab
kubectl describe vsr webapp-v2-routes -n app-lab
```

Expected event:

```text
Normal AddedOrUpdated Configuration for app-lab/webapp was added or updated
```

Test:

```bash
curl -k --resolve webapp.lab.local:$HTTPS_PORT:$NIC_IP https://webapp.lab.local:$HTTPS_PORT/v1
curl -k --resolve webapp.lab.local:$HTTPS_PORT:$NIC_IP https://webapp.lab.local:$HTTPS_PORT/v2
```

Expected:

```text
webapp version v1
webapp version v2
```

---

## 10. Advanced Routing Situation: Traffic Splitting

This demonstrates a canary-style split between `v1` and `v2` for one application.

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
  curl -k -s --resolve webapp.lab.local:$HTTPS_PORT:$NIC_IP https://webapp.lab.local:$HTTPS_PORT/
done
```

Expected: most responses should be `v1`, and some should be `v2`.

---

## 11. Access Control Situation Using Policy CRD

Create a Policy to allow only a source CIDR. For lab, replace `172.16.2.0/24` with the real client CIDR.

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

Attach it to VirtualServer:

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
curl -k -i --resolve webapp.lab.local:$HTTPS_PORT:$NIC_IP https://webapp.lab.local:$HTTPS_PORT/
```

---

## 12. WAF Policy Implementation, Lab-Only Open Source Option

### 12.1 Limitation statement

F5 WAF for NGINX requires NGINX Plus. Because this lab condition requests **NGINX Open Source**, the WAF section uses a separate **community ingress-nginx + ModSecurity + OWASP CRS** lab only.

Use this section only to demonstrate an open-source WAF concept. Do not present it as F5 WAF for NGINX or NGINX App Protect WAF.

### 12.2 Install community ingress-nginx WAF lab controller

Create a separate namespace and separate ingress class to avoid conflict with the main NGINX OSS Ingress Controller.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

kubectl create namespace nginx-waf-lab
```

Create `ingress-nginx-waf-values.yaml`:

```bash
cat > ingress-nginx-waf-values.yaml <<'YAML'
controller:
  replicaCount: 1
  ingressClassResource:
    name: nginx-waf
    enabled: true
    default: false
  ingressClass: nginx-waf
  service:
    type: NodePort
    nodePorts:
      http: 31080
      https: 31443
  config:
    enable-modsecurity: "true"
    enable-owasp-modsecurity-crs: "true"
    modsecurity-snippet: |
      SecRuleEngine On
      SecRequestBodyAccess On
      SecAuditEngine RelevantOnly
      SecAuditLog /dev/stdout
YAML
```

Install:

```bash
helm install ingress-nginx-waf ingress-nginx/ingress-nginx \
  -n nginx-waf-lab \
  -f ingress-nginx-waf-values.yaml
```

Verify:

```bash
kubectl get pods -n nginx-waf-lab -o wide
kubectl get svc -n nginx-waf-lab -o wide
kubectl get ingressclass
```

### 12.3 Create WAF test Ingress for the same sample application

```bash
cat > webapp-waf-ingress.yaml <<'YAML'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-waf
  namespace: app-lab
  annotations:
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
spec:
  ingressClassName: nginx-waf
  rules:
  - host: webapp-waf.lab.local
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

kubectl apply -f webapp-waf-ingress.yaml
```

Test normal request:

```bash
export WAF_NODE_IP=$(kubectl get pod -n nginx-waf-lab -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].status.hostIP}')

curl -i -H "Host: webapp-waf.lab.local" http://$WAF_NODE_IP:31080/
```

Test simple malicious-looking request:

```bash
curl -i -H "Host: webapp-waf.lab.local" "http://$WAF_NODE_IP:31080/?id=1%20UNION%20SELECT%20password%20FROM%20users"
```

Check WAF logs:

```bash
kubectl logs -n nginx-waf-lab deploy/ingress-nginx-waf-controller --tail=200 | egrep -i 'modsecurity|crs|attack|warning|union|select|949|blocked'
```

Expected: the WAF should log CRS/ModSecurity detection. Depending on CRS behavior and anomaly score, the response may be blocked or logged. For a stronger blocking demonstration, tune CRS threshold or add a specific test rule.

### 12.4 Optional custom block rule for lab clarity

Add a custom rule that blocks a clear test string `blockme`:

```bash
kubectl patch configmap ingress-nginx-waf-controller -n nginx-waf-lab --type merge -p '{
  "data": {
    "modsecurity-snippet": "SecRuleEngine On\nSecRequestBodyAccess On\nSecRule ARGS test \"@streq blockme\" \"id:1000001,phase:2,deny,status:403,msg:'"'"'LAB custom WAF block rule'"'"'\""
  }
}'

kubectl rollout restart deployment ingress-nginx-waf-controller -n nginx-waf-lab
kubectl rollout status deployment ingress-nginx-waf-controller -n nginx-waf-lab
```

Test:

```bash
curl -i -H "Host: webapp-waf.lab.local" "http://$WAF_NODE_IP:31080/?test=blockme"
```

Expected:

```text
HTTP/1.1 403 Forbidden
```

---

## 13. Logging Integration with Open Source Tools

This section uses stdout logs plus optional Loki/Promtail/Grafana as open-source logging.

### 13.1 Check NGINX Ingress Controller logs

```bash
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller --tail=100
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller -f
```

Generate traffic:

```bash
curl -k --resolve webapp.lab.local:$HTTPS_PORT:$NIC_IP https://webapp.lab.local:$HTTPS_PORT/
```

Check logs again:

```bash
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller --tail=100
```

### 13.2 Install Loki stack, optional lab logging

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace logging

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
kubectl port-forward -n logging svc/loki-grafana 3000:80
```

Get Grafana admin password:

```bash
kubectl get secret -n logging loki-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Open Grafana from your workstation:

```text
http://<admin-node-ip>:3000
```

Search logs using labels such as namespace `nginx-ingress` and pod name containing `nginx-ingress-controller`.

---

## 14. Validation Checklist

### 14.1 Controller validation

```bash
kubectl get pods -n nginx-ingress -o wide
kubectl get deploy -n nginx-ingress
kubectl get svc -n nginx-ingress -o wide
kubectl get ingressclass
```

Expected:

```text
NGINX Ingress Controller pods: 2/2 Running
Service NodePort: HTTP 30080, HTTPS 30443
IngressClass: nginx
```

### 14.2 Application validation

```bash
kubectl get deploy,svc,pods -n app-lab -o wide
kubectl get ingress -n app-lab
kubectl get vs,vsr -n app-lab -o wide
```

### 14.3 TLS validation

```bash
curl -k -I --resolve webapp.lab.local:$HTTPS_PORT:$NIC_IP https://webapp.lab.local:$HTTPS_PORT/
```

Expected:

```text
HTTP/1.1 200 OK
```

### 14.4 CRD validation

```bash
kubectl describe vs webapp -n app-lab
kubectl describe vsr webapp-routes -n app-lab
```

Expected event:

```text
AddedOrUpdated
```

### 14.5 Logging validation

```bash
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller --tail=100
kubectl get pods -n logging -o wide
```

### 14.6 WAF lab validation

```bash
kubectl get pods -n nginx-waf-lab -o wide
kubectl get ingress -n app-lab webapp-waf
curl -i -H "Host: webapp-waf.lab.local" "http://$WAF_NODE_IP:31080/?test=blockme"
```

Expected for custom rule:

```text
HTTP/1.1 403 Forbidden
```

---

## 15. Cleanup

Delete the sample application and NGINX OSS Ingress Controller:

```bash
kubectl delete namespace app-lab
helm uninstall nic -n nginx-ingress
kubectl delete namespace nginx-ingress
```

Delete optional WAF lab:

```bash
helm uninstall ingress-nginx-waf -n nginx-waf-lab
kubectl delete namespace nginx-waf-lab
```

Delete optional logging stack:

```bash
helm uninstall loki -n logging
kubectl delete namespace logging
```

Keep CRDs if you plan to reuse the controller. If not, delete carefully:

```bash
kubectl delete -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.5.0/deploy/crds.yaml
```

---

## 16. Quick Command Summary

```bash
# Pre-check
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes -o wide
kubectl get pods -A -o wide

# Helm
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update

# Namespace and CRDs
kubectl create namespace nginx-ingress
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.5.0/deploy/crds.yaml

# Install NGINX OSS Ingress Controller
helm install nic nginx-stable/nginx-ingress -n nginx-ingress -f nic-oss-values.yaml
kubectl get pods -n nginx-ingress -o wide
kubectl get svc -n nginx-ingress -o wide

# Deploy sample app
kubectl create namespace app-lab
kubectl apply -f app-v1.yaml
kubectl apply -f app-v2.yaml

# Test standard Ingress
kubectl apply -f webapp-ingress-tls.yaml
curl -k --resolve webapp.lab.local:$HTTPS_PORT:$NIC_IP https://webapp.lab.local:$HTTPS_PORT/

# Test CRDs
kubectl apply -f webapp-vsr.yaml
kubectl apply -f webapp-vs.yaml
kubectl get vs,vsr -n app-lab -o wide

# Logs
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller --tail=100
```

---

## 17. Reference Links

### F5 DevCentral lab references

- Basic Ingress: <https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/1.basic-ingress/README.md>
- Advanced Routing: <https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/2.advanced-routing/README.md>
- Authentication: <https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/3.authentication/README.md>
- Traffic Splitting: <https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/4.traffic-splitting/README.md>
- Access Control: <https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/5.access-control/README.md>
- Rate Limiting: <https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/6.rate-limiting/README.md>
- WAF: <https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/7.waf/README.md>
- WAF Precompiled: <https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/8.waf-precompiled/README.md>

### Official / community references

- NGINX Ingress Controller CRD installation: <https://docs.nginx.com/nginx-ingress-controller/install/manifests/>
- VirtualServer and VirtualServerRoute: <https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/>
- NGINX Ingress Controller Policy resources: <https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/>
- F5 WAF for NGINX requirement: <https://docs.nginx.com/nginx-ingress-controller/integrations/app-protect-waf/installation/>
- ingress-nginx ModSecurity / OWASP CRS: <https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/modsecurity/>
- ingress-nginx ConfigMap: <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/>
