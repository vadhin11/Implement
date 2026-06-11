# NGINX Plus Ingress Controller Project Lab / Reference Guideline

**Purpose:** This guideline is a practical lab and project reference for implementing **NGINX One / NGINX Plus Ingress Controller** on an existing Kubernetes cluster. It is aligned with the provided SOW and uses the F5 DevCentral NGINX Ingress Controller lab topics as implementation references.

**Target use case:** Onsite installation and migration of **NGINX One: Ingress Controller** for one Kubernetes cluster, with up to five applications, SSL/TLS termination, advanced routing, optional CRDs, WAF policy, logging, monitoring, tuning, documentation, and on-the-job training.

---

## 1. Scope Mapping to SOW

| SOW Item | Lab / Implementation Reference |
|---|---|
| Conduct kickoff meeting and confirm readiness | Project readiness checklist |
| Install NGINX Plus Ingress Controller to 1 Kubernetes cluster | Install NGINX Plus Ingress Controller |
| Production 2 instances / 2 servers | Controller replica planning |
| License activation and usage reporting | NGINX Plus license and registry secret |
| Application routing up to 5 applications | Application onboarding template |
| Advanced configuration: annotations and ConfigMap | Advanced configuration |
| Enable CRDs: VirtualServer / VirtualServerRoute | CRD installation |
| SSL/TLS termination | TLS termination |
| WAF policy up to 5 applications | WAF policy implementation |
| Logging integration if syslog is available | Logging integration |
| Monitoring and tuning | Monitoring, validation, and tuning |
| Software update / patch / hotfix | Patch and upgrade workflow |
| Documentation and OJT | Handover checklist |

---

## 2. Project Readiness Checklist

Use this checklist during the customer kickoff meeting.

### 2.1 Cluster Inventory

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get svc -A -o wide
kubectl get ingressclass
kubectl get ingress -A
kubectl get crd | egrep 'nginx|virtualserver|policy|appprotect'
```

Record:

| Item | Required Information |
|---|---|
| Cluster name | Production / UAT / DEV/TEST |
| Kubernetes version | `kubectl version` |
| CNI | Calico / Cilium / others |
| Node count | Control plane and worker nodes |
| Worker node IPs | Customer-provided |
| Existing ingress controller | ingress-nginx / Traefik / none |
| Application namespaces | Up to 5 in-scope applications |
| DNS domains | Example: `app.example.com` |
| TLS certificates | Customer-provided or generated for lab |
| External access method | NodePort / LoadBalancer / external LB / F5 BIG-IP / firewall NAT |
| Syslog server | IP, port, protocol |
| Monitoring system | Prometheus / Grafana / NGINX One / NIM |
| WAF requirement | Blocking / transparent / staging mode |
| Change window | Date and time |
| Rollback requirement | Required or not |

### 2.2 Current Lab Cluster Example

The current lab cluster already has a healthy Kubernetes control plane, Calico CNI, CoreDNS, and worker nodes. Validate using:

```bash
kubectl get all -A -o wide
kubectl get nodes -o wide
kubectl get tigerastatus
```

Expected indicators:

```text
calico-node       READY 5/5
kube-proxy        READY 5/5
coredns           2/2 Running
calico            AVAILABLE True
```

---

## 3. Preparation on Admin Machine

Install required tools:

```bash
apt update
apt install -y curl vim jq openssl apache2-utils git
```

Install Helm if not already installed:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Set kubeconfig:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes -o wide
```

Create working directory:

```bash
mkdir -p ~/nginx-ingress-project
cd ~/nginx-ingress-project
```

---

## 4. NGINX Plus License and Registry Secret

NGINX Plus Ingress Controller requires valid subscription credentials to pull the NGINX Plus image from the F5 registry and to run NGINX Plus.

### 4.1 Prepare NGINX JWT License

Copy the customer-provided JWT file to the admin machine:

```bash
mkdir -p ~/nginx-license
cp nginx-repo.jwt ~/nginx-license/nginx-repo.jwt
```

Create the namespace and license secret:

```bash
kubectl create namespace nginx-ingress

kubectl create secret generic license-token \
  --from-file=license.jwt=~/nginx-license/nginx-repo.jwt \
  -n nginx-ingress
```

Verify:

```bash
kubectl get secret -n nginx-ingress
```

### 4.2 Create F5 Registry Pull Secret

