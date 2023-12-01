# SPIFFE demo

## minikube

See <https://minikube.sigs.k8s.io>.

```sh
minikube start
minikube tunnel

# check if everything is up and running
minikube status
kubectl cluster-info
```

## Gateway API

See <https://gateway-api.sigs.k8s.io>.

```sh
# deploy Gateway API CRDs
gateway_api_version="v1.0.0"
gateway_api_channel="standard" # or experimental
gateway_api_crds_url="https://github.com/kubernetes-sigs/gateway-api/releases"
gateway_api_crds_url="${gateway_api_crds_url}/download/${gateway_api_version}"
gateway_api_crds_url="${gateway_api_crds_url}/${gateway_api_channel}"
gateway_api_crds_url="${gateway_api_crds_url}-install.yaml"

kubectl apply --filename "${gateway_api_crds_url}"

# check if Gateway API CRDs have been established
kubectl wait --for condition=established \
  crd/gateways.gateway.networking.k8s.io
kubectl wait --for condition=established \
  crd/httproutes.gateway.networking.k8s.io
```

## SPIRE

See <https://spiffe.io>.

```sh
helm repo add spiffe https://spiffe.github.io/helm-charts-hardened/

# deploy SPIRE CRDs
# https://artifacthub.io/packages/helm/spiffe/spire-crds
helm upgrade spire-crds spiffe/spire-crds \
  --install \
  --version 0.2.0 \
  --namespace spire \
  --create-namespace \
  --atomic

# check if SPIRE CRDs have been established
kubectl wait --for condition=established \
  crd/clusterfederatedtrustdomains.spire.spiffe.io
kubectl wait --for condition=established \
  crd/clusterspiffeids.spire.spiffe.io
kubectl wait --for condition=established \
  crd/clusterstaticentries.spire.spiffe.io
kubectl wait --for condition=established \
  crd/controllermanagerconfigs.spire.spiffe.io

# deploy SPIRE
# https://artifacthub.io/packages/helm/spiffe/spire
helm upgrade spire spiffe/spire \
  --install \
  --version 0.16.0 \
  --namespace spire \
  --create-namespace \
  --atomic \
  --values spire.yaml

# you may need this quite often
alias spire-server='kubectl exec spire-server-0 \
  --namespace spire \
  --container spire-server \
  -- \
  /opt/spire/bin/spire-server'

# check if SPIFFE workload registration entries exist
spire-server entry count
spire-server entry show
```

## Istio

See <https://istio.io>.

```sh
# deploy Istio
# <https://istio.io/latest/docs/setup/install/istioctl/>
istioctl install -f istio.yaml

istioctl dashboard controlz deployment/istiod.istio-system

# deploy Istio add-ons
istio_version="release-1.20"
istio_url="https://raw.githubusercontent.com/istio/istio/${istio_version}"
istio_addons_url="${istio_url}/samples/addons"

kubectl apply --filename "${istio_addons_url}/kiali.yaml"
kubectl apply --filename "${istio_addons_url}/grafana.yaml"
kubectl apply --filename "${istio_addons_url}/jaeger.yaml"
kubectl apply --filename "${istio_addons_url}/prometheus.yaml"

# show dashboards for Istio add-ons
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger
istioctl dashboard prometheus

# verify Istio installation
istioctl verify-install -f istio.yaml

# enforce mutual TLS for entire mesh
cat <<EOD | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOD
```

## cert-manager

See <https://cert-manager.io>.

