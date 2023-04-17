# open5gs-central-certs
Running 5G Core using a central PKI solution for certificate generation.

Blog post: https://futuredon.medium.com/5g-core-sbi-mtls-using-external-certificate-pki-4ffc02ac7728

## KIND Cluster Install

***wget https://github.com/kubernetes-sigs/kind/releases/download/v0.18.0/kind-linux-amd64***

***chmod +x kind-linux-amd64 && sudo mv kind-linux-amd64 /usr/local/bin/kind***

Create kind config file:

```
cat <<EOF >kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.23.13@sha256:ef453bb7c79f0e3caba88d2067d4196f427794086a7d0df8df4f019d5e336b61
- role: worker
  image: kindest/node:v1.23.13@sha256:ef453bb7c79f0e3caba88d2067d4196f427794086a7d0df8df4f019d5e336b61
EOF
```

***kind create cluster -n core5g --config kind-config.yaml***

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.23.13@sha256:ef453bb7c79f0e3caba88d2067d4196f427794086a7d0df8df4f019d5e336b61
- role: worker
  image: kindest/node:v1.23.13@sha256:ef453bb7c79f0e3caba88d2067d4196f427794086a7d0df8df4f019d5e336b61
```

```
kubectl get no
NAME                   STATUS   ROLES           AGE   VERSION
core5g-control-plane   Ready    control-plane   74s   v1.26.3
core5g-worker          Ready    <none>          50s   v1.26.3
```

## Install Smallstep CA
```
helm repo add smallstep  https://smallstep.github.io/helm-charts
helm repo update
helm -n pki install autocert --set step-certificates.ca.runAsRoot=true smallstep/autocert --create-namespace

NAME: autocert
LAST DEPLOYED: Mon Apr 17 15:08:12 2023
NAMESPACE: pki
STATUS: deployed
REVISION: 1
NOTES:
Thanks for installing Autocert.

1. Enable Autocert in your namespaces:
   kubectl label namespace pki autocert.step.sm=enabled

2. Check the namespaces where Autocert is enabled:
   kubectl get namespace -L autocert.step.sm

3. Get the PKI and Provisioner secrets running these commands:
   kubectl get -n pki -o jsonpath='{.data.password}' secret/autocert-step-certificates-ca-password | base64 --decode
   kubectl get -n pki -o jsonpath='{.data.password}' secret/autocert-step-certificates-provisioner-password | base64 --decode

4. Get the CA URL and the root certificate fingerprint running this command:
   kubectl -n pki logs job.batch/autocert

5. Delete the configuration job running this command:
   kubectl -n pki delete job.batch/autocert
```

## Create Open5gs namespace and label it for SmallStep
```
kubectl create ns open5gs
kubectl label namespace open5gs autocert.step.sm=enabled
```

# Install Open5gs

```
helm -n open5gs upgrade --install core5g open5gs-helm-charts/


Release "core5g" does not exist. Installing it now.
NAME: core5g
LAST DEPLOYED: Mon Apr 17 15:21:37 2023
NAMESPACE: open5gs
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Status after deployment:

$(kubectl --namespace open5gs get all)

kubectl -n open5gs get po
NAME                                      READY   STATUS    RESTARTS   AGE
core5g-amf-deployment-544667f9c7-z4bj6    2/2     Running   0          42m
core5g-ausf-deployment-668fb8d8f8-zn628   2/2     Running   0          42m
core5g-bsf-deployment-57fcfb768d-rlffq    2/2     Running   0          42m
core5g-mongodb-5c8964b55d-7wmnc           1/1     Running   0          42m
core5g-nrf-deployment-65db5975b5-b79jm    2/2     Running   0          42m
core5g-nssf-deployment-7956d984d4-67sdz   2/2     Running   0          42m
core5g-pcf-deployment-d76fcd97-bq9hv      2/2     Running   0          42m
core5g-scp-deployment-cdb486c95-x6tml     2/2     Running   0          42m
core5g-smf-deployment-8474b579df-5fg5g    2/2     Running   0          42m
core5g-udm-deployment-848c5bf848-xx2jc    2/2     Running   0          42m
core5g-udr-deployment-5797cc596f-g2rmw    2/2     Running   0          42m
core5g-ue-import-8556598584-k4gt4         1/1     Running   0          42m
core5g-ueransim-gnb-7bcd5cfd6c-jbnqg      1/1     Running   0          42m
core5g-ueransim-ue-78ddfb8874-sf5pm       1/1     Running   0          42m
core5g-upf-deployment-696f8c6597-22kkp    2/2     Running   0          42m
core5g-webui-b44ddc56-rvm7d               1/1     Running   0          42m

```

Check SmallStep autocert side-car logs:

```
kubectl -n open5gs logs deploy/core5g-amf-deployment -c autocert-renewer
INFO: 2023/04/17 16:26:06 first renewal in 19m21s
INFO: 2023/04/17 16:45:27 core5g-amf.open5gs.svc.cluster.local certificate renewed, next in 18m36s
INFO: 2023/04/17 17:04:03 core5g-amf.open5gs.svc.cluster.local certificate renewed, next in 18m51s
```

Test certificate from the AMF to the NRF:

```
kubectl -n open5gs exec -ti deploy/core5g-amf-deployment -c amf bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@core5g-amf-deployment-544667f9c7-z4bj6:/# curl -v --http2-prior-knowledge \
--cacert /var/run/autocert.step.sm/root.crt \
--cert /var/run/autocert.step.sm/site.crt \
--key /var/run/autocert.step.sm/site.key \
https://core5g-amf.open5gs.svc.cluster.local/nnrf-nfm/v1/nf-instances
*   Trying 10.96.120.203:443...
* Connected to core5g-amf.open5gs.svc.cluster.local (10.96.120.203) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
*  CAfile: /var/run/autocert.step.sm/root.crt
*  CApath: /etc/ssl/certs
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (OUT), TLS header, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Finished (20):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.3 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.3 (OUT), TLS handshake, CERT verify (15):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=core5g-amf.open5gs.svc.cluster.local
*  start date: Apr 17 16:25:05 2023 GMT
*  expire date: Apr 17 16:56:05 2023 GMT
*  subjectAltName: host "core5g-amf.open5gs.svc.cluster.local" matched cert's "core5g-amf.open5gs.svc.cluster.local"
*  issuer: O=Step Certificates; CN=Step Certificates Intermediate CA
*  SSL certificate verify ok.
```

