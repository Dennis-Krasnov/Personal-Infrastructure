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
# minikube addons enable ingress (this starts nginx)

# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Log into Argo CD
ARGOCD_PASSWORD=$(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2)
argocd login $ARGOCD_PASSWORD
argocd account update-password # TODO

# Bootstrap cluster
argocd app create guestbook --repo https://github.com/Dennis-Krasnov/Personal-Infrastructure.git --path bootstrap --dest-server https://kubernetes.default.svc --dest-namespace default
argocd app get guestbook
argocd app sync guestbook

# Configure private image repository # TODO: move before install argo # TODO: figure out how this works with minikube
# https://www.digitalocean.com/docs/images/container-registry/how-to/use-registry-docker-kubernetes/
doctl registry kubernetes-manifest | kubectl apply -f -
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "registry-krasnov"}]}'

# minikube service <load-balancer-service>

# Delete local cluster
minikube delete

# Return to normal kubernetes context
kubectl config get-contexts
kubectl config use-context do-tor1-personal
```

# Minikube

```bash
minikube dashboard
minikube status
minikube stop
minikube addons list
```

# Traefik
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000

# ArgoCD
# Dashboard: kubectl port-forward svc/argocd-server -n argocd 8080:443
# TODO: create ingress route for this!

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
