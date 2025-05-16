# arkhn-test
Repository for arkhn technical test

## Minikube
Will allow the deployment and configuration of a local kubernetes cluster with docker (docker packages already installed):
```
# Install minikube CLI
$ curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64                                                                         $ sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

# Deploy a basic cluster
$ minikube start --profile "arkhn"
```
  
## MetalLB
Will allow me to deploy Traefik with a service-type load-balancer without having to make some `minikube tunnel` command
  
I'm using the metallb addon as it streamline the deployment and configuration for a quick test:
```
# Enable metallb addon
minikube --profile=arkhn addons enable metallb

# Get minikube IP
$ minikube --profile=arkhn ip
192.168.49.2

# Configure metallb addon
$ minikube --profile=arkhn addons configure metallb
-- Enter Load Balancer Start IP: 192.168.49.100
-- Enter Load Balancer End IP: 192.168.49.110
    ▪ Using image quay.io/metallb/speaker:v0.9.6
    ▪ Using image quay.io/metallb/controller:v0.9.6
✅  metallb was successfully configured

# Check config
$ k -n metallb-system describe cm config
Name:         config
Namespace:    metallb-system
Labels:       <none>
Annotations:  <none>

Data
====
config:
----
address-pools:
- name: default
  protocol: layer2
  addresses:
  - 192.168.49.100-192.168.49.110
```

## Vault
Will generate the CA needed for the Mtls part
  
Install:
```
$ k create ns vault

$ helm repo add hashicorp https://helm.releases.hashicorp.com

$ helm search repo hashicorp/vault
-> Get the latest version

$ helm upgrade --install --version 0.30.0 --namespace vault vault hashicorp/vault -f vault/dev-mode.yaml

$ helm -n vault list
NAME 	NAMESPACE	REVISION	UPDATED                                 	STATUS  	CHART       	APP VERSION
vault	vault    	1       	2025-05-13 14:30:02.368371257 +0200 CEST	deployed	vault-0.30.0	1.19.0
```
  
PKI configuration:
```
$ k exec -n vault -it vault-0 -- sh

/ $ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/

/ $ vault write pki/root/generate/internal common_name="arkhn.test" issuer_name="arkhn-labs"

/ $ vault write pki/config/urls \
    issuing_certificates="http://vault.vault:8200/v1/pki/ca" \
    crl_distribution_points="http://vault.vault:8200/v1/pki/crl"

/ $ vault write pki/roles/arkhn-dot-test \
    allowed_domains=arkhn.test \
    allow_subdomains=true \
    max_ttl=72h

# Certificate generation and put as secret as it was easier for me
/ $ vault write -format=json pki/issue/arkhn-dot-test \
    common_name=hello-world.arkhn.test

$ <pc> CERT=$(cat $HOME/Downloads/cert.json| jq -r .data.certificate)
$ <pc> KEY=$(cat $HOME/Downloads/cert.json| jq -r .data.private_key)
$ <pc> CA=$(cat $HOME/Downloads/cert.json| jq -r .data.issuing_ca)

/ $ CERT="XX"
/ $ KEY="XX"
/ $ CA="XX"

/ $ vault kv put secret/hello-world-cert \
    cert="$CERT" \
    key="$KEY" \
    ca="$CA"

======== Secret Path ========
secret/data/hello-world-cert

======= Metadata =======
Key                Value
---                -----
created_time       2025-05-14T15:17:05.403058163Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

# Verif only
/ $ vault kv get secret/hello-world-cert

```
  
Secret needed for ESO:
```
$ k -n vault logs vault-0

$ k create ns demo-app

$ k -n demo-app create secret generic vault-token --from-literal=token=<root_token>
```
  