Use the JWT as the password for the F5 registry.

```bash
JWT=$(cat ~/nginx-license/nginx-repo.jwt)

kubectl create secret docker-registry regcred \
  --docker-server=private-registry.nginx.com \
  --docker-username=$JWT \
  --docker-password=none \
  -n nginx-ingress
```

Verify:

```bash
kubectl get secret regcred -n nginx-ingress
```

> Adjust the registry secret method according to the customer’s NGINX subscription credential type and official NGINX documentation.

---

## 5. Install NGINX Plus Ingress Controller with Helm

### 5.1 Add Helm Repository

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm search repo nginx-stable/nginx-ingress
```

### 5.2 Create Helm Values File

Create `nic-values.yaml`:

```bash
cat > nic-values.yaml <<'YAML'
controller:
  nginxplus: true
  image:
    repository: private-registry.nginx.com/nginx-ic/nginx-plus-ingress
    tag: "latest"
    pullPolicy: IfNotPresent

  serviceAccount:
    imagePullSecretName: regcred

  replicaCount: 2

  ingressClass:
    name: nginx
    create: true
    setAsDefaultIngress: false

  enableCustomResources: true
  enableOIDC: true

  service:
    type: NodePort
    httpPort:
      nodePort: 30080
    httpsPort:
      nodePort: 30443

  config:
    name: nginx-config

  appprotect:
    enable: false

  logLevel: info
  nginxStatus:
    enable: true

prometheus:
  create: true
  port: 9113
YAML
```

> For production, replace `tag: latest` with the approved supported version.

### 5.3 Install Controller

```bash
helm install nic nginx-stable/nginx-ingress \
  -n nginx-ingress \
  -f nic-values.yaml
```

Verify:

```bash
kubectl get pods -n nginx-ingress -o wide
kubectl get svc -n nginx-ingress -o wide
kubectl get ingressclass
```

Expected:

```text
nic-nginx-ingress-controller-*   1/1 Running
```

### 5.4 Replica Planning for SOW

For the SOW requirement of Production 2 instances:

```yaml
controller:
  replicaCount: 2
```

Confirm:

```bash
kubectl get deploy -n nginx-ingress
kubectl get pods -n nginx-ingress -o wide
```

The two controller pods should preferably run on different worker nodes.

---

## 6. Enable CRDs

The SOW says CRDs may be enabled if required, especially:

- `VirtualServer`
- `VirtualServerRoute`

Apply CRDs if installing by manifest or if Helm did not install them:

```bash
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.5.0/deploy/crds.yaml
```

Verify:

```bash
kubectl get crd | egrep 'virtualserver|virtualserverroute|transportserver|policy|globalconfiguration'
```

Expected resources include:

```text
virtualservers.k8s.nginx.org
virtualserverroutes.k8s.nginx.org
policies.k8s.nginx.org
globalconfigurations.k8s.nginx.org
```

---

## 7. Basic Application Routing Lab

This lab publishes sample `coffee` and `tea` applications with URI-based routing and TLS termination.

### 7.1 Clone Reference Lab

```bash
cd ~
git clone https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab.git
cd ~/NGINX-Ingress-Controller-Lab/labs/1.basic-ingress
```

### 7.2 Get NGINX Ingress Controller NodePort

```bash
export NIC_IP=$(kubectl get pod -l app.kubernetes.io/instance=nic -n nginx-ingress -o json | jq -r '.items[0].status.hostIP')
export HTTP_PORT=$(kubectl get svc nic-nginx-ingress-controller -n nginx-ingress -o jsonpath='{.spec.ports[0].nodePort}')
export HTTPS_PORT=$(kubectl get svc nic-nginx-ingress-controller -n nginx-ingress -o jsonpath='{.spec.ports[1].nodePort}')