```sh
helm repo add jetstack https://charts.jetstack.io

# deploy CRDs
cert_manager_version="v1.13.2"
cert_manager_url="https://github.com/cert-manager/cert-manager/releases"
cert_manager_url="${cert_manager_url}/download/${cert_manager_version}"
cert_manager_crds_url="${cert_manager_url}/cert-manager.crds.yaml"
kubectl apply --filename "${cert_manager_crds_url}"

# check if cert-manager CRDs have been established
kubectl wait --for condition=established \
  crd/certificaterequests.cert-manager.io
kubectl wait --for condition=established \
  crd/certificates.cert-manager.io
kubectl wait --for condition=established \
  crd/challenges.acme.cert-manager.io
kubectl wait --for condition=established \
  crd/clusterissuers.cert-manager.io
kubectl wait --for condition=established \
  crd/issuers.cert-manager.io
kubectl wait --for condition=established \
  crd/orders.acme.cert-manager.io

# deploy cert-manager
helm upgrade cert-manager jetstack/cert-manager \
  --install \
  --version "${cert_manager_version}" \
  --namespace cert-manager \
  --create-namespace \
  --atomic \
  --values cert-manager.yaml

# create CA key pair
# TODO: create CA guide
cat <<EOD | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ca-key-pair
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMrVENDQWVHZ0F3SUJBZ0lKQUtQR3dLRGwvNUhuTUEwR0NTcUdTSWIzRFFFQkN3VUFNQk14RVRBUEJnTlYKQkFNTUNHcHZjMmgyWVc1c01CNFhEVEU1TURneU1qRTJNRFUxT0ZvWERUSTVNRGd4T1RFMk1EVTFPRm93RXpFUgpNQThHQTFVRUF3d0lhbTl6YUhaaGJtd3dnZ0VpTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElCRHdBd2dnRUtBb0lCCkFRQ3doU0IvcVc2L2tMYjJ6cHUrRUp2RDl3SEZhcStRQS8wSkgvTGxseW83ekFGeCtISHErQ09BYmsrQzhCNHQKL0hVRXNuczVSTDA5Q1orWDRqNnBiSkZkS2R1UHhYdTVaVllua3hZcFVEVTd5ZzdPU0tTWnpUbklaNzIzc01zMApSNmpZbi9Ecmo0eFhNSkVmSFVEcVllU1dsWnIzcWkxRUZhMGM3ZlZEeEgrNHh0WnROTkZPakg3YzZEL3ZXa0lnCldRVXhpd3Vzc2U2S01PV2pEbnYvNFZyamVsMlFnVVlVYkhDeWVaSG1jdGkrSzBMV0Nmby9SZzZQdWx3cmJEa2gKam1PZ1l0MzBwZGhYME9aa0F1a2xmVURIZnA4YmpiQ29JMnRhWUFCQTZBS2pLc08zNUxBRVU3OUNMMW1MVkh1WgpBQ0k1VWppamEzVlBXVkhTd21KUEp5dXhBZ01CQUFHalVEQk9NQjBHQTFVZERnUVdCQlFtbDVkVEFaaXhGS2hqCjkzd3VjUldoYW8vdFFqQWZCZ05WSFNNRUdEQVdnQlFtbDVkVEFaaXhGS2hqOTN3dWNSV2hhby90UWpBTUJnTlYKSFJNRUJUQURBUUgvTUEwR0NTcUdTSWIzRFFFQkN3VUFBNElCQVFCK2tsa1JOSlVLQkxYOHlZa3l1VTJSSGNCdgpHaG1tRGpKSXNPSkhac29ZWGRMbEcxcFpORmpqUGFPTDh2aDQ0Vmw5OFJoRVpCSHNMVDFLTWJwMXN1NkNxajByClVHMWtwUkJlZitJT01UNE1VN3ZSSUNpN1VPbFJMcDFXcDBGOGxhM2hQT2NSYjJ5T2ZGcVhYeVpXWGY0dDBCNDUKdEhpK1pDTkhCOUZ4alNSeWNiR1lWaytUS3B2aEphU1lOTUdKM2R4REthUDcrRHgzWGNLNnNBbklBa2h5SThhagpOVSttdzgvdG1Sa1A0SW4va1hBUitSaTBxVW1Iai92d3ZuazRLbTdaVXkxRllIOERNZVM1TmtzbisvdUhsUnhSClY3RG5uMDM5VFJtZ0tiQXFONzJnS05MbzVjWit5L1lxREFZSFlybjk4U1FUOUpEZ3RJL0svQVRwVzhkWAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBc0lVZ2Y2bHV2NUMyOXM2YnZoQ2J3L2NCeFdxdmtBUDlDUi95NVpjcU84d0JjZmh4CjZ2Z2pnRzVQZ3ZBZUxmeDFCTEo3T1VTOVBRbWZsK0krcVd5UlhTbmJqOFY3dVdWV0o1TVdLVkExTzhvT3praWsKbWMwNXlHZTl0N0RMTkVlbzJKL3c2NCtNVnpDUkh4MUE2bUhrbHBXYTk2b3RSQld0SE8zMVE4Ui91TWJXYlRUUgpUb3grM09nLzcxcENJRmtGTVlzTHJMSHVpakRsb3c1Ny8rRmE0M3Bka0lGR0ZHeHdzbm1SNW5MWXZpdEMxZ242ClAwWU9qN3BjSzJ3NUlZNWpvR0xkOUtYWVY5RG1aQUxwSlgxQXgzNmZHNDJ3cUNOcldtQUFRT2dDb3lyRHQrU3cKQkZPL1FpOVppMVI3bVFBaU9WSTRvMnQxVDFsUjBzSmlUeWNyc1FJREFRQUJBb0lCQUNFTkhET3JGdGg1a1RpUApJT3dxa2UvVVhSbUl5MHlNNHFFRndXWXBzcmUxa0FPMkFDWjl4YS96ZDZITnNlanNYMEM4NW9PbmtrTk9mUHBrClcxVS94Y3dLM1ZpRElwSnBIZ09VNzg1V2ZWRXZtU3dZdi9Fb1V3eHFHRVMvcnB5Z1drWU5WSC9XeGZGQlg3clMKc0dmeVltbXJvM09DQXEyLzNVVVFiUjcrT09md3kzSHdUdTBRdW5FSnBFbWU2RXdzdWIwZzhTTGp2cEpjSHZTbQpPQlNKSXJyL1RjcFRITjVPc1h1Vm5FTlVqV3BBUmRQT1NrRFZHbWtCbnkyaVZURElST3NGbmV1RUZ1NitXOWpqCmhlb1hNN2czbkE0NmlLenUzR0YwRWhLOFkzWjRmeE42NERkbWNBWnphaU1vMFJVaktWTFVqbVlQSEUxWWZVK3AKMkNYb3dNRUNnWUVBMTgyaU52UEkwVVlWaUh5blhKclNzd1YrcTlTRStvVi90U2ZSUUNGU2xsV0d3KzYyblRiVwpvNXpoL1RDQW9VTVNSbUFPZ0xKWU1LZUZ1SWdvTEoxN1pvWjN0U1czTlVtMmRpT0lPSHorcTQxQzM5MDRrUzM5CjkrYkFtVmtaSFA5VktLOEMraS9tek5mSkdHZEJadGIweWtTM2t3OUIxTHdnT3o3MDhFeXFSQ2tDZ1lFQTBXWlAKbzF2MThnV2tMK2FnUDFvOE13eDRPZlpTN3dKY3E0Z0xnUWhjYS9pSkttY0x0RFN4cUJHckJ4UVo0WTIyazlzdQpzTFVrNEJobGlVM29iUUJNaUdtMGtITHVBSEFRNmJvdWZBMUJwZjN2VFdHSkhSRjRMeFJsNzc2akw4UXI4VnpxClpURVBtY0R0T0hpYjdwb2I1Z2IzSDhiVGhYeUhmdGZxRW55alhFa0NnWUVBdk9DdDZZclZhTlQrWThjMmRFYk4Kd3dJOExBaUZtdjdkRjZFUjlCODJPWDRCeGR0WTJhRDFtNTNqN2NaVnpzNzFYOE1TN25FcDN1dkFqaElkbDI3KwpZbTJ1dUUyYVhIbDN5VTZ3RzBETFpUcnVIU0Z5TVI4ZithbHRTTXBDd0s1NXluSGpHVFp6dXpYaVBBbWpwRzdmCk1XbVRncE1IK3puc3UrNE9VNFBHUW9FQ2dZQWNqdUdKbS84YzlOd0JsR2lDZTJIK2JGTHhSTURteStHcm16QkcKZHNkMENqOWF3eGI3aXJ3MytjRGpoRUJMWExKcjA5YTRUdHdxbStrdElxenlRTG92V0l0QnNBcjVrRThlTVVBcAp0djBmRUZUVXJ0cXVWaldYNWlaSTNpMFBWS2ZSa1NSK2pJUmVLY3V3aWZKcVJpWkw1dU5KT0NxYzUvRHF3Yk93CnRjTHAwUUtCZ0VwdEw1SU10Sk5EQnBXbllmN0F5QVBhc0RWRE9aTEhNUGRpL2dvNitjSmdpUmtMYWt3eUpjV3IKU25QSG1TbFE0aEluNGMrNW1lbHBDWFdJaklLRCtjcTlxT2xmQmRtaWtYb2RVQ2pqWUJjNnVGQ1QrNWRkMWM4RwpiUkJQOUNtWk9GL0hOcHN0MEgxenhNd1crUHk5Q2VnR3hhZ0ZCekxzVW84N0xWR2h0VFFZCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
EOD

# deploy cluster issuer
cat <<EOD | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: ca-key-pair
EOD
```