## External Secret Operator
Will synchronize the Vault object with a Kubernetes secret, allowing Traefik to use it (as it seems that it can't gather the vault object directly)

```
$ helm repo add external-secrets https://charts.external-secrets.io

$ helm search repo external-secrets
-> Get the latest version for external-secrets/external-secrets

$ helm upgrade --install --version 0.16.2 --namespace external-secrets --create-namespace external-secrets external-secrets/external-secrets 

$ helm -n external-secrets list
NAME            	NAMESPACE       	REVISION	UPDATED                                 	STATUS  	CHART                  	APP VERSION
external-secrets	external-secrets	1       	2025-05-13 11:40:30.486093567 +0200 CEST	deployed	external-secrets-0.16.2	v0.16.2
```
  
Then create the SecretStore object to talk with the hashicorp Vault:
```
$ k apply -f external-secret/vault-secret-store.yaml

$ k -n demo-app describe secretstores.external-secrets.io vault-secret-store
Status:
  Capabilities:  ReadWrite
  Conditions:
    Last Transition Time:  2025-05-13T12:50:01Z
    Message:               store validated
    Reason:                Valid
    Status:                True
    Type:                  Ready
```
  
## Traefik
Will handle external Web access to the cluster.
  
First step is to create a `values.yaml` file to enable the GatewayAPI capability:
```
# values.yaml
providers:
  kubernetesIngress:
    enabled: true
  # Enable the GatewayAPI provider
  kubernetesGateway:
    enabled: true
# Allow the Gateway to expose HTTPRoute from all namespaces
gateway:
  namespacePolicy: All
logs:
    general:
      format: json
    access:
      enabled: true
      format: json
```
  
Then deploy Traefik:
```
$ helm repo add traefik https://traefik.github.io/charts

$ helm repo update

$ k create namespace traefik

$ helm search repo traefik
-> Get the latest version for traefik/traefik

$ helm upgrade --install --version 35.2.0 --namespace traefik traefik traefik/traefik -f traefik/values.yaml

$ helm -n traefik list
NAME   	NAMESPACE	REVISION	UPDATED                                 	STATUS  	CHART         	APP VERSION
traefik	traefik  	3       	2025-05-12 16:37:29.369783129 +0200 CEST	deployed	traefik-35.2.0	v3.3.6

# Check that the load-balancer service got an external-ip via metallb
$ k -n traefik get svc
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
traefik   LoadBalancer   10.107.121.31   192.168.49.100   80:31708/TCP,443:32080/TCP   13m
```
  
## Test app
Deploy snake game:
```
$ k create ns demo-app

$ k apply -f demo-app/deploy.yaml
$ k apply -f demo-app/svc.yaml
```
  
Add the ExternalSecret object to gather the certificate needed:
```
$ k apply -f external-secret/external-secret.yaml

$ k -n demo-app get secret hello-world-cert
NAME               TYPE                DATA   AGE
hello-world-cert   kubernetes.io/tls   3      2m11s

```

And a specific Traefik IngressRoute to access it:
```
$ k apply -f demo-app/traefik-ingressroute.yaml
```
  
Edit host file to add the configured dns record:
```
$ vi /etc/hosts
## Arkhn - The svc url of traefik
192.168.49.100 hello-world.arkhn.test
```

## Browser CA
Add the generated CA to the browser to avoid https error

```
Get the CA output from the vault and put it in a file "arkhn_test.crt"
$ echo $CA > arkhn_test.crt

Add it under "settings -> Privacy and security -> Security -> Manage certificate -> Authorities -> Import -> Trust this certificate for identifying websites"
```

-> Test web access, should be good
  
## Mtls
Adding an Mtls configuration to secure the client connection
  
```
$ k apply -f traefik/tls-option-mtls.yaml

# Client certificate generation
$ k exec -n vault -it vault-0 -- sh
/ $ vault write -format=json pki/issue/arkhn-dot-test \
    common_name=client-hello-world.arkhn.test
-> save output to a file "client_cert.json"

# Ajout du mtls option sur l'ingress
-> uncomment the "options" section of the "traefik-ingressroute.yaml" file and apply again
$ k apply -f demo-app/traefik-ingressroute.yaml

# Get certificate detail for client
$ <pc> cat $HOME/Downloads/client_cert.json| jq -r .data.issuing_ca > $HOME/Downloads/client_ca.crt
$ <pc> cat $HOME/Downloads/client_cert.json| jq -r .data.certificate > $HOME/Downloads/client_cert.crt
$ <pc> cat $HOME/Downloads/client_cert.json| jq -r .data.private_key > $HOME/Downloads/client_key.key

# Test accès KO sans cert via navigateur
ERR_BAD_SSL_CLIENT_AUTH_CERT

# Test accès OK avec cert
$ curl -k https://hello-world.arkhn.test --cert $HOME/Downloads/client_cert.crt --key $HOME/Downloads/client_key.key
<!DOCTYPE html>
<html>
<head>
  <title>Kubernetes Application Example</title>

-> Navigateur :
$ cat $HOME/Downloads/client_cert.crt $HOME/Downloads/client_key.key > $HOME/Downloads/pkcs12.pem
$ openssl pkcs12 -in $HOME/Downloads/pkcs12.pem -export -out $HOME/Downloads/pkcs12.p12

Lors de l'accès il va me demander de choisir un cert, et ensuite je passe

```