echo -e "NIC address: $NIC_IP\nHTTP port  : $HTTP_PORT\nHTTPS port : $HTTPS_PORT"
```

### 7.3 Deploy Sample Applications

```bash
kubectl apply -f 0.cafe.yaml
kubectl get all
```

### 7.4 Create TLS Secret

```bash
kubectl apply -f 1.cafe-secret.yaml
kubectl get secret
```

### 7.5 Publish with Standard Ingress

```bash
kubectl apply -f 2.cafe-ingress.yaml
kubectl get ingress
```

Test:

```bash
curl --insecure --connect-to cafe.example.com:$HTTPS_PORT:$NIC_IP https://cafe.example.com:$HTTPS_PORT/coffee
curl --insecure --connect-to cafe.example.com:$HTTPS_PORT:$NIC_IP https://cafe.example.com:$HTTPS_PORT/tea
```

### 7.6 Publish with VirtualServer

Delete the standard Ingress first:

```bash
kubectl delete -f 2.cafe-ingress.yaml
```

Apply VirtualServer:

```bash
kubectl apply -f 3.cafe-virtualserver.yaml
kubectl get vs -o wide
kubectl describe vs cafe
```

Test:

```bash
curl --insecure --connect-to cafe.example.com:$HTTPS_PORT:$NIC_IP https://cafe.example.com:$HTTPS_PORT/coffee
curl --insecure --connect-to cafe.example.com:$HTTPS_PORT:$NIC_IP https://cafe.example.com:$HTTPS_PORT/tea
```

---

## 8. Advanced Configuration: Annotations and ConfigMap

### 8.1 Ingress Annotations Example

Use annotations for application-specific behavior such as timeout and upload size.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    nginx.org/client-max-body-size: "50m"
    nginx.org/proxy-connect-timeout: "30s"
    nginx.org/proxy-read-timeout: "60s"
    nginx.org/proxy-send-timeout: "60s"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-svc
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f ingress-annotations.yaml
kubectl describe ingress app-ingress
```

### 8.2 ConfigMap Example

Edit global NGINX Ingress Controller behavior:

```bash
kubectl get configmap -n nginx-ingress
kubectl edit configmap nginx-config -n nginx-ingress
```

Example keys:

```yaml
data:
  client-max-body-size: "50m"
  proxy-connect-timeout: "30s"
  proxy-read-timeout: "60s"
  proxy-send-timeout: "60s"
  error-log-level: "info"
```

Restart controller if required:

```bash
kubectl rollout restart deployment nic-nginx-ingress-controller -n nginx-ingress
kubectl rollout status deployment nic-nginx-ingress-controller -n nginx-ingress
```

---

## 9. SSL/TLS Termination

### 9.1 Create TLS Secret

For a lab certificate:

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout app.example.com.key \
  -out app.example.com.crt \
  -subj "/CN=app.example.com/O=Lab"

kubectl create secret tls app-tls \
  --cert=app.example.com.crt \
  --key=app.example.com.key \
  -n default
```

Verify:

```bash
kubectl get secret app-tls -n default
```

### 9.2 Test TLS

```bash
curl -k --resolve app.example.com:$HTTPS_PORT:$NIC_IP https://app.example.com:$HTTPS_PORT/
```

For production, use customer-provided certificate and private key.

---

## 10. Advanced Routing Lab

Use the advanced routing lab for path-based routing, route-level configuration, and upstream behavior.

```bash
cd ~/NGINX-Ingress-Controller-Lab/labs/2.advanced-routing
kubectl apply -f .
kubectl get vs -o wide
kubectl describe vs
```

Test with the host and routes defined in the lab:

```bash
curl -i -H "Host: <host-from-lab-yaml>" http://$NIC_IP:$HTTP_PORT/<path>
```

Clean up:

```bash
kubectl delete -f .
```

---

## 11. Authentication and Access Control Labs

### 11.1 Authentication Lab

```bash
cd ~/NGINX-Ingress-Controller-Lab/labs/3.authentication
kubectl apply -f .
kubectl get policy
kubectl get vs -o wide
curl -i -H "Host: <host-from-lab-yaml>" http://$NIC_IP:$HTTP_PORT/
kubectl delete -f .
```

### 11.2 Access Control Lab

```bash
cd ~/NGINX-Ingress-Controller-Lab/labs/5.access-control
kubectl apply -f .
kubectl get policy
kubectl get vs -o wide
curl -i -H "Host: <host-from-lab-yaml>" http://$NIC_IP:$HTTP_PORT/
kubectl delete -f .
```

---

## 12. Traffic Splitting Lab

Use the traffic-splitting lab to demonstrate blue/green, canary, or weighted routing.

```bash
cd ~/NGINX-Ingress-Controller-Lab/labs/4.traffic-splitting
kubectl apply -f .
kubectl get vs -o wide
kubectl describe vs
```

Run multiple requests to observe traffic distribution:

```bash
for i in {1..20}; do
  curl -s -H "Host: <host-from-lab-yaml>" http://$NIC_IP:$HTTP_PORT/
