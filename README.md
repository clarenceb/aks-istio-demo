# AKS Istio Demo

## Pre-requisites

* Install the [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
* Login to your azure subscription:

```sh
az login
```

* Install AKS CLI tools (`kubectl`, `kubelogin`):

```sh
az aks install-cli
# or to a specific directory in yur $PATH (e.g. $HOME/.tools/)
az aks install-cli --install-location $HOME/.tools/kubectl --kubelogin-install-location $HOME/.tools/kubelogin
```

Create AKS cluster with Istio add-on enabled
--------------------------------------------

```sh
# Update aks-istio-env.sh with youre values.
source ./aks-istio-env.sh

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

# If using Entra ID auth on AKS
kubelogin convert-kubeconfig -l azurecli

kubectl get pods -n aks-istio-system
```

(Optional) Enable Istio add-on on an existing cluster
-----------------------------------------------------

```sh
az aks get-credentials --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}

# If using Entra ID auth on AKS
kubelogin convert-kubeconfig -l azurecli

# Enable the istio mesh add-on
az aks mesh enable --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}
REVISION=$(az aks show --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}  --query 'serviceMeshProfile.istio.revisions[0]' -o tsv)

# Or install a specific version for your AKS version

# See available service mesh versions
az aks mesh get-revisions --location $LOCATION -o table
REVISION="asm-1-24"
az aks mesh enable --resource-group ${RESOURCE_GROUP} --name ${CLUSTER} --revision ${REVISION}

az aks show --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}  --query 'serviceMeshProfile.mode'
```

## (Optional) Install `istioctl` CLI tool

```sh
ISTIO_VERSION="$(kubectl get deploy istiod-${REVISION} -n aks-istio-system -o yaml | grep image: | egrep -o '[0-9]+\.[0-9]+\.[0-9]+')"

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=$ISTIO_VERSION TARGET_ARCH=x86_64 sh -
sudo cp  "istio-${ISTIO_VERSION}/bin/istioctl" /usr/local/bin
rm -rf "./istio-${ISTIO_VERSION}/"

istioctl -i aks-istio-system version
```

## Enable external Ingress Gateway

```sh
az aks mesh enable-ingress-gateway --resource-group $RESOURCE_GROUP --name $CLUSTER --ingress-gateway-type external
kubectl get svc aks-istio-ingressgateway-external -n aks-istio-ingress
```

## Configure Secure Istio Ingress Gateway

Turn TLS on on our Ingress Gateway.

### Obtain/Generate TLS certificate and private key

You'll need a TLS certificate and private key.
You can generate a self-signed one like described here **[Secure ingress gateway for Istio service mesh add-on for Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/aks/istio-secure-gateway#required-clientserver-certificates-and-keys)**
or follow a more detailed setup to get a free TLS cert from Let's Encrypt: **[Deploy cert-manager on Azure Kubernetes Service (AKS) and use Let's Encrypt to sign a certificate for an HTTPS website](https://cert-manager.io/docs/tutorials/getting-started-aks-letsencrypt/)**

The rest of this demo assumes a wildcard TLS cert and private key named:

- `istio-demo.crt`
- `istio-demo.key`

and your wildcard domain in `$DOMAIN`

```sh
# Change to your domain
DOMAIN=apps.example.com
```

### Set up Azure Key Vault and sync secrets to the cluster

```sh
export AKV_NAME=istio-demo-kv  
az keyvault create --name $AKV_NAME --resource-group $RESOURCE_GROUP --location $LOCATION

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
CURRENT_USER=$(az ad signed-in-user show --query id -o tsv)

az aks enable-addons --addons azure-keyvault-secrets-provider --resource-group $RESOURCE_GROUP --name $CLUSTER

AKV_SECRET_PROVIDER_CLIENT_ID=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query 'addonProfiles.azureKeyvaultSecretsProvider.identity.clientId' -o tsv)
TENANT_ID=$(az keyvault show --resource-group $RESOURCE_GROUP --name $AKV_NAME --query 'properties.tenantId' -o tsv)

az role assignment create --role "Key Vault Secrets User" --assignee ${AKV_SECRET_PROVIDER_CLIENT_ID} --scope /subscriptions/${SUBSCRIPTION_ID}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${AKV_NAME}
az role assignment create --role "Key Vault Secrets Officier" --assignee ${CURRENT_USER} --scope /subscriptions/${SUBSCRIPTION_ID}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${AKV_NAME}

TLS_CERT_FILE=istio-demo.crt
TLS_KEY_FILE=istio-demo.key

az keyvault secret set --vault-name $AKV_NAME --name istio-demo-key --file ./${TLS_KEY_FILE}
az keyvault secret set --vault-name $AKV_NAME --name istio-demo-crt --file ./${TLS_CERT_FILE}

# Define secret provider class to fetch Azure Key Vault Secrets
cat <<EOF > secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: istiodemo-credential-spc
  namespace: aks-istio-ingress
spec:
  provider: azure
  secretObjects:
  - secretName: istiodemo-credential
    type: kubernetes.io/tls
    data:
    - objectName: istio-demo-key
      key: tls.key
    - objectName: istio-demo-crt
      key: tls.crt
  parameters:
    useVMManagedIdentity: "true"
    userAssignedIdentityID: $AKV_SECRET_PROVIDER_CLIENT_ID 
    keyvaultName: $AKV_NAME
    cloudName: ""
    objects:  |
      array:
        - |
          objectName: istio-demo-key
          objectType: secret
          objectAlias: "istio-demo-key"
        - |
          objectName: istio-demo-crt
          objectType: secret
          objectAlias: "istio-demo-crt"
    tenantId: $TENANT_ID
EOF

kubectl apply -f secret-provider-class.yaml

cat <<EOF > secrets-store-sync-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secrets-store-sync-istiodemo
  namespace: aks-istio-ingress
spec:
  containers:
    - name: busybox
      image: mcr.microsoft.com/oss/busybox/busybox:1.33.1
      command:
        - "/bin/sleep"
        - "10"
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "istiodemo-credential-spc"
EOF

kubectl apply -f secrets-store-sync-pod.yaml
```

### (Optional) Create TLS secret directly in Kubernetes

If you don't want top set up Azure Key Vault and sync the secret you can just import it directly:

```sh
cat <<EOF > istio-ingress-tls-secret.yaml
apiVersion: v1
data:
  tls.crt: $(cat istio-demo.crt | base64 -w 0)
  tls.key: $(cat istio-demo.key | base64 -w 0)
kind: Secret
metadata:
  name: istiodemo-credential
  namespace: aks-istio-ingress
type: kubernetes.io/tls
EOF

kubectl apply -f istio-ingress-tls-secret.yaml
```

### Create TLS Istio Gateway and Virtual Service for your domain

```sh
kubectl create ns bookinfo

cat <<EOF > bookinfo-ingress.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: istiodemo-credential
    hosts:
    - productpage.$DOMAIN
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: productpage-vs
  namespace: bookinfo
spec:
  hosts:
  - productpage.$DOMAIN
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        port:
          number: 9080
        host: productpage
EOF

kubectl apply -f bookinfo-ingress.yaml
```

## Deploy Bookinfo sample application

```sh
kubectl create ns bookinfo
kubectl label namespace bookinfo istio.io/rev=${REVISION}

ISTIO_RELEASE=$(echo $ISTIO_VERSION | cut -f1,2 -d .)

kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/destination-rule-all.yaml -n bookinfo
kubectl get services -n bookinfo
kubectl get pods -n bookinfo

kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}' -n bookinfo)" -n bookinfo -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

# If not using TLS ingress
kubectl apply -f bookinfo-vs-external.yaml -n bookinfo
kubectl apply -f bookinfo-gateway-external.yaml -n bookinfo

export INGRESS_HOST_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL_EXTERNAL=$INGRESS_HOST_EXTERNAL:$INGRESS_PORT_EXTERNAL
PRODUCT_PAGE="http://${GATEWAY_URL_EXTERNAL}/productpage"
echo $PRODUCT_PAGE

# otherwise, if using TLS ingress
kubectl apply -f bookinfo-ingress.yaml
PRODUCT_PAGE="https://productpage.$DOMAIN/productpage"
echo $PRODUCT_PAGE

curl -s $PRODUCT_PAGE | grep -o "<title>.*</title>"
```

## Deploy observability add-ons

```sh
# Prometheus - metrics
curl -s https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/addons/prometheus.yaml | sed 's/istio-system/aks-istio-system/g' | kubectl apply -f -

# Grafana - monitoring and metrics dashboards
curl -s https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/addons/grafana.yaml | sed 's/istio-system/aks-istio-system/g' | kubectl apply -f -

# Jaeger - distributed tracing
curl -s https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/addons/jaeger.yaml | sed 's/istio-system/aks-istio-system/g' | kubectl apply -f -

# Kiali installation
helm repo add kiali https://kiali.org/helm-charts
helm repo update

helm install \
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

## Generate some app traffic to observe in Kiali UI

```sh
for i in $(seq 1 100); do curl -o /dev/null -s -w "Request: ${i}, Response: %{http_code}\n" "$PRODUCT_PAGE"; done
```

Browse to Kiali UI: [http://localhost:20001](http://localhost:20001)

## View Prometheus Metrics

```sh
kubectl port-forward -n aks-istio-system svc/prometheus 9090:9090
```

Browse to Prometheus UI: [http://localhost:9090](http://localhost:9090)

View the total Istio requests metric:

```promql
sum(istio_requests_total)
```

[Prometheus metrics shortcut link](http://localhost:9090/graph?g0.expr=sum(istio_requests_total)&g0.tab=0&g0.stacked=0&g0.show_exemplars=0&g0.range_input=15m)

## Setup up Grafana

```sh
kubectl port-forward -n aks-istio-system svc/grafana 3000:3000
```

Browse to Grafana: [http://localhost:3000](http://localhost:3000)

Add a datasource for Prometheus:

* Name: prometheus-istio
* URL: http://prometheus.aks-istio-system.svc.cluster.local:9090
* Set this datasource as the default
* Save and Test

The Istio dashboards should already be loaded into Grafana.

Choose one of the dashboards to view.

## View distributed traces in Jaeger

```sh
JAEGER_POD=$(kubectl get pods -n aks-istio-system --no-headers  --selector app=jaeger | awk 'NR==1{print $1}')
kubectl port-forward -n aks-istio-system $JAEGER_POD 16686:16686

# Generate some traffic
for i in $(seq 1 100); do curl -o /dev/null -s -w "Request: ${i}, Response: %{http_code}\n" "$PRODUCT_PAGE"; done
```

Browse to Jaeger UI: [http://localhost:16686](http://localhost:16686)

* Select service `productpage.bookinfo`
* Click **Find Traces**
* Explore one of the traces
* Select **Trace Graph** from the top right drop down
* Click **Search** at the top left menu bar
* Click **Deep dependency graph**
* Click **System Architecture** / **DAG**

## Request Routing

Requests by default will cycle through all versions.

```sh
# Route all requests to v1 of each service
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-all-v1.yaml -n bookinfo

# Route all traffic from a user named Jason the service reviews:v2
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml -n bookinfo
```

Log in as user: `jason`, password: `jason`

* Refresh product page to see black stars (review:v2)
* You can see user "`jason`" in the session cookie by decoding the JWT token at [https://jwt.io/](https://jwt.io/)

Logout user

* Refresh product page to see no stars (review:v1)

## Traffic Shifting

Shift traffic from reviews:v1 to reviews:v3

```sh
# 50% split v1 and v3
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml -n bookinfo

# 100% to v3
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-reviews-v3.yaml -n bookinfo
```

## Fault Injection

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-all-v1.yaml -n bookinfo
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml -n bookinfo
```

With the above configuration, this is how requests flow:

* productpage → reviews:v2 → ratings (only for user jason)
* productpage → reviews:v1 (for everyone else)

### Inject HTTP delay fault

```sh
# Inject a 7s delay between the reviews:v2 and ratings microservices only for user 'jason'
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml -n bookinfo
```

* Refresh the product page as user 'json'
* Error "Sorry, product reviews are currently unavailable for this book." is displayed as ratings delay of 7 sec exceed reviews:v2 timeout of 6 seconds

```sh
kubectl apply -f ./virtual-service-ratings-test-delay-2s.yaml -n bookinfo
```

* Refresh the product page as user 'json'
* Product page now loads as the delay is less than the product page timeout

### Inject HTTP abort fault

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-all-v1.yaml -n bookinfo
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml -n bookinfo

kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml -n bookinfo
```

* Log in as user 'json'
* Refresh product page
* "Ratings service is currently unavailable" is shown for rating
* Logout
* Refresh product page
* Ratings appear again
* Run some requests as a logged out user:

```sh
for i in $(seq 1 100); do curl -o /dev/null -s -w "Request: ${i}, Response: %{http_code}\n" "${PRODUCT_PAGE}"; done
```

* Show reviews:v1 and reviews:v2 in Kiali graph to see v1 succeeding and v2 failing

## L7 traffic authorization policies

```sh
# Round-robin reviews v1/v2/v3
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-all-v1.yaml -n bookinfo

# Deny all traffic in the mesh
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-nothing
  namespace: bookinfo
spec:
  {}
EOF
```

* Refresh productpage: "RBAC: access denied"

```sh
# Allow access to product page for all clients
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "productpage-viewer"
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - to:
    - operation:
        methods: ["GET"]
EOF
```

* Refresh productpage: page loads but product details and reviews fail

```sh
# Allow product page to fetch product details
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "details-viewer"
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
EOF

# Allow product page to fetch reviews
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "reviews-viewer"
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
EOF

# Allow reviews to fetch ratings
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "ratings-viewer"
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-reviews"]
    to:
    - operation:
        methods: ["GET"]
EOF
```

## Reset app state

```sh
kubectl delete authorizationpolicy.security.istio.io/allow-nothing -n bookinfo
kubectl delete authorizationpolicy.security.istio.io/productpage-viewer -n bookinfo
kubectl delete authorizationpolicy.security.istio.io/details-viewer -n bookinfo
kubectl delete authorizationpolicy.security.istio.io/reviews-viewer -n bookinfo
kubectl delete authorizationpolicy.security.istio.io/ratings-viewer -n bookinfo

kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-all-v1.yaml -n bookinfo
```

## Cleanup

```sh
kubectl delete AuthorizationPolicy allow-nothing -n bookinfo
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/virtual-service-all-v1.yaml -n bookinfo
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/networking/destination-rule-all.yaml -n bookinfo
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-${ISTIO_RELEASE}/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
kubectl delete ns bookinfo

az aks mesh disable --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}
kubectl delete crd $(kubectl get crd -A | grep "istio.io" | awk '{print $1}')

az group delete --name ${RESOURCE_GROUP} --yes --no-wait
```

## Resources

* [Istio docs](https://istio.io/latest/docs/)

## TODO

* Switch Grafana and Prometheus to Azure managed versions, see [Istio Service Mesh AKS Add-on](https://www.youtube.com/watch?v=CifKWSnX8C8) on YouTube at 39:39 min:sec mark or [Collect metrics for Istio service mesh add-on workloads for Azure Kubernetes Service in Azure Managed Prometheus](https://learn.microsoft.com/en-us/azure/aks/istio-metrics-managed-prometheus)
* Egress rules with REGISTRY_ONLY outbound policy and service entries
