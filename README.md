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

# Kiali installation
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

# Port forward to Istio service to access on http://localhost:20001
kubectl port-forward svc/kiali 20001:20001 -n aks-istio-system
```

Generate some app traffic to observe in Kiali UI
------------------------------------------------

```sh
for i in $(seq 1 100); do curl -o /dev/null -s -w "Request: ${i}, Response: %{http_code}\n" "http://$GATEWAY_URL_EXTERNAL/productpage"; done
```

Browse to Kiali UI: [http://localhost:20001](http://localhost:20001)

View Prometheus Metrics
-----------------------

```sh
kubectl port-forward -n aks-istio-system svc/prometheus 9090:9090
```

Browse to Prometheus UI: [http://localhost:9090](http://localhost:9090)

View the total Istio requests metric:

```promql
sum(istio_requests_total)
```

[Prometheus metrics shortcut link](http://localhost:9090/graph?g0.expr=sum(istio_requests_total)&g0.tab=0&g0.stacked=0&g0.show_exemplars=0&g0.range_input=15m)

Setup up Grafana
----------------

```sh
kubectl port-forward -n aks-istio-system svc/grafana 3000:3000
```

Browse to Grafana: [http://localhost:3000](http://localhost:3000)

Add a datasource for Prometheus:

* URL: http://prometheus.aks-istio-system.svc.cluster.local:9090
* Set this datasource as the default
* Save and Test

The Istio dashboards should already be loaded into Grafana.

Choose one of the dashboards to view.

View distributed traces in Jaeger
---------------------------------

```sh
JAEGER_POD=$(kubectl get pods -n aks-istio-system --no-headers  --selector app=jaeger | awk 'NR==1{print $1}')
kubectl port-forward -n jaeger $JAEGER_POD  16686:16686
```

Browse to Jaeger UI: [http://localhost:16686](http://localhost:16686)

* Select service `productpage.bookinfo`
* Click **Find Traces**
* Explore one of the traces
* Select **Trace Graph** from the top right drop down
* Click **Search** at the top left menu bar
* Click **Deep dependency graph**
* Click **System Architecture** / **DAG**

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

* 

TODO
----

* Switch Grafana and Prometheus to Azure managed versions, see [Istio Service Mesh AKS Add-on](https://www.youtube.com/watch?v=CifKWSnX8C8) on YouTube at 39:39 min:sec mark