done
```

Clean up:

```bash
kubectl delete -f .
```

---

## 13. Rate Limiting Lab

Use the rate-limiting lab to validate policy enforcement.

```bash
cd ~/NGINX-Ingress-Controller-Lab/labs/6.rate-limiting
kubectl apply -f 0.webapp.yaml
kubectl apply -f 1.rate-limit.yaml
kubectl apply -f 2.virtual-server.yaml
kubectl get vs -o wide
kubectl describe vs webapp
```

Test once:

```bash
curl -i -H "Host: webapp.example.com" http://$NIC_IP:$HTTP_PORT
```

Test twice quickly:

```bash
curl -i -H "Host: webapp.example.com" http://$NIC_IP:$HTTP_PORT; \
curl -i -H "Host: webapp.example.com" http://$NIC_IP:$HTTP_PORT
```

Expected second response:

```text
HTTP/1.1 429 Too Many Requests
```

Clean up:

```bash
kubectl delete -f .
```

---

## 14. WAF Policy Implementation

The SOW includes WAF policy implementation for up to five applications.

### 14.1 Enable WAF in Helm Values

Create `nic-waf-values.yaml`:

```yaml
controller:
  nginxplus: true
  appprotect:
    enable: true
```

Upgrade:

```bash
helm upgrade nic nginx-stable/nginx-ingress \
  -n nginx-ingress \
  -f nic-values.yaml \
  -f nic-waf-values.yaml
```

Verify:

```bash
kubectl get pods -n nginx-ingress -o wide
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller | egrep -i 'app protect|waf|error'
```

### 14.2 Apply WAF Lab

```bash
cd ~/NGINX-Ingress-Controller-Lab/labs/7.waf
kubectl apply -f .
kubectl get policy
kubectl get vs -o wide
kubectl describe vs
```

Test normal and attack requests according to the lab examples.

Clean up:

```bash
kubectl delete -f .
```

### 14.3 Precompiled WAF Policy Lab

```bash
cd ~/NGINX-Ingress-Controller-Lab/labs/8.waf-precompiled
kubectl apply -f .
kubectl get policy
kubectl get vs -o wide
kubectl delete -f .
```

### 14.4 Project WAF Checklist per Application

| Item | Value |
|---|---|
| Application name | |
| Namespace | |
| Hostname | |
| Service name / port | |
| TLS secret | |
| WAF mode | Blocking / Transparent |
| WAF policy file | |
| False positive tuning required | Yes / No |
| Sign-off owner | |

---

## 15. Logging Integration

### 15.1 Controller Logs

```bash
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller --tail=100 -f
```

### 15.2 Access Log Format via ConfigMap

Edit ConfigMap:

```bash
kubectl edit configmap nginx-config -n nginx-ingress
```

Example:

```yaml
data:
  log-format: |
    $remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$request_time" "$upstream_response_time"
```

### 15.3 Syslog Integration

If the customer provides a syslog server, configure NGINX access/error logs to forward to the syslog destination, subject to the supported NGINX Ingress Controller version and customer change window.

Required customer input:

| Item | Required |
|---|---|
| Syslog IP/FQDN | Yes |
| Syslog port | Yes |
| Protocol | TCP / UDP |
| Log format | Standard / JSON / SIEM format |
| Firewall path | Customer responsibility per SOW |

---

## 16. Monitoring and Tuning

### 16.1 Controller Health

```bash
kubectl get pods -n nginx-ingress -o wide
kubectl describe pod -n nginx-ingress -l app.kubernetes.io/instance=nic
kubectl get events -n nginx-ingress --sort-by=.lastTimestamp
```

### 16.2 Kubernetes Health

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get --raw='/readyz?verbose'
```

### 16.3 NGINX Metrics

If Prometheus metrics are enabled:

```bash
kubectl get svc -n nginx-ingress
kubectl port-forward -n nginx-ingress svc/nic-nginx-ingress-controller 9113:9113
```

From another terminal:

```bash
curl http://127.0.0.1:9113/metrics | head
```

### 16.4 Basic Tuning Items

