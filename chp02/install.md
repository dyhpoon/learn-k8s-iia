# requirements
- kubernetes v1.13.0
- istio 1.0.5

# install
```
minikube start --memory=4096 --disk-size=30g --kubernetes-version="v1.10.0"
k apply -f install/kubernetes/istio-demo.yaml
```

# check if install properly
```
# queried the Istio control plane and asked for the overall mesh configuration and the endpoints
kubectl run -i --rm --restart=Never dummy --image=tutum/curl:alpine \
-n istio-system --command \
-- curl -v 'http://istio-pilot.istio-system:8080/v1/registration'
```

# deploy our first example
```
kubectl create namespace istioinaction
kubectl config set-context $(kubectl config current-context) \
 --namespace=istioinaction

istioctl kube-inject -f ./catalog-service/catalog-deployment.yaml

kubectl create -f <(istioctl kube-inject -f ./catalog-service/catalog-all.yaml)
```

# query our istio's service
```
kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty --command -- sh -c 'curl -s catalog:8080/api/catalog'
```

# deploy Gateway API service
```
kubectl create -f <(istioctl kube-inject -f ./apigateway-service/apigateway-all.yaml)
```

# invoke our gateway service
```
kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty --command -- sh -c 'curl -s apigateway:8080/api/products'
```

# create an ingress to receive traffic
```
kubectl create -f ./ingress/ingress-gateway.yaml
```

# curl our service through ingress
```
URL=$(minikube ip):$(kubectl -n istio-system get service istio-ingressgateway \
 -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
curl $URL/api/products
```

# if anything goes wrong, make sure the gateway properly has a route setup to our api gateway
# we can use istio debugging tool to check
```
istioctl -n istio-system proxy-config routes $(kubectl get pod -n istio-system | grep ingress | cut -d ' ' -f 1)
```
# or check gateway & virtualService
```
kubectl get gateway
kubectl get virtualservice
```

# check our grafana
```
GRAFANA=$(kubectl -n istio-system get pod | grep -i running | grep grafana | cut -d ' ' -f 1)
kubectl port-forward -n istio-system $GRAFANA 8080:3000

while true; do curl $URL/api/products; sleep .5; done
```

# check our openTracing (Jaegar)
```
TRACING=$(kubectl -n istio-system get pod | grep istio-tracing | cut -d ' ' -f 1)
kubectl port-forward -n istio-system $TRACING 8181:16686
```

# simulate network failure, the `catalog-spring-boot` service checks aginst the header and simulate the failure
```
curl $URL/api/products -H "failure-percentage: 100"
```

# simulate a retry mechanism in istio
```
while true; do curl $URL/api/products -H "failure-percentage: 50"; sleep .5; done
```
# and you should see intermittent success and failures from apigateway

# lets create a virtualService and adding retries to it
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
    retries:
      attempts: 3
      perTryTimeout: 2s
```
```
kubectl create -f <(istioctl kube-inject -f ./catalog-service/catalog-svc-v2.yaml)
```

# simulate intermittent failures again, you should see a much stable network connections
```
while true; do curl $URL/api/products -H "failure-percentage: 50"; sleep .5; done
```
