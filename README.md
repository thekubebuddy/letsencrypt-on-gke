# Managing SSL certificates with Let's encrypt+cert-manager+nginx-controller on GKE

* Let's Encrypt provides a browser-trusted certificate for web services.
* In combination with **cert-manager**, a Kubernetes add-on, the management and issuance of TLS certificates from Let's Encrypt will be completely automated.
* With Let's Encrypt, we have access to a free, automated, and open certificate authority (CA), run for the **public's benefit**.
* As GKE does not provide a managed HTTPS offering, so it can be a bit daunting trying to take on the task of obtaining a valid TLS certificate without prior experience.
* And since GKE also lacks built-in HTTP to HTTPs redirect for Google Cloud Load Balancers (GCLB), an NGINX ingress will be deployed to handle HTTP to HTTPs redirect.

### Clone the Repo
* Run the following command to clone the Github repository:
```
$ git clone https://github.com/thekubebuddy/letsencrypt-on-gke.git
$ cd ./letsencrypt-on-gke/
```

### GKE cluster creation
```
$ gcloud container clusters create demo-cluster --machine-type "e2-medium" --num-nodes 2 --zone us-central1-a --preemptible --labels=owner=ishaq --subnetwork=us-central1  --network=default

us-ct1-pvt-app-subnet-ishaq
```

### Connecting to the GKE cluster 
```
$ gcloud container clusters get-credentials demo-cluster --zone us-central1-a
```

### Deployment of the "hello-app" 
```
$ kubectl apply -f ./hello-app-dpl-svc.yaml
```

### Installation of the nginx-ingress controller

* Assign yourself the cluster-admin role by running the following command:
```
$ kubectl create clusterrolebinding cluster-admin-binding \
--clusterrole cluster-admin --user $(gcloud config get-value account)

# deploy the nginx-ingress-controller(v0.35.0) using the
$ kubectl apply -f ./nginx-ingress-controller.yaml 
```

**Note: Static IP can also be used for the ingress-nginx-controller service, using "loadBalancerIP" attribute in the nginx-ingress-controller.yaml manifest file.**

**Also For private GKE cluster, you will need to either add an additional firewall rule that allows master nodes access to port 8443/tcp on worker nodes, or change the existing rule that allows access to ports 80/tcp, 443/tcp and 10254/tcp to also allow access to port 8443/tcp.**

For reserving static IP in GCP use the below gcloud command
```
$ gcloud compute addresses create nginx-ing-ip --region us-central1
$ gcloud compute addresses list

# for the deleting the ext-ip
$ gcloud compute addresses delete nginx-ing-ip --region us-central1
```
* Verify that nginx-controller was installed properly:
```
$ kubectl get pods,svc --namespace ingress-nginx
```

### Installation of the ExternalDNS for managing the DNS Mapping(Optional) 


### Setting-up the Let's Encrypt

* Installing the cert-manager on the GKE cluster
```
# Kubernetes 1.16+ (apiVersion: cert-manager.io/v1alpha2 is used)
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager.yaml


# Kubernetes <1.16 (apiVersion: cert-manager.io/v1 is used)
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager-legacy.yaml
```

* Verify that cert-manager was installed properly and all of the pods are running:
```
$ kubectl get pods --namespace cert-manager
```

* Deploy the  Let's Encrypt issuer to issue TLS certificates. Deploy the Issuer manifest using the following commands:
```
$ export EMAIL_ID="your-email@your-domain.com"

# Deploying the clusterissuer manifest
$ sed "s/EMAIL_ID/$EMAIL_ID/g" letsencrypt-issuer.yaml.tpl | kubectl apply -f-

# verify that the clusterissuer properly configured & status is set to True
$ kubectl get clusterissuers.cert-manager.io 
```

### Certificate & ingress creation 

* (Required) Map the DNS(from cloud DNS if used) for the hostname with the Ingress IP so that the Certificate can be easily provisioned 
Or
If its being managed by the externalDNS then no need to map the DNS.

* Open the cert-ingress.yaml file and make changes for the   `commonName`,`dnsNames`,`domains` attribute as per the ingress host name. Finally apply the certificate

* The above certificate will generate the TLS secret **"hello-app-tls"** within the namespace.

* Apply the final ingress with TLS enabled and SSL redirect 
```
$ kubectl apply -f ./cert-ingress.yaml --validate=false
```

* Confirm the endpoint with the just pasting the ingress host, if should automatically redirect to the HTTPS. 


### Cleanup
```
$ gcloud container clusters delete demo-cluster --zone us-central1-a
```


#### Reference
https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke

https://cert-manager.io/docs/installation/kubernetes/