# Linkerd and TrafficSplit

## Set up Ingress on Minikube with the NGINX Ingress Controller

[Reference](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)

Start minikube with the ingress addon enabled:

```shell
minikube start --driver=virtualbox --addons=ingress
```

Deploy applications and services:

```shell
kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment web --type=NodePort --port=8080

kubectl create deployment web2 --image=gcr.io/google-samples/hello-app:2.0
kubectl expose deployment web2 --port=8080 --type=NodePort
```

Verify the deployment of the applications and that results from them differ:

```shell
curl $(minikube service web --url)
curl $(minikube service web2 --url)
```

Create Ingress resource allowing nginx to route requests to the web service (ie, to version 1.0.0):

```shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: hello-world.info
    http:
      paths:
      - path: /
        backend:
          serviceName: web
          servicePort: 8080
EOF
```

Verify access via Ingress:

```shell
curl $(minikube ip) -H 'Host: hello-world.info'
```

## Install linkerd

[Reference](https://linkerd.io/2.10/getting-started/)

```shell
linkerd install | kubectl apply -f -
linkerd viz install | kubectl apply -f -

linkerd viz dashboard &
```

## Inject linkerd

```shell
kubectl get deploy -o yaml \
  | linkerd inject - \
  | kubectl apply -f -

kubectl get deployment ingress-nginx-controller -n ingress-nginx -o yaml \
  | linkerd inject --ingress - \
  | kubectl apply -f -
```

Verify continued access:

```shell
curl $(minikube ip) -H 'Host: hello-world.info'
```

## Add TrafficSplit

```shell
cat <<EOF | kubectl apply -f -
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: web-trafficsplit
spec:
  service: web
  # Services inside the namespace with their own selectors, endpoints and configuration.
  backends:
  - service: web
    weight: 50
  - service: web2
    weight: 50
EOF
```

# Expectation

Expect that repeated requests:

```shell
curl $(minikube ip) -H 'Host: hello-world.info'
```

would alternate between returning results from version 1.0.0 and version 2.0.0.

Instead just see results from version 1.0.0.
