# Chaining of the Wallarm and additional Ingress Controllers in the same Kubernetes cluster

These instructions provide you with the steps to deploy the Wallarm Ingress controller to your K8s cluster and chain it with other Controllers that are already running in your environment.

## The issue addressed by the solution

Wallarm offers its node software in different form-factors, including [Ingress Controller built on top of the Community Ingress NGINX Controller](installation-kubernetes-en.md).

If you already use an Ingress controller, it might be challenging to replace the existing Ingress controller with the Wallarm controller (e.g. if using AWS ALB Ingress Controller). In this case, you can explore the [Wallarm Sidecar proxy solution](../installation/kubernetes/sidecar-proxy/deployment.md) but if it also does not fit your infrastructure, it is possible to chain several Ingress controllers.

Ingress controller chaining enables you to utilize an existing controller to get end-user requests to a cluster, and deploy an additional Wallarm Ingress controller to provide necessary application protection.

## Requirements

* Kubernetes platform version 1.23-1.25
* [Helm](https://helm.sh/) package manager
* Access to the account with the **Administrator** role in Wallarm Console for the [US Cloud](https://us1.my.wallarm.com/) or [EU Cloud](https://my.wallarm.com/)
* Access to `https://us1.api.wallarm.com` for working with US Wallarm Cloud or to `https://api.wallarm.com` for working with EU Wallarm Cloud
* Access to `https://charts.wallarm.com` to add the Wallarm Helm charts. Ensure the access is not blocked by a firewall
* Access to the Wallarm repositories on Docker Hub `https://hub.docker.com/r/wallarm`. Make sure the access is not blocked by a firewall
* Access to [GCP storage addresses](https://www.gstatic.com/ipranges/goog.json) to download an actual list of IP addresses registered in [allowlisted, denylisted, or graylisted](../user-guides/ip-lists/overview.md) countries, regions or data centers
* Deployed Kubernetes cluster running an Ingress controller

## Deploying the Wallarm Ingress controller and chaining it with an additional Ingress Controller

To deploy the Wallarm Ingress controller and chain it with additional controllers:

1. Deploy the official Wallarm controller Helm chart using an Ingress class value different from the existing Ingress controller.
1. Create the Wallarm-specific Ingress object with:

    * The Ingress class similar to the value in the Wallarm Ingress controller configuration.
    * External ELB/ALB load balancers disabled, so the Wallarm Ingress controller will be not exposed to the Internet.
    * Ingress controller requests routing rules configured in the same way as the existing Ingress controller.
1. Reconfigure the existing Ingress controller to forward incoming requests to the new Wallarm Ingress controller instead of application services.
1. Test the Wallarm Ingress controller operation.

### Step 1: Deploy the Wallarm Ingress controller

1. Go to Wallarm Console → **Nodes** via the link below:
    * https://us1.my.wallarm.com/nodes for the US Cloud
    * https://my.wallarm.com/nodes for the EU Cloud
1. Create a filtering node with the **Wallarm node** type and copy the generated token.
    
    ![!Creation of a Wallarm node](../images/user-guides/nodes/create-wallarm-node-name-specified.png)
1. Add the [Wallarm chart repository](https://charts.wallarm.com/):
    ```
    helm repo add wallarm https://charts.wallarm.com
    ```
1. Create the new K8s namespace, e.g. `wallarm-ingress`:

    ```bash
    kubectl create namespace wallarm-ingress
    ```
1. Create the `values.yaml` file with the following Wallarm configuration:

    === "US Cloud"
        ```bash
        controller:
          wallarm:
            enabled: "true"
            token: "<NODE_TOKEN>"
            apiHost: "us1.api.wallarm.com"
          ingressClass: "wallarm-ingress"
          ingressClassResource:
            name: "wallarm-ingress"
          service
            type: "ClusterIP"
        nameOverride: "wallarm-ingress"
        ```
    === "EU Cloud"
        ```bash
        controller:
          wallarm:
            enabled: "true"
            token: "<NODE_TOKEN>"
          ingressClass: "wallarm-ingress"
          ingressClassResource:
            name: "wallarm-ingress"
          service
            type: "ClusterIP"
        nameOverride: "wallarm-ingress"
        ```    
    
    `<NODE_TOKEN>` is the Wallarm node token.

    To learn more configuration options, please use the [link](configure-kubernetes-en.md).
1. Install the Wallarm packages:

    ``` bash
    helm install --version 4.4.0 <INGRESS_CONTROLLER_NAME> wallarm/wallarm-ingress -n wallarm-ingress -f <PATH_TO_VALUES>
    ```

    * `<INGRESS_CONTROLLER_NAME>` is the name for the Wallarm Ingress controller
    * `<PATH_TO_VALUES>` is the path to the `values.yaml` file
1. Verify that the Wallarm ingress controller is up and running: 

    ```bash
    kubectl get pods -A|grep wallarm
    ```

    Each pod status should be **STATUS: Running** or **READY: N/N**. For example:

    ```
    NAME                                                              READY     STATUS    RESTARTS   AGE
    ingress-controller-nginx-ingress-controller-675c68d46d-cfck8      4/4       Running   0          5m
    ingress-controller-nginx-ingress-controller-wallarm-tarantljj8g   4/4       Running   0          5m
    ```

### Step 2: Create the Wallarm-specific Ingress object

Create the Wallarm-specific Ingress object, e.g.:

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/wallarm-application: "1"
    nginx.ingress.kubernetes.io/wallarm-mode: monitoring
    kubernetes.io/ingress.class: "wallarm-ingress"
  name: wallarm-ingress
  namespace: default
spec:
  rules:
  - host: www.example.com
    http:
      paths:
      - backend:
          service:
            name: myapp
            port:
              number: 80
        path: /
```

### Step 3: Reconfigure the existing Ingress controller to forward requests to Wallarm

Reconfigure the existing Ingress controller to forward incoming requests to the new Wallarm Ingress controller instead of application services, e.g.:

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: nginx-ingress
  namespace: wallarm-ingress
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - backend:
              service:
                name: wallarm-ingress-controller
                port:
                  number: 80
            path: /
```

### Step 4: Test the Wallarm Ingress controller operation

Send a test request to the existing Ingress controller address and verify that the system is working as expected:

```bash
curl http://<INGRESS_CONTROLLER_IP>/etc/passwd
```
