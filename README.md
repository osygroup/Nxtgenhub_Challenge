# Nxtgenhub DevOps Engineer Coding Challenge

## Runbook for "Hello, World!" Web Server on Kubernetes (AKS)

### Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Deployment](#deployment)
4. [Scaling](#scaling)
5. [Logging](#logging)
6. [Monitoring](#monitoring)
7. [Troubleshooting](#troubleshooting)

### 1. Introduction
This runbook provides a comprehensive guide for deploying, maintaining, and operating the "Hello, World!" web server on an Azure Kubernetes Service cluster. It includes steps for deployment, scaling, monitoring, and troubleshooting.

### 2. Prerequisites
- Kubernetes cluster up and running.
- kubectl and Helm installed and configured.
- Docker Hub access to the web server image.
- Helm charts for Nginx Ingress controller and Cert-manager.
- Domain Name Service provider for the webserver's url.

### 3. Deployment

#### Step 1: Deploy Nginx Ingress Controller
1. **Install Nginx Ingress Controller** 
Use the ingress-nginx helm chart in the helm_charts directory to install the Nginx Ingress controller:
   ```sh
   helm upgrade --install ingress-nginx ingress-nginx \
   --namespace ingress-nginx --create-namespace
   ```
2. ** Get the External IP address of the Ingress controller**
It can take a minute or two for Azure to provide and link a public IP address to the Load Balancer that would be created. When it is complete, you can see the external IP address using the kubectl command:
   ```sh
   kubectl get svc -n ingress-nginx
   ```
3. **Assign a DNS name**
The external IP that is allocated to the ingress-controller is the IP to which all incoming traffic to the web server should be routed. To enable this, add it to a DNS provider you control, for example as in this case [helloworld.logeesti.co](https://helloworld.logeesti.co).

![Screenshot 2024-07-18 at 16.21.41.png](/.attachments/Screenshot%202024-07-18%20at%2016.21.41-714db98c-d6c1-4f8e-a09e-f6b523d8d057.png)

#### Step 2: Deploy "Hello, World!" Web Server
1. **Create a namespace 'hello-world' for the "Hello, World!" web server deployment:**
   ```sh
   kubectl create ns hello-world
   ```

2. **Create a Helm Chart for the Web Server:**
The helm chart for the deployment of the web server can be found in the helm_charts directory, but for understanding of how to create and use helm charts:
   ```sh
   helm create hello-world
   ```

2. **Update the values.yaml in the created hello-world directory to update the necessary kubernetes resources needed:** 
   ```yaml
   image:
     repository: infrastructureascode/hello-world
     pullPolicy: Always
     tag: latest

   service:
     type: ClusterIP
     port: 8080

   ingress:
     enabled: true
     annotations:
       cert-manager.io/issuer: "letsencrypt-prod"
       kubernetes.io/ingress.class: "nginx"
     hosts:
       - host: helloworld.logeesti.co
         paths:
           - path: /
             pathType: ImplementationSpecific
     tls:
       - hosts:
           - helloworld.logeesti.co
         secretName: hello-world-tls
   ```

3. **Install the Helm Chart:**
   ```sh
   helm upgrade --install hello-world ./hello-world -n hello-world
   ```
   At this point, the web server is now accessible over a browser with the url http://helloworld.logeesti.co, but the web server is unsecured due to a lack of an SSL certificate. The next step secures the web server with an SSL certificate (HTTPS). 

#### Step 3: Deploy Cert-Manager
1. **Add Helm Repo:**
   ```sh
   helm repo add jetstack https://charts.jetstack.io --force-update
   ```

2. **Install Cert-Manager:**
   ```sh
   helm install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.15.1 \
    --set crds.enabled=true
   ```

3. **Configure LetsEncrypt Issuer:**
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       #update email address
       email: your-email@example.com
       privateKeySecretRef:
         name: letsencrypt-prod
       solvers:
       - http01:
           ingress:
             class: nginx
   ```

   ```sh
   kubectl apply -f letsencrypt-issuer.yaml -n hello-world
   ```
### 4. Scaling

#### Horizontal Pod Autoscaler
1. **Create Horizontal Pod Autoscaler for the Web Server:**
Update the values.yaml in the hello-world helm chart to create the HPA resource: 
   ```yaml
   resources: 
     limits:
       cpu: 100m
       memory: 128Mi
   requests:
     cpu: 100m
     memory: 128Mi

   autoscaling:
     enabled: true
     minReplicas: 1
     maxReplicas: 100
     targetCPUUtilizationPercentage: 80
     targetMemoryUtilizationPercentage: 80
   ```

2. **Re-install the Helm Chart to create the HPA resource:**
   ```sh
   helm upgrade --install hello-world ./hello-world -n hello-world
   ```

### 5. Logging (NOTE: This was not configured, but is to be done for production clusters)

#### Enable Logging
1. **Deploy EFK Stack (Elasticsearch, Fluentd, Kibana):**
   ```sh
   helm repo add elastic https://helm.elastic.co
   helm repo update
   helm install elasticsearch elastic/elasticsearch
   helm install kibana elastic/kibana
   helm install fluentd stable/fluentd
   ```

2. **Configure Fluentd to collect logs from the web server pods.**

### 6. Monitoring

#### Prometheus and Grafana
1. **Install Prometheus and Grafana:**
   ```sh
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update

   helm install prometheus prometheus-community/kube-prometheus-stack
   helm install grafana grafana/grafana
   ```

2. **Access Grafana Dashboard:**
   - Get Grafana admin password:
     ```sh
     kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode
     ```
   - Access Grafana via the browser using the Kubernetes cluster IP and port.

### 7. Troubleshooting

#### Common Issues
1. **Ingress Not Working:**
   - Check Nginx Ingress controller logs:
     ```sh
     kubectl logs -l app.kubernetes.io/name=ingress-nginx
     ```
   - Verify Ingress resource configuration.

2. **Certificate Issues:**
   - Check Cert-manager logs:
     ```sh
     kubectl logs -l app.kubernetes.io/name=cert-manager
     ```
   - Ensure DNS is correctly pointing to the Kubernetes cluster.

3. **Pod Not Running:**
   - Describe the pod to get detailed information:
     ```sh
     kubectl describe pod <pod-name>
     ```

#### Diagnostic Commands
1. **Check Pod Status:**
   ```sh
   kubectl get pods
   ```
2. **View Logs:**
   ```sh
   kubectl logs <pod-name>
   ```
3. **Check Services:**
   ```sh
   kubectl get svc
   ```

#### Contact Support
- If issues persist, contact Kubernetes cluster or cloud provider support.

---

This runbook provides a solid foundation for administering and maintaining a "Hello, World!" web server on a Kubernetes cluster, ensuring smooth operation and quick resolution of common issues.