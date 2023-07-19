AKS Istio Demo
==============

Create AKS cluster with Istio add-on enabled
--------------------------------------------

```sh
source ./aks-istio-env.sh

az extension add --name aks-preview
az extension update --name aks-preview

az feature register --namespace "Microsoft.ContainerService" --name "AzureServiceMeshPreview"
az feature show --namespace "Microsoft.ContainerService" --name "AzureServiceMeshPreview"

az provider register --namespace Microsoft.ContainerService

az group create --name ${RESOURCE_GROUP} --location ${LOCATION}

az aks create \
--resource-group ${RESOURCE_GROUP} \
--name ${CLUSTER} \
--enable-asm \
--network-plugin azure \
--node-count 3 \
--kubernetes-version $K8S_VERSION \
--generate-ssh-keys

az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}
az aks show --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}  --query 'serviceMeshProfile.mode'
kubectl get pods -n aks-istio-system
```

Install `istioctl` CLI tool
---------------------------

```sh
ISTIO_VERSION="$(kubectl get deploy istiod-asm-1-17 -n aks-istio-system -o yaml | grep image: | egrep -o '[0-9]+\.[0-9]+\.[0-9]+')"

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=$ISTIO_VERSION TARGET_ARCH=x86_64 sh -
sudo cp  "istio-${ISTIO_VERSION}/bin/istioctl" /usr/local/bin
rm -rf "./istio-${ISTIO_VERSION}/"

istioctl -i aks-istio-system version
```

Enable external Ingress Gateway
-------------------------------

```sh
az aks mesh enable-ingress-gateway --resource-group $RESOURCE_GROUP --name $CLUSTER --ingress-gateway-type external
kubectl get svc aks-istio-ingressgateway-external -n aks-istio-ingress
```

Deploy Bookinfo sample application
----------------------------------

```sh
kubectl create ns bookinfo
kubectl label namespace bookinfo istio.io/rev=asm-1-17
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.17/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
kubectl get services -n bookinfo
kubectl get pods -n bookinfo

kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}' -n bookinfo)" -n bookinfo -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

kubectl apply -f bookinfo-vs-external.yaml -n bookinfo
kubectl apply -f bookinfo-gateway-external.yaml -n bookinfo

export INGRESS_HOST_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL_EXTERNAL=$INGRESS_HOST_EXTERNAL:$INGRESS_PORT_EXTERNAL

echo "http://$GATEWAY_URL_EXTERNAL/productpage"
curl -s "http://${GATEWAY_URL_EXTERNAL}/productpage" | grep -o "<title>.*</title>"
```

Deploy observability add-ons
----------------------------

```sh
# Prometheus - metrics
curl -s https://raw.githubusercontent.com/istio/istio/release-1.17/samples/addons/prometheus.yaml | sed 's/istio-system/aks-istio-system/g' | kubectl apply -f -

# Grafana - monitoring and metrics dashboards
curl -s https://raw.githubusercontent.com/istio/istio/release-1.17/samples/addons/grafana.yaml | sed 's/istio-system/aks-istio-system/g' | kubectl apply -f -

# Jaeger - distributed tracing
curl -s https://raw.githubusercontent.com/istio/istio/release-1.17/samples/addons/jaeger.yaml | sed 's/istio-system/aks-istio-system/g' | kubectl apply -f -

helm install \
    --version=1.63.1 \
    --set cr.create=true \
    --set cr.namespace=aks-istio-system \
    --namespace aks-istio-system \
    --create-namespace \
    kiali-operator \
    kiali/kiali-operator

# Generate a short-lived token to login to Kiali UI
kubectl -n aks-istio-system create token kiali-service-account
```

Cleanup
-------

```sh
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.17/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
kubectl delete ns bookinfo

az aks mesh disable --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}
kubectl delete crd $(kubectl get crd -A | grep "istio.io" | awk '{print $1}')

az group delete --name ${RESOURCE_GROUP} --yes --no-wait
```

Resources
---------

TODO
----

- Switch Grafana and Prometheus to Azure managed PaaS versions
