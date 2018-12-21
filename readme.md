# Ingress with Traefik Testing
This repository will go through the steps for using Kubernetes ingress with Traefik as the ingress controller. We will deploy a sample Elastic Stack application and use Traefik to route the requests to both the Kibana and Traefik dashboards. The sample Elastic Stack application can be found at https://github.com/cheuklau/elastic-stack-logging.

## What is Ingress?
Ingress exposes HTTP/HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined in the ingress resource. An ingress controller (e.g., Traefik) is responsible for fulfilling the ingress.

## Installation (Unsecured)
1. Deploy the Elastic Stack application using Helm:
```
helm install --tiller-namespace=tiller-ns --namespace=default elastic-demo/
```
2. Create a Namespace for Traefick:
- Defined in `traefik-namespace.yaml`
3. Create a ServiceAccount for Traefick:
- Defined in `traefik-service-account.yaml`
4. Create a ClusterRole and ClusterRoleBinding for the Traefik ServiceAccount:
- Defined in `traefik-cluster-role.yaml`
- Note: May want to go to per-namespace RoleBinding to follow the least-privilege principle
5. Create Deployment and Service for Traefik:
- Defined in `traefik-deployment.yaml`
- Notes:
    * Traefik pod exposed on ports 80 and 8080
    * Port 80 exposes the Traefik API which is bound to NodePort 35080
    * Port 8080 exposes the Traefik UI
    * We may want to increase the number of replicas in the future
6. Verify Traefik pod is running: `kubectl get pods --namespace=ingress-traefik`
7. Create ingress rules:
- Defined in `traefik-ingress.yaml`
- We have defined:
    * Host `localhost` with path `\kibana` to route to Kibana UI
    * Host `localhost` with path `\traefik` to route to Traefik UI
8. Verify the dashboards can be viewed on your browser:
- Kibana: `localhost:35080/kibana`
- Traefik: `localhost:35080/traefik`
- Notes:
    * `localhost:35080` routes to the port that Traefik API is listening on
    * All future Traefik routing will be done through `localhost:35080`
    * On production clusters `localhost` will be replaced with the hostname associated with the load balancer in front of the Kubernetes nodes

## A note on Kibana with Traefik
Kibana requires configuration to work with Traefik. The following options set in `kibana.yml` was successful:
- `server.host: 0.0.0.0`
- `server.basePath: /kibana`
- `server.rewriteBasePath: true`

## Installation (Secure)
- Future work: Let's Encrypt, TCP, Basic Authentication

## Resources
1. https://success.docker.com/article/how-to-configure-traefik-as-a-layer-7-ingress-controller-for-kubernetes
2. https://kubernetes.io/docs/concepts/services-networking/ingress/
3. https://medium.com/@dusansusic/traefik-ingress-controller-for-k8s-c1137c9c05c4
4. https://github.com/containous/traefik
5. https://docs.traefik.io/user-guide/kubernetes/