## Hello world

TODO: add limit range

```sh
# create namespace
kubectl create namespace hello-world

# label namespace for Istio sidecar injection
kubectl label namespace hello-world istio-injection=enabled

# create service account, deployment and service
kubectl apply -k hello-world/core

# create Istio gateway and virtual service
# <https://istio.io/latest/docs/reference/config/networking/gateway/>
# <https://istio.io/latest/docs/reference/config/networking/virtual-service/>
kubectl apply -k hello-world/istio

# create Gateway API resources
# <https://gateway-api.sigs.k8s.io/api-types/httproute/>
# <https://gateway-api.sigs.k8s.io/api-types/gateway/>
kubectl apply -k hello-world/gateway-api

# create shell pod for diagnostics
kubectl apply -k hello-world/shell

# view Istio proxy dashboard
istioctl dashboard proxy deployment/hello-world.hello-world

# cURL with custom resolver using Gateway API via Istio
# TODO: remove --insecure after CA guide has been completed
curl \
  --insecure \
  --verbose \
  --resolve "hello-world.minikube.h5k.io:443:$( \
    kubectl get service hello-world-istio \
    --namespace hello-world \
    --output jsonpath='{.status.loadBalancer.ingress[0].ip}' \
  )" \
  https://hello-world.minikube.h5k.io

# cURL with custom resolver using Istio ingress gateway
# TODO: remove --insecure after CA guide has been completed
curl \
  --insecure \
  --verbose \
  --resolve "hello-world.minikube.h5k.io:443:$( \
    kubectl get service istio-ingressgateway \
    --namespace istio-system \
    --output jsonpath='{.status.loadBalancer.ingress[0].ip}' \
  )" \
  https://hello-world.minikube.h5k.io

# generate some load ;)
while true; do
  curl \
    --insecure \
    --silent \
    --resolve "hello-world.minikube.h5k.io:443:$( \
      kubectl get service istio-ingressgateway \
      --namespace istio-system \
      --output jsonpath='{.status.loadBalancer.ingress[0].ip}' \
    )" \
    https://hello-world.minikube.h5k.io >/dev/null
    date
done

# check that the workload identity was issued by SPIRE
# expected output:
# X509v3 Subject Alternative Name: critical
#   URI:spiffe://minikube.h5k.io/ns/hello-world/sa/hello-world
istioctl proxy-config secret "$( kubectl get pod \
  --selector app=hello-world,version=v1 \
  --output jsonpath='{.items[0].metadata.name}'
)" --output json | \
  jq '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain' | \
  jq --raw-output '.inlineBytes' | \
  base64 --decode | \
  openssl x509 -in /dev/stdin -noout -ext subjectAltName

# create debug container, e.g. to run tcpdump
kubectl debug "$( kubectl get pod \
  --selector app=hello-world,version=v1 \
  --output jsonpath='{.items[0].metadata.name}'
)" \
  --stdin \
  --tty \
  --image alpine \
  --target hello-world

# inside debug container, install tcpdump
apk add --update tcpdump

# in a shell pod, run curl
while true; do curl hello-world; done

# inside debug container, run tcpdump and filter for shell pod IP address
# expected output: encrypted data, when mTLS enabled, cleartext otherwise
tcpdump -vvvv -A '(dst POD_IP_ADDRESS)'
```