| Area | Example |
|---|---|
| Timeout | `proxy-read-timeout`, `proxy-send-timeout` |
| Upload size | `client-max-body-size` |
| Header size | Large client header buffers |
| Rate limit | Policy resource |
| TLS | TLS version / ciphers |
| WAF | Enforcement mode, signatures, exclusions |
| Logging | Format and log destination |

---

## 17. Software Update / Patch / Hotfix Workflow

### 17.1 Pre-Update Backup

```bash
mkdir -p ~/backup-nginx-ingress-$(date +%F)

kubectl get all -n nginx-ingress -o yaml > ~/backup-nginx-ingress-$(date +%F)/nginx-ingress-all.yaml
kubectl get configmap -n nginx-ingress -o yaml > ~/backup-nginx-ingress-$(date +%F)/configmaps.yaml
kubectl get secret -n nginx-ingress -o yaml > ~/backup-nginx-ingress-$(date +%F)/secrets.yaml
kubectl get ingress -A -o yaml > ~/backup-nginx-ingress-$(date +%F)/ingress-all.yaml
kubectl get vs,vsr,policy -A -o yaml > ~/backup-nginx-ingress-$(date +%F)/nginx-crds.yaml
helm get values nic -n nginx-ingress > ~/backup-nginx-ingress-$(date +%F)/helm-values.yaml
helm list -n nginx-ingress > ~/backup-nginx-ingress-$(date +%F)/helm-list.txt
```

### 17.2 Helm Upgrade

```bash
helm repo update
helm upgrade nic nginx-stable/nginx-ingress \
  -n nginx-ingress \
  -f nic-values.yaml
```

Monitor:

```bash
kubectl rollout status deployment nic-nginx-ingress-controller -n nginx-ingress
kubectl get pods -n nginx-ingress -o wide
```

### 17.3 Rollback

```bash
helm history nic -n nginx-ingress
helm rollback nic <REVISION> -n nginx-ingress
kubectl rollout status deployment nic-nginx-ingress-controller -n nginx-ingress
```

---

## 18. Application Onboarding Template

Use this for each in-scope application.

### 18.1 Required Input

| Field | Example |
|---|---|
| Application name | app1 |
| Namespace | app1 |
| Hostname | app1.example.com |
| Backend service | app1-svc |
| Backend port | 80 |
| TLS secret | app1-tls |
| WAF policy | app1-waf-policy |
| Rate limit | Required / Not required |
| Auth | Required / Not required |
| Source IP restriction | Required / Not required |
| Logging requirement | Syslog / stdout / SIEM |
| Monitoring requirement | Prometheus / NGINX One |

### 18.2 Basic VirtualServer Template

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: app1
  namespace: app1
spec:
  host: app1.example.com
  tls:
    secret: app1-tls
  upstreams:
  - name: app1
    service: app1-svc
    port: 80
  routes:
  - path: /
    action:
      pass: app1
