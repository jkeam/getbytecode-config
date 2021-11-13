# GetByteCode Config
Using Knative for all deployments because:

1.  It's easier to deploy
2.  I can scale to 0
3.  I can learn the intricacies of Knative

## Knative

### Installation

#### Prerequisites
1.  Monitoring Stack configured, specifically with Prometheus
2.  Default Ingress configured for the cluster

#### Installation via YAML
```
# install crds
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.0.0/serving-crds.yaml

# core serving
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.0.0/serving-core.yaml

# install networking layer
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.0.0/kourier.yaml
kubectl patch configmap/config-network --namespace knative-serving --type merge --patch '{"data":{"ingress.class":"kourier.ingress.networking.knative.dev"}}'
# or edit
# edit url created
kubectl edit configmap config-domain -n knative-serving

# fetch external IP
kubectl --namespace kourier-system get service kourier
```

#### Operator Based Install
```
# install operator
kubectl apply -f https://github.com/knative/operator/releases/download/knative-v1.0.0/operator.yaml

# verify operator installation
kubectl config set-context --current --namespace=default  # set namespace
kubectl get deployment knative-operator  # verify
kubectl logs -f deploy/knative-operator

# create namespace
kubectl apply -f ./knative/1-serving-namespace.yaml

# create serving
kubectl apply -f ./knative/2-serving.yaml

# verify
kubectl get deployment -n knative-serving
# see if the knative service object is ready, look for True
kubectl get KnativeServing knative-serving -n knative-serving

# fetch ip or cname for dns later
kubectl --namespace knative-serving get service kourier

# change default url used by knative (look at ./knative/3-serving-config-map.yaml as an example)
kubectl edit configmap config-domain -n knative-serving
kubectl edit configmap/config-network -n knative-serving
  # replace _example with getbytecodeapi.com: ""

# will be skipping eventing
# kubectl apply -f ./knative/eventing.yaml
```

### TLS Certs

```
# install cert manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.0/cert-manager.yaml

# install knative certmanager connector
kubectl apply --filename https://github.com/knative/net-certmanager/releases/download/knative-v1.0.0/release.yaml

# must install 
kubectl apply -f ./secrets/digital-ocean-api-token.yaml
kubectl apply -f ./knative/4-serving-tls.yaml
```

Edit the configs:

`kubectl get configmap config-certmanager --namespace knative-serving --output yaml`

```
data:
  issuerRef: |
    kind: ClusterIssuer
    name: letsencrypt-dns-issuer
  getbytecodeapi.com: |
    issuerRef: |
      kind: ClusterIssuer
      name: letsencrypt-dns-issuer
```

`kubectl edit configmap config-network --namespace knative-serving`

```
data:
  autoTLS: "Enabled"
  httpProtocol: "Redirected"
  getbytecodeapi.com: |
    autoTLS: "Enabled"
    http-protocol: "Redirected"
    default-external-scheme: "https"
```


### Teardown
```
# uninstall serving
kubectl delete KnativeServing knative-serving -n knative-serving

# uninstall eventing
# kubectl delete KnativeEventing knative-eventing -n knative-eventing

# uninstall operator
kubectl delete -f https://github.com/knative/operator/releases/download/knative-v1.0.0/operator.yaml
```


### Example App
```
# set namesapce
kubectl create namespace jon
kubectl config set-context --current --namespace=jon

kn service create hello --image gcr.io/knative-samples/helloworld-go --env TARGET=Knative

kn service list

kn service update hello --env TARGET=Kn

kn service describe hello

kn service delete hello

kubectl delete secrets
kubectl delete namespace jon
```

`service update` or `service create` flags for scaling.

```
Flag    Description
--concurrency-limit int
  Hard limit of concurrent requests to be processed by a single replica.

--concurrency-target int
  Recommendation for when to scale up based on the concurrent number of incoming requests. Defaults to --concurrency-limit.

--max-scale int Maximum number of replicas.
--min-scale int Minimum number of replicas.
```

### Things I learned
1.  If an endpoint is unavailable, like an internal URL for a webhook, and I'm tailing the logs of the endpoint/controller and nothing happens -> Just kill the pod and usually it'll start working again.

2.  Prerequisites not mentioned are the Monitoring Stack (Prometheus specifically, used for autoscaling) and a default ingress for the cluster


## Deploy Apps

```
kubectl config set-context --current --namespace=diss

# specific one
kubectl apply -k ./apps/overlays/ruby/ruby25

# all ruby
kubectl apply -k ./apps/overlays/ruby

# all languages
kubectl apply -k ./apps/overlays
```
