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
--generate-ssh-keys

az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}
az aks show --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}  --query 'serviceMeshProfile.mode'
```

Install `istioctl` CLI tool
-------------------------

```sh
ISTIO_VERSION=1.17.2

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=$ISTIO_VERSION TARGET_ARCH=x86_64 sh -
sudo cp  istio-1.17.2/bin/istioctl /usr/local/bin
rm -rf ./istio-1.17.2/

istioctl -i aks-istio-system version
kubectl get pods -n aks-istio-system
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
kuebctl create ns bookinfo
kubectl label namespace bookinfo istio.io/rev=asm-1-17
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.17/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
kubectl get services -n bookinfo
kubectl get pods -n bookinfo

export INGRESS_HOST_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL_EXTERNAL=$INGRESS_HOST_EXTERNAL:$INGRESS_PORT_EXTERNAL

echo "http://$GATEWAY_URL_EXTERNAL/productpage"
curl -s "http://${GATEWAY_URL_EXTERNAL}/productpage" | grep -o "<title>.*</title>"
```

Deploy observability add-ons
----------------------------

```sh
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
