# APISIX API Gateway Demo

This guide demonstrates how to set up and test APISIX API Gateway in a Kubernetes environment.

## Prerequisites

- Minikube or Kind cluster
- kubectl
- Helm
- kustomize

## Setup with Minikube

1. Start Minikube cluster:

# init minikube
```bash
colima start --cpu 6 --memory 4 --mount $HOME:w || colima kubernetes stop 
minikube start --driver=docker 
```

## enable ingress minikube addon
```bash
minikube addons enable ingress 
```
### get nodes
```bash
kgn
```

## install hello-app and apisix controller
```bash
kustomize build --enable-helm . | kubectl apply -f - 
```
### alt enable helm permanently export KUSTOMIZE_ENABLE_HELM=true
### get all
```bash
kg all -A
```

### enable minikube ingress
```bash
minikube tunnel >/dev/null 2>&1 &
minikube service -n ingress-apisix release-name-apisix-gateway --url http://127.0.0.1:58479
http://127.0.0.1:65188
```

## install ingress
```bash
kubectl apply -f 01-ingress.yaml
```

### get all ingress
```bash
kg ingress -A
```

## hit ingress
### nginx
```bash
curl -v -H "Host: hello-v1-nginx.minikube.local" http://127.0.0.1
### apisix
curl -v -H "Host: hello-v1-apisix.minikube.local" http://127.0.0.1:65188
```

## test apisix plugins
```bash
kubectl apply -f 03-ingress-plugin.yaml
curl -v -H "Host: hello-v1-apisix.minikube.local" http://127.0.0.1:65188

< Server: APISIX/3.11.0
< testkey1: testval1
```

## test RateLimit
```bash
kubectl apply -f 04-apisixroute-limitcount.yaml
kg ApisixRoute
curl -v -H "Host: hello-v1-apisixroute.minikube.local" http://127.0.0.1:65188

< X-RateLimit-Limit: 1
< X-RateLimit-Remaining: 0
< X-RateLimit-Reset: 0
< Server: APISIX/3.11.0
```

# useful

## install kind cluster
```bash
cat <<EOF | kind create cluster --config -
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## install ingress-nginx for non-minikube
# Add the ingress-nginx repository
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

# Install the ingress-nginx controller
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.hostPort.enabled=true
```

# get logs
```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

# ingress test
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: hello.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 80
EOF

# Add to /etc/hosts:
# 127.0.0.1 hello.local
curl -H "Host: hello.local" http://localhost
```