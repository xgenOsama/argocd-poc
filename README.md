```shell
# install argocd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# Using a Helm Chart
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd-demo argo/argo-cd --namespace argocd -f argocd-custom-values.yaml
# create certificate for argocd
openssl genrsa -out selfsigned.key 2048
openssl req -new -key selfsigned.key -out selfsigned.csr -config csr.conf
openssl x509 -req -days 365 -in selfsigned.csr -signkey selfsigned.key -out selfsigned.crt
helm install argocd-demo argo/argo-cd --namespace argocd -f argocd-custom-values.yaml
# create the certificate secret for ingress controller for argocd
kubectl create secret tls dev-tls-secret \
  --cert=./selfsigned.crt \
  --key=./selfsigned.key \
  -n argocd
# to update the helm release for argocd
helm upgrade argocd-demo argo/argo-cd --namespace argocd -f argocd-custom-values.yaml

#output
NAME: argocd-demo
LAST DEPLOYED: Thu Nov 21 22:44:37 2024
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-demo-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)

# create certificate for dev env
openssl genrsa -out dev.key 2048
openssl req -new -key dev.key -out dev.csr -config csr_dev.conf
openssl x509 -req -days 365 -in dev.csr -signkey dev.key -out dev.crt
# create the certificate secret for ingress controller for dev
kubectl create secret tls dev-tls-secret \
  --cert=./dev.crt \
  --key=./dev.key \
  -n dev-majls

# check values of helm deployment
helm template ./nginx-app --name-template dev-majls --namespace dev-majls --kube-version 1.30 --values nginx-app/values_dev.yaml --include-crds --debug

kubectl get secret dev-tls-secret -n dev-majls -o yaml  # check the certificate

# Verify the Installation
kubectl get pods -n argocd
# Forward the ArgoCD Server Port
kubectl port-forward svc/argocd-demo-server -n argocd 8080:443
#  Login to the Dashboard https://localhost:8080
# The default username is admin, and to retrieve the password, use:
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath={.data.password} | base64 -d
# users 
htpasswd -bnBC 10 "" "password" | tr -d ':\n'
#output
$2y$10$zN7Mvi5ObciLiZOpG04F..yFqafw7l8yEHIKdwupxetHh/AiirZwG
# Create a secret for user1
kubectl -n argocd create secret generic argocd-user1 \
  --from-literal=password='$2y$10$zN7Mvi5ObciLiZOpG04F..yFqafw7l8yEHIKdwupxetHh/AiirZwG' \
  --from-literal=username='user1' \
  --dry-run=client -o yaml | kubectl apply -f -

# Create a secret for user2
kubectl -n argocd create secret generic argocd-user2 \
  --from-literal=password='$2y$10$zN7Mvi5ObciLiZOpG04F..yFqafw7l8yEHIKdwupxetHh/AiirZwG' \
  --from-literal=username='user2' \
  --dry-run=client -o yaml | kubectl apply -f -

#  login 

argocd login localhost:8080
# add more users 
kubectl get configmap argocd-cm -n argocd -o yaml > argocd-cm.yml
# update this in file 
data:
  accounts.user1: apiKey, login
  accounts.user2: apiKey, login
# then apply it 
kubectl apply -f argocd-cm.yml 
argocd account update-password --account user1 --new-password password
argocd account update-password --account user2 --new-password password
argocd account update-password --account user3 --new-password password

# Annotate the Secrets for ArgoCD
kubectl -n argocd label secret argocd-user1 "argocd.argoproj.io/secret-type=account"
kubectl -n argocd label secret argocd-user2 "argocd.argoproj.io/secret-type=account"

# restart argocd 
kubectl rollout restart deployment argocd-demo-server -n argocd

argocd login <ARGOCD_SERVER> --username user1 --password user1password
argocd login <ARGOCD_SERVER> --username user2 --password user2password

# you can create apps from ui but here we are using CRD of argocd to create apps
kubectl create namesapce dev-majls
kubectl apply -f apps/majls/dev.yaml

helm create nginx-app
#create users and acl you can update this yaml with users an groups
kubectl get configmap argocd-cm -n argocd -o yaml > argocd-cm.ym
kubectl get configmap argocd-rbac-cm -n argocd -o yaml > argocd-rbac.yml
# install ingress nginx controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
kubectl create namespace ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx -f nginx-ingress-controller-values.yaml
# output 
NAME: ingress-nginx
LAST DEPLOYED: Fri Nov 22 13:51:22 2024
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls

kubectl port-forward svc/ingress-nginx-controller 8085:80 -n ingress-nginx
# this is nginx loadbalancer
# check the status of nginx ingress
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get svc ingress-nginx-controller -n ingress-nginx

# check values of helm deployment
helm template ./nginx-app --name-template dev-majls --namespace dev-majls --kube-version 1.30 --values nginx-app/values_dev.yaml --include-crds --debug
```