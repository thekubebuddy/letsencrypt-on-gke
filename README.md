# Managing SSL with Let's encrypt and nginx-controller on GKE

* Let's Encrypt provides a browser-trusted certificate for web services.
* In combination with **cert-manager**, a Kubernetes add-on, the management and issuance of TLS certificates from Let's Encrypt will be completely automated.
* With Let's Encrypt, we have access to a free, automated, and open certificate authority (CA), run for the **public's benefit**.
* As GKE does not provide a managed HTTPS offering, so it can be a bit daunting trying to take on the task of obtaining a valid TLS certificate without prior experience.
* And since GKE also lacks built-in HTTP to HTTPs redirect for Google Cloud Load Balancers (GCLB), an NGINX ingress will be deployed to handle HTTP to HTTPs redirect.

### Create a GKE cluster
```
$ gcloud container clusters create demo-cluster --machine-type "n1-standard-1" --num-nodes 1 --zone us-central1-a --preemptible 
```

### Connect to the cluster 
```
$ gcloud container clusters get-credentials demo-cluster --zone us-central1-a
```

### Deploy the hello-app deployment and the service manifest
```
$ kubectl apply -f hello-dpl-svc.yaml
```

### Install the nginx ingress controller

* Assign yourself the cluster-admin role by running the following command:
```
$ kubectl create clusterrolebinding cluster-admin-binding \
--clusterrole cluster-admin --user $(gcloud config get-value account)

# deploy the nginx-ingress-controller(v0.35.0) using the
$ kubectl apply -f nginx-ingress-controller.yaml 
```

Note: Static Ip can also be used for the ingress-nginx-controller service, using "loadBalancerIP" attribute in the manifest file.
For reserving static IP in GCP
```
$ gcloud compute addresses create nginx-ing-ip --region us-central1
$ gcloud compute addresses list

# for the deleting the ext-ip
$ gcloud compute addresses delete nginx-ing-ip --region us-central1
```

### Deploying the ingress for the hello-app without TLS 

* Before deploying properly change the host as per needed within the ingress file
```
k apply -f ./ingress.yaml
```
* Validate the ingress endpoint either by mapping the hostname to the **/et/hosts** file or more like mapping the ingress hostname with the ingress IP in the respective cloud DNS provider(or CloudDNS if primarily using)


### Setting the Let's Encrypt

* Installing the cert-manager on the GKE cluster
```
# Kubernetes 1.16+ (apiVersion: cert-manager.io/v1alpha2 is used)
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager.yaml


# Kubernetes <1.16 (apiVersion: cert-manager.io/v1 is used)
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager-legacy.yaml
```

* Verify that cert-manager was installed:
```
kubectl get pods --namespace cert-manager
```

* Deploy the  Let's Encrypt issuer to issue TLS certificates. Deploy the Issuer manifest using the following commands:
```
$ export EMAIL="your-email@your-domain.com"

# Deploying the clusterissuer manifest
$ cat letsencrypt-issuer.yaml | sed "s/email: ''/email: $EMAIL/g" | kubectl apply -f-

# verify that the clusterissuer properly configured
$ k get clusterissuers.cert-manager.io 
```

### Creating the certificate for the ingress

* Map the DNS(from cloud DNS if used) for the hostname with the Ingress IP so that the Certificate can be easily provisioned 

* Open the certificate file and make changes for the   `commonName`,`dnsNames`,`domains` attribute as per the ingress host name. Finally apply the certificate
```
$ kubectl apply -f ./certificate.yaml --validate=false
```
* The above certificate will generate the TLS secret within the namespace.

* Apply the final ingress with TLS enabled and SSL redirect 
```
$ kubectl apply -f ./ingress-tls.yaml --validate=false
```

* Confirm the endpoint with the just pasting the ingress host, if should automatically redirect to the HTTPS. 


### Cleanup
```
$ gcloud container clusters delete demo-cluster --zone us-central1-a
```


#### Reference
https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke
https://cert-manager.io/docs/installation/kubernetes/