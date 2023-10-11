# Bookinfo example
This is the Bookinfo example, which is the most fundamental official tutorial of Istio.

The setup environment utilizes `EKS` and is configured using `Terraform`. However, it is written with the aim of gaining understanding through actual **usage of Istio.**

### Reference
- [Istio GitHub](https://github.com/istio/istio/tree/master/samples/bookinfo) 

### Set-Up EKS, Istio (Terraform)
> Explanation for the Terraform code has been omitted. This covers the basic EKS creation code.
1. **terraform apply**
```
$ terraform init

$ terraform plan

$ terraform apply
```

2. **kubeconfig**
```
aws eks update-kubeconfig --region ap-northeast-2 --name eks-cluster
```

3. **istio**
```
# CAUTION!! Execute in the directory where terraform or kubeconfig resides.

$ curl -L https://istio.io/downloadIstio | sh -
$ cd istio-1.19.1
$ echo "export PATH=$PWD/bin:$PATH" >> ~/.bash_profile
$ source ~/.bash_profile

# Verify the installation
$ istioctl
# If something appears, the installation is successful.

# Install the demo profile
$ istioctl install --set profile=demo

# Pay attention to the path and often need to execute twice.
$ kubectl apply -f samples/addons 

# Check the status
$ kubectl get pod,svc -n istio-system

# Enable istio sidecar injection
$ kubectl label namespace default istio-injection=enabled    
```
- The scope of support varies by `profile`, and in the `demo`, elements like the ingress and egress gateways are installed by default.
- The `ingressgateway` is of the LoadBalancer type, so one can confirm the addition of the LoadBalancer in the AWS console.
- Detailed information about the `ingressgateway` is covered in the bookinfo example section.


4. **kialili dashboard**

The Kiali dashboard is useful for visualizing traffic flow and other aspects when learning Istio.
``` 
$ istioctl dashboard kialili

# When using SSH, access the dashboard using SSH tunneling.
$ ssh -L 20001:localhost:20001 royroyee@10.10.0.170
```

---

### Bookinfo Example
Once the setup is complete, launch applications and gateways using Istio.

1. **bookinfo application (service, deployment)**
````
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
````

- bookinfo.yaml
    - Reference to the yaml file: https://github.com/istio/istio/blob/master/samples/bookinfo/platform/kube/bookinfo.yaml
    - Services and deployments for `detail`, `productpage`, `reviews`, and `ratings`.
    - Service: All specified with the same port 9080, using labels instead of target ports.
    - Container port: All specified with the same port 9080.


- architecture
![bookinfo.png](..%2Fimg%2Fbookinfo.png)

2. **bookinfo gateway**
```
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

- bookinfo-gateway.yaml
```yaml
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

> Note : In Istio, traffic is controlled through the Gateway and Virtual Service.

- `istio-ingressgateway` : Ingress gateway installed when setting up the demo profile:
  - The entry point (frontmost layer) of the cluster.
  - For instance, operates in a manner similar to the Kubernetes nginx ingress controller.
  - Type of LoadBalancer (Service).
  - Functions as a pod.


- `Gateway` : Sets up ports, protocols, and TLS authentication.
  - `VirtualService` is combined and used with the `Gateway`.
  - In this case, all hosts are allowed.
  - In this case, the `istio-ingressgateway` is used (must specify the ingress gateway).


- `VirtualService` : Configures routing to determine which service it will connect to.
  - Since it's combined with the `Gateway`, the `Gateway` to be used must be specified.
