# Prerequisites
- Kubectl (yay -S ...)
- Helm 3 (yay -S ... &&  helm repo add traefik https://helm.traefik.io/traefik && helm repo update)
- Virtualbox (yay -S virtualbox virtualbox-host-dkms && reboot)
- Minikube (yay -S minikube)
- Doctl (yay -S doctl && doctl login)
- ArgoCD (yay -S argocd-cli)

# Instructions

```bash
# Start local cluster
minikube start --driver=virtualbox
# minikube addons enable ingress # FIXME

# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait until Argo CD starts
watch -n5 -d "kubectl get pods -A"

# Log into Argo CD
ARGOCD_PASSWORD=$(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2)
# ANOTHER TAB: kubectl port-forward svc/argocd-server -n argocd 8080:443
echo y | argocd login localhost:8080 --username admin --password $ARGOCD_PASSWORD
argocd account update-password

# Bootstrap cluster
argocd app create bootstrap --repo https://github.com/Dennis-Krasnov/Personal-Infrastructure.git --path bootstrap --dest-server https://kubernetes.default.svc --dest-namespace default
argocd app sync bootstrap
argocd app sync ingress-controller
# TODO: and other apps that I need

# ANOTHER TAB: minikube tunnel

sudo nano /etc/hosts # add: <ip> argocd.example.com

# Configure private image repository # TODO: move before install argo # TODO: figure out how this works with minikube
# https://www.digitalocean.com/docs/images/container-registry/how-to/use-registry-docker-kubernetes/
doctl registry kubernetes-manifest | kubectl apply -f -
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "registry-krasnov"}]}'

# Delete local cluster
minikube delete

# Return to normal kubernetes context
kubectl config get-contexts
kubectl config use-context do-tor1-personal
```

# Traefik
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000

# ArgoCD
# TODO: create ingress route for this!

# Minikube
```bash
minikube dashboard
minikube status
minikube stop
minikube addons list
minikube cache list
minikube service list
```

## Metrics
```
minikube addons enable metrics-server
kubectl top nodes
kubectl top pods # -A OR -n <namespace>
```

## Cache
ctrl-f `image:` on https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```bash
minikube cache add argoproj/argocd:v1.7.6
# minikube cache add quay.io/dexidp/dex:v2.22.0 (doesn't work...)
minikube cache add redis:5.0.8
```

# random stuff

      VIRTUAL_HOST: "sgbiotec.com"
      LETSENCRYPT_HOST: "sgbiotec.com"
      LETSENCRYPT_EMAIL: "igor.k@sgbiotec.com"
      
      #!/usr/bin/env bash

docker-compose -f docker-compose.yml up -d --build


FROM nginx:mainline-alpine
COPY default.conf /etc/nginx/conf.d
server {
	listen 80;
    server_name localhost;
    return 301 https://sgbiotec.com$request_uri;
}





-------------------------



https://marketplace.digitalocean.com/apps/kubernetes-metrics-server

k top nodes

k top pod --all-namespaces



---


export ARGOCD_SERVER=argocd.mycompany.com
export ARGOCD_AUTH_TOKEN=<JWT token generated from project>
curl -sSL -o /usr/local/bin/argocd https://${ARGOCD_SERVER}/download/argocd-linux-amd64
argocd app sync guestbook
argocd app wait guestbook
