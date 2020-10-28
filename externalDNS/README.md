# ExternalDNS on GKE

* ExternalDNS allows to control DNS records dynamically via Kubernetes resources in a DNS provider-agnostic way.
* It makes Kubernetes resources discoverable via public DNS servers. 
* Like Kubernetes' KubeDNS, it retrieves a list of resources (Services, Ingresses, etc.) from the Kubernetes API to determine a desired list of DNS records. 
* Unlike KubeDNS, however, it's not a DNS server itself, but merely configures other DNS providers accordingly —e.g. AWS Route 53 or Google Cloud DNS.


### Installation on GKE

* Connect to the GKE cluster

* Create a namespace `external-dns` and deploy the manifest
```
$ kubectl create ns external-dns
```
* Change the values for the below argument within the install-manifest.yaml
```
....
....
        - --domain-filter=ext-dns.mydomain.org # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
        ...
        - --google-project=<gcp-project-name> # Use this to specify a project different from the one external-dns is running inside
....
....
```
* Apply the install manifest file

```
$ kubectl apply -f ./install-manifest.yaml -n external-dns 
```

* Check the logs as well as the status for the externalDNS pod
```
$ kubectl get po -n external-dns 
$ kubectl logs -n external-dns -l app=external-dns -f 
```

**Note: Make sure that the GKE cluster SA has the DNSAdmin Role attach else the controller wont be able to do the CRUD opertions.**

**For more secure method we can have a secret with the json key for the custom SA & use in the controller manifest file by using volumeMounts**

### Usage 

1. For ingress objects ExternalDNS will create a DNS record based on the host specified for the ingress object.

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx
spec:
  rules:
  - host: nginx-ing.external-dns-test.gcp.zalan.do # just a sample DNS 
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
```

2. For services ExternalDNS will look for the annotation `external-dns.alpha.kubernetes.io/hostname` on the service and use the corresponding value.

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx-svc.external-dns-test.gcp.zalan.do # just a sample DNS 
  name: nginx
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

