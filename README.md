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
# going to apply a modified version of the nginx controller that sets service type to ClusterIP instead of LoadBalancer default, sets appProtocol to HTTP2 (instead of HTTPS), and omits externalTrafficPolicy
# also, added use-http2: "true" to the nginx controller configMap to allow for HTTP2
# finally, disabled use-proxy-protocol: "true" via the same configMap in nginx/controller-v1.11.2-deploy.yaml
kubectl apply -f nginx/
```

### create dummy self-signed cert to use for this demo

```
# because this is self signed, will need to use -K with curl - also initially tried this with size of 4096 but got size errors 
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 \
 -subj "/CN=whereami.example.com/O=Edge2Mesh Inc" \
 -addext "subjectAltName = DNS:whereami.example.com" \
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

### create example Cloud Armor policy

```
gcloud compute security-policies create edge-fw-policy \
  --description "Block XSS attacks"

gcloud compute security-policies rules create 1000 \
    --security-policy edge-fw-policy \
    --expression "evaluatePreconfiguredExpr('xss-stable')" \
    --action "deny-403" \
    --description "XSS attack filtering"

kubectl apply -f cloud-armor/
```

### set up gateway resources

```
# reserve static IP - this way if your LB gets deleted you can recreate a new one with the same public IP address
gcloud compute addresses create whereami-nginx-demo --global # --project=$PROJECT_ID # if needed

kubectl apply -f gateway/
kubectl apply -f httproute/
```

### test endpoint 

```
# get IP address of load balancer
GALB_IP="$(gcloud compute addresses describe whereami-nginx-demo --global --format='value(address)')"

curl -k --header 'Host: whereami.example.com' https://$GALB_IP

# sample output
#$ curl -k --header 'Host: whereami.example.com' https://$GALB_IP -s | jq 
#{
#  "cluster_name": "gateway-nginx-test",
#  "gce_instance_id": "6857840697694670280",
#  "gce_service_account": "spanner-reporter-01.svc.id.goog",
#  "host_header": "whereami.example.com",
#  "metadata": "whereami-nginx-demo",
#  "node_name": "gk3-gateway-nginx-test-pool-2-b6d0d56f-z8l6",
#  "pod_ip": "10.92.128.12",
#  "pod_name": "whereami-nginx-demo-5459d7c85d-t7m2d",
#  "pod_name_emoji": "🗑️",
#  "pod_namespace": "whereami-nginx-demo",
#  "pod_service_account": "whereami-nginx-demo",
#  "project_id": "spanner-reporter-01",
#  "timestamp": "2024-09-13T04:03:15",
#  "zone": "us-central1-f"
#}
```