```

Apply:

```bash
kubectl apply -f app1-virtualserver.yaml
kubectl get vs -n app1 -o wide
kubectl describe vs app1 -n app1
```

Test:

```bash
curl -k --resolve app1.example.com:$HTTPS_PORT:$NIC_IP https://app1.example.com:$HTTPS_PORT/
```

---

## 19. Production Migration Workflow

### 19.1 Pre-Migration

1. Export current ingress resources from the existing controller.
2. Map existing annotations to NGINX Ingress Controller annotations or VirtualServer resources.
3. Confirm DNS TTL and rollback method.
4. Confirm SSL/TLS certificate ownership.
5. Confirm firewall/NAT/LB path is prepared by the customer.
6. Confirm application owners are ready for testing.

Commands:

```bash
kubectl get ingress -A -o yaml > current-ingress-backup.yaml
kubectl get svc -A -o wide > current-service-inventory.txt
kubectl get endpoints -A -o wide > current-endpoints.txt
```

### 19.2 Migration

For each application:

1. Create namespace if required.
2. Create TLS secret.
3. Create Ingress or VirtualServer.
4. Apply WAF policy if required.
5. Test using `curl --resolve`.
6. Switch DNS or external LB mapping.
7. Monitor logs and application response.

### 19.3 Rollback

1. Revert DNS or external LB mapping to the old ingress controller.
2. Delete or disable the new VirtualServer/Ingress.
3. Confirm application response through the old path.
4. Collect logs and document reason.

---

## 20. Validation Checklist

### 20.1 Platform

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

### 20.2 NGINX Ingress Controller

```bash
kubectl get pods -n nginx-ingress -o wide
kubectl get svc -n nginx-ingress -o wide
kubectl get ingressclass
helm list -n nginx-ingress
```

### 20.3 Application

```bash
kubectl get ingress -A
kubectl get vs -A -o wide
kubectl get policy -A
curl -k --resolve <host>:<port>:<nic-ip> https://<host>:<port>/
```

### 20.4 WAF

```bash
kubectl get policy -A
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller | egrep -i 'waf|app protect|security|blocked'
```

### 20.5 Logging

```bash
kubectl logs -n nginx-ingress deploy/nic-nginx-ingress-controller --tail=100
```

---

## 21. Handover and OJT Topics

Cover these topics during on-the-job training:

1. NGINX Ingress Controller architecture.
2. Difference between standard Ingress and VirtualServer.
3. TLS secret creation and certificate rotation.
4. ConfigMap vs annotations.
5. WAF policy deployment and tuning boundary.
6. Rate limiting and access control policy.
7. Log collection and troubleshooting.
8. Helm upgrade and rollback.
9. Operational checklist.
10. Scope exclusions and change control.

---

## 22. SOW Boundary Notes

The following items are outside the implementation scope unless explicitly added to the SOW:

- Network and firewall configuration.
- Switch configuration.
- LAN cabling, fiber, SFP modules, wiring, and labeling.
- Application code changes or application tuning.
- Database modification.
- Re-deployment of applications.
- Operating system license, OS patching, or OS upgrade.
- Pentest/VA remediation, retest, rescan, or compliance reporting.
- Third-party application installation and configuration.
- UAT document creation.
- Hard copy documentation.

---

## 23. Quick Command Summary

```bash
# Validate cluster
kubectl get nodes -o wide
kubectl get pods -A -o wide

# Add Helm repo
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update

# Create namespace and license secrets
kubectl create namespace nginx-ingress
kubectl create secret generic license-token --from-file=license.jwt=./nginx-repo.jwt -n nginx-ingress
kubectl create secret docker-registry regcred --docker-server=private-registry.nginx.com --docker-username="$(cat ./nginx-repo.jwt)" --docker-password=none -n nginx-ingress

# Install NGINX Plus Ingress Controller
helm install nic nginx-stable/nginx-ingress -n nginx-ingress -f nic-values.yaml

# Verify
kubectl get pods -n nginx-ingress -o wide
kubectl get svc -n nginx-ingress -o wide
kubectl get ingressclass

# Clone lab
git clone https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab.git

# Run basic lab
cd ~/NGINX-Ingress-Controller-Lab/labs/1.basic-ingress
kubectl apply -f 0.cafe.yaml
kubectl apply -f 1.cafe-secret.yaml
kubectl apply -f 3.cafe-virtualserver.yaml
kubectl get vs -o wide

# Clean lab
kubectl delete -f .
```

---

## 24. Reference Links

### F5 DevCentral Lab

- Basic Ingress: https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/1.basic-ingress/README.md
- Advanced Routing: https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/2.advanced-routing/README.md
- Authentication: https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/3.authentication/README.md
- Traffic Splitting: https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/4.traffic-splitting/README.md
- Access Control: https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/5.access-control/README.md
- Rate Limiting: https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/6.rate-limiting/README.md
- WAF: https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/7.waf/README.md
- WAF Precompiled: https://github.com/f5devcentral/NGINX-Ingress-Controller-Lab/blob/main/labs/8.waf-precompiled/README.md

### Official NGINX Documentation

- NGINX Ingress Controller overview: https://docs.nginx.com/nginx-ingress-controller/
- Helm installation: https://docs.nginx.com/nginx-ingress-controller/install/helm/
- License secret: https://docs.nginx.com/nginx-ingress-controller/install/license-secret/
- CRD installation: https://docs.nginx.com/nginx-ingress-controller/install/manifests/
- VirtualServer and VirtualServerRoute: https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/
- Annotations: https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/advanced-configuration-with-annotations/
- ConfigMap: https://docs.nginx.com/nginx-ingress-controller/configuration/global-configuration/configmap-resource/
- Policy resources: https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/
- Logging: https://docs.nginx.com/nginx-ingress-controller/logging-and-monitoring/logging/
- F5 WAF for NGINX: https://docs.nginx.com/nginx-ingress-controller/integrations/app-protect-waf-v5/configuration/
