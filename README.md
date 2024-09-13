# galb-nginx-ca-sample-01
Sample of integrating gcloud managed ALB (provisioned via Gateway API) with nginx proxies and referencing Cloud Armor security policies.

### make sure your GKE cluster has Gateway API enabled on it

```
# check for gatewayClass
kubectl get gatewayclass

# sample output
#$ kubectl get gatewayclass
#NAME                                  CONTROLLER                    ACCEPTED   AGE
#gke-l7-global-external-managed        networking.gke.io/gateway     True       274d
#gke-l7-global-external-managed-mc     networking.gke.io/gateway     True       274d
#gke-l7-gxlb                           networking.gke.io/gateway     True       274d
#gke-l7-gxlb-mc                        networking.gke.io/gateway     True       274d
#gke-l7-regional-external-managed      networking.gke.io/gateway     True       274d
#gke-l7-regional-external-managed-mc   networking.gke.io/gateway     True       274d
#gke-l7-rilb                           networking.gke.io/gateway     True       274d
#gke-l7-rilb-mc                        networking.gke.io/gateway     True       274d

# if no gatewayClasses are found, you need to enable it for your cluster - follow commands at https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#enable-gateway-existing-cluster
```

### set up sample app whereami

```
kubectl create namespace whereami-nginx-demo
kubectl apply -k whereami-nginx-demo/variant
```

### set up nginx ingress controller

```
# going to apply a modified version of the nginx controller that sets service type to ClusterIP instead of LoadBalancer default and omits externalTrafficPolicy
kubectl apply -f nginx/
```

### create dummy self-signed cert to use for this demo

```
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
 -subj "/CN=whereami.example.com/O=Edge2Mesh Inc" \
 -keyout whereami.example.com.key \
 -out whereami.example.com.crt

kubectl -n whereami-nginx-demo create secret tls whereami-nginx-demo \
 --key=whereami.example.com.key \
 --cert=whereami.example.com.crt

# put the cert in ingress-nginx as well for gateway config
kubectl -n ingress-nginx create secret tls whereami-nginx-demo \
 --key=whereami.example.com.key \
 --cert=whereami.example.com.crt
```
### create ingress spec for nginx

```
kubectl apply -f ingress/
```

### 