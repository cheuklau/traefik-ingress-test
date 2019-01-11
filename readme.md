# Ingress with Traefik Testing
This repository will go through the steps for using Kubernetes [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) with [Traefik](https://traefik.io/) as the ingress controller. We will deploy a sample [Elastic Stack](https://www.elastic.co/products) application to demonstrate ingress routing with Traefik. [Let's Encrypt](https://letsencrypt.org/) will be used to encrypt communication (via HTTPS) with external traffic. [NGINX with LDAP](https://github.com/tiagoapimenta/nginx-ldap-auth) will be used to authenticate and authorize users to access specific services.

## Introduction

### What is Ingress?
Ingress exposes HTTP/HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined by ingress objects. An ingress controller (e.g., Traefik) is responsible for fulfilling the ingress.

### What is Let's Encrypt?
Let's Encrypt is a certificate authority (CA) which signs a certificate establishing the identity of a server. This certificate is used as part of the [SSL/TLS handshake](https://www.ssl.com/app/uploads/2015/07/SSLTLS_handshake.png?x32239) to establish an encrypted channel between a client and server:
1. Client sends request to server to set up an encrypted session.
2. Server sends back CA-signed certificate and public key.
3. Client verifies certificate is signed by a trusted CA then extracts public key. Client uses public key to encrypt a new key and sends it to the server.
4. Server uses private key to decrypt new key.
5. Client and server use the new key to compute a shared secret key which is used to encrypt the remainder of the session.
Note: TLS (transport layer security) is just an updated version of SSL (secured sockets layer) and HTTPS (hyper text transfer protocol secure) appears in the URL when a website is secured by a TLS/SSL certificate.

### How is Authentication and Authorization Handled?
Authentication means confirming a user's identity whereas authorization means granting a user access to a particular service. [Authentication](https://docs.traefik.io/user-guide/kubernetes/#basic-authentication) is handled by specifying the URL of the [authentication server](https://docs.traefik.io/configuration/entrypoints/#forward-authentication) in the Ingress object. We will run [NGINX with LDAP](https://github.com/tiagoapimenta/nginx-ldap-auth) as a separate Kubernetes deployment to be our authentication server. We will run separate authentication servers for each unique combination of LDAP groups needed across all services. For example, only LDAP users in the `k8s-admin` group should access the Traefik UI whereas only LDAP users in the `devops` group should access the Kibana UI. In this scenario, we will have two deployments to handle the two different criteria. Note: Be on the lookout for a way to dynamically filter groups based on ingress. This would allow us to run just a single NGINX deployment!

## Installation (Native Let's Encrypt without LDAP)
Note: Files are in `native_lets_encrypt/`.
1. Deploy the Elastic Stack application using [Helm](https://docs.helm.sh/using_helm/):
```
helm install --tiller-namespace=tiller-ns --namespace=default elastic-demo/
```
2. Create a Namespace for Traefik:
- Defined in `traefik-namespace.yaml`
3. Create a ServiceAccount for Traefik:
- Defined in `traefik-service-account.yaml`
4. Create a ClusterRole and ClusterRoleBinding for the Traefik ServiceAccount:
- Defined in `traefik-cluster-role.yaml`
- Note: May want to go to per-namespace RoleBinding to follow the least-privilege principle
5. Create PersistentVolume and PersistentVolumeClaim for Traefik to store Let's Encrypt certificates:
- Defined in `persistent-volume-test.yaml` and `persistent-volume-claim-test.yaml`
- Notes:
    * PersistentVolume should be set up by cluster admin
    * If PersistentVolume is not used then a new certificate is generated each time the Traefik pod is spun up and you may hit Let's Encrypt limit of 20 certificates per week.
6. Create Secrets required by Traefik Deployment:
- Defined in `traefik-scrubbed.yaml` (fill in as appropriate)
- Note: This includes proxy address and Google DNS account information required for Let's Encrypt DNS challenge for issuing a certificate.
7. Create ConfigMap required by Traefik Deployment:
- Defined in `traefik-config.yaml`
- Note: Defines the Let's Encrypt options.
8. Create Deployment and Service for Traefik:
- Defined in `traefik-deployment.yaml`
- Notes: 
    * Traefik pod exposed on ports 80 (http), 443 (https) and 8080 (web-ui)
    * Each port is bound to their respective port on the node
    * We may want to increase the number of replicas
    * Traefik pod is running on a node that the persistent volume is defined on since we are using local storage
9. Verify Traefik pod is running: `kubectl get pods --namespace=ingress-traefik`
10. Create ingress rules:
- Defined in `traefik-ingress.yaml`
- We have defined:
    * Path `/kibana` to route to Kibana UI
    * Path `/traefik` to route to Traefik UI
11. Verify the dashboards can be viewed on your browser and Let's Encrypt certificates are generated:
- Kibana: `https://<host-name>/kibana`
- Traefik: `https://<host-name>/traefik`

### A note on Daemonset with Traefik
We can also deploy Traefik as a DaemonSet where each node runs a single Traefik pod. However, this method was not tested since [local persistent volumes](https://kubernetes.io/blog/2018/04/13/local-persistent-volumes-beta/) require pods to run on nodes where the persistent volumes are defined. Therefore, pods that are required to run on nodes that do not have local persistent volumes will be stuck in pending mode. This problem is avoided if we generate the certificates externally and load them to Kubernetes as Secrets, which is done in the next [section](#readme.md/Installation-External-Lets-Encrypt).

### A note on Kibana with Traefik
Kibana requires configuration to work with Traefik. The following options set in `kibana.yml` was successful:
- `server.host: 0.0.0.0`
- `server.basePath: /kibana`
- `server.rewriteBasePath: true`

## Installation (External Let's Encrypt without LDAP)
Note: Files are in `external_lets_encrypt/`.
1. Deploy the Elastic Stack application using [Helm](https://docs.helm.sh/using_helm/):
```
helm install --tiller-namespace=tiller-ns --namespace=default elastic-demo/
```
2. Create a Namespace for Traefik:
- Defined in `traefik-namespace.yaml`
3. Create a ServiceAccount for Traefik:
- Defined in `traefik-service-account.yaml`
4. Create a ClusterRole and ClusterRoleBinding for the Traefik ServiceAccount:
- Defined in `traefik-cluster-role.yaml`
- Note: May want to go to per-namespace RoleBinding to follow the least-privilege principle
5. Install [certbot](https://certbot.eff.org/lets-encrypt/ubuntuxenial-other.html) on local machine (Ubuntu 16.04 instructions):
```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot
```
6. Generate Let's Encrypt certificate:
```
sudo certbot -d <host-name> --manual --preferred-challenges dns certonly
```
- Note: This will require you to go to DNS provider and add a DNS TXT record. Make sure that you verify the DNS TXT record has [propagated](https://mxtoolbox.com/txtlookup.aspx) before proceeding with the Let's Encrypt DNS challenge.
7. Add Let's Encrypt certificate as a Kubernetes Secret:
```
kubectl -n default create secret tls tls-secret --key=/path/to/privkey.pem --cert=/path/to/cert.pem
kubectl -n ingress-traefik create secret tls tls-secret --key=/path/to/privkey.pem --cert=/path/to/cert.pem
```
- Note: Certificate needs to be added to each Namespace that Traefik routes to. In our case we need to it in `default` to route to the Kibana dashboard and `ingress-traefik` to route to the Traefik dashboard.
8. Create Daemonset and Service for Traefik:
- Defined in `traefik-daemonset.yaml`
- Notes: 
    * Traefik pod exposed on ports 80 (http), 443 (https) and 8080 (web-ui)
    * Each port is bound to their respective port on the node
    * Traefik pod is running on each worker node in the cluster. We used node selector with label `type: worker` to make sure that Traefik pods are not deployed on the master nodes. This is important because the Docker EE UCP is listening on port 443 which coincides with the Traefik HTTPS port!
9. Verify Traefik pod is running: `kubectl get pods --namespace=ingress-traefik`
10. Create ingress rules:
- Defined in `traefik-ingress.yaml`
- We have defined:
    * Path `/kibana` to route to Kibana UI
    * Path `/traefik` to route to Traefik UI
11. Verify the dashboards can be viewed on your browser and Let's Encrypt certificates are generated:
- Kibana: `https://<host-name>/kibana`
- Traefik: `https://<host-name>/traefik`

### Note on Certificate Renewal
Currently, we have to perform steps 6 and 7 every time we need to renew the Let's Encrypt certificate (by default Let's Encrypt certificates are valid for three months). Alternatively, we can download certbot onto a Docker node and allow it to run the built-in cron job to renew the certificate. We can then add another cron job to update the Kubernetes secret with the renewed certificate.

## Installation (External Let's Encrypt with LDAP)
Note: Files are in `external_lets_encrypt_ldap/`.
1. Deploy the Elastic Stack application using [Helm](https://docs.helm.sh/using_helm/):
```
helm install --tiller-namespace=tiller-ns --namespace=default elastic-demo/
```
2. Create a Namespace for Traefik:
- Defined in `traefik-namespace.yaml`
3. Create a ServiceAccount for Traefik:
- Defined in `traefik-service-account.yaml`
4. Create a ClusterRole and ClusterRoleBinding for the Traefik ServiceAccount:
- Defined in `traefik-cluster-role.yaml`
- Note: May want to go to per-namespace RoleBinding to follow the least-privilege principle
5. Install [certbot](https://certbot.eff.org/lets-encrypt/ubuntuxenial-other.html) on local machine (Ubuntu 16.04 instructions):
```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot
```
6. Generate Let's Encrypt certificate:
```
sudo certbot -d <host-name> --manual --preferred-challenges dns certonly
```
- Note: This will require you to go to DNS provider and add a DNS TXT record. Make sure that you verify the DNS TXT record has [propagated](https://mxtoolbox.com/txtlookup.aspx) before proceeding with the Let's Encrypt DNS challenge.
7. Add Let's Encrypt certificate as a Kubernetes Secret:
```
kubectl -n default create secret tls tls-secret --key=/path/to/privkey.pem --cert=/path/to/cert.pem
kubectl -n ingress-traefik create secret tls tls-secret --key=/path/to/privkey.pem --cert=/path/to/cert.pem
```
- Note: Certificate needs to be added to each Namespace that Traefik routes to. In our case we need to it in `default` to route to the Kibana dashboard and `ingress-traefik` to route to the Traefik dashboard.
8. Create Daemonset and Service for Traefik:
- Defined in `traefik-daemonset.yaml`
- Notes: 
    * Traefik pod exposed on ports 80 (http), 443 (https) and 8080 (web-ui)
    * Each port is bound to their respective port on the node
    * Traefik pod is running on each worker node in the cluster
9. Verify Traefik pod is running: `kubectl get pods --namespace=ingress-traefik`
10. Create ServiceAccount for Nginx:
- Defined in `nginx-service-account.yaml`
- Note: Nginx will reside in the same Namespace as Traefik (`ingress-traefik`).
11. Create Role and RoleBinding for Nginx ServiceAccount:
- Defined in `nginx-role.yaml`
12. Add LDAP connection information as a Kubernetes Secret:
```
kubectl create secret -n ingress-traefik generic nginx-ldap-auth --from-file=config.yaml=nginx-scrubbed.yaml
```
- Note: Fill in the missing infomration in `nginx-scrubbed.yaml` as it pertains to your specific LDAP setup. 
13. Create Deployment and Service for Nginx:
- Defined in `nginx-deployment.yaml`
14. Create ingress rules:
- Defined in `traefik-ingress.yaml`
- We have defined:
    * Path `/kibana` to route to Kibana UI (no authentication)
    * Path `/traefik` to route to Traefik UI (with LDAP authentication)
15. Verify the dashboards can be viewed on your browser and Let's Encrypt certificates are generated:
- Kibana: `https://<host-name>/kibana`
- Traefik: `https://<host-name>/traefik` (requires LDAP authentication)

### Note on Authorization
If only care about authentication, then we can create a single Nginx deployment and forward all Ingress objects to that server. If we care about authorization, then we have the option to filer by group. This requires creating an Nginx deployment for each unique combination of groups and specifying the correct Nginx server URL in `traefik.ingress.kubernetes.io/auth-url` for each Ingress object. Finer control (e.g., specify by user) may be possible by altering the Nginx-LDAP [Docker image](https://hub.docker.com/r/tpimenta/nginx-ldap-auth).

## Resources
1. https://success.docker.com/article/how-to-configure-traefik-as-a-layer-7-ingress-controller-for-kubernetes
2. https://kubernetes.io/docs/concepts/services-networking/ingress/
3. https://medium.com/@dusansusic/traefik-ingress-controller-for-k8s-c1137c9c05c4
4. https://github.com/containous/traefik
5. https://docs.traefik.io/user-guide/kubernetes/
6. https://letsencrypt.org/
7. https://certbot.eff.org/lets-encrypt/ubuntuxenial-other.html
8. https://github.com/tiagoapimenta/nginx-ldap-auth