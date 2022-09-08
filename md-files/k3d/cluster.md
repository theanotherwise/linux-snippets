```bash
CLUSTER_NAME="seems"

k3d cluster delete "${CLUSTER_NAME}"

SERVERS=3
AGENTS=1

k3d cluster create \
  --servers "${SERVERS}" \
  --agents "${AGENTS}" \
  --k3s-arg "--cluster-cidr=10.100.0.0/16@server:*" \
  --k3s-arg "--service-cidr=10.200.0.0/16@server:*" \
  --k3s-arg "--disable=traefik@server:*" \
  --k3s-arg "--disable=servicelb@server:*" \
  --k3s-arg "--disable=metrics-server@server:*" \
  --no-lb \
  ${CLUSTER_NAME}

for i in `seq 0 $(("${SERVERS}"-1))` ; do
  kubectl taint nodes "k3d-${CLUSTER_NAME}-server-$i" dedicated=control-plane:NoSchedule
done
# NoSchedule- untaint node
``` 

## Contexts

```bash
kubectl create namespace workspace

kubectl config set-context k3d-${CLUSTER_NAME}-workspace \
                --cluster k3d-${CLUSTER_NAME} \
                --user admin@k3d-${CLUSTER_NAME} \
                --namespace workspace

kubectl config use-context k3d-${CLUSTER_NAME}-workspace

kubectl config delete-context k3d-${CLUSTER_NAME}

kubectl config set-context k3d-${CLUSTER_NAME}-default \
                --cluster k3d-${CLUSTER_NAME} \
                --user admin@k3d-${CLUSTER_NAME} \
                --namespace kube-default
                
kubectl config set-context k3d-${CLUSTER_NAME}-kube-system \
                --cluster k3d-${CLUSTER_NAME} \
                --user admin@k3d-${CLUSTER_NAME} \
                --namespace kube-system

kubectl config set-context k3d-${CLUSTER_NAME}-metallb-system \
                --cluster k3d-${CLUSTER_NAME} \
                --user admin@k3d-${CLUSTER_NAME} \
                --namespace metallb-system
                
kubectl config set-context k3d-${CLUSTER_NAME}-cert-manager-system \
                --cluster k3d-${CLUSTER_NAME} \
                --user admin@k3d-${CLUSTER_NAME} \
                --namespace cert-manager-system

kubectl config set-context k3d-${CLUSTER_NAME}-nginx-ingress-system \
                --cluster k3d-${CLUSTER_NAME} \
                --user admin@k3d-${CLUSTER_NAME} \
                --namespace nginx-ingress-system
```

## Load Balancer

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update

kubectl create namespace metallb-system

helm upgrade --install metallb metallb/metallb \
  --version 0.13.4 \
  --namespace metallb-system

METALLB_CIDR=`docker network inspect k3d-${CLUSTER_NAME} | jq -r ".[0].IPAM.Config[0].Subnet" | awk -F'.' '{print $1"."$2"."$3"."128"/"25}'`

cat <<EndOfMessage | kubectl -n metallb-system apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: docker-host
spec:
  addresses:
    - "${METALLB_CIDR}"
  avoidBuggyIPs: true
EndOfMessage

cat <<EndOfMessage | kubectl -n metallb-system apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: docker-host
spec:
  ipAddressPools:
  - docker-host
EndOfMessage
```

## Certificates

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl create namespace cert-manager-system

helm upgrade --install cert-manager jetstack/cert-manager \
  --version v1.7.2 \
  --namespace cert-manager-system \
  --set installCRDs=true
```

## Ingress

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update

kubectl create namespace nginx-ingress-system

helm upgrade --install nginx-ingess nginx-stable/nginx-ingress \
  --namespace nginx-ingress-system
```

```
kubectl port-forward service/nginx-ingess-nginx-ingress \
  --namespace nginx-ingress-system \
  8080:80
```