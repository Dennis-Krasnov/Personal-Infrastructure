# Prerequisites
## Kubernetes
```bash
yay -S kubectl helm argocd-cli

helm repo add traefik https://helm.traefik.io/traefik
helm repo update
```

## Digital Ocean
```bash
yay -S doctl
doctl login
```

## Minikube and kvm2
```bash
yay -S minikube

# https://gist.github.com/grugnog/caa118205ad498423266f26150a5d555
yay -S libvirt qemu gnome-boxes ebtables docker-machine-driver-kvm2
systemctl enable --now libvirtd.service
sudo usermod -a -G libvirt $(whoami)
sudo virsh net-autostart default
reboot

virt-host-validate
```

# Instructions

```bash
# Start local cluster
minikube start
minikube addons enable ingress

# Create secrets
kubectl create namespace ingress-controller
kubectl -n ingress-controller create secret generic traefik --from-literal="DO_AUTH_TOKEN=$(cat ~/.secrets/digital_ocean/traefik_kubernetes_pat.txt | tr -d '\n')"
kubectl -n ingress-controller get secret traefik -o jsonpath="{.data.DO_AUTH_TOKEN}" | base64 --decode

# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f argocd-install.yaml
kubectl wait -n argocd --for=condition=ready pod -l app.kubernetes.io/name=argocd-server

# Log into Argo CD
# ANOTHER TAB: kubectl port-forward svc/argocd-server -n argocd 8088:80
ARGOCD_PASSWORD=$(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2)
echo y | argocd login localhost:8088 --username admin --password $ARGOCD_PASSWORD
argocd account update-password

# Bootstrap cluster
# ANOTHER TAB: minikube tunnel
argocd app create bootstrap \
  --repo https://github.com/Dennis-Krasnov/Personal-Infrastructure.git \
  --path bootstrap \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
argocd app sync bootstrap
argocd app wait bootstrap
argocd app sync ingress-controller
argocd app wait ingress-controller
kubectl apply -n argocd -f argocd-dashboard-ingress.yaml

for app in sgbiotec portfolio; do
  # Configure imagePullSecrets from private image repository
  kubectl create namespace $app
  doctl registry kubernetes-manifest --namespace $app | kubectl apply -f -
  kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "registry-krasnov"}]}' -n $app

  # Sync application
  argocd app sync $app
done

# Update /etc/hosts
LOAD_BALANCER_IP=$(kgs -n ingress-controller ingress-controller-traefik -o json | jq -r ".status.loadBalancer.ingress[0].ip")
sudo nano /etc/hosts # add: <LOAD_BALANCER_IP> argocd.example.com

# Delete local cluster
minikube delete

# Return to normal kubernetes context
kubectl config get-contexts
kubectl config use-context do-tor1-personal
```

# Traefik
# kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
kubectl port-forward -n ingress-controller $(kubectl get pods -n ingress-controller --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
http://localhost:9000/dashboard/#/

TODO: don't include let's encrypt staging for production


# ArgoCD
# TODO: create ingress route for this!

# Minikube
```bash
minikube dashboard
minikube status
minikube logs
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

---

helm dependency update 
helm install traefik .
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
helm uninstall traefik

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



watch -n5 -d "kubectl get pods -A"

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


kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment web --port=8080

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: tmp-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: tmp.denniskrasnov.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web
          servicePort: 8080



kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: helloworld-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: default
spec:
  rules:
    - host: helloworld.com
      http:
        paths:
          - path: /helloworld
            backend:
              serviceName: helloworld-service
              servicePort: 8080

dashboard:
  enabled: true
  domain: traefik-ui.